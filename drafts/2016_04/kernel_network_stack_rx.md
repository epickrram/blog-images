# Navigating the Linux kernel network stack: receive path

## Background

At [work](https://lmax.com) we practice continuous integration in terms of [performance testing](http://epickrram.blogspot.co.uk/2014/05/performance-testing-at-lmax-part-one.html) 
alongside different stages of functional testing.

In order to do this, we have a performance environment that fully replicates the hardware and software used in our production environments. 
This is necessary in order to be able to find the limits of our system in terms of throughput and latency, and means that we make sure that the environments are 
identical, right down to the network cables.

Since we like to be ahead of the curve, we are constantly trying to push the boundaries of our system to find out where it will fall over, and the nature of the failure mode.

This involves [running production-like load](http://epickrram.blogspot.co.uk/2014/08/performance-testing-at-lmax-part-three.html) 
against the system at a much higher rate than we have ever seen in production. 
We currently aim to be able to handle a constant throughput of 2-5 times
the maximum peak throughput ever observed in production. 
We believe that this will give us enough headroom to handle future capacity requirements.

Our Performance & Capacity team has a constant background task of:

1. increase load applied to the system until it breaks
2. find and fix the bottleneck

Using this process, we aim to ensure that we are able to handle spikes in demand, 
and increases in user numbers, while still achieving a consistent and low latency-profile.

There is actually a third step to this process, which is something like 'buy new hardware' or 
'modify the system to do less work', since there is only so much than tuning will buy you.

## Identifying a bottleneck

During the latest iteration of the break/fix cycle, we identified one particular service as problematic. 
The service in question is one of those responsible for consuming the output of our matching engine, 
which can peak at 250,000 messages per second at our current performance-test load.

When we have a bottleneck in one of our services, it usually follows a familiar pattern of back-pressure from the
application processing thread, resulting in packets being dropped by the networking card.

In this case however, we could see from our monitoring that we were not suffering from any processing back-pressure, 
so it was necessary to delve a little deeper into the network packet receive path in order to understand the problem.

## Understanding the data flow

The Linux kernel provides a number of counters that can give an indication of any problems in the network stack. 
Since we are concerned with throughput, we will be most interested in things like queue depths and drop counts.

Before looking at the available statistics, let's take a look at how a packet is handled once it is pulled off the wire.

The journey begins in the network driver code; this is vendor-specific and in the majority of cases open source.
In this example, we're working with an Intel 10Gb card, which uses the ixgbe driver. You can find out the driver used by a 
network interface by using ethtool:

    ethtool -i <device-name>

This will generate output that looks something like this:

    driver: ixgbe
    version: 3.19.1-k
    firmware-version: 0x546d0001
    bus-info: 0000:41:00.0
    supports-statistics: yes
    supports-test: yes
    supports-eeprom-access: yes
    supports-register-dump: yes
    supports-priv-flags: no

The driver code back be found in the Linux kernel source [here](http://lxr.free-electrons.com/source/drivers/net/ethernet/intel/ixgbe/?v=4.0).

### NAPI

NAPI, or New API is a mechanism introduced into the kernel several years ago. More background can be read [here](http://www.linuxfoundation.org/collaborate/workgroups/networking/napi),
but in summary, NAPI increases network receive performance by changing packet receipt from interrupt-driven to polling-mode.

Previous to the introduction of NAPI, network cards would typically fire a hardware interrupt for each received packet. 
Since an interrupt on a CPU will always cause suspension of the executing software, a high interrupt rate can interfere with software performance.
NAPI addresses this by exposing a poll method to the kernel, which is periodically executed (actually via an interrupt). While the poll method is executing,
receive interrupts for the network device are disabled. The effect of this is that the kernel can drain potentially multiple packets from the network device 
receive buffer, thus increasing throughput at the same time as reducing the interrupt overhead.

### Interrupt handling

When the network device driver is initially configured, it first associates a handler function with the receive interrupt. 
For the card that we're look at, this happens in a method called [ixgbe_request_msix_irqs](http://lxr.free-electrons.com/source/drivers/net/ethernet/intel/ixgbe/ixgbe_main.c?v=4.0#L2740):

    request_irq(entry->vector, &ixgbe_msix_clean_rings, 0,
       q_vector->name, q_vector);

The ixgbe_msix_clean_rings method simply [schedules a NAPI poll](http://lxr.free-electrons.com/source/drivers/net/ethernet/intel/ixgbe/ixgbe_main.c?v=4.0#L2675), 
and returns IRQ_HANDLED:

    static irqreturn_t ixgbe_msix_clean_rings(int irq, void *data)
    {
        struct ixgbe_q_vector *q_vector = data;
        ...
        if (q_vector->rx.ring || q_vector->tx.ring)
            napi_schedule(&q_vector->napi);

        return IRQ_HANDLED;
    }

Scheduling the NAPI poll entails [adding some work](http://lxr.free-electrons.com/source/net/core/dev.c?v=4.0#L3022) to the per-cpu poll list maintained in the softnet_data structure:

    static inline void ____napi_schedule(struct softnet_data *sd,
                                         struct napi_struct *napi)
    {
        list_add_tail(&napi->poll_list, &sd->poll_list);
        __raise_softirq_irqoff(NET_RX_SOFTIRQ);
    }

and then raising a softirq event.

### softirq processing

For more background on interrupt handling, the [Linux Device Drivers](https://lwn.net/Kernel/LDD3/) book has a chapters dedicated to this topic.
Suffice to say, doing work inside of a hardware interrupt context is generally avoided within the kernel. One mechanism for dealing with this is 
to use softirqs.

Each CPU in the system has a bound process called ksoftirqd/<cpu_number>, which is responsible for processing softirq events.

In this manner, when a hardware interrupt is received, the driver raises a softIRQ to be processed on the ksoftirqd process. So it is this 
process that will be responsible for calling the drivers poll method.


The softirq handler [net_rx_action](http://lxr.free-electrons.com/source/net/core/dev.c?v=4.0#L7475) is configured for network packet receive events during device initialisation.

So, having followed the code this far, we can say that when a network packet is in the device's receive ring-buffer, the net_rx_action function will be the top-level 
entry point for packet processing.

### net_rx_action

At this point, it is instructive to look at a function trace of the ksoftirqd process. 
This trace was generated using [ftrace](https://www.kernel.org/doc/Documentation/trace/ftrace.txt), and gives a high-level overview of the functions involved
in processing the available packets on the network device.


    net_rx_action() {
      ixgbe_poll() {
        ixgbe_clean_tx_irq();
        ixgbe_clean_rx_irq() {
          ixgbe_fetch_rx_buffer() {
            ... // allocate buffer for packet
          } // returns the buffer containing packet data
          ... // housekeeping
          napi_gro_receive() {
            // generic receive offload
            dev_gro_receive() {
              inet_gro_receive() {
                udp4_gro_receive() {
                  udp_gro_receive();
                }
              }
            }
            netif_receive_skb_internal() {
              __netif_receive_skb() {
                __netif_receive_skb_core() {
                  ...
                  ip_rcv() {
                    ...
                    ip_rcv_finish() {
                      ...
                      ip_local_deliver() {
                        ip_local_deliver_finish() {
                          raw_local_deliver();
                          udp_rcv() {
                            __udp4_lib_rcv() {
                              __udp4_lib_mcast_deliver() {
                                ...
                                // clone skb & deliver
                                flush_stack() {
                                  udp_queue_rcv_skb() {
                                    ... // data preparation
                                    // deliver UDP packet
                                    // http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1497
                                    // check if buffer is full
                                    // http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1584
                                    __udp_queue_rcv_skb() {
                                      // deliver to socket queue
                                      // http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1453
                                      // check for delivery error
                                      // http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1464
                                      sock_queue_rcv_skb() {
                                        ...
                                        _raw_spin_lock_irqsave();
                                        // enqueue packet to socket buffer list
                                        // http://lxr.free-electrons.com/source/include/linux/skbuff.h?v=4.0#L1481
                                        _raw_spin_unlock_irqrestore();
                                        // wake up listeners
                                        // http://lxr.free-electrons.com/source/net/core/sock.c?v=4.0#L474
                                        sock_def_readable() {
                                          __wake_up_sync_key() {
                                            _raw_spin_lock_irqsave();
                                            __wake_up_common() {
                                              ep_poll_callback() {
                                                ...
                                                _raw_spin_unlock_irqrestore();
                                              }
                                            }
                                            _raw_spin_unlock_irqrestore();
                                          }
    ...


The softirq handler performs the following steps:

1.  Call the driver's poll method (in this case ixgbe_poll)
2.  Perform some GRO functions to group packets together into a larger work unit
3.  Call the packet type's [handler function](http://lxr.free-electrons.com/source/net/core/dev.c?v=4.0#L1735) (ip_rcv) to walk down the protocol chain
4.  Parse IP headers, perform checksumming then call ip_rcv_finish
5.  The buffer's destination function is invoked, in this case udp_rcv 
6.  Since these are multicast packets, __udp4_lib_mcast_deliver is called
7.  The packet is copied and delivered to each registered UDP socket queue
8.  In udp_queue_rcv_skb, buffers are checked and if space remains, the skb is added to the end of the socket's queue



## Monitoring back-pressure

When attempting to increase the throughput of an application, we need to understand where back-pressure is coming from.

At this point in the data receive path, we could have throughput issues for two reasons:

1.  The softirq handling mechanism cannot dequeue packets from the network device fast enough
2.  The application processing the destination socket is not dequeuing packets from the socket buffer fast enough


### softirq back-pressure

For the first case, we need to look at softnet stats (/proc/net/softnet_stat), which are updated in the network receive stack.

The softnet stats are defined [here](http://lxr.free-electrons.com/source/include/linux/netdevice.h?v=4.0#L2444) as the per-cpu struct softnet_data, 
which contains a few fields of interest: processed, time_squeeze and dropped.


time_squeeze is updated if the softirq process cannot process all packets available in the network device ring-buffer before its cpu-time is up.
The process is limited to 2 jiffies of processing time, or a certain amount of 'work'. There are a couple of sysctls that control these parameters:

1.  net.core.netdev_budget - the total amount of processing to be done in one invocation of net_rx_action
2.  net.core.dev_weight - an indicator to the network driver of how much work to do per invocation of its napi poll method

The softirq daemon will continue to [call napi_poll](http://lxr.free-electrons.com/source/net/core/dev.c?v=4.0#L4655) until either the time has run out,
or the amount of work reported as completed by the driver exceeds the value of net.core.netdev_budget.

This behaviour will be driver-specific; in the Intel 10Gb driver, completed work will always be [reported as net.core.dev_weight]
(http://lxr.free-electrons.com/source/drivers/net/ethernet/intel/ixgbe/ixgbe_main.c?v=4.0#L2727) if there are still packets 
to be processed at the end of a poll invocation.


Given some example numbers, we can determine how many times the napi_poll function will be called for a softIRQ event:

    net.core.netdev_budget = 300
    net.core.dev_weight = 64

    poll_count = (300 / 64) + 1 => 5


If there are still packets to be processed in the network device ring-buffer, then the time_squeeze counter will be incremented for the 
given CPU.


The dropped counter is only used when the softirq process is attemping to add a packet to the backlog queue of another CPU.
This can happen if [Receive Packet Steering](https://www.kernel.org/doc/Documentation/networking/scaling.txt) is enabled,
but since we are only looking at UDP multicast without RPS, I won't go into the detail.

So if our kernel helper thread is unable to move packets from the network device receive queue to the socket's receive buffer 
fast enough, we can expect the time_squeeze column in /proc/net/softnet_stat to increase.

The only tunable that we have at our disposal is the netdev_budget value. Increasing this will allow the softirq process to do more work.
The process will still be limited by a total processing time of 2 jiffies though, so there will be an upper ceiling to packet throughput.

Given the speeds that modern processors are capable of, it is unlikely that the softirq daemon will be unlikely to keep up with the flow of
data. In order to give the kernel the best chance, make sure that there is no contention for CPU resources by assigning network interrupts to 
a number of cores, and then using isolcpus to make sure that no other processes will be running on them.


If the softirq daemon is squeezed frequently enough, or is just unable to get CPU time, then the network device will be forced to drop 
packets from the wire. In this case, we can use ethtool to find the rx_missed count:

    ethtool -S em1 | grep rx_missed
    rx_missed_errors: 0

alternatively, the same data can be found by looking at the following file:

    /sys/class/net/<device-name>/statistics/rx_missed_errors


For a full description of each of the statistics reported by ethtool, refer to [this document](http://lxr.free-electrons.com/source/Documentation/ABI/testing/sysfs-class-net-statistics).

### Application back-pressure

It is far more likely that our user programs will be the bottleneck here, and in order to determine whether that is the case, we need to have a look at the next stage 
in the message receipt path. A continuation of this post will explore in more detail.


## Summary

For UDP-multicast traffic, we have seen in detail the code paths involved in moving an inbound network packet from a network device to a socket's input buffer.
This stage can be broadly summarised as follows:

1.  On packet receipt, the network device fires a hardware interrupt to the configured CPU
2.  The hardware interrupt handler schedules a softIRQ on the same CPU
3.  The softIRQ handler thread (ksoftirqd) will disable receive interrupts and poll the card for received data
4.  Data will be copied from the network device's receive buffer into the destination socket's input buffer
5.  After a certain amount of work has been done, or no inbound packets remain, the softirq daemon will re-enable receive interrupts and return

In order to optimise for throughput, there are a couple of things to try tuning:

1.  Increase the amount of work that the softirq daemon is allowed to do (net.core.netdev_budget)
2.  Make sure that the ksoftirq process is not contending for CPU resource or being descheduled due to other hardware interrupts
3.  Increase the size of the network device's ring-buffer (ethtool -g <device-name>)


As with all performance-related experiments, never attempt to tune the system without being able to measure the impact of any changes in isolation.
First, make sure that you know what the problem is (i.e. rx_missed_errors or time_squeeze is increasing), the add the relevant monitoring.
For this particular case, we would want to be able to correlate the application experiencing message loss with a change in the 
relevant counters, so recording and charting the numbers would be a good start.

Once this has been done, changes can be made to system configuration to see if an improvement can be made.

Lastly, any changes to the tuning parameters that I've mentioned MUST be configured via automation. We have sadly lost a fair 
amount of time to manual changes being made on machines that have not persisted across reboots.

It is all too easy (and I speak from experience) to make adjustments, but the optimal configuration, and then move on to something
else. Do yourself and your colleagues a favour and automate!


__netif_receive_skb_core increments softnetdata.processed (http://lxr.free-electrons.com/source/net/core/dev.c?v=4.0#L3646)
ip_rcv updates received, discarded, checksum errors, truncated packets (http://lxr.free-electrons.com/source/net/ipv4/ip_input.c?v=4.0#L363)
flush_stack updates sock net rcvbuff, input errors (http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1628)
udp_queue_rcv_skb updates rcvbuff errors on sock net (http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1585)


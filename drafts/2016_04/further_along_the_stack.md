# Navigating the Linux kernel network stack: into user land

This is a continuation of my [previous post](http://epickrram.blogspot.co.uk/2016/05/navigating-linux-kernel-network-stack.html)
in which we follow the path of an inbound multicast packet to the user application.

  
  
At the end of the last post, I mentioned that application back-pressure was likely to be the cause of receiver packet-loss.
As we continue through the code path taken by inbound packets up into user-land, we will see the various points at which
data will be discarded due to slow application processing, and what metrics can shed light on these events.
  
  

## Protocol mapping

  
  
First of all, let's take a quick look at how a received data packet is matched up to its 
[handler function](http://lxr.free-electrons.com/source/net/core/dev.c?v=4.0#L1735).
  
  
We can see from the stack-trace below that the top-level function calls when dealing with the 
packets are `packet_rcv` and `ip_rcv`. 
  
  

    __netif_receive_skb_core() {
        packet_rcv() {
            skb_push();
            __bpf_prog_run();
            consume_skb();
        }
        bond_handle_frame() {
            bond_3ad_lacpdu_recv();
        }
        packet_rcv() {
            skb_push();
            __bpf_prog_run();
            consume_skb();
        }
        packet_rcv() {
            skb_push();
            __bpf_prog_run();
            consume_skb();
        }
        ip_rcv() {
            ...
  
  
This trace captures the [part of the receive path](http://lxr.free-electrons.com/source/net/core/dev.c?v=4.0#L3671) where 
the inbound packet is passed to each registered handler function. In this way, the kernel handles things like VLAN tagging,
interface bonding, and packet-capture. Note the `__bpf_prog_run` function call, which indicates that this is the 
entry point for `pcap` packet capture and filtering.
  
  
The protocol-to-handler mapping can be viewed in the file `/proc/net/ptype`:
  
    [pricem@metal]$ cat /proc/net/ptype 
    Type Device      Function
    ALL  em1      packet_rcv
    0800          ip_rcv
    0011          llc_rcv [llc]
    0004          llc_rcv [llc]
    88f5          mrp_rcv [mrp]
    0806          arp_rcv
    86dd          ipv6_rcv

  
Comparing against [this reference](https://en.wikipedia.org/wiki/EtherType#Notable_values), it is clear that the kernel
reads the value of the ethernet frame's `EtherType` field and dispatches to the corresponding function. There are also
some functions that will be executed for all packet types (signified by type `ALL`).
  
  
So our inbound packet will be processed by the capture hooks, even if no packet capture is running 
(this then must presumably be very cheap), before being passed to the correct protocol handler, in this case `ip_rcv`.
  
  


## The socket buffer
  

As discussed previously, each socket on the system has a receive-side FIFO queue that is written to by the kernel,
and read from by the user application.
  

[`ip_rcv`](http://lxr.free-electrons.com/source/net/ipv4/ip_input.c?v=4.0#L376) starts by getting its own copy of the
incoming packet, copying if the packet is already shared. If the copy fails due to lack of memory, then the packet is
discarded, and the `Discard` count of ipstats is incremented. Other checks made at this point include the IP checksum,
header-length check and truncation check, each of which update the relevant metrics in the ipstats table.
  
  
Before calling the `ip_rcv_finish` function, the packet is diverted through the `netfilter` module where software-based
network filtering can be applied.


Assuming that the packet is not dropped by a filter, 
[`ip_rcv_finish`](http://lxr.free-electrons.com/source/net/ipv4/ip_input.c?v=4.0#L312) 
passes the packet on to the next protocol-handler in the chain.
  
  
  
In the internals of the [`udp_rcv`](http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1749) function, we
finally get to the point of accessing the socket FIFO queue.
  
  
Simple validation performed at this point includes packet-length check and checksum. Failure of these checks
will cause the relevant statistics to be updated in the udpstats table.
  
  

Because the traffic we're tracing [is multicast](http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1801), 
the next handler function is [`__udp4_lib_mcast_deliver`](http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1660).

With some exotic locking in place, the kernel determines how many different sockets this multicast packet needs to be delivered to.
It is worth noting here that there is a smallish number (`256 / sizeof( struct sock)`) of sockets that can be listening to 
a given multicast address before the possibility of greater lock-contention creeps in.

An effort is made to enumerate the registered sockets with a lock held, then perform the packet dispatching without a lock. 
However, if the number of registered sockets exceeds the specified threshold, then packet dispatch (the `flush_stack` function)
will be handled while a lock is held.
  
  
Once in [`flush_stack`](http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1614), the packet is copied for each
registered socket, and pushed onto the FIFO queue.

If the kernel is unable to allocate memory in order to copy the buffer, the socket's drop-count will be incremented, along
with the RcvBufErrors and InErrors metrics in the udpstats table. 
  
  

After another checksum test, we are finally at the point where socket-buffer overflow 
[is tested](http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1584):

    if (sk_rcvqueues_full(sk, sk->sk_rcvbuf)) {
        UDP_INC_STATS_BH(sock_net(sk), UDP_MIB_RCVBUFERRORS,
                         is_udplite);
        goto drop;
    }

If the size of the socket's backlog queue, plus the memory used in the receive queue is greater than the socket receive buffer size, then the 
RcvBufferErrors and InErrors metrics are updated in the udpstats table, along with the socket's drop count.
  
  
To [safely handle multi-threaded access to the socket buffer](http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1594), 
if an application is currently reading from a socket, then the inbound packet will be queued to the socket's 
[backlog queue](http://lxr.free-electrons.com/source/include/net/sock.h?v=4.0#L335). The backlog queue will be processed 
during the handling of the next sortIRQ event.

Otherwise, pending more checks that socket memory limits have not be exceeded, the packet is added to the socket's 
[`sk_receive_queue`](http://lxr.free-electrons.com/source/net/core/sock.c?v=4.0#L439).

  
  
Finally, once the packet has been delivered to the socket's receive queue, notification is 
[sent to any interested listeners](http://lxr.free-electrons.com/source/net/core/sock.c?v=4.0#L474).
  

## Kernel statistics


Now that we have seen where the various capacity checks are made, we can make some observations in the `proc` file system.
  

There are protocol-specific files in `/proc/net` that are updated by the kernel whenever data is received.

### `/proc/net/snmp`

This file contains statistics on multiple protocols, and is the target for updates to the UDP receive buffer errors.
So when we want to know about system-wide receive errors, this is the place to look. When this file is read, 
a [function call](http://lxr.free-electrons.com/source/net/ipv4/proc.c?v=4.0#L374) iterates through the stored values
and writes them to the supplied destination. Such is the magic of [`procfs`](https://en.wikipedia.org/wiki/Procfs).
  
  
In the output below, we can see that there have been several receive errors (`RcvbufErrors`), and the number is matched by the `InErrors` metric:

    Udp: InDatagrams NoPorts InErrors OutDatagrams RcvbufErrors SndbufErrors InCsumErrors
    Udp: 281604018683 4      261929024 42204463516 261929024    0            0

Each of the columns maps to a constant specified [here](http://lxr.free-electrons.com/source/net/ipv4/proc.c?v=4.0#L176), so to determine when these
values are updated, we simply need to 
[search for the constant name](https://www.google.co.uk/search?q=UDP_MIB_RCVBUFERRORS&sitesearch=lxr.free-electrons.com/source).
  
  
There is a subtle difference between `InErrors` and `RcvbufErrors`, but in our case when looking for socket buffer overflow, 
we only care about `RcvbufErrors`.


### `/proc/net/udp`


This file contains socket-specific statistics, which only live for the lifetime of a socket. As we have seen from the 
code so far, each time the `RcvbufErrors` metric has been incremented, so too has the corresponding `sk_drops` value
for the target socket.
  
  
So if we need to differentiate between drop events on a per-socket basis, this file is what we need to observe.
The contents below can be explained by looking at the 
[handler function](http://lxr.free-electrons.com/source/net/ipv4/udp.c#L2517)
for socket data:

    [pricem@metal ~]# cat /proc/net/udp
    sl  local_address rem_address   st tx_queue rx_queue ... inode ref pointer drops
    47: 00000000:B87A 00000000:0000 07 00000000:00000000 ...  239982339 2 ffff88052885c440 0     
    95: 4104C1EF:38AA 00000000:0000 07 00000000:00000000 ...  239994712 2 ffff8808507ed800 0     
    95: BC04C1EF:38AA 00000000:0000 07 00000000:00000000 ...  175113818 2 ffff881054a8b080 0     
    95: BE04C1EF:38AA 00000000:0000 07 00000000:00000000 ...  175113817 2 ffff881054a8b440 0     
    ...
  
  
For observing back-pressure, we are interested in the drops column, and the rx_queue column, which is a snapshot of the amount of queued data 
at the time of reading. Local and remote addresses are hex-encoded `ip:port` addresses.

Monitoring this file can tell us if our application is falling behind the packet arrival rate (increase in `rx_queue`), or 
whether the kernel was unable to copy a packet into the socket buffer due to it already being full (increase in `drops`).



 
 


## Tracepoints

If we wish to capture more data about packets that are being dropped, there a three tracepoints that are of interest.


1.  [`sock:sock_rcvqueue_full`](http://lxr.free-electrons.com/source/net/core/sock.c?v=4.0#L447): called when the amount of 
memory already allocated to the socket buffer is greater than or equal to the configured socket buffer size.
1.  [`sock:sock_exceed_buf_limit`](http://lxr.free-electrons.com/source/net/core/sock.c?v=4.0#L2084): called when the 
kernel is unable to allocate more memory for the socket buffer.
1.  [`udp:udp_fail_queue_rcv_skb`](http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1473): called shortly after 
the `sock_rcvqueue_full` event trace for the same reason.

  
  
These events can be captured using Linux kernel tracing tools such as 
[`perf`](https://perf.wiki.kernel.org/index.php/Main_Page) or 
[`ftrace`](https://www.kernel.org/doc/Documentation/trace/ftrace.txt).




TODO: how does /sys/class/net/p3p1/statistics/rx_dropped get populated by the kernel?
TODO: double-check the code that updates device-side drops/rx-ring overflow.
TODO: image showing backlog and rcvqueue of socket buffer.








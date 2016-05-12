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
a given multicast address before the possibility of lock-contention creeps in.

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

If the socket's backlog queue, plus the already-allocated memory is greater than the socket receive buffer size, then the 
RcvBufferErrors and InErrors metrics are updated in the udpstats table, along with the socket's drop count.
 
 
To [safely handle multi-threaded access to the socket buffer](http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1594), 
if an application is currently reading from a socket, then the inbound packet will be queued to the socket's backlog queue.

Otherwise, pending more checks that socket memory limits have not be exceeded, the packet is added to the socket's 
[`sk_receive_queue`](http://lxr.free-electrons.com/source/net/core/sock.c?v=4.0#L439).

 
 
Finally, once the packet has been delivered to the socket's receive queue, notification is 
[sent to any interested listeners](http://lxr.free-electrons.com/source/net/core/sock.c?v=4.0#L474).
 
 
 
`/proc/net/udp`: http://lxr.free-electrons.com/source/net/ipv4/udp.c#L2538

`/proc/net/snmp`: http://lxr.free-electrons.com/source/net/ipv4/proc.c?v=4.0#L176

Diff between InErrors and RevbufErrors:

Udp: InDatagrams NoPorts InErrors OutDatagrams RcvbufErrors SndbufErrors InCsumErrors
Udp: 275707389845 4 261757187 41217478199 261757187 0 0


InErrors > RcvbufErrors -> allocation failure when cloning skb.http://lxr.free-electrons.com/source/net/ipv4/udp.c?v=4.0#L1626




TODO: how does /sys/class/net/p3p1/statistics/rx_dropped get populated by the kernel?










As covered previously, the data structure containing information about network-related softIRQ processing on each CPU
is `softnet_data`, exposed in the `proc` filesystem as `/proc/net/softnet_stat`.

In the receive loop of [`__netif_receive_skb_core`](http://lxr.free-electrons.com/source/net/core/dev.c?v=4.0#L3619), we can see
the `processed` count [being incremented](http://lxr.free-electrons.com/source/net/core/dev.c?v=4.0#L3646) 
for each socket buffer handled:

    __this_cpu_inc(softnet_data.processed);

So this is the first column in the `softnet_stat` file.

/proc/net/snmp: http://lxr.free-electrons.com/source/net/ipv4/ip_input.c?v=4.0#L388


http://lxr.free-electrons.com/source/net/ipv4/proc.c?v=4.0#L374

[pricem@ares buck-all]$ cat /proc/net/ptype 
Type Device      Function
ALL  em1      packet_rcv
0800          ip_rcv
0011          llc_rcv [llc]
0004          llc_rcv [llc]
88f5          mrp_rcv [mrp]
0806          arp_rcv
86dd          ipv6_rcv







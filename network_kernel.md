This file shows the function invocations in the kernel when transmiting or receiving packets.
# Transmit a packet
## Functions from APP layer to TCP layer
There are three userspace functions for sockets to transmit packets: send(), sendmsg() and sendto(). <br>

SYSCALL_DEFINE4(send, int, fd, void __user *, buff, size_t, len, unsigned int, flags) <br>
-> __sys_sendto() <br>
```
struct socket sock = sockfd_lookup_light(fd, ...);
```
SYSCALL_DEFINE6(recvfrom, int, fd, void __user *, ubuf, size_t, size,
		unsigned int, flags, struct sockaddr __user *, addr, int __user *, addr_len) <br>
-> __sys_recvfrom() <br>
SYSCALL_DEFINE6(sendto, int, fd, void __user *, buff, size_t, len,
		unsigned int, flags, struct sockaddr __user *, addr, int, addr_len) <br>
-> __sys_sendto() <br>

-> sock_sendmsg() <br>
-> sock_sendmsg_nosec() <br>
-> sock->ops->sendmsg() = inet_sendmsg() <br>
-> sk->sk_prot->sendmsg() = tcp_sendmsg() <br>
-> tcp_sendmsg_locked() <br>
-> tcp_push_one() <br>
-> tcp_write_xmit() <br>
-> __tcp_transmit_skb() (from this function on, skb is one of the arguments) <p>

## Functinon from TCP layer to Ethernet
Include kernel functions from __tcp_transmit_skb() in the TCP layer to ixgbe_xmit_frame_ring() in the ethernet. <br>
__tcp_transmit_skb() <br>
-> ip_queue_xmit() <br>
-> ip_local_out() <br>
-> __ip_local_out() <br>
-> dst_output() <br>
-> ip_output() <br>
-> ip_finish_output() <br>
-> ip_finish_output_gso() <br>
-> ip_finish_output2() <br>
-> dst_neigh_output() <br>
-> neigh_hh_output() <br>
-> dev_queue_xmit() <br>
-> __dev_queue_xmit() -> dev_hard_start_xmit() (loopback) <br>
-> __dev_xmit_skb() -> qdisc->enqueue() <br>
-> sch_direct_xmit() <br>
-> dev_hard_start_xmit() <br> 
-> xmit_one() <br>
-> netdev_start_xmit() <br>
-> __netdev_start_xmit() <br>
-> ixgbe_xmit_frame() <br>
-> __ixgbe_xmit_frame() <br>
-> ixgbe_xmit_frame_ring() <br>

# Receive a packet
## Functions from Ethernet layer to TCP layer
Include kernel functions from ixgbe_poll() in the ethernet to __skb_queue_tail() in the TCP layer. <br>
ixgbe_poll() <br>
-> ixgbe_clean_rx_irq() <br>
-> |- ixgbe_process_skb_fields() (from this function on, skb is one of the arguments) -> eth_type_trans() <br>
-> |- ixgbe_rx_skb() -> <br>
(napi_gro_receive()) <br>
-> netif_rx() <br>
-> netif_rx_action() <br>
-> netif_receive_skb_internal() <br>
->  __netif_receive_skb() <br>
->  __netif_receive_skb_core() <br>
-> deliver_skb() <br>
-> ip_rcv() <br>
-> ip_rcv_finish() <br>
-> ip_route_input_noref() <br>
-> tcp_v4_rcv() -> tcp_v4_fill_cb() <br>
-> tcp_v4_do_rcv() <br>
-> tcp_rcv_established() (sk->sk_state == TCP_ESTABLISHED) <br>
-> tcp_data_queue() <br>
-> tcp_queue_rcv() -> tcp_event_data_recv() <br>
-> |- __skb_queue_tail <br>
&emsp; |- skb_set_owner_r() <br>

## Functions from APP layer to TCP layer
There are three userspace functions for sockets to receive packets: recv(), recvmsg() and recvfrom(). <br>
SYSCALL_DEFINE4(recv, int, fd, void __user *, ubuf, size_t, size, unsigned int, flags) <br>
-> __sys_recvfrom() <br>
```
struct socket sock = sockfd_lookup_light(fd, ...);
```
SYSCALL_DEFINE3(recvmsg, int, fd, struct user_msghdr __user *, msg, unsigned int, flags) <br>
-> __sys_recvmsg() <br>
SYSCALL_DEFINE6(recvfrom, int, fd, void __user *, ubuf, size_t, size,
		unsigned int, flags, struct sockaddr __user *, addr, int __user *, addr_len) <br>
-> __sys_recvfrom() <br>

-> sock_recvmsg() <br>
-> sock_recvmsg_nosec() <br>
-> sock->ops->recvmsg() = inet_recvmsg() <br>
-> sk->sk_prot->recvmsg() = tcp_recvmsg() <br>
-> |- skb_queue_walk() <br>
&emsp; |- skb_copy_datagram_msg() <br>
&emsp; &emsp; -> skb_copy_datagram_iter() <br>

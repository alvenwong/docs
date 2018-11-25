This file shows the function invocations in the kernel when transmitting or receiving packets.
# Transmit Packets
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

## Functinon from TCP layer to net device
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
-> __dev_xmit_skb() -> qdisc->enqueue() (I/O Interrupt) <br>
-> sch_direct_xmit() <br>
-> dev_hard_start_xmit() <br> 
-> xmit_one() <br>
-> netdev_start_xmit() <br>
-> __netdev_start_xmit() <br>

## I/O Interrupts
```
static int __init net_dev_init(void)
{
	open_softirq(NET_TX_SOFTIRQ, net_tx_action);
	open_softirq(NET_RX_SOFTIRQ, net_rx_action);
}
```
The hardware devices periodically raise signals (NET_RX_SOFTIRQ) to transmit packets from QDisc.
net_tx_action() <br>
-> qdisc_run() <br>
-> __qdisc_run <br>
-> qdisc_restart() <br>
-> dequeue_skb() <br>
-> sch_direct_xmit() <br>
-> dev_hard_start_xmit() <br> 
-> xmit_one() <br>
-> netdev_start_xmit() <br>
-> __netdev_start_xmit() <br>

# Receive Packets

## I/O Interrupts
<img align="center" src="https://github.com/alvenwong/docs/blob/master/IO_Interrupt.png" width="500"> <p>
This figure displays the process that network devices receive packets and send signals to handle the I/O interrupts. <p>
When a packet arrives, the hardware device raises signal on an IRQ line (IRQn), where n is the index for the interrupts. For example, the IRQ n for Keyboard signal is 2 and Network interface is 11. <p>

The Programmable Interrupt Controller (PIC) monitors the IRQ lines, checking for the raised signals. If a signal occurs on an IRQ line, PIC converts the raised signal into a corresponding vector and stores the vector in an Interrupt Controller I/O port, thus allowing the CPU to read it via the data bus. Then PIC sends a raised signal to the processor INTR, issuing an interrupt. The INT number is the IRQ number plus 32. For example, the IRQ n for Keyboard signal is 34 and Network interface is 43. <p>

PIC waits until the CPU acknowledges the interrupt signal by writing into one of the Programmable Interrupt Controllers (PIC) I/O ports; when this occurs, clears the INTR line. <p>

When receiving an interrupt signal, do_IRQ(n) will be invoked to handle this interrupt. 
do_IRQ() <br>
// update task cpu time, irq time and preempt counter <br>
-> |- irq_enter() -> account_irq_enter_time() <br>
// handle hardware interrups <br>
-> |- handle_irq(irq, regs) <br>
// handle softirq <br>
-> |- irq_exit() -> account_irq_exit_time() <br>
-> invoke_softirq() <br>
-> __do_softirq() <br>
-> struct softirq_action->action() <br>
-> net_rx_action() <br>
```
static int __init net_dev_init(void)
{
	open_softirq(NET_TX_SOFTIRQ, net_tx_action);
	open_softirq(NET_RX_SOFTIRQ, net_rx_action);
}
```

net_rx_action() <br>
-> napi_poll() <br>
// skb list is in struct napi_struct->gro_list
-> gro_cell_poll() <br>
-> napi_gro_recevie() <br>
-> napi_gro_complete() <br>
-> netif_receive_skb_internal() <br>
-> __netif_receive_skb() <br>
-> __netif_receive_skb_core() <br>
-> deliver_skb() <br>
-> ip_rcv() <br>

## Functions from IP layer to TCP layer

ip_rcv() <br>
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

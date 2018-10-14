This file displays the process flow for RTT calculation in the kernel.

# Transmitting a packet
tcp_sendmsg() <br>
-> tcp_sendmsg_locked() <br>
-> tcp_push_one() <br>
-> tcp_write_xmit() <br>
-> |- tcp_mstamp_refresh() <br>
&emsp;   |- __tcp_transmit_skb() <p>
```
static inline void tcp_mstamp_refresh(struct tcp_sock *tp)
{
	u64 val = tcp_clock_us();

	if (val > tp->tcp_mstamp)
		tp->tcp_mstamp = val;
}
```
```
static int __tcp_transmit_skb(struct sock *sk, struct sk_buff *skb, ...)
: skb->skb_mstamp = tp->tcp_mstamp;
```

# Receiving a packet
tcp_v4_rcv() <br>
-> tcp_v4_do_rcv() <br>
-> tcp_rcv_state_process() <br>
-> |- tcp_mstamp_refresh() <br>
&emsp; |- tcp_ack() <br>
&emsp; &emsp; -> tcp_clean_rtx_queue() <br>
&emsp; &emsp; -> |- tcp_stamp_us_delta() <br>
&emsp; &emsp; &emsp; |- tcp_ack_update_rtt() <br>
&emsp; &emsp; &emsp; -> tcp_rtt_estimator() <br>
```
static int tcp_clean_rtx_queue(struct sock *sk, ...)
: last_ackt = skb->skb_mstamp;
  WARN_ON_ONCE(last_ackt == 0);
  if (!first_ackt)
	  first_ackt = last_ackt;

  seq_rtt_us = tcp_stamp_us_delta(tp->tcp_mstamp, first_ackt);
  rtt_update = tcp_ack_update_rtt(sk, flag, seq_rtt_us, ...);
```
```
static void tcp_rtt_estimator(struct sock *sk, long mrtt_us)
: m -= (srtt >> 3);
  srtt += m;		/* rtt = 7/8 rtt + 1/8 new */
```

---
title: tcp nagle + ack delay导致的延迟问题
date: 2025-02-24 11:11:00 +0800
categories: [Blogging, network, tcp]
tags: [writing]
---

+ [我们来说一说TCP神奇的40ms](https://zhuanlan.zhihu.com/p/53169705)

文章写的很糙，nagle下小于MSS的小数据包会等没有未确认数据包的时候发送，这个算法的设计的初衷是频繁的发送小字节的包会导致网络拥塞

tcp delay ack开启之后，当接收到数据包时，TCP 不会立即发送 ACK，而是等待一段时间看看是否有数据可以一起发送。如果在这段时间内有数据需要发送，ACK 会与数据一起发送，从而减少单独发送 ACK 包的次数

具体代码在/net/ipv4/tcp_input.c

```c
/*
 * Check if sending an ack is needed.
 */
static void __tcp_ack_snd_check(struct sock *sk, int ofo_possible)
{
	struct tcp_sock *tp = tcp_sk(sk);
	unsigned long rtt, delay;


    // rcv_nxt下一个期望接收的序列号，rcv_wup最后一次发送ACK时的接收窗口右边缘。
	    /* More than one full frame received... */
	if (((tp->rcv_nxt - tp->rcv_wup) > inet_csk(sk)->icsk_ack.rcv_mss &&
	     /* ... and right edge of window advances far enough.
	      * (tcp_recvmsg() will send ACK otherwise).
	      * If application uses SO_RCVLOWAT, we want send ack now if
	      * we have not received enough bytes to satisfy the condition.
	      */
      // copied_seq应用程序已经读取的数据的序列号。
      // sk_rcvlowat接收低水位标记，表示应用程序希望接收的最小数据量。
	    (tp->rcv_nxt - tp->copied_seq < sk->sk_rcvlowat ||
	     __tcp_select_window(sk) >= tp->rcv_wnd)) ||
	    /* We ACK each frame or... */
	    tcp_in_quickack_mode(sk) ||
	    /* Protocol state mandates a one-time immediate ACK */
	    inet_csk(sk)->icsk_ack.pending & ICSK_ACK_NOW) {
		/* If we are running from __release_sock() in user context,
		 * Defer the ack until tcp_release_cb().
     * 用户持有，且允许延迟ack
		 */
		if (sock_owned_by_user_nocheck(sk) &&
		    READ_ONCE(sock_net(sk)->ipv4.sysctl_tcp_backlog_ack_defer)) {
			set_bit(TCP_ACK_DEFERRED, &sk->sk_tsq_flags);
			return;
		}
send_now:
		tcp_send_ack(sk);
		return;
	}

	if (!ofo_possible || RB_EMPTY_ROOT(&tp->out_of_order_queue)) {
		tcp_send_delayed_ack(sk);
		return;
	}

	if (!tcp_is_sack(tp) ||
	    tp->compressed_ack >= READ_ONCE(sock_net(sk)->ipv4.sysctl_tcp_comp_sack_nr))
		goto send_now;

	if (tp->compressed_ack_rcv_nxt != tp->rcv_nxt) {
		tp->compressed_ack_rcv_nxt = tp->rcv_nxt;
    // dup_ack_counter重复ACK的计数器，用于快速重传。
		tp->dup_ack_counter = 0;
	}
	if (tp->dup_ack_counter < TCP_FASTRETRANS_THRESH) {
		tp->dup_ack_counter++;
		goto send_now;
	}
  // compressed_ack压缩ACK的计数器。
	tp->compressed_ack++;
	if (hrtimer_is_queued(&tp->compressed_ack_timer))
		return;

	/* compress ack timer : 5 % of rtt, but no more than tcp_comp_sack_delay_ns */

	rtt = tp->rcv_rtt_est.rtt_us;
	if (tp->srtt_us && tp->srtt_us < rtt)
		rtt = tp->srtt_us;

	delay = min_t(unsigned long,
		      READ_ONCE(sock_net(sk)->ipv4.sysctl_tcp_comp_sack_delay_ns),
		      rtt * (NSEC_PER_USEC >> 3)/20);
	sock_hold(sk);
	hrtimer_start_range_ns(&tp->compressed_ack_timer, ns_to_ktime(delay),
			       READ_ONCE(sock_net(sk)->ipv4.sysctl_tcp_comp_sack_slack_ns),
			       HRTIMER_MODE_REL_PINNED_SOFT);
}
```

这里规则可以看成几段

1. 最开始的几个判断
    >  条件1：接收到的数据超过一个MSS（最大分段大小），并且接收窗口的右边缘已经推进得足够远。
    
    >  条件2：应用程序设置了 SO_RCVLOWAT（接收低水位标记），并且接收到的数据不足以满足条件。
    
    >  条件3：TCP处于快速确认模式（Quick ACK Mode）。
    
    >  条件4：协议状态要求立即发送一次性的ACK

2. 然后他判断了有无乱序数据包，没有的话可以delay ack

	```c
	if (!ofo_possible || RB_EMPTY_ROOT(&tp->out_of_order_queue)) {
		tcp_send_delayed_ack(sk);
		return;
	}
	```

3. 之后处理sack,如果启用了SACK（选择性确认），并且满足以下条件，则立即发送ACK：

	```c
	// 如果未启用SACK，或者压缩ACK的次数超过系统配置的阈值，则立即发送ACK。
	if (!tcp_is_sack(tp) ||
			tp->compressed_ack >= READ_ONCE(sock_net(sk)->ipv4.sysctl_tcp_comp_sack_nr))
			goto send_now;
	```

4. 如果接收到的数据包是乱序的，并且重复ACK的次数小于快速重传的阈值（TCP_FASTRETRANS_THRESH），则增加重复ACK计数器，并立即发送ACK：

	```c
	if (tp->dup_ack_counter < TCP_FASTRETRANS_THRESH) {
			tp->dup_ack_counter++;
			goto send_now;
	}
	```

5. 上述都不满足，压缩ACK定时器

	如果上述条件都不满足，则启动一个压缩ACK定时器，延迟发送ACK

	```c
	delay = min_t(unsigned long,
								READ_ONCE(sock_net(sk)->ipv4.sysctl_tcp_comp_sack_delay_ns),
								rtt * (NSEC_PER_USEC >> 3)/20);
	sock_hold(sk);
	hrtimer_start_range_ns(&tp->compressed_ack_timer, ns_to_ktime(delay),
												READ_ONCE(sock_net(sk)->ipv4.sysctl_tcp_comp_sack_slack_ns),
												HRTIMER_MODE_REL_PINNED_SOFT);
	```

+ 定时器的延迟时间取以下两个值的最小值：
  + 系统配置的压缩ACK延迟时间（tcp_comp_sack_delay_ns）。
  + RTT（往返时间）的5%。
+ 启动定时器后，ACK会延迟发送。

nagle + tcp ack delay开启下，小于MSS的包挺多的话，延迟还是比较大的

缓解手段也比较简单

client侧

1. 关nagle
2. 合并小包

server侧

1. 关delay ack
2. quick ack

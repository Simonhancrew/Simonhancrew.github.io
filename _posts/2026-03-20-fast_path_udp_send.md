---
title: udp_sendmsg里的fastpath
date: 2026-03-20 00:00:00 +0800
categories: [Blogging, linux, socket, io, udp]
tags: [writing]
---


当时在做multipath的时候看kernel里udp发包的代码，发现udp_sendmsg里有个connect的状态，而且这个状态的判断分支里sock的state是TCP_ESTABLISHED，这个状态是TCP协议的状态，UDP协议里是没有这个状态的。

后来看了下代码, 发现这个是复用的, 这里的connect使用了ip4_datagram_connect, 这里直接设置了sk_state = TCP_ESTABLISHED，虽然名字叫TCP，但这是Linux内核的一种复用设计，用来表示socket已绑定了对端地址。

```c
struct proto udp_prot = {
	.name			= "UDP",
	.owner			= THIS_MODULE,
	.close			= udp_lib_close,
	.pre_connect		= udp_pre_connect,
	.connect		= ip4_datagram_connect,
	.disconnect		= udp_disconnect,
	.ioctl			= udp_ioctl,
	.init			= udp_init_sock,
	.destroy		= udp_destroy_sock,
	.setsockopt		= udp_setsockopt,
	.getsockopt		= udp_getsockopt,
	.sendmsg		= udp_sendmsg,
	.recvmsg		= udp_recvmsg,
	.sendpage		= udp_sendpage,
	.release_cb		= ip4_datagram_release_cb,
	.hash			= udp_lib_hash,
	.unhash			= udp_lib_unhash,
	.rehash			= udp_v4_rehash,
	.get_port		= udp_v4_get_port,
#ifdef CONFIG_BPF_SYSCALL
	.psock_update_sk_prot	= udp_bpf_update_proto,
#endif
	.memory_allocated	= &udp_memory_allocated,
	.sysctl_mem		= sysctl_udp_mem,
	.sysctl_wmem_offset	= offsetof(struct net, ipv4.sysctl_udp_wmem_min),
	.sysctl_rmem_offset	= offsetof(struct net, ipv4.sysctl_udp_rmem_min),
	.obj_size		= sizeof(struct udp_sock),
	.h.udp_table		= &udp_table,
	.diag_destroy		= udp_abort,
};
```

内部代码形如

```c
int ip4_datagram_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
	int res;

	lock_sock(sk);
	res = __ip4_datagram_connect(sk, uaddr, addr_len);
	release_sock(sk);
	return res;
}

int __ip4_datagram_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
	struct inet_sock *inet = inet_sk(sk);
	struct sockaddr_in *usin = (struct sockaddr_in *) uaddr;
	struct flowi4 *fl4;
	struct rtable *rt;
	__be32 saddr;
	int oif;
	int err;


	if (addr_len < sizeof(*usin))
		return -EINVAL;

	if (usin->sin_family != AF_INET)
		return -EAFNOSUPPORT;

	sk_dst_reset(sk);

	oif = sk->sk_bound_dev_if;
	saddr = inet->inet_saddr;
	if (ipv4_is_multicast(usin->sin_addr.s_addr)) {
		if (!oif || netif_index_is_l3_master(sock_net(sk), oif))
			oif = inet->mc_index;
		if (!saddr)
			saddr = inet->mc_addr;
	}
	fl4 = &inet->cork.fl.u.ip4;
	rt = ip_route_connect(fl4, usin->sin_addr.s_addr, saddr,
			      RT_CONN_FLAGS(sk), oif,
			      sk->sk_protocol,
			      inet->inet_sport, usin->sin_port, sk);
	if (IS_ERR(rt)) {
		err = PTR_ERR(rt);
		if (err == -ENETUNREACH)
			IP_INC_STATS(sock_net(sk), IPSTATS_MIB_OUTNOROUTES);
		goto out;
	}

	if ((rt->rt_flags & RTCF_BROADCAST) && !sock_flag(sk, SOCK_BROADCAST)) {
		ip_rt_put(rt);
		err = -EACCES;
		goto out;
	}
	if (!inet->inet_saddr)
		inet->inet_saddr = fl4->saddr;	/* Update source address */
	if (!inet->inet_rcv_saddr) {
		inet->inet_rcv_saddr = fl4->saddr;
		if (sk->sk_prot->rehash)
			sk->sk_prot->rehash(sk);
	}
	inet->inet_daddr = fl4->daddr;
	inet->inet_dport = usin->sin_port;
	reuseport_has_conns(sk, true);
	sk->sk_state = TCP_ESTABLISHED;
	sk_set_txhash(sk);
	inet->inet_id = prandom_u32();

	sk_dst_set(sk, &rt->dst);
	err = 0;
out:
	return err;
}
```

可以看到的是这里的sk->sk_state = TCP_ESTABLISHED;是直接设置的，并没有判断协议类型，导致了udp_sendmsg里会有这个状态的判断分支。

```c
int udp_sendmsg(struct sock *sk, struct msghdr *msg, size_t len)
{
	struct inet_sock *inet = inet_sk(sk);
	struct udp_sock *up = udp_sk(sk);
	DECLARE_SOCKADDR(struct sockaddr_in *, usin, msg->msg_name);
	struct flowi4 fl4_stack;
	struct flowi4 *fl4;
	int ulen = len;
	struct ipcm_cookie ipc;
	struct rtable *rt = NULL;
	int free = 0;
	int connected = 0;
	__be32 daddr, faddr, saddr;
	__be16 dport;
	u8  tos;
	int err, is_udplite = IS_UDPLITE(sk);
	int corkreq = READ_ONCE(up->corkflag) || msg->msg_flags&MSG_MORE;
	int (*getfrag)(void *, char *, int, int, int, struct sk_buff *);
	struct sk_buff *skb;
	struct ip_options_data opt_copy;
  // ...

  	/*
	 *	Get and verify the address. 如果dst有地址的时候
	 */
	if (usin) {
		if (msg->msg_namelen < sizeof(*usin))
			return -EINVAL;
		if (usin->sin_family != AF_INET) {
			if (usin->sin_family != AF_UNSPEC)
				return -EAFNOSUPPORT;
		}

		daddr = usin->sin_addr.s_addr;
		dport = usin->sin_port;
		if (dport == 0)
			return -EINVAL;
	} else {
    // 如果没带地址 (说明调用的是 send 或已 connect 的 sendto)
    // 检查 socket 状态是否为 TCP_ESTABLISHED (即 UDP 已 connect)
		if (sk->sk_state != TCP_ESTABLISHED)
			return -EDESTADDRREQ;
    // 直接从内核管理的 sock 结构体中获取缓存的地址和端口
		daddr = inet->inet_daddr;
		dport = inet->inet_dport;
		/* Open fast path for connected socket.
		   Route will not be used, if at least one option is set.
		 */
		connected = 1;
	}
  // ...
  if (connected)
		rt = (struct rtable *)sk_dst_check(sk, 0);

	if (!rt) {
		struct net *net = sock_net(sk);
		__u8 flow_flags = inet_sk_flowi_flags(sk);

		fl4 = &fl4_stack;

		flowi4_init_output(fl4, ipc.oif, ipc.sockc.mark, tos,
				   RT_SCOPE_UNIVERSE, sk->sk_protocol,
				   flow_flags,
				   faddr, saddr, dport, inet->inet_sport,
				   sk->sk_uid);

		security_sk_classify_flow(sk, flowi4_to_flowi_common(fl4));
		rt = ip_route_output_flow(net, fl4, sk);
		if (IS_ERR(rt)) {
			err = PTR_ERR(rt);
			rt = NULL;
			if (err == -ENETUNREACH)
				IP_INC_STATS(net, IPSTATS_MIB_OUTNOROUTES);
			goto out;
		}

		err = -EACCES;
		if ((rt->rt_flags & RTCF_BROADCAST) &&
		    !sock_flag(sk, SOCK_BROADCAST))
			goto out;
		if (connected)
			sk_dst_set(sk, dst_clone(&rt->dst));
	}
  // ...
}
```


### 性能考量

**如果不 connect (使用 sendto)**： 内核在每次调用 sendto同时指定地址时(dst不传NULL)，都必须执行以下步骤：
+ 连接检查：检查套接字状态。
+ 路由查找：根据目的 IP 查找路由表，确定数据包从哪个网卡出、下一跳是谁。
+ 资源开销：路由表查询（Route Lookup）是原子操作，涉及锁和哈希计算，高并发下开销明显。
+ 释放路径：发送完后，刚才查到的路由信息会被立即丢弃（除非有全局缓存）。

**如果 connect 之后 (使用 send)**： 内核在 connect 调用期间，只做一次路由查找，并把结果（dst_entry）直接挂载到这个 socket 结构体上。
+ 后续 send 时，内核发现状态是 TCP_ESTABLISHED，直接取出缓存的路由信息。ip_route_output_flow会被跳过
+ BPF cgroup挂钩被跳过


### REF

1. [linux kernel source](https://elixir.bootlin.com/linux/v5.15.4/source/net/ipv4/udp.c#L1041)
2. [Everything you ever wanted to know about UDP sockets but were afraid to ask, part 1](https://blog.cloudflare.com/everything-you-ever-wanted-to-know-about-udp-sockets-but-were-afraid-to-ask-part-1/)

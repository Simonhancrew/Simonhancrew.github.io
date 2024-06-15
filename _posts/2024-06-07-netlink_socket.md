---
title: netlink socket
date: 2024-06-07 12:29:00 +0800
categories: [Blogging, linux, netlink]
tags: [writing]
---

Netlink 是一种进程间通信（IPC）机制，主要用于内核和用户空间进程之间的通信，特别是在网络子系统中, 可以用来获取和设置内核信息，包括网络路由表、接口信息等。

如果为了看代码怎么动态获取网络接口的变更，可以参考下面的代码。

```cpp
#include <arpa/inet.h>
#include <cstring>
#include <iostream>
#include <linux/netlink.h>
#include <linux/rtnetlink.h>
#include <sys/socket.h>
#include <unistd.h>

#define BUFFER_SIZE 8192

/*
  @description: This is a helper function that parses the attributes of a
  netlink message.
  @param: tb is an array of pointers to rtattr structures.
  @param: max is the maximum number of attributes that can be stored in tb.
  @param: rta is a pointer to the first attribute in the message.
  @param: len is the length of the message.
*/
void ParseAddr(struct rtattr **tb, int max, struct rtattr *rta, int len) {
  memset(tb, 0, sizeof(struct rtattr *) * (max + 1));
  // RTA_OK: A macro that checks if the current attribute is valid.
  while (RTA_OK(rta, len)) {
    if (rta->rta_type <= max) {
      tb[rta->rta_type] = rta;
    }
    rta = RTA_NEXT(rta, len);
  }
}

/*
  @description: This function handles a new address message from the netlink.
  @param: nlh is a pointer to the netlink message header.
*/
void HandleNewAddr(struct nlmsghdr *nlh) {
  // NLMSG_DATA: A macro that returns a pointer to the data in the message.
  auto *ifa = reinterpret_cast<struct ifaddrmsg *>(NLMSG_DATA(nlh));
  // IFA_RTA: A macro that returns a pointer to the first attribute in the message.
  struct rtattr *rth = IFA_RTA(ifa);
  // IFA_PAYLOAD: A macro that returns the length of the message.
  int rtl = IFA_PAYLOAD(nlh);
  struct rtattr *tb[IFA_MAX + 1];
  // IFA_MAX is the maximum number of attributes that can be stored in tb.
  ParseAddr(tb, IFA_MAX, rth, rtl);

  if (tb[IFA_LOCAL]) {
    char ip[INET6_ADDRSTRLEN];
    if (ifa->ifa_family == AF_INET) {
      inet_ntop(AF_INET, RTA_DATA(tb[IFA_LOCAL]), ip, sizeof(ip));
    } else if (ifa->ifa_family == AF_INET6) {
      inet_ntop(AF_INET6, RTA_DATA(tb[IFA_LOCAL]), ip, sizeof(ip));
    }
    std::cout << "New address added: " << ip << std::endl;
  }
}

int main() {
  int sock_fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE);
  if (sock_fd < 0) {
    std::cerr << "Failed to create netlink socket" << std::endl;
    return -1;
  }

  struct sockaddr_nl sa {};
  /*
    AF_NETLINK: The address family for netlink sockets.
  */
  sa.nl_family = AF_NETLINK;
  /*
    RTMGRP_LINK: Receive link layer notifications.
    RTMGRP_IPV4_IFADDR: Receive IPv4 address change notifications.
    RTMGRP_IPV6_IFADDR: Receive IPv6 address change notifications.
  */
  sa.nl_groups = RTMGRP_LINK | RTMGRP_IPV4_IFADDR | RTMGRP_IPV6_IFADDR;

  if (bind(sock_fd, reinterpret_cast<struct sockaddr *>(&sa), sizeof(sa)) < 0) {
    std::cerr << "Failed to bind netlink socket" << std::endl;
    close(sock_fd);
    return -1;
  }
  /*
    The netlink message format is as follows:
    +-------------------+
    | nlmsghdr          |
    +-------------------+
    | ifaddrmsg         |
    +-------------------+
    | rtattr            |
    +-------------------+
    | rtattr            |
    +-------------------+
    | ...               |
    +-------------------+
    | rtattr            |
    +-------------------+
    BUFFER_SIZE is 8192 bytes.
  */
  char buffer[BUFFER_SIZE];
  while (true) {
    auto len = recv(sock_fd, buffer, sizeof(buffer), 0);
    if (len < 0) {
      std::cerr << "Failed to receive netlink message" << std::endl;
      break;
    }
    /*
      NLMSG_OK: A macro that checks if the current netlink message is valid.
      NLMSG_NEXT: A macro that returns a pointer to the next netlink message.
    */
    for (auto *nh = reinterpret_cast<struct nlmsghdr *>(buffer);
         NLMSG_OK(nh, len); nh = NLMSG_NEXT(nh, len)) {
      // NLMSG_DONE is a macro that indicates the end of a multi-part message
      if (nh->nlmsg_type == NLMSG_DONE) {
        break;
      }
      // RTM_NEWADDR: A new address has been added.
      // RTM_DELADDR: An address has been removed.
      if (nh->nlmsg_type == RTM_NEWADDR || nh->nlmsg_type == RTM_DELADDR) {
        HandleNewAddr(nh);
      }
    }
  }

  close(sock_fd);
  return 0;
}

```

## REF

1. [Netlink Library (libnl)](https://www.infradead.org/~tgr/libnl/doc/core.html)
2. [Introduction to Netlink](https://www.kernel.org/doc/html/v6.7/userspace-api/netlink/intro.html)

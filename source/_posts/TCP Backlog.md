---
title: TCP Backlog
date: 2021-05-17 15:30:41
tags: tcp
---

# TCP backlog

TCP 3 way handshake

```sequence
client -> server: syn j
server -> client: syn k, ack j+1
client -> server: ack k+1
```

server會維護有兩個queue
* incomplete connection (SYN_RECV) queue
* complete connection (ESTABLISHED) queue

server收到syn並回覆syn/ack後將connection放到SYN_RECV queue，SYN_RECV queue大小由net.ipv4.tcp_max_syn_backlog控制(net.ipv4.tcp_syncookies 0)。

當server收到ack之後，將connection從SYN_RECV queue移到ESTABLISHED queue，connection在ESTABLISHED queue被accept後移出。ESTABLISHED queue大小由listen的backlog參數控制，如果backlog小於net.core.somaxconn會用net.core.somaxconn的設定

> The behavior of the backlog argument on TCP sockets changed with Linux 2.2. Now it specifies the queue length for completely established sockets waiting to be accepted, instead of the number of incomplete connection requests. The maximum length of the queue for incomplete sockets can be set using /proc/sys/net/ipv4/tcp_max_syn_backlog. When syncookies are enabled there is no logical maximum length and this setting is ignored. See tcp(7) for more information.

> If the backlog argument is greater than the value in /proc/sys/net/core/somaxconn, then it is silently truncated to that value.  Since Linux 5.4, the default in this file is 4096; in earlier kernels, the default value is 128. In kernels before 2.4.25, this limit was a hard coded value, SOMAXCONN, with the value 128.

當ESTABLISHED queue滿了無法將connection從SYN_RECV queue移過來且net.ipv4.tcp_abort_on_overflow 1時，server收到ack會回rst；net.ipve.tcp_abort_on_overflow 0時，會直接丟棄ack，此時從client角度會認為connection被接受並不知道ack被丟棄(connection稱做half-open)，client便直接送data，connection對於server來說還不是ESTABLISHED因此丟棄data。在SYN_RECV queue的connection會有timeout重送syn/ack機制(exponential backoff)，timeout時重送syn/ack直到收到ack並成功將connection移往ESTABLISHED queue或是超過net.ipv4.tcp_synack_retries，超過net.ipv4.tcp_synack_retries server回覆rst。ESTABLISHED queue滿了也會丟棄一定程度的syn即使SYN_RECV queue沒有滿

## Reference

* [How TCP backlog works in Linux](http://veithen.io/2014/01/01/how-tcp-backlog-works-in-linux.html)
* [listen(2) — Linux manual page](https://man7.org/linux/man-pages/man2/listen.2.html)
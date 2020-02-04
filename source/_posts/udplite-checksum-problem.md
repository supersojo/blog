---
title: udplite checksum problem
date: 2019-07-16 22:18:10
tags:
- udplite
- checksum
categories:
- network
---

## 问题

在linux 3.10.0上面，遇到一个udplite校验失败问题。

## 分析

udp报文格式:
```
MAC|IP|UDP|Payload
```

而如下的udplite报文被drop了。
```
MAC|IP|UDP|Payload
14  20  8   61

udplite_rcv
->__udp4_lib_rcv
->udp4_csum_init
->udplite_checksum_init
->skb_checksum_init_zero_check
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~    报bad checksum
```

在udplite_checksum_init的开始处:
```
skb->len = 61 + 8 =69
udphdr->len = 64
```
UDPLITE协议中，udphdr中len表示对报文多少进行了校验。
即UDP_SKB_CB(skb)->partial_cov=1



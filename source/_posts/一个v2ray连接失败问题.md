---
title: 一个v2ray连接失败问题
date: 2020-10-22 20:10:13
tags:
- v2ray

categories:
- linux
---

这个问题我已经遇到好几次，每次都要到网上搜索一番，这里记录一下。


```
2020/10/26 01:01:33 tcp:127.0.0.1:50610 accepted tcp:beacons.gcp.gvt2.com:443 [proxy] 
2020/10/26 01:01:35 [Warning] failed to handler mux client connection > v2ray.com/core/proxy/vmess/outbound: connection ends > v2ray.com/core/proxy/vmess/outbound: failed to read header > v2ray.com/core/proxy/vmess/encoding: failed to read response header > websocket: close 1000 (normal)

```
v2ray报这个错主要是时间不同步问题，只要把客户端时间设置下就解决了。

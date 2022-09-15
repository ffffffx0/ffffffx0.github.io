---
layout: post
title: "《TCP/IP详解卷1：协议》笔记"
description: "Datascience Guide"
category: "Protocols"
tags: ["TCP/IP","Protocols","NetWork"]
---

### 分层

* 链路层

 操作系统中的设备驱动程序和计算机对应的网络接口,ARP(地址解析协议)和RARP(逆地址解析协议)
 
* 网络层

 IP协议（网际协议），ICMP(Internet互联网控制报文协议),IGMP(Internet组管理协议)
 
* 运输层

 TCP（传输控制协议），UDP(用户数据报协议)
 
* 应用层

 Telnet,FTP,SMTP,SNTP
 
### 工具
 
##### 1,tcpdump

```
14:21:13.735001 IP 10.3.142.10.http > 10.3.142.62.31580: Flags [S.], seq 1882046784, ack 245205655, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
14:21:13.735654 IP 10.3.142.10.http > 10.3.142.62.31580: Flags [.], ack 381, win 237, length 0
14:21:13.744116 IP 10.3.142.10.http > 10.3.142.62.31581: Flags [S.], seq 4167052849, ack 2571110651, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
14:21:13.744748 IP 10.3.142.10.http > 10.3.142.62.31581: Flags [.], ack 379, win 237, length 0
14:21:13.746761 IP 10.3.142.10.http > 10.3.142.62.31580: Flags [P.], seq 1:449, ack 381, win 237, length 448
14:21:13.747423 IP 10.3.142.10.http > 10.3.142.62.31580: Flags [F.], seq 449, ack 382, win 237, length 0
14:21:13.751565 IP 10.3.142.10.http > 10.3.142.62.31578: Flags [.], seq 4057624839:4057627759, ack 3715079403, win 237, length 2920
14:21:13.751614 IP 10.3.142.10.http > 10.3.142.62.31578: Flags [P.], seq 2920:2977, ack 1, win 237, length 57
14:21:13.752526 IP 10.3.142.10.http > 10.3.142.62.31578: Flags [F.], seq 2977, ack 2, win 237, length 0
14:21:13.762168 IP 10.3.142.10.http > 10.3.142.62.31581: Flags [P.], seq 1:628, ack 379, win 237, length 627
14:21:13.763615 IP 10.3.142.10.http > 10.3.142.62.31581: Flags [F.], seq 628, ack 380, win 237, length 0
14:21:13.768904 IP 10.3.142.10.http > 10.3.142.62.31584: Flags [S.], seq 2698062966, ack 3290428620, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
14:21:13.769580 IP 10.3.142.10.http > 10.3.142.62.31584: Flags [.], ack 462, win 237, length 0
14:21:13.778808 IP 10.3.142.10.http > 10.3.142.62.31584: Flags [P.], seq 1:481, ack 462, win 237, length 480
14:21:13.779459 IP 10.3.142.10.http > 10.3.142.62.31584: Flags [F.], seq 481, ack 463, win 237, length 0
14:21:13.823270 IP 10.3.142.10.http > 10.3.142.62.31587: Flags [S.], seq 2687230459, ack 3482940275, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
14:21:13.823914 IP 10.3.142.10.http > 10.3.142.62.31587: Flags [.], ack 471, win 237, length 0
14:21:13.830794 IP 10.3.142.10.http > 10.3.142.62.31588: Flags [S.], seq 1796206633, ack 2662870446, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
14:21:13.831450 IP 10.3.142.10.http > 10.3.142.62.31588: Flags [.], ack 467, win 237, length 0
14:21:13.839749 IP 10.3.142.10.http > 10.3.142.62.31587: Flags [.], seq 1:2921, ack 471, win 237, length 2920
14:21:13.839833 IP 10.3.142.10.http > 10.3.142.62.31587: Flags [P.], seq 2921:3380, ack 471, win 237, length 459
14:21:13.840700 IP 10.3.142.10.http > 10.3.142.62.31587: Flags [F.], seq 3380, ack 472, win 237, length 0
14:21:13.852202 IP 10.3.142.10.http > 10.3.142.62.31588: Flags [.], seq 1:2921, ack 467, win 237, length 2920
14:21:13.852227 IP 10.3.142.10.http > 10.3.142.62.31588: Flags [P.], seq 2921:3724, ack 467, win 237, length 803
14:21:13.853330 IP 10.3.142.10.http > 10.3.142.62.31588: Flags [F.], seq 3724, ack 468, win 237, length 0
14:21:13.866645 IP 10.3.142.10.http > 10.3.142.62.31592: Flags [S.], seq 3544149494, ack 1340127701, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
14:21:13.867311 IP 10.3.142.10.http > 10.3.142.62.31592: Flags [.], ack 470, win 237, length 0
14:21:13.870886 IP 10.3.142.10.http > 10.3.142.62.31592: Flags [P.], seq 1:489, ack 470, win 237, length 488
14:21:13.871564 IP 10.3.142.10.http > 10.3.142.62.31592: Flags [F.], seq 489, ack 471, win 237, length 0
14:21:13.980743 IP 10.3.142.10.http > 10.3.142.62.31599: Flags [S.], seq 2813159308, ack 4229199650, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
14:21:13.981352 IP 10.3.142.10.http > 10.3.142.62.31599: Flags [.], ack 376, win 237, length 0
14:21:14.026220 IP 10.3.142.10.http > 10.3.142.62.31602: Flags [S.], seq 3209688867, ack 3217793971, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
14:21:14.026861 IP 10.3.142.10.http > 10.3.142.62.31602: Flags [.], ack 346, win 237, length 0
14:21:14.028618 IP 10.3.142.10.http > 10.3.142.62.31602: Flags [P.], seq 1:416, ack 346, win 237, length 415
14:21:14.029249 IP 10.3.142.10.http > 10.3.142.62.31602: Flags [F.], seq 416, ack 347, win 237, length 0
14:21:14.034137 IP 10.3.142.10.http > 10.3.142.62.31599: Flags [P.], seq 1:629, ack 376, win 237, length 628
14:21:14.034881 IP 10.3.142.10.http > 10.3.142.62.31599: Flags [F.], seq 629, ack 377, win 237, length 0
14:21:14.056644 IP 10.3.142.10.http > 10.3.142.62.31604: Flags [S.], seq 2676426302, ack 714186577, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
14:21:14.057629 IP 10.3.142.10.http > 10.3.142.62.31604: Flags [.], ack 457, win 237, length 0
14:21:14.069155 IP 10.3.142.10.http > 10.3.142.62.31604: Flags [P.], seq 1:1277, ack 457, win 237, length 1276
14:21:14.069881 IP 10.3.142.10.http > 10.3.142.62.31604: Flags [F.], seq 1277, ack 458, win 237, length 0
14:21:14.070129 IP 10.3.142.10.http > 10.3.142.62.31606: Flags [S.], seq 2426352170, ack 2408227546, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
14:21:14.070747 IP 10.3.142.10.http > 10.3.142.62.31606: Flags [.], ack 379, win 237, length 0
14:21:14.072803 IP 10.3.142.10.http > 10.3.142.62.31606: Flags [P.], seq 1:432, ack 379, win 237, length 431
14:21:14.073406 IP 10.3.142.10.http > 10.3.142.62.31606: Flags [F.], seq 432, ack 380, win 237, length 0
14:21:14.101003 IP 10.3.142.10.http > 10.3.142.62.31607: Flags [S.], seq 2210610393, ack 875667818, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
14:21:14.101639 IP 10.3.142.10.http > 10.3.142.62.31607: Flags [.], ack 454, win 237, length 0
```
 
##### 2,traceroute
 

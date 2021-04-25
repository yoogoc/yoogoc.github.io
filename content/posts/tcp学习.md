---
title: tcp学习
tags:
- 计算机网络
date: 2021-01-16 16:58:00
---

# what is TCP

## 简介

**传输控制协议**（英语：**T**ransmission **C**ontrol **P**rotocol，缩写：**TCP**）是一种面向连接的、可靠的、基于[字节流](https://zh.wikipedia.org/wiki/字節流)的[传输层](https://zh.wikipedia.org/wiki/传输层)通信协议，由[IETF](https://zh.wikipedia.org/wiki/IETF)的[RFC](https://zh.wikipedia.org/wiki/RFC) [793](https://tools.ietf.org/html/rfc793)定义。在简化的计算机网络[OSI模型](https://zh.wikipedia.org/wiki/OSI模型)中，它完成第四层传输层所指定的功能。[用户数据报协议](https://zh.wikipedia.org/wiki/用户数据报协议)（UDP）是同一层内另一个重要的传输协议。

## 概念

数据在TCP层称为流（Stream），数据分组称为分段（Segment）。作为比较，数据在IP层称为Datagram，数据分组称为分片（Fragment）。 UDP 中分组称为Message。

**CWR(Congestion Window Reduce)：拥塞窗口减少标志被发送主机设置，用来表明它接收到了设置ECE标志的TCP包，发送端通过降低发送窗口的大小来降低发送速率

**ECE(ECN Echo)**：ECN响应标志被用来在TCP3次握手时表明一个TCP端是具备ECN功能的，并且表明接收到的TCP包的IP头部的ECN被设置为11。更多信息请参考RFC793。

**URG(Urgent)**：该标志位置位表示紧急(The urgent pointer) 标志有效。该标志位目前已经很少使用参考后面流量控制和窗口管理部分的介绍。

**ACK(Acknowledgment)**：取值1代表Acknowledgment Number字段有效，这是一个确认的TCP包，取值0则不是确认包。后续文章介绍中当ACK标志位有效的时候我们称呼这个包为ACK包，使用大写的ACK称呼。

**PSH(Push)**：该标志置位时，一般是表示发送端缓存中已经没有待发送的数据，接收端不将该数据进行队列处理，而是尽可能快将数据转由应用处理。在处理 telnet 或 rlogin 等交互模式的连接时，该标志总是置位的。

**RST(Reset)**：用于复位相应的TCP连接。通常在发生异常或者错误的时候会触发复位TCP连接。

**SYN(Synchronize**)：同步序列编号(Synchronize Sequence Numbers)有效。该标志仅在三次握手建立TCP连接时有效。它提示TCP连接的服务端检查序列编号，该序列编号为TCP连接初始端(一般是客户端)的初始序列编号。在这里，可以把TCP序列编号看作是一个范围从0到4，294，967，295的32位计数器。通过TCP连接交换的数据中每一个字节都经过序列编号。在TCP报头中的序列编号栏包括了TCP分段中第一个字节的序列编号。类似的后续文章介绍中当这个SYN标志位有效的时候我们称呼这个包为SYN包。

**FIN(Finish)**：带有该标志置位的数据包用来结束一个TCP会话，但对应端口仍处于开放状态，准备接收后续数据。当FIN标志有效的时候我们称呼这个包为FIN包。

## 运行

TCP协议的运行可划分为三个阶段：连接创建(*connection establishment*)、数据传送（*data transfer*）和连接终止（*connection termination*）。操作系统将TCP连接抽象为[套接字](https://zh.wikipedia.org/wiki/Berkeley套接字)表示的本地端点（local end-point），作为编程接口给程序使用。在TCP连接的生命期内，本地端点要经历一系列的[状态](https://zh.wikipedia.org/wiki/传输控制协议#状态编码)改变。

### 连接创建（3次握手）

#### 第一次握手(client -> server)

```sh
IP 127.0.0.1.63515 > 127.0.0.1.8888: Flags [S], seq 332753718, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 1572533652 ecr 0,sackOK,eol], length 0
	0x0000:  4500 0040 0000 4000 4006 0000 7f00 0001
	0x0010:  7f00 0001 f81b 22b8 13d5 6b36 0000 0000
	0x0020:  b002 ffff fe34 0000 0204 3fd8 0103 0306
	0x0030:  0101 080a 5dba f594 0000 0000 0402 0000
```

过程：客户端（通过执行connect函数）向服务器端发送一个SYN包，请求一个主动打开。该包携带客户端为这个连接请求而设定的随机数（seq=332753718,hex(seq)=13d56b36）作为消息序列号。暂且称此次数据包的seq为s1

意义：确认客户端发送能力正常

#### 第二次握手(server -> client)

```sh
IP 127.0.0.1.8888 > 127.0.0.1.63515: Flags [S.], seq 1886023911, ack 332753719, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 1572533652 ecr 1572533652,sackOK,eol], length 0
	0x0000:  4500 0040 0000 4000 4006 0000 7f00 0001
	0x0010:  7f00 0001 22b8 f81b 706a 70e7 13d5 6b37
	0x0020:  b012 ffff fe34 0000 0204 3fd8 0103 0306
	0x0030:  0101 080a 5dba f594 5dba f594 0402 0000
```

过程：服务器端收到一个合法的SYN包后，把该包放入SYN队列中；回送一个SYN/ACK。ACK的确认码应为第一次握手的s1+1，SYN/ACK包本身携带一个随机产生的seq。暂且称此次数据包的ACK为a1，暂且称此次数据包的seq为s2

意义：服务端的接收、发送能力正常

第三次 握手(client -> server,  server -> client)

```sh
IP 127.0.0.1.63515 > 127.0.0.1.8888: Flags [.], ack 1, win 6379, options [nop,nop,TS val 1572533652 ecr 1572533652], length 0
	0x0000:  4500 0034 0000 4000 4006 0000 7f00 0001
	0x0010:  7f00 0001 f81b 22b8 13d5 6b37 706a 70e8
	0x0020:  8010 18eb fe28 0000 0101 080a 5dba f594
	0x0030:  5dba f594
IP 127.0.0.1.8888 > 127.0.0.1.63515: Flags [.], ack 1, win 6379, options [nop,nop,TS val 1572533652 ecr 1572533652], length 0
	0x0000:  4500 0034 0000 4000 4006 0000 7f00 0001
	0x0010:  7f00 0001 22b8 f81b 706a 70e8 13d5 6b37
	0x0020:  8010 18eb fe28 0000 0101 080a 5dba f594
	0x0030:  5dba f594
```

过程：客户端收到SYN/ACK包后，发送一个ACK包，该包的seq被设定为a1+1，而ACK的确认码则为s2+1。然后客户端的connect函数成功返回。当服务器端收到这个ACK包的时候，把请求帧从SYN队列中移出，放至ACCEPT队列中；这时accept函数如果处于阻塞状态，可以被唤醒，从ACCEPT队列中取出ACK包，重新创建一个新的用于双向通信的sockfd，并返回。

意义： 客户端接收能力正常

我自己感觉虽然是3次握手，但是由于tcp协议的对称性，即有发送就会有响应，所以真实是4次？

### 数据传输

```
IP 127.0.0.1.63515 > 127.0.0.1.8888: Flags [P.], seq 1:1060, ack 1, win 6379, options [nop,nop,TS val 1572533652 ecr 1572533652], length 1059
	0x0000:  4500 0457 0000 4000 4006 0000 7f00 0001  E..W..@.@.......
	0x0010:  7f00 0001 f81b 22b8 13d5 6b37 706a 70e8  ......"...k7pjp.
	0x0020:  8018 18eb 024c 0000 0101 080a 5dba f594  .....L......]...
	0x0030:  5dba f594 504f 5354 202f 6261 7365 2f6c  ]...POST./base/l
	0x0040:  6f67 696e 2048 5454 502f 312e 310d 0a48  ogin.HTTP/1.1..H
	0x0050:  6f73 743a 2031 3237 2e30 2e30 2e31 3a38  ost:.127.0.0.1:8
	0x0060:  3838 380d 0a43 6f6e 6e65 6374 696f 6e3a  888..Connection:
	0x0070:  206b 6565 702d 616c 6976 650d 0a43 6f6e  .keep-alive..Con
	0x0080:  7465 6e74 2d4c 656e 6774 683a 2039 380d  tent-Length:.98.
	0x0090:  0a61 6363 6570 743a 2061 7070 6c69 6361  .accept:.applica
	0x00a0:  7469 6f6e 2f6a 736f 6e0d 0a55 7365 722d  tion/json..User-
	0x00b0:  4167 656e 743a 204d 6f7a 696c 6c61 2f35  Agent:.Mozilla/5
	0x00c0:  2e30 2028 4d61 6369 6e74 6f73 683b 2049  .0.(Macintosh;.I
	0x00d0:  6e74 656c 204d 6163 204f 5320 5820 3131  ntel.Mac.OS.X.11
	0x00e0:  5f31 5f30 2920 4170 706c 6557 6562 4b69  _1_0).AppleWebKi
	0x00f0:  742f 3533 372e 3336 2028 4b48 544d 4c2c  t/537.36.(KHTML,
	0x0100:  206c 696b 6520 4765 636b 6f29 2043 6872  .like.Gecko).Chr
	0x0110:  6f6d 652f 3837 2e30 2e34 3238 302e 3134  ome/87.0.4280.14
	0x0120:  3120 5361 6661 7269 2f35 3337 2e33 360d  1.Safari/537.36.
	0x0130:  0a43 6f6e 7465 6e74 2d54 7970 653a 2061  .Content-Type:.a
	0x0140:  7070 6c69 6361 7469 6f6e 2f6a 736f 6e0d  pplication/json.
	0x0150:  0a4f 7269 6769 6e3a 2068 7474 703a 2f2f  .Origin:.http://
	0x0160:  3132 372e 302e 302e 313a 3838 3838 0d0a  127.0.0.1:8888..
	0x0170:  5365 632d 4665 7463 682d 5369 7465 3a20  Sec-Fetch-Site:.
	0x0180:  7361 6d65 2d6f 7269 6769 6e0d 0a53 6563  same-origin..Sec
	0x0190:  2d46 6574 6368 2d4d 6f64 653a 2063 6f72  -Fetch-Mode:.cor
	0x01a0:  730d 0a53 6563 2d46 6574 6368 2d44 6573  s..Sec-Fetch-Des
	0x01b0:  743a 2065 6d70 7479 0d0a 5265 6665 7265  t:.empty..Refere
	0x01c0:  723a 2068 7474 703a 2f2f 3132 372e 302e  r:.http://127.0.
	0x01d0:  302e 313a 3838 3838 2f73 7761 6767 6572  0.1:8888/swagger
	0x01e0:  2f69 6e64 6578 2e68 746d 6c0d 0a41 6363  /index.html..Acc
	0x01f0:  6570 742d 456e 636f 6469 6e67 3a20 677a  ept-Encoding:.gz
	0x0200:  6970 2c20 6465 666c 6174 652c 2062 720d  ip,.deflate,.br.
	0x0210:  0a41 6363 6570 742d 4c61 6e67 7561 6765  .Accept-Language
	0x0220:  3a20 7a68 2d43 4e2c 7a68 3b71 3d30 2e39  :.zh-CN,zh;q=0.9
	0x0230:  0d0a 436f 6f6b 6965 3a20 5f67 613d 4741  ..Cookie:._ga=GA
	0x0240:  312e 312e 3131 3434 3731 3235 3531 2e31  1.1.1144712551.1
	0x0250:  3630 3939 3430 3830 323b 205f 686f 6d65  609940802;._home
	0x0260:  6c61 6e64 5f73 6573 7369 6f6e 3d73 4962  land_session=sIb
	0x0270:  7a4b 2532 4243 776b 4525 3242 5954 3978  zK%2BCwkE%2BYT9x
	0x0280:  425a 3675 5041 7877 4641 4b46 4869 7732  BZ6uPAxwFAKFHiw2
	0x0290:  6861 7150 3951 796b 5968 7371 7647 4d70  haqP9QykYhsqvGMp
	0x02a0:  3033 5872 4470 5330 6a38 4b4f 6535 6c36  03XrDpS0j8KOe5l6
	0x02b0:  6e48 2532 4654 4770 5151 324e 6157 7154  nH%2FTGpQQ2NaWqT
	0x02c0:  654a 4f71 4944 6a6b 3272 537a 347a 4e54  eJOqIDjk2rSz4zNT
	0x02d0:  6c63 586a 5843 4a64 3537 4b57 6642 4142  lcXjXCJd57KWfBAB
	0x02e0:  7861 6463 5657 3070 586f 4177 3933 7444  xadcVW0pXoAw93tD
	0x02f0:  5331 7a51 6663 6458 6d4d 3869 7263 3337  S1zQfcdXmM8irc37
	0x0300:  7834 6c77 3179 514d 635a 6c33 4771 616e  x4lw1yQMcZl3Gqan
	0x0310:  4462 5639 5a58 6464 7375 6725 3246 7452  DbV9ZXddsug%2FtR
	0x0320:  5655 4766 5953 3550 3874 6d44 5161 3362  VUGfYS5P8tmDQa3b
	0x0330:  4968 4e7a 6a25 3242 6555 506a 6866 3725  IhNzj%2BeUPjhf7%
	0x0340:  3246 7671 5446 4353 4c31 6a72 3268 4146  2FvqTFCSL1jr2hAF
	0x0350:  2532 4249 4225 3242 5255 4834 5551 6453  %2BIB%2BRUH4UQdS
	0x0360:  6b76 6750 6f7a 6d74 6e48 6172 6241 4f6b  kvgPozmtnHarbAOk
	0x0370:  4e6f 5250 2532 4676 596a 5376 3261 4f37  NoRP%2FvYjSv2aO7
	0x0380:  506a 7358 7178 6248 5932 4a44 3866 5a51  PjsXqxbHY2JD8fZQ
	0x0390:  6453 6c45 6d70 577a 4c41 5844 6745 7665  dSlEmpWzLAXDgEve
	0x03a0:  3536 5755 4f69 6869 3559 6876 3564 4b76  56WUOihi5Yhv5dKv
	0x03b0:  6171 7047 6d37 5736 534f 716d 436f 2533  aqpGm7W6SOqmCo%3
	0x03c0:  442d 2d4d 5271 6745 4f6b 3343 4e45 4543  D--MRqgEOk3CNEEC
	0x03d0:  5a6e 462d 2d72 3166 7154 3148 7830 5764  ZnF--r1fqT1Hx0Wd
	0x03e0:  796e 7444 3357 3744 5951 6725 3344 2533  yntD3W7DYQg%3D%3
	0x03f0:  440d 0a0d 0a7b 0a20 2022 6361 7074 6368  D....{..."captch
	0x0400:  6122 3a20 2273 7472 696e 6722 2c0a 2020  a":."string",...
	0x0410:  2263 6170 7463 6861 4964 223a 2022 7374  "captchaId":."st
	0x0420:  7269 6e67 222c 0a20 2022 7061 7373 776f  ring",..."passwo
	0x0430:  7264 223a 2022 7374 7269 6e67 222c 0a20  rd":."string",..
	0x0440:  2022 7573 6572 6e61 6d65 223a 2022 7374  ."username":."st
	0x0450:  7269 6e67 220a 7d                        ring".}
IP 127.0.0.1.8888 > 127.0.0.1.63515: Flags [.], ack 1060, win 6363, options [nop,nop,TS val 1572533652 ecr 1572533652], length 0
	0x0000:  4500 0034 0000 4000 4006 0000 7f00 0001  E..4..@.@.......
	0x0010:  7f00 0001 22b8 f81b 706a 70e8 13d5 6f5a  ...."...pjp...oZ
	0x0020:  8010 18db fe28 0000 0101 080a 5dba f594  .....(......]...
	0x0030:  5dba f594
IP 127.0.0.1.8888 > 127.0.0.1.63515: Flags [P.], seq 1:550, ack 1060, win 6363, options [nop,nop,TS val 1572533671 ecr 1572533652], length 549
	0x0000:  4500 0259 0000 4000 4006 0000 7f00 0001  E..Y..@.@.......
	0x0010:  7f00 0001 22b8 f81b 706a 70e8 13d5 6f5a  ...."...pjp...oZ
	0x0020:  8018 18db 004e 0000 0101 080a 5dba f5a7  .....N......]...
	0x0030:  5dba f594 4854 5450 2f31 2e31 2032 3030  ]...HTTP/1.1.200
	0x0040:  204f 4b0d 0a41 6363 6573 732d 436f 6e74  .OK..Access-Cont
	0x0050:  726f 6c2d 416c 6c6f 772d 4372 6564 656e  rol-Allow-Creden
	0x0060:  7469 616c 733a 2074 7275 650d 0a41 6363  tials:.true..Acc
	0x0070:  6573 732d 436f 6e74 726f 6c2d 416c 6c6f  ess-Control-Allo
	0x0080:  772d 4865 6164 6572 733a 2043 6f6e 7465  w-Headers:.Conte
	0x0090:  6e74 2d54 7970 652c 4163 6365 7373 546f  nt-Type,AccessTo
	0x00a0:  6b65 6e2c 582d 4353 5246 2d54 6f6b 656e  ken,X-CSRF-Token
	0x00b0:  2c20 4175 7468 6f72 697a 6174 696f 6e2c  ,.Authorization,
	0x00c0:  2054 6f6b 656e 2c58 2d54 6f6b 656e 2c58  .Token,X-Token,X
	0x00d0:  2d55 7365 722d 4964 0d0a 4163 6365 7373  -User-Id..Access
	0x00e0:  2d43 6f6e 7472 6f6c 2d41 6c6c 6f77 2d4d  -Control-Allow-M
	0x00f0:  6574 686f 6473 3a20 504f 5354 2c20 4745  ethods:.POST,.GE
	0x0100:  542c 204f 5054 494f 4e53 2c44 454c 4554  T,.OPTIONS,DELET
	0x0110:  452c 5055 540d 0a41 6363 6573 732d 436f  E,PUT..Access-Co
	0x0120:  6e74 726f 6c2d 416c 6c6f 772d 4f72 6967  ntrol-Allow-Orig
	0x0130:  696e 3a20 6874 7470 3a2f 2f31 3237 2e30  in:.http://127.0
	0x0140:  2e30 2e31 3a38 3838 380d 0a41 6363 6573  .0.1:8888..Acces
	0x0150:  732d 436f 6e74 726f 6c2d 4578 706f 7365  s-Control-Expose
	0x0160:  2d48 6561 6465 7273 3a20 436f 6e74 656e  -Headers:.Conten
	0x0170:  742d 4c65 6e67 7468 2c20 4163 6365 7373  t-Length,.Access
	0x0180:  2d43 6f6e 7472 6f6c 2d41 6c6c 6f77 2d4f  -Control-Allow-O
	0x0190:  7269 6769 6e2c 2041 6363 6573 732d 436f  rigin,.Access-Co
	0x01a0:  6e74 726f 6c2d 416c 6c6f 772d 4865 6164  ntrol-Allow-Head
	0x01b0:  6572 732c 2043 6f6e 7465 6e74 2d54 7970  ers,.Content-Typ
	0x01c0:  650d 0a43 6f6e 7465 6e74 2d54 7970 653a  e..Content-Type:
	0x01d0:  2061 7070 6c69 6361 7469 6f6e 2f6a 736f  .application/jso
	0x01e0:  6e3b 2063 6861 7273 6574 3d75 7466 2d38  n;.charset=utf-8
	0x01f0:  0d0a 4461 7465 3a20 5361 742c 2031 3620  ..Date:.Sat,.16.
	0x0200:  4a61 6e20 3230 3231 2030 393a 3336 3a34  Jan.2021.09:36:4
	0x0210:  3420 474d 540d 0a43 6f6e 7465 6e74 2d4c  4.GMT..Content-L
	0x0220:  656e 6774 683a 2034 340d 0a0d 0a7b 2263  ength:.44....{"c
	0x0230:  6f64 6522 3a37 2c22 6461 7461 223a 7b7d  ode":7,"data":{}
	0x0240:  2c22 6d73 6722 3a22 e9aa 8ce8 af81 e7a0  ,"msg":"........
	0x0250:  81e9 9499 e8af af22 7d                   ......."}
IP 127.0.0.1.63515 > 127.0.0.1.8888: Flags [.], ack 550, win 6371, options [nop,nop,TS val 1572533671 ecr 1572533671], length 0
	0x0000:  4500 0034 0000 4000 4006 0000 7f00 0001  E..4..@.@.......
	0x0010:  7f00 0001 f81b 22b8 13d5 6f5a 706a 730d  ......"...oZpjs.
	0x0020:  8010 18e3 fe28 0000 0101 080a 5dba f5a7  .....(......]...
	0x0030:  5dba f5a7
```

可以看出http协议中的get,post...本质上是没有区别的，请求数据放在query，header，body本质是没有区别的，只是要用合适方式去做合适的事

### 关闭连接

```
IP 127.0.0.1.8888 > 127.0.0.1.63515: Flags [F.], seq 550, ack 1060, win 6363, options [nop,nop,TS val 1572533681 ecr 1572533671], length 0
	0x0000:  4500 0034 0000 4000 4006 0000 7f00 0001  E..4..@.@.......
	0x0010:  7f00 0001 22b8 f81b 706a 730d 13d5 6f5a  ...."...pjs...oZ
	0x0020:  8011 18db fe28 0000 0101 080a 5dba f5b1  .....(......]...
	0x0030:  5dba f5a7                                ]...
IP 127.0.0.1.63515 > 127.0.0.1.8888: Flags [.], ack 551, win 6371, options [nop,nop,TS val 1572533681 ecr 1572533681], length 0
	0x0000:  4500 0034 0000 4000 4006 0000 7f00 0001  E..4..@.@.......
	0x0010:  7f00 0001 f81b 22b8 13d5 6f5a 706a 730e  ......"...oZpjs.
	0x0020:  8010 18e3 fe28 0000 0101 080a 5dba f5b1  .....(......]...
	0x0030:  5dba f5b1                                ]...
IP 127.0.0.1.63515 > 127.0.0.1.8888: Flags [.], ack 551, win 6371, length 0
	0x0000:  4500 0028 6b78 0000 4006 0000 7f00 0001  E..(kx..@.......
	0x0010:  7f00 0001 f81b 22b8 13d5 6f59 706a 730e  ......"...oYpjs.
	0x0020:  5010 18e3 fe1c 0000                      P.......
IP 127.0.0.1.8888 > 127.0.0.1.63515: Flags [.], ack 1060, win 6363, options [nop,nop,TS val 1572578966 ecr 1572533681], length 0
	0x0000:  4500 0034 0000 4000 4006 0000 7f00 0001  E..4..@.@.......
	0x0010:  7f00 0001 22b8 f81b 706a 730e 13d5 6f5a  ...."...pjs...oZ
	0x0020:  8010 18db fe28 0000 0101 080a 5dbb a696  .....(......]...
	0x0030:  5dba f5b1                                ]...
IP 127.0.0.1.63515 > 127.0.0.1.8888: Flags [F.], seq 1060, ack 551, win 6371, options [nop,nop,TS val 1572607233 ecr 1572578966], length 0
	0x0000:  4500 0034 0000 4000 4006 0000 7f00 0001  E..4..@.@.......
	0x0010:  7f00 0001 f81b 22b8 13d5 6f5a 706a 730e  ......"...oZpjs.
	0x0020:  8011 18e3 fe28 0000 0101 080a 5dbc 1501  .....(......]...
	0x0030:  5dbb a696                                ]...
IP 127.0.0.1.8888 > 127.0.0.1.63515: Flags [.], ack 1061, win 6363, options [nop,nop,TS val 1572607233 ecr 1572607233], length 0
	0x0000:  4500 0034 0000 4000 4006 0000 7f00 0001  E..4..@.@.......
	0x0010:  7f00 0001 22b8 f81b 706a 730e 13d5 6f5b  ...."...pjs...o[
	0x0020:  8010 18db fe28 0000 0101 080a 5dbc 1501  .....(......]...
	0x0030:  5dbc 1501                                ]...
```

连接终止使用了四路握手过程（或称四次握手，four-way handshake），在这个过程中连接的每一侧都独立地被终止。当一个端点要停止它这一侧的连接，就向对侧发送FIN，对侧回复ACK表示确认。因此，拆掉一侧的连接过程需要一对FIN和ACK，分别由两侧端点发出。

首先发出FIN的一侧，如果给对侧的FIN响应了ACK，那么就会超时等待2*MSL时间，然后关闭连接。在这段超时等待时间内，本地的端口不能被新连接使用；避免延时的包的到达与随后的新连接相混淆。RFC793定义了MSL为2分钟，Linux设置成了30s。参数tcp_max_tw_buckets控制并发的TIME_WAIT的数量，默认值是180000，如果超限，那么，系统会把多的TIME_WAIT状态的连接给destory掉，然后在日志里打一个警告（如：time wait bucket table overflow）

连接可以工作在[TCP半开](https://zh.wikipedia.org/w/index.php?title=TCP半开&action=edit&redlink=1)状态。即一侧关闭了连接，不再发送数据；但另一侧没有关闭连接，仍可以发送数据。已关闭的一侧仍然应接收数据，直至对侧也关闭了连接。

也可以通过测三次握手关闭连接。主机A发出FIN，主机B回复FIN & ACK，然后主机A回复ACK.

> https://zh.wikipedia.org/wiki/%E4%BC%A0%E8%BE%93%E6%8E%A7%E5%88%B6%E5%8D%8F%E8%AE%AE

> tcp抓包工具：tcpdump

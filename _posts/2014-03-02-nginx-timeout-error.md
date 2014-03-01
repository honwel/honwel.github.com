---
layout: post
title: Nginx 出现 timeout 错误
description: "Nginx 出现 timeout 错误"
category: 技术
tags: [Nginx, timeout, conntrack_max, 内核参数]
---
{% include JB/setup %}

前一阵子在测试环境中发现Nginx的日志中出现了大量的timeout错误，通过多方排除都没有发现具体的原因，我查看了系统内核参数，如tcp.tw.reuse，tcp.tw.recycle等，也查看了后端服务器的日志，其并发量也在接受的正常范围，Nginx本身也没有问题，虽然添加了自己开发的模块，但不会照成该错误，这种问题有时候排查起来还是要费些功夫的。从日志上看，问题很明显，就是网络超时了，客户端请求经过Nginx转发后没有收到正确的结果。

我尝试从Nginx所在的前端机器向后端服务器发起ping命令，果然有些异常，有丢包：

![图1 ping]({{ site.img_url }}/ping.png)

然后查看了系统日志，发现有很多nf_conntrack: table full, dropping packet的错误，看来应该就是这个问题导致的，我将conntrack_max的参数按照系统内存的调整到合适的大小（sysctl -w），然后重新进行几个小时的测试，这次没有再出现timeout错误，而且ping后端服务器时也没有出现上面的丢包现象。

![图1 ping]({{ site.img_url }}/nf_conntrack.png)

其实conntrack_max参数与开启的iptable有关，默认值是65536，表示[连接跟踪表](http://baike.baidu.com)的最大数，具体含义是：“Linux为每一个经过网络堆栈的数据包，生成一个新的连接记录项 （Connection entry）。此后，所有属于此连接的数据包都被唯一地分配给这个连接，并标识连接的状态。连接跟踪是防火墙模块（iptable）的状态检测的基础，同时也是地址转换中实现SNAT和DNAT的前提。在Linux的iptable中的每一个数据，都有“来源”与“目的”主机，发起连接的主机称为“来源”，响应“来源”的请求的主机即为目的，所谓生成记录项，就是对每一个这样的连接的产生、传输及终止进行跟踪记录。由所有记录项产生的表，即称为连接跟踪表”。

至此，问题解决。


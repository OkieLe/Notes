# sed命令进阶

> 版权声明： 本文章内容在非商业使用前提下可无需授权任意转载、发布。
> 转载、发布请务必注明作者和其微博、微信公众号地址，以便读者询问问题和甄误反馈，共同进步。
> 微博ID：**orroz**
> 微信公众号：**Linux系统技术**

#### 前言

本文主要介绍sed的高级用法，在阅读本文之前希望读者已经掌握sed的基本使用和正则表达式的相关知识。本文主要可以让读者学会**如何使用sed处理段落内容**。

#### 问题举例

日常工作中我们都经常会使用sed命令对文件进行处理。最普遍的是以行为单位处理，比如做替换操作，如：
```
[root@TENCENT64 ~]# head -3 /etc/passwd
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
[root@TENCENT64 ~]# head -3 /etc/passwd |sed 's/bin/xxx/g'
root:x:0:0:root:/root:/xxx/bash
xxx:x:1:1:xxx:/xxx:/sxxx/nologin
daemon:x:2:2:daemon:/sxxx:/sxxx/nologin
```

于是每一行中只要有”bin”这个关键字的就都被替换成了”xxx”。以”行”为单位的操作，了解sed基本知识之后应该都能处理。我们下面通过一个对段落的操作，来进一步学习一下sed命令的高级用法。

工作场景下有时候遇到的文本内容并不仅仅是以行为单位的输出。举个例子来说，比如ifconfig命令的输出：
```
[root@TENCENT64 ~]# ifconfig
eth1 Link encap:Ethernet HWaddr 40:F2:E9:09:FC:45
inet addr:10.0.0.1 Bcast:10.213.123.255 Mask:255.255.252.0
UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1
RX packets:4699092093 errors:0 dropped:0 overruns:0 frame:0
TX packets:4429422167 errors:0 dropped:0 overruns:0 carrier:0 collisions:0 txqueuelen:0
RX bytes:621383727756 (578.7 GiB) TX bytes:987104190070 (919.3 GiB)

eth2 Link encap:Ethernet HWaddr 40:F2:E9:09:FC:45
UP BROADCAST RUNNING MULTICAST MTU:1500 Metric:1
RX packets:421719237 errors:0 dropped:0 overruns:0 frame:0
TX packets:0 errors:0 dropped:0 overruns:0 carrier:0 collisions:0 txqueuelen:0
RX bytes:17982386416 (16.7 GiB) TX bytes:0 (0.0 b)

lo Link encap:Local Loopback
inet addr:127.0.0.1 Mask:255.0.0.0
UP LOOPBACK RUNNING MTU:16436 Metric:1
RX packets:3094508 errors:0 dropped:0 overruns:0 frame:0
TX packets:3094508 errors:0 dropped:0 overruns:0 carrier:0 collisions:0 txqueuelen:0
RX bytes:1579253954 (1.4 GiB) TX bytes:1579253954 (1.4 GiB)

br0 Link encap:Ethernet HWaddr FE:BD:B8:D5:79:46
UP BROADCAST RUNNING PROMISC MULTICAST MTU:1500 Metric:1
RX packets:118073976200 errors:0 dropped:0 overruns:0 frame:0
TX packets:129141892891 errors:0 dropped:0 overruns:0 carrier:0 collisions:0 txqueuelen:1000
RX bytes:13271406153198 (12.0 TiB) TX bytes:21428348510630 (19.4 TiB)
```
这是一个有很多网卡的服务器。观察以上输出我们会发现，每个网卡的信息都有多行内容，组成一段。网卡名称和MAC地址在一行，网卡名称IP地址不在一行。类似的还有网卡的收发包数量的信息，以及收发包的字节数。那么对于类似这样的文本内容，当我们想要使用sed将输出处理成：网卡名对应IP地址或者 网卡名对应收包字节数，将不在同一行的信息放到同一行再输出该怎么处理呢？类似这样：
```
网卡名:收包字节数
eth1:621383727756
eth2:17982386416
...
```
这样的需求对于一般的sed命令来说显然做不到了。我们需要引入更高级的sed处理功能来处理类似问题。大家可以先将上述代码保存到一个文本文件里以备后续实验，我们先给出答案：

网卡名对应RX字节数：
```
[root@zorrozou-pc0 zorro]# sed -n '/^[^ ]/{s/^\([^ ]*\) .*/\1/g;h;: top;n;/^$/b;s/^.*RX bytes:\([0-9]\{1,\}\).*/\1/g;T top;H;x;s/\n/:/g;p}' ifconfig.out
eth1:621383727756
eth2:17982386416
eth3:16934321897
eth4:5929444543
eth5:743435149333809
eth6:0
eth7:0
eth8:0
eth9:0
lo:1579253954
br0:13271406153198
br1:216934635012821
br2:146402818737176
br3:216934635012821
br4:216934635013645
br5:1355936046078
```
网卡名对应ip地址：
```
[root@zorrozou-pc0 zorro]# sed -n '/^[^ ]/{s/^\([^ ]*\) .*/\1/g;h;: top;n;/^$/b;s/^.*inet addr:\([0-9]\{1,\}\.[0-9]\{1,\}\.[0-9]\{1,\}\.[0-9]\{1,\}\).*/\1/g;T top;H;x;s/\n/:/g;p}' ifconfig.out
eth1:10.0.0.1
lo:127.0.0.1
```
我们还会发现显示IP的sed命令很智能的过滤掉了没有IP地址的网卡，只把有IP的网卡显示出来。相信一部分人看到这个命令已经晕了。先不要着急，讲解这个命令的含义之前，我们需要先来了解一下sed的工作原理。


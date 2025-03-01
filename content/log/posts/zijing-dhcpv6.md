+++
title = '紫荆宿舍区 DHCPv6 有线网指北'
date = 2021-10-08
draft = false
+++


## TLDR

0. 优先使用 Tsinghua-Secure 无线网
1. 若需使用有线网络
   0. 更改 DUID Type 为 LLT
   1. 更改 DUID Data 为网卡 MAC 地址
   2. 开启 Anonymize
2. 无法获取地址时可以先睡觉

___

上周 [Tunight](https://tuna.moe/event/2021/welcome-and-debian/) 现场本应有 Installation Party，没有合适设备只好播放续老师录像。回宿舍后没忍住还是滚了个 [bullseye](https://www.debian.org/releases/bullseye/)，先后遭遇各种问题，包括但不限于 DHCPv4 拿不到地址，无线网络也无法连接等。

后来发现下雨天人们都宅在宿舍，确实没有 IPv4 地址可以分配~~抢了一个就好了~~，无线是因为把 Peking_Secure 打成 Peking-Secure 搜索不到。安装常用软件包后发现 DHCPv6 一直拿不到地址，和群友一起进行了复杂的分析，总结成这篇紫荆区 DHCPv6 指北。

## 问题排查

贴一下非常常见的上网方式：本人使用 networkd 配置网络，根据 [信息化技术中心说明](https://thu.services/file/CampusNetworkLectureNotes201909.pdf) 紫荆宿舍区 IPv6 为有状态网，因此均使用 DHCP 服务器获取地址，配置文件 `/etc/network/interfaces` 如下

```bash
# enp89s0
auto enp89s0
allow-hotplug enp89s0
iface enp89s0 inet dhcp
iface enp89s0 inet6 dhcp
```

这样可以正常通过 DHCP 拿到 IPv4 地址，但却无法获取 IPv6 地址，手动复查

```bash
sudo dhclient -d enp89s0
sudo dhclient -6 -d enp89s0
```

发现 DHCPv4 正常，DHCPv6 无法得到地址。

## DHCPv6

无状态 IPv6 地址分配（SLAAC）非常简单：由于 IPv6 地址资源过于充沛，可以通过监听路由广播获取全局 /64 前缀，通过物理地址通过一定规则映射拼接成完整地址，若发送请求无响应则说明可用。

有状态 IPv6 地址分配继承了 DHCPv4 协议。为了更好定位问题，首先学习 DHCPv6 协议：[RFC3315](https://tools.ietf.org/html/rfc3315) 定义了 DHCPv6 协议，并在 [RFC8415](https://tools.ietf.org/html/rfc8415) 改进，报文格式如下

```
(1) SOLICIT      DHCPDISCOVER
(2) ADVERTISE    DHCPOFFER
(3) REQUEST      DHCPREQUEST
(4) REPLY        DHCPACK
```

左侧 DHCPv6 报文分别与右边的 DHCPv4 报文对应，因此并不难理解。

```bash
sudo dhclient -6 -d enp89s0
sudo tcpdump -i enp89s0 -n -vv '(udp port 546 or 547) or icmp6'
```

通过抓包发现确有发送 SOLICIT 包，但一直无 ADVERTISE 包回应。

## 对比验证

第一想法是和 IPv4 一样，雨天宅宿舍的人太多，导致没有足够地址分配；再一想这是 IPv6，资源枯竭的时候人类早就实现共产主义了，这不可能。查阅了相关 bug tracker, networkd 在 DHCPv6 上并没有值得关注的问题，于是陷入困境。

突然想到 Windows 笔记本一直能获取 IPv6，于是首先尝试接入 Tsinghua-Secure 无线网，这里使用续老师提供的 `wpa_supplicant.conf` 配置片段，networkd 配置与有线网相同

```shell
network={
    ssid="Tsinghua-Secure"
    proto=RSN
    key_mgmt=WPA-EAP
    pairwise=CCMP
    eap=PEAP
    identity="redacted"
    password="redacted"
    phase2="auth=MSCHAPV2"
}     
```

成功接入，发现能正常获取 IPv4 和 IPv6 地址，于是确认本机 DHCPv6 工作正常。找出笔记本附赠的远古转接线，在 Windows 下尝试有线网连接，发现也能获得 IPv6 地址。

怎会如此？

在谭院士建议下抓包复查。Windows 下已经协商得到地址，热插拔时只能捕捉到 RENEW 和 REPLY 包，因此学习 ipconfig 的使用方式

```bash
ipconfig /release6
ipconfig /renew6
```

成功捕捉 SOLICIT 与 ADVERTISE 报文，SOLICIT 如下

```
0000   33 33 00 01 00 02 00 e0 4c 60 42 e4 86 dd 60 0f   33......L`B...`.
0010   1c 8d 00 5e 11 01 fe 80 00 00 00 00 00 00 b4 32   ...^...........2
0020   05 8f 9c b7 ed 6f ff 02 00 00 00 00 00 00 00 00   .....o..........
0030   00 00 00 01 00 02 02 22 02 23 00 5e 41 df 01 de   .......".#.^A...
0040   28 cf 00 08 00 02 00 00 00 01 00 0e 00 01 00 01   (...............
0050   28 6b 02 a8 50 76 af 6a 7d ae 00 03 00 0c 38 00   (k..Pv.j}.....8.
0060   e0 4c 00 00 00 00 00 00 00 00 00 27 00 08 00 06   .L.........'....
0070   6c 79 70 2d 70 63 00 10 00 0e 00 00 01 37 00 08   lyp-pc.......7..
0080   4d 53 46 54 20 35 2e 30 00 06 00 08 00 11 00 17   MSFT 5.0........
0090   00 18 00 27                                       ...'
```

发现 Windows 使用 [LLT](https://tools.ietf.org/html/rfc8415#section-11.2) (Link-layer Address Plus Time) 作为 DUID，而 networkd 默认使用 vendor 作为 DUID

## DUID

查阅 [RFC6355](https://tools.ietf.org/html/rfc6355) 学习 DUID，和 DHCPv4 一致为设备唯一标识符，最小长度为 12 字节，最长 20 字节，格式如下

```
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          DUID-Type (4)        |    UUID (128 bits)            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               |
|                                                               |
|                                                               |
|                                -+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-
```

其中前 16 位表示 DUID Type，主要分为 LLT, LL 与 VENDOR 三种，之后为 DUID Raw Data，搜索记忆似乎确有[类似提示](https://thu.services/services/#rfc-dhcpv6)

> 你校的 DHCPv6 server 会不承认某些 DUID，对于这样的 DHCP 请求会不予回应。即使向学校反映该问题，学校尝试让厂商修复后，该问题仍然存在。根据相关人士消息，只有 DUID Type 1，也就是 DUID-LLT 被承认。

于是修改 networkd 配置文件 `/etc/systemd/networkd.conf` 尝试解决

```systemd
[DHCP]
DUIDType=link-layer-time
DUIDRawData=<MAC>
```

抓包如下

``` 
0000   33 33 00 01 00 02 1c 69 7a a1 d1 25 86 dd 60 0b   33.....iz..%..`.
0010   d8 8f 00 40 11 01 fe 80 00 00 00 00 00 00 1e 69   ...@...........i
0020   7a ff fe a1 d1 25 ff 02 00 00 00 00 00 00 00 00   z....%..........
0030   00 00 00 01 00 02 02 22 02 23 00 40 67 08 01 4f   .......".#.@g..O
0040   80 91 00 01 00 0e 00 01 00 01 28 f0 6d 39 1c 69   ..........(.m9.i
0050   7a a1 d1 25 00 06 00 08 00 17 00 18 00 27 00 1f   z..%.........'..
0060   00 08 00 02 00 00 00 03 00 0c 7a a1 d1 25 00 00   ..........z..%..
0070   0e 10 00 00 15 18                                 ......
```

此时 DUID 已经更改为 LLT 格式，不过仍然收不到 ADVERTISE 响应。

怎会如此？

## Windows

谭院士提示再看看 Windows 报文，于是仔细研读，发现 Linux 包长度为 118 字节，而 Windows 包长度为 148 字节，包含了一些额外的信息，但这些信息根据 RFC 都是可选的。

考虑到紫荆区 DHCPv6 服务器甚至只响应一种类型 DUID 对应的请求，说不定也有关系。和妮可草一起阅读 networkd 文档，发现奇妙参数 [Anonymize=](https://systemd.networkAnonymize=)

> With this option enabled DHCP requests will <strong>mimic those generated by Microsoft Windows</strong>, in order to reduce the ability to fingerprint and recognize installations.

怎会如此？

不过以上内容已经折腾了足够长时间，熄灯后无设备工作无法尝试，群内讨论猜想如下：

1. 更改 DUID Type 为 LLT
2. 使用正确 MAC 地址填充 DUID Data
3. 使用 Anonymize 模拟 Windows 行为

切换到 systemd-networkd 并更改 `/etc/systemd/network/enp89s0.network` 参数如下

```systemd
[Match]
Name=enp89s0

[Link]
MacAddressPolicy=random

[Network]
DHCP=yes

[DHCPv4]
Anonymize=true

[DHCPv6]
DUIDType=link-layer-time
DUIDRawData=<MAC>
```

妮可草提出猜想，非常认可

<blockquote>
Nick Cao | 想吃狐狐, [Oct 7, 2021 at 12:14 AM]

说明厂商就是对着 Windows 抓包瞎写的实现

<br>

Nick Cao | 想吃狐狐, [Oct 7, 2021 at 12:14 AM]

压根没看 rfc
</blockquote>

随后睡觉。

## 神奇收尾

第二天上午，部分群友发现并没有改配置，但得到了正确的 IPv6 地址。本地在更改 Anonymize 后，发现无论 DUID Type 和是否 Anonymize，均能正常获得地址，且 IPv4 没有冲突，抓包一切正常。

谭院士反复观察后做出指示

> Aha! Well-behaved twd2, [Oct 7, 2021 at 12:47 AM]
> 
> 没人知道学校 DHCPv6 Server 怎么工作的
> <br><br>
> Aha! Well-behaved twd2, [Oct 7, 2021 at 10:16 AM]
> 
> 可能发 v6 是办公室老师亲自审批的
> <br><br>
> Aha! Well-behaved twd2, [Oct 7, 2021 at 10:16 AM]
> 
> 亲自阅请求，亲自签发地址壹个

因此提出在紫荆宿舍区使用 DHCPv6 指北：

~~不要使用 IPv6 / 全面转向 Windows / 去信息化技术中心手动签发~~

<break>

0. 优先使用 Tsinghua-Secure 无线网
1. 若需使用有线网络
   0. 更改 DUID Type 为 LLT
   1. 更改 DUID Data 为网卡 MAC 地址
   2. 开启 Anonymize
2. 无法获取地址时可以先睡觉

## Extras

一些可能有用的链接

> https://tools.ietf.org/html/rfc3315
> 
> https://tools.ietf.org/html/rfc6355
> 
> https://tools.ietf.org/html/rfc7844
> 
> https://tools.ietf.org/html/rfc8415
> 
> https://manpages.debian.org/bullseye/systemd/networkd.conf.5
> 
> https://systemd.network#Anonymize=

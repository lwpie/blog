+++
title = '机房运维备忘录'
date = 2022-11-22
draft = false
toc = true
tocBorder = true
+++

作为专注云计算的实验室，一个月前组里只有一台服务器，直接接入校园网，日常也没什么人用。为了~~高效花完本年度科研经费~~提供更多组内计算资源，我全程负责了实验室基础设施体系的搭建，在蒙民伟科技南楼地下机房度过无数日夜。哈利、坏人和 TUNA 的小伙伴们在此过程提供了许多指导，希望这篇文章不仅为自己备忘，也能为你们提供帮助。

## 网络

### 接口

常用的网络接口包含电口和光口。

![接口](/log/sysop/interfaces.png)

#### 电口

大部分家用设备（个人电脑、路由器等）使用电口，如图中最左侧的 ETH / BOOT，通过网线连接使用。五类网线可以提供千兆速率，万兆一般需要六类屏蔽网线，六类也可以进行尝试。

#### 光口

光口是对各种光纤端口的总称，如图中右侧四个接口。网线有效传输距离一般不到百米，最高提供 10/40 Gbps 速率；光纤距离以公里计算，速率可达 200/400 Gbps, 这使机房和交换机中光口更加普遍。电口可以直接接入网线使用，光口需要插入模块后接入传输介质。

![模块](/log/sysop/modules.jpg)

**电模块**

插入电模块后，光口可以作为电口接入网线使用。以下几点需要注意：

1. 电模块温度很高，需避免相邻使用，并小心烫伤手指；
2. 生产电模块的厂商常见 TP, Marvell, Broadcom 三种，其余均为贴牌产物
    1. 华为、H3C 等交换机限制模块型号，第三方联系店家刷码
    2. TP 专注家用，兼容 10/5/2.5/1 Gbps 速率，不建议在机房使用
    3. Marvell 功率 1.6W, Broadcom 功率 2.0W, 可用距离更长
3. IPMI 通常只能运行在电模块下

**光模块**

光模块常见规格有 SFP+(10G) / SFP28(25G) / QSFP+(40G) / QSFP28(100G)，机房内接入双模光纤一般即可满足传输距离，相比电模块发热量小很多。

**DAC / AOC**

DAC / AOC 可以理解为线和模块的结合，前者类似电模块加铜缆，后者光模块加光纤。
从使用体验看，DAC 发热量最小，AOC 与光模块光纤类似。

关于接口更多文档可以参考

> https://jia.je/hardware/2020/12/27/ethernet-physical-interfaces/
> 
> https://support.huawei.com/hedex/hdx.do?docid=EDOC1100156553&id=ZH-CN_TOPIC_0250303640&lang=zh

### 交换机

交换机工作在二层，路由器（网关）工作在三层。为了帮助理解，可以把交换机看作网口拓展器，接入交换机的设备均可被独立分配校园网 IP，但互相访问的数据包只经过交换机转发；路由器用于提供 NAT 服务，组建形如 192.168.1.0/24 局域网，并负责到校园网的 NAT 转发。

最近新发售的高端家用路由器（如 XTR10890、AX89X 等）提供各一个万兆光口电口，可以在机房使用。电口接入机房提供的校园网，光口与交换机相连，其余千兆电口用于 IPMI 等电口服务。

交换机一般开箱即用，网口指示灯亮即表示正常工作。如果服务器网卡为双口，且希望使用 LACP (802.3ad) 对不同 TCP 连接负载均衡，在 H3C 上如下设置

```shell
<H3C> system-view
[H3C] display interface brief

[H3C] interface Bridge-Aggregation 1
[H3C-Bridge-Aggregation1] quit
[H3C] interface range Ten-GigabitEthernet 1/0/3 to Ten-GigabitEthernet 1/0/4
[H3C-if-range] port link-aggregation group 1
[H3C-if-range] quit
[H3C] interface Bridge-Aggregation 1
[H3C-Bridge-Aggregation1] port link-type access
[H3C-Bridge-Aggregation1] port access vlan 1
[H3C-Bridge-Aggregation1] link-aggregation mode dynamic
[H3C-Bridge-Aggregation1] quit

[H3C] save force
[H3C] display interface brief
[H3C] display mac-address
```

并在服务器 `/etc/network/interfaces` 上配置 bonding

```bash
auto bond0
iface bond0 inet static
	address 192.168.1.101
	netmask 255.255.255.0
	gateway 192.168.1.1
	bond-slaves enp161s0f0 enp161s0f1
	bond-mode 4
	bond-miimon 100
	bond-downdelay 200
	bond-updelay 200
	bond-lacp-rate 1
	bond-xmit-hash-policy layer3+4
```

测试时首先在交换机查看聚合速率，再使用 iperf3 -P 20 多线程测速。

### InfiniBand

更高网络速度需要使用和以太网协议平行的 IB 交换机，不过经费用完了还没买。

## 硬件

服务器一般由准系统、CPU、内存、硬盘、网卡（和阵列卡、显卡）组成。

### 准系统

主板和机箱在服务器被称为准系统，目前接触到 Supermicro, Gigabyte, Dell 三家产品。

## 管理

### IPMI

IPMI 是通过网络管理设备的接口，是 web 协议的 BIOS 界面，可以避免前往机房使用键盘、显示器调试设备。常用的服务器准系统均支持 IPMI，在 BIOS 设置或操作系统中设置，常用的工具有 ipmitool.

```bash
ipmitool lan 0 print

ipmitool lan set 1 ipaddr 192.168.1.201
ipmitool lan set 1 netmask 255.255.255.0
ipmitool lan set 1 defgw ipaddr 192.168.1.1

ipmitool user list 1
ipmitool user set password 2 Foo@bar114
```

配置为 Shared 模式以便操作系统使用同一块网卡

```bash
ipmitool raw 0x30 0x70 0x0c 1 1
```

然后可以访问 https://192.168.1.201 或使用 ipmitool 进行管理。

### RAID

管理 RAID 阵列卡建议在 BIOS 中操作，也可以使用 MegaRAID 等提供的 cli 工具

> https://docs.broadcom.com/doc/12352476

### NFS

使用小容量 SSD 组建 RAID1 作为系统盘，用网络存储在服务器之间共享数据。NFS 挂载仅取决于 IP，挂载后的权限在服务器间通过 uid 对应。一种解决方式是使用 LDAP 服务统一 uid，另一种方式是通过规则将用户 squash 到对应用户，如把所有用户均映射到特定有权限账户。

```bash
mount.nfs4 192.168.1.3:/volume1/public /public

## /etc/fstab
192.168.1.3:/volume1/public /public nfs4 defaults 0 0
```

### NVIDIA

没有显卡是很快乐的事，但很不幸一台服务器需要配置 Nvidia GPU.

#### CUDA

如果只需使用 CUDA 环境，如 pytorch 等，推荐使用官方 non-free 源安装

```bash
# non-free
sudo apt install nvidia-cuda-dev nvidia-cuda-toolkit
```

#### CUDNN

使用 tensorflow 需要 CUDNN 支持，通过 Nvidia 源实现

```bash
# https://docs.nvidia.com/cuda/cuda-installation-guide-linux
# https://docs.nvidia.com/deeplearning/cudnn/install-guide
wget https://developer.download.nvidia.com/compute/cuda/repos/debian11/x86_64/cuda-keyring_1.0-1_all.deb
sudo dpkg -i cuda-keyring_1.0-1_all.deb
sudo apt install cuda
sudo apt install libcudnn8 libcudnn8-dev
```

### LDAP

LDAP 是通用的统一认证服务，绝大多数应用均支持 LDAP 认证协议。古老的传闻是亲自配置 openLDAP 会延毕，等存储服务器安装之后再试着部署。

## 网络

使用 WireGuard 搭建内网。

## 调试

奇奇怪怪的硬件需要调试。

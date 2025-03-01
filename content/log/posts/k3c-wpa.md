+++
title = '解决 K3(c) 低安全性'
date = 2021-09-27
draft = false
+++

很久之前发现 iOS 提示宿舍无线网络不安全，无线覆盖范围自主可控一直没管，今天强迫症发作尝试解决。首先定位不安全来源，发现默认连接方式支持 WPA-PSK 在 TKIP 标准下加密通信，而二者均被发现存在安全风险。斐讯管理界面没有修改加密功能，因而进行手动尝试。

官改固件刷机过程暂不赘述，首先进入 SSH 终端

```bash
ssh admin@192.168.2.1   
```

其中 192.168.2.1 为路由器 IP，密码为管理界面密码。

定位到路由器配置文件目录

```bash
cd /opt/lantiq/wave/confs
```

在双频合一模式下，无线配置文件主要为以下四者

```bash
# 2.4GHz
hostapd_wlan2.conf
hostapt_vap_wlan2.conf

# 5GHz
hostapd_wlan0.conf
hostapd_vap_wlan0.conf
```

其原本配置为

```bash
wpa=3
wpa_pairwise=TKIP
```

表示默认兼容 WPA1/2，使用 TKIP，修改如下

```bash
wpa=2
wpa_pairwise=CCMP
```

更改为仅允许 WPA2，使用 AES 协议，然后重启路由器问题解决。

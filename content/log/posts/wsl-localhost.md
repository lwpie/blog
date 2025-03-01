+++
title = 'WSL2 访问 Windows 宿主机 IP'
date = 2021-09-18
draft = false
+++

今天帮同学看代理问题，发现网络配置非常扭曲：在 Windows 下使用 Clash GUI 连接代理，希望在 WSL2 下也能访问，又不想在 WSL2 中配置 clash，于是尝试连接 Windows 主机代理端口。

经尝试 localhost, 127.0.0.1, ::1 均无法连接，后思考 WSL2 与 Windows 是否不共享同一个 localhost，发现由于 WSL2 通过 Hyper-V 实现，确实不共享，且每次重启均会发生改变。

尝试如下方法可以在默认路由中找到宿主机

```bash
ip route
```

提取宿主机 IP

```bash
ip route | grep default | awk '{print $3}'
```

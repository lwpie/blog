+++
title = '通过 YubiKey 使用 PGP 与 SSH 证书'
date = 2021-09-16
draft = false
+++

返校后整理书柜时找到一枚 YubiKey 5 NFC，是疫情之前党主席帮忙带的。当时由于还在使用 Windows, GPG tools 支持并不完善，可以使用的功能不多于是闲置。现在切换到 macOS，并且发现 Windows 下 gpg 套件更新得越来越好用，于是重新配置 YubiKey 使之支持 PGP 签名与 SSH 认证。

## Prerequisites

对于大部分 Linux 发行版，什么都不用做

在 macOS 下安装 gnupg 与 pinentry-mac 包，后者为 PIN 弹窗组件

```bash
brew install gnupg
brew install pinentry-mac
```

在 Windows 下安装 [Git Bash](https://git-scm.com/downloads) 并将其添加到环境变量高优先级；安装后需要确定系统使用的工具均由 Git Bash 提供，并使用 PowerShell 作为终端

```bash
> which gpg
/usr/bin/gpg

> which ssh
/usr/bin/ssh
```

插入 YubiKey 后通过 gpg 确定其正常工作

```bash
gpg --card-status
```

如果 YubiKey 曾经被设置过，建议通过 [YubiKey Manager](https://www.yubico.com/support/download/yubikey-manager/) 恢复出厂设置，默认 PIN 为 123456，管理员 PIN 也即 PUK 为 12345678.

## 设置 GPG key 并导出

推荐在 YubiKey 上生成密钥对以确保安全性，注意生成的私钥无法通过任何手段导出，因此需妥善保管实体，避免因丢失或损坏导致私钥无法使用，或者购买两块 YubiKey 导入密钥互为备份。

首先进入 gpg card 交互模式，并启用管理员功能

```bash
gpg --card-edit
> admin
```

建议重置 PIN 与 PUK，牢记密码，PIN 的验证通常只有三次尝试机会

```bash
> passwd
```

然后设置卡片密钥类型，对于 YubiKey 5 建议使用 RSA 4096

```bash
> key-attr
```

之后生成密钥，输入的信息与正常通过 gpg 生成相同，注意离卡备份无法恢复 YubiKey

```bash
> generate
```

生成可能持续很长一段时间，这是因为 YubiKey 的计算速度远没有电脑快，可以观察到灯光闪烁。

之后可以查看生成密钥对的编号，并导出 PGP 公钥

```bash
gpg --list-secret-keys --keyid-format=long
gpg --armor --export <key-id>
gpg --armor --export-ssh-key <key-id>
```

之后可以把 PGP 公钥上传到 [GitHub](https://docs.github.com/cn/github/authenticating-to-github/managing-commit-signature-verification/generating-a-new-gpg-key) 并告诉 git 使用私钥对在提交时签名

```bash
git config user.signingkey <key-id>
git commit -S -m "<message>"
```

## 配置 SSH 与 gpg-agent

之后通过 YubiKey 认证登陆 SSH 服务器。PGP 和 SSH 证书内容等价可以互相转化，认证时使用 gpg-agent 通信。首先导出 SSH 证书，也可以通过 PGP 证书转换，这里采用前者

```bash
gpg --armor --export-ssh-key <key-id>
```

将导出的 SSH 公钥放入目标服务器的 `.ssh/authorized_keys` 中，然后配置本机的 gpg-agent，首先编辑配置文件 `.gnupg/gpg-agent.conf` 如下

```bash
pinentry-program <path-to-pinentry>
enable-ssh-support
default-cache-ttl 60
max-cache-ttl 120
```

其中对于 macOS 为 pinentry-mac，对于 Windows 与 Linux 均为 pinentry，特别地对于 Windows，应填入 Git Bash 路径也即 `/usr/bin/pinentry`.

然后启动 gpg-agent

```bash
gpgconf --launch gpg-agent
```

设置环境变量，对于 bash 与 zsh，设置 tty 窗口与 ssh 代理配置

```bash
export GPG_TTY="$(tty)"
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
```

对于 Powershell，语法比较特殊，并且由于 pinentry 使用 Qt 实现，无需设置 tty

```bash
$Env:SSH_AUTH_SOCK = $(gpgconf --list-dirs agent-ssh-socket)
```

可以把以上语句加入 .zshrc 或 $PROFILE 中，然后即可正常通过 ssh 登陆服务器！

如果报错

```bash
sign_and_send_pubkey: signing failed: agent refused operation
```

可以尝试

```bash
gpg-connect-agent updatestartuptty /bye
```

更具体的讨论见 [gpg agent refusing ssh agent work](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=835394).

## Extras

一些可能有用的参考链接

```
https://support.yubico.com/hc/en-us/articles/360013757959-Resetting-Your-YubiKey-5-series-to-Factory-Defaults
https://www.gnupg.org/gph/en/manual.html
https://brew.sh
https://docs.microsoft.com/en-us/windows/terminal/get-started
https://git-scm.com/doc
https://rnorth.org/gpg-and-ssh-with-yubikey-for-mac
```
+++
title = 'V2Ray 配置记录'
date = 2022-04-08
draft = false
+++

> Across the Great Wall we can reach every corner in the world.

多说无益，直接切入正题，本文为配置记录。

> https://twitter.com/qiushuiyibing/status/1331440933825437697
> <br><br>
> v2ray 是著名开源网络工具程序名，Project V 则是其曾经的项目名。
> <br><br>
> v2fly 是 v2ray 的创始人突然离开后社区重组的开源组织名。
> <br><br>
> Project X 是在 v2ray 无法合并 xtls 的情况下（开源协议许可证问题）另行组建的组织名。

## 安装服务

```bash
# https://github.com/v2fly/fhs-install-v2ray
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
```

## 配置文件

```json
// /usr/local/etc/v2ray/config.json

{
  "inbounds": [
    {
      "port": 8987,
      "listen": "127.0.0.1",
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "12a7d218-fffb-4fdb-a11b-e65f5ce57065",
            "alterId": 0
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "wsSettings": {
          "path": "/e65f5ce57065"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
```

HTTP 开销大，TCP 无法被 CDN 转发，使用 WebSocket 作为伪装。

id 为任意字符串，类似 passphrase，推荐使用 `/proc/sys/kernel/random/uuid`.

alterId 设置为 0 使用 AEAD 确保安全，path 为监听路径，也可任意设置。

## Nginx 反代

申请有效证书

```bash
# acme.sh
acme.sh --issue -d *.<domain> -d <domain> --dns dns_cf

# /etc/nginx/snippets/pwecat.conf
ssl_certificate /root/.acme.sh/*.pwe.cat/fullchain.cer;
ssl_certificate_key /root/.acme.sh/*.pwe.cat/*.pwe.cat.key;
```

配置 Nginx 反代

```nginx
# /etc/nginx/sites-available/ws

server {
	listen 443 ssl;
	listen [::]:443 ssl;

	include snippets/pwecat.conf;

	root /var/www/ws;

	index index.html;

	server_name ws.<domain>;

	location / {
		try_files $uri $uri/ =404;
	}

	location /e65f5ce57065 {
		if ($http_upgrade != "websocket") {
			return 404;
		}
		proxy_redirect off;
		proxy_pass http://127.0.0.1:8987;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	}

	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;
}
```

然后开启服务

```shell
sudo systemctl enable --now v2ray
sudo ln -s /etc/nginx/sites-available/ws /etc/nginx/sites-enabled/
sudo nginx -s reload
```

## 客户端

推荐使用 Shadowrocket

```
vmess://<token>?obfsParam=ws.<domain>&path=/e65f5ce57065&obfs=websocket&tls=1&peer=ws.<domain>&alterId=0
```

或 Clash 作为 macOS 客户端

```yaml
# config.yaml

proxies:
  - name: PROXY
    type: vmess
    server: ws.<domain>
    port: 443
    uuid: 12a7d218-fffb-4fdb-a11b-e65f5ce57065
    alterId: 0
    cipher: auto
    tls: true
    servername: ws.pwe.cat
    network: ws
    ws-opts:
      path: /e65f5ce57065
      headers:
        Host: ws.pwe.cat

rule-providers:
    # https://github.com/Loyalsoldier/clash-rules
```

## CDN

使用 WebSocket 的另一个好处是可以使用 CDN

- Cloudflare 使用 proxy 无需额外配置
- 腾讯云支持已备案域名，选择流媒体加速
- 阿里云使用全站 DCDN 并企业认证提交工单

使用需谨慎。

## 后记

近来梯子频繁遭遇不测，TCP reset 和流量黑洞使得国际互联网访问趋于不稳定。具体来说，在特定重要时间段，除去已知白名单协议如 WEB 和 SSH 等，其余 TCP 连接不分端口均会被定期阻断；在阿里云、移动宽带等特殊网络环境，这种策略不分时段全时执行，严重干扰了正常生活学术。传统协议如 Shadowsocks 强调去特征，已经不能满足需求，还需对流量进行伪装。

常见大流量（且在白名单内的）网络协议无非 WEB，第一反应即为 HTTP，它内容简单、简洁易懂，同时也十分可靠。但缺点也同样明显，每个 HTTP 报文都包含了完整头部，并且数据单向传播，这会带来很大的额外开销，对晚高峰低 QoS 是致命的。在 HTTP 之外，流媒体传播时常用 WS，也即 WebSocket，它支持双向通信，overhead 更小，并且可以扩展协议，可玩性想必更强。经过查询，发现 V2Ray 支持基于 WS 伪装流量，决定一试。

V2Ray 的文档写得一坨狗屎，新人看了不知道怎么快速配置，有经验的玩家又很难找到系统架构，属实非常头大。但这也是好的，使用量越大攻击面越大，Shadowsocks 正因为它的用户基数被一次次盯上。简单来说，和 Shadowsocks 一样，V2Ray 使用了自研通信协议即 VMess，并构建了一个数据处理框架，这使得外层伪装变得简洁。因此出现了 HTTP、TCP、WS 等多种伪装方式，原理大同小异。

## Extras

一些可能有用的链接

> https://guide.v2fly.org
> 
> https://nginx.org/en/docs/beginners_guide.html

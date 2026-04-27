---
title: Vaultwarden 本地部署密码管理服务
tags: self-hosted docker caddy vaultwarden
excerpt: 用 Docker + Caddy 在局域网内部署私有密码管理器 Vaultwarden，通过 HTTPS 安全访问。
---

[Vaultwarden](https://github.com/dani-garcia/vaultwarden) 是 Bitwarden 服务端的轻量级 Rust 实现，可以自托管到本地，数据完全自己掌控。

## 目录结构

```
vaultwarden/
├── compose.yaml      # Docker Compose 配置
├── Caddyfile         # Caddy 反向代理配置
├── caddy-config/     # Caddy 运行时配置（自动生成）
├── caddy-data/       # Caddy 证书等数据（自动生成）
└── vw-data/          # Vaultwarden 数据（自动生成）
```

## compose.yaml

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      DOMAIN: "https://xxxx001.mynetgear.com"
      SIGNUPS_ALLOWED: "true"
    volumes:
      - ./vw-data:/data

  caddy:
    image: caddy:2
    container_name: caddy
    restart: always
    ports:
      - 80:80       # ACME HTTP-01 challenge
      - 443:443
      - 443:443/udp # HTTP/3
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy-config:/config
      - ./caddy-data:/data
    environment:
      DOMAIN: "https://xxxx001.mynetgear.com"
      EMAIL: "xxxx@hotmail.com"
      LOG_FILE: "/data/access.log"
```

Caddy 会自动通过 ACME 申请 TLS 证书，所以 Vaultwarden 容器本身不需要暴露端口，只让 Caddy 对外。

## Caddyfile

```
{$DOMAIN} {
  log {
    level INFO
    output file {$LOG_FILE} {
      roll_size 10MB
      roll_keep 10
    }
  }

  # 通过 ACME HTTP-01 自动申请证书
  tls {$EMAIL}

  encode zstd gzip

  reverse_proxy vaultwarden:80 {
    # 将真实 IP 传给 Vaultwarden，供 fail2ban 使用
    header_up X-Real-IP {remote_host}
  }
}
```

## 启动

```bash
docker compose up -d
```

## 更新

```bash
docker compose pull
docker compose stop
docker compose up -d
```

## 局域网 mDNS 广播（可选）

如果想在局域网内通过 mDNS 让其他设备自动发现这个 HTTPS 服务，可以用以下脚本（macOS）：

```python
from Foundation import NSObject, NSRunLoop, NSNetService

class ServiceDelegate(NSObject):
    def netServiceDidPublish_(self, sender):
        print(f"Published: {sender.name()}")

    def netService_didNotPublish_(self, sender, errorDict):
        print(f"Failed to publish: {errorDict}")

if __name__ == "__main__":
    service = NSNetService.alloc().initWithDomain_type_name_port_(
        "local.",
        "_https._tcp.",
        "MyHTTPSService",
        443
    )
    delegate = ServiceDelegate.alloc().init()
    service.setDelegate_(delegate)
    service.publish()

    NSRunLoop.currentRunLoop().run()
```

运行后，局域网内支持 mDNS 的设备可以发现该服务。

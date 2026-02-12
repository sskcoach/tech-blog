---
title: "VPS 보안 기초: 방화벽과 포트 관리"
date: 2026-02-12
description: "UFW 설정, SSH 포트 변경, Cloudflare Tunnel을 활용한 서버 보안 구성기"
tags:
  - Linux
  - Security
  - VPS
  - Cloudflare
categories:
  - Infrastructure
---

## 오늘 배운 것

VPS 서버의 기본 보안을 설정했다. 방화벽 없이 운영되고 있던 서버에 UFW를 적용하고, 불필요한 포트를 차단했다.

## 현재 상태 파악

```bash
ss -tlnp | grep -v '127.0.0'
```

외부에 노출된 포트를 확인하니 SSH(22), sslh(443) 외에도 Dokploy(3000), WordPress(8080, 8081), Docker Swarm(2377, 7946)이 열려 있었다.

## UFW 설정

```bash
# 필요한 포트만 허용
sudo ufw allow 22/tcp

# 방화벽 활성화
sudo ufw --force enable
```

Cloudflare Tunnel을 사용하면 웹 서비스는 터널 경유로 접근하므로, 외부에 직접 포트를 열 필요가 없다.

## SSH 포트 변경

기존에 sslh를 통해 443 포트로 SSH를 우회하는 구조였는데, Cloudflare Tunnel로 SSH 접속이 가능해지면서 표준 포트(22)로 변경했다.

```bash
# /etc/ssh/sshd_config
Port 22
PermitRootLogin prohibit-password
```

## 핵심 정리

- 방화벽은 기본 중의 기본이다
- 사용하지 않는 포트는 반드시 닫을 것
- Cloudflare Tunnel을 쓰면 웹 서비스 포트를 외부에 노출할 필요가 없다
- `fail2ban`으로 브루트포스 공격을 자동 차단할 수 있다

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

```
 ╔═════════════════════════════════════════════╗
 ║                                             ║
 ║   ___ ___ ___ ___ _ _ _  _  _   _           ║
 ║  | __||_ _|| _ \| __|| | | |/ \ | |  | |    ║
 ║  | _|  | | |   /| _| | V V / _ \| |__| |__  ║
 ║  |_|  |___||_|_\|___| \_/\_/_/ \_\____|____| ║
 ║                                             ║
 ║   VPS Security: Firewall & Port Management  ║
 ║                                             ║
 ╚═════════════════════════════════════════════╝
```

## TL;DR

- 방화벽(UFW) 없이 서버 운영하고 있었다 — 위험
- 외부에 Dokploy(3000), WordPress(8080/8081), Docker Swarm(2377/7946) 노출 상태
- wp-login.php 브루트포스 공격이 IP 직접 접근으로 들어오고 있었음
- UFW 켜서 SSH(22)만 허용, 나머지 전부 차단
- Cloudflare Tunnel 경유 서비스는 영향 없음 (localhost 통신)

---

## 오늘 배운 것

VPS 서버의 기본 보안을 설정했다. 방화벽 없이 운영되고 있던 서버에 UFW를 적용하고, 불필요한 포트를 차단했다.

## 현재 상태 파악

```bash
ss -tlnp | grep -v '127.0.0'
```

외부에 노출된 포트를 확인하니 SSH(22), sslh(443) 외에도 Dokploy(3000), WordPress(8080, 8081), Docker Swarm(2377, 7946)이 열려 있었다.

```
포트    서비스           필요    위험도
──────────────────────────────────────
22      SSH              O      브루트포스 (fail2ban 있음)
443     sslh → SSH       O      회사 우회용
3000    Dokploy          X      관리패널 노출
8080    WP 한국어        X      직접 노출 불필요
8081    WP 영어          X      직접 노출 불필요
2377    Docker Swarm     X      클러스터 관리 포트
7946    Docker Swarm     X      노드 통신 포트
```

WordPress 로그를 확인하니 외부 IP에서 직접 wp-login.php를 때리고 있었다.

```
161.35.157.115 - POST /wp-login.php  ← 반복 브루트포스
159.223.228.106 - POST /wp-login.php ← 반복 브루트포스
```

## UFW 설정

```bash
# 필요한 포트만 허용
sudo ufw allow 22/tcp

# 방화벽 활성화
sudo ufw --force enable

# 확인
sudo ufw status
```

```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
22/tcp (v6)                ALLOW       Anywhere (v6)
```

## Cloudflare Tunnel과의 관계

Cloudflare Tunnel을 사용하면 웹 서비스는 터널 경유로 접근한다. 터널 데몬(cloudflared)이 localhost에서 각 서비스로 연결하므로 외부 포트를 열 필요가 없다.

```
[사용자] → Cloudflare Edge → Tunnel → localhost:3000 (Dokploy)
                                    → localhost:8080 (WP KO)
                                    → localhost:8081 (WP EN)
```

UFW가 차단하는 건 IP 직접 접근뿐이다. 도메인 경유 접근은 터널을 타므로 영향 없음.

## SSH 포트 변경

기존에 sslh를 통해 443 포트로 SSH를 우회하는 구조였는데, Cloudflare Tunnel로 SSH 접속이 가능해지면서 표준 포트(22)로 변경했다.

```bash
# /etc/ssh/sshd_config
Port 22
PermitRootLogin prohibit-password
```


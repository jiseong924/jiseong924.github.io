---
layout: post
title: "재부팅 후 사이트가 안 열렸다: 범인은 롤백된 default route였다"
date: 2026-08-20 18:22:49 +0900
categories: [troubleshooting]
tags: [network, linux, netplan, cloud-init, routing]
---

사내 서버실 장비가 꺼졌다 켜졌다. 서비스는 다 정상으로 올라왔는데 사이트는 외부에서 안 열렸다. 원인은 그보다 한 계층 아래, default gateway에 있었다.

> **TL;DR**
>
> - 증상: 재부팅 후 서버 안은 전부 정상인데 외부에서 사이트 접속 불가
> - 원인: default gateway가 `10.1.1.2` → `10.1.1.1`로 롤백되어 응답이 다른 공인 IP를 달고 나감 (비대칭 라우팅)
> - 진단: `curl -s ifconfig.me`와 `dig +short` 결과가 다르면 비대칭 라우팅
> - 해결: cloud-init의 네트워크 관리를 끄고 `50-cloud-init.yaml`을 직접 수정

## 1. 구성과 증상

```text
인바운드:        외부 → <공인IP-A>(X.X.X.184) → (방화벽 NAT) → 10.1.1.141
아웃바운드(정상): 10.1.1.141 → 10.1.1.2 → X.X.X.184
아웃바운드(장애): 10.1.1.141 → 10.1.1.1 → X.X.X.183   ← 불일치
```

두 공인 IP는 끝자리 하나 차이다. 요청은 `.184`로 들어오는데 응답은 `.183`을 달고 나가니, 클라이언트 입장에서는 SYN을 보낸 상대와 SYN-ACK을 보낸 상대가 다르다. 연결이 성립할 수가 없다.

## 2. 진단 — IP 3종 비교

처음에는 서버 안을 들여다봤다. 웹 서버 → 앱 → DB 순으로 파고들었는데 전부 정상이었고, 시간만 썼다. 실제로 답을 준 건 서버 밖을 본 이 세 줄이었다.

```bash
curl -s ifconfig.me           # X.X.X.183      ← 아웃바운드 공인 IP
dig +short www.example.com    # X.X.X.184      ← DNS A 레코드
ip -4 addr show scope global  # 10.1.1.141/20
sudo ufw status verbose       # inactive       ← 서버 방화벽 무관
```

`ifconfig.me`와 `dig` 결과가 다르다. 나가는 IP와 들어오는 IP가 다르니 비대칭 라우팅이 확정됐다.

```bash
ip route show default
# default via 10.1.1.1 dev enp4s0   ← .2여야 하는데 롤백됨
```

과거에 게이트웨이를 `.2`로 바꿔 둔 이력이 있었다. 문제는 그게 `ip route`로 준 런타임 명령이었다는 점이다.

```bash
sudo ip route replace default via 10.1.1.2 dev enp4s0
```

이 방식은 커널 라우팅 테이블만 그 자리에서 바꾼다. `/etc/netplan` 아래 설정 파일은 그대로이므로 재부팅하면 원래 값인 `.1`로 되돌아간다. 그동안은 서버를 끌 일이 없어서 몇 달간 정상으로 보였고, 정전으로 처음 재부팅되면서 드러났다.

## 3. 해결

같은 실수를 반복하지 않으려면 설정 파일을 고쳐야 한다. cloud-init의 네트워크 관리를 끄고 `50-cloud-init.yaml`을 직접 수정했다. 이 파일은 cloud-init이 매 부팅마다 재생성하므로, 비활성화를 빼면 파일을 고쳐도 똑같이 원복된다.

```bash
# 1) cloud-init의 네트워크 관리 차단 (이걸 빼면 재부팅 시 원복)
echo 'network: {config: disabled}' \
  | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg

# 2) 원본 백업 후 수정
sudo cp /etc/netplan/50-cloud-init.yaml /root/50-cloud-init.yaml.bak
sudo nano /etc/netplan/50-cloud-init.yaml   # via: 10.1.1.1 → 10.1.1.2

# 3) 문법 검증 (출력이 없어야 정상)
sudo netplan generate

# 4) 적용
sudo netplan try
```

최종 설정은 이렇게 됐다.

```yaml
network:
  version: 2
  ethernets:
    enp4s0:
      addresses:
        - 10.1.1.141/20
      nameservers:
        addresses:
          - 10.1.1.11
        search: []
      routes:
        - to: default
          via: 10.1.1.2
```

민망한 건, 답이 이미 그 파일 맨 위 주석에 적혀 있었다는 점이다.

```text
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

### 재부팅 후 검증

```bash
ip route show default
# default via 10.1.1.2 dev enp4s0 proto static   ← proto static = netplan 적용

curl -s ifconfig.me
# X.X.X.184                                       ← DNS와 일치
```

## 4. 참고 자료

- [`netplan` 설정 레퍼런스](https://netplan.readthedocs.io/en/stable/netplan-yaml/) — `routes`, `to: default`
- [`netplan try`](https://netplan.readthedocs.io/en/stable/netplan-try/) — 자동 롤백 동작
- [cloud-init 네트워크 설정 비활성화](https://cloudinit.readthedocs.io/en/latest/reference/network-config.html#disabling-network-configuration) — `99-disable-network-config.cfg`
- [`ip-route(8)`](https://man7.org/linux/man-pages/man8/ip-route.8.html) — `ip route show default`

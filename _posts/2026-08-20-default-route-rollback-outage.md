---
layout: post
title: "재부팅 후 사이트가 안 열렸다: 범인은 롤백된 default route였다"
date: 2026-08-20 18:00:00 +0900
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

과거에 누군가 게이트웨이를 `.2`로 바꾼 이력이 있었다. 그런데 그게 `ip route` 런타임 명령이었거나, `50-cloud-init.yaml`을 고쳤지만 cloud-init을 비활성화하지 않은 상태였다. 어느 쪽이든 재부팅하면 원래 설정으로 되돌아간다.

## 3. 해결

`ip route`로 런타임만 고치면 다음 재부팅에 또 되돌아간다. cloud-init의 네트워크 관리를 끄고 `50-cloud-init.yaml`을 직접 수정했다.

```bash
# 1) cloud-init의 네트워크 관리 차단 (이걸 빼면 재부팅 시 원복)
echo 'network: {config: disabled}' \
  | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg

# 2) 원본 백업 후 수정
sudo cp /etc/netplan/50-cloud-init.yaml /root/50-cloud-init.yaml.bak
sudo nano /etc/netplan/50-cloud-init.yaml   # via: 10.1.1.1 → 10.1.1.2

# 3) 문법 검증 (출력이 없어야 정상)
sudo netplan generate

# 4) 적용 — apply 아님, try
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

### 검증

```bash
ip route show default
# default via 10.1.1.2 dev enp4s0 proto static   ← proto static = netplan 적용

curl -s ifconfig.me
# X.X.X.184                                       ← DNS와 일치
```

## 4. netplan 관련해서 알게 된 것들

- **`netplan try` vs `apply`**: SSH 원격 작업이면 반드시 `try`. 120초 안에 Enter로 확정하지 않으면 자동 롤백되어 접속 유실을 막아준다. 반대로 Enter를 안 누르면 적용도 안 된다. 이것 때문에 한 번 헛돌았다.
- **`netplan get routes`가 `null`인 건 정상**: 최상위 `network.routes`를 조회하는 명령이다. 인터페이스 하위 route는 `netplan get ethernets.enp4s0.routes`로 봐야 한다.
- **`WARNING:root:Cannot call Open vSwitch`**: OVS를 안 쓰는 환경에서 나오는 정상 메시지다. 무시해도 된다.
- **netplan 파일 권한**: `chmod 600`을 안 하면 경고가 뜬다.
- **`gateway4`와 `routes: to: default` 동시 선언 금지**: `Conflicting default route declarations` 에러가 난다. 게이트웨이는 한 곳에만 적는다.

## 5. 회고

증상은 위에서 보이는데 원인은 아래에 있었다. 서비스 계층을 위에서만 훑는 습관이 이번 시간 낭비의 대부분을 만들었다. 서버 안이 다 정상이면 다음은 서버가 세상에 어떤 IP로 나가는지 확인할 차례다.

"수동으로 고쳤다"는 영속성을 보장하지 않는다는 것도 다시 확인했다. `ip route`는 휘발성이고, cloud-init 관리 하의 파일 수정도 결국 휘발성이다. 재부팅으로 검증하지 않았다면 고쳤다고 말할 수 없다.

남은 과제는 두 가지다. 게이트웨이 `10.1.1.1`(일반망)과 `10.1.1.2`(DMZ·NAT망)의 역할 구분을 팀 문서에 명시하는 것, 그리고 온프레미스 NAT/라우팅 구간이 재부팅 한 번에 서비스 전체를 내리는 SPOF라는 점이다. 후자는 클라우드로 옮기면 보안그룹과 라우팅 테이블로 코드화되면서 상당 부분 해소될 영역이다.

## 6. 참고 자료

- [`netplan` 설정 레퍼런스](https://netplan.readthedocs.io/en/stable/netplan-yaml/) — `routes`, `to: default`
- [`netplan try`](https://netplan.readthedocs.io/en/stable/netplan-try/) — 자동 롤백 동작
- [cloud-init 네트워크 설정 비활성화](https://cloudinit.readthedocs.io/en/latest/reference/network-config.html#disabling-network-configuration) — `99-disable-network-config.cfg`
- [`ip-route(8)`](https://man7.org/linux/man-pages/man8/ip-route.8.html) — `ip route show default`

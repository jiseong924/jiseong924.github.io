---
layout: post
title: "재부팅 후 사이트가 안 열렸다: 범인은 롤백된 default route였다"
date: 2026-08-20 18:00:00 +0900
categories: [troubleshooting]
tags: [network, linux, netplan, cloud-init, nginx, pm2]
---

사내 서버실 장비가 꺼졌다 켜졌다. PM2 프로세스는 배포 파이프라인으로 되살렸는데 사이트는 여전히 외부에서 안 열렸다. nginx도 앱도 정상이었다. 원인은 그보다 한 계층 아래, default gateway에 있었다.

> **TL;DR**
>
> - 증상: 재부팅 후 프로세스는 전부 `online`인데 외부에서 사이트 접속 불가
> - 원인: default gateway가 `10.1.1.2` → `10.1.1.1`로 롤백되어 응답이 다른 공인 IP를 달고 나감 (비대칭 라우팅)
> - 해결: cloud-init의 네트워크 관리를 끄고 `50-cloud-init.yaml`을 직접 수정
> - 진단 3줄: `curl -s ifconfig.me` / `dig +short www.example.com` / `ip route show default`

## 1. 구성과 사건 개요

```text
인바운드:        외부 → <공인IP-A>(X.X.X.184) → (방화벽 NAT) → 10.1.1.141
아웃바운드(정상): 10.1.1.141 → 10.1.1.2 → X.X.X.184
아웃바운드(장애): 10.1.1.141 → 10.1.1.1 → X.X.X.183   ← 불일치
```

두 공인 IP는 끝자리 하나 차이다. 요청은 `.184`로 들어오는데 응답은 `.183`을 달고 나가니, 클라이언트 입장에서는 SYN을 보낸 상대와 SYN-ACK을 보낸 상대가 다르다. 연결이 성립할 수가 없다.

물론 이걸 처음부터 알았던 건 아니다. 여기까지 오는 데 오진을 세 번 했다.

## 2. 1차 오진 — "PM2가 문제겠지"

재부팅 직후 `pm2 list`가 텅 비어 있었다. 원인을 찾았다고 생각하고 배포 파이프라인을 재실행했다. 프로세스는 전부 `online`으로 올라왔는데 사이트는 그대로 안 열렸다.

PM2의 `online`은 "프로세스가 살아있다"는 뜻일 뿐 "정상 동작 중"이 아니다. 크래시 루프에 빠져 있어도 목록에서는 `online`으로 보이니, 재시작 카운트(`↺`)를 같이 봐야 상태를 판단할 수 있다. 이번엔 `↺ 0`이었으므로 앱은 무죄였는데, 그 판단을 하는 데 시간을 썼다.

## 3. 2차 오진 — "SSL 에러 로그가 범인이다"

`/var/log/nginx/error.log`를 열었더니 `[crit]`이 잔뜩 찍혀 있었다.

```text
SSL_do_handshake() failed (SSL: error:0A00006C:SSL routines::bad key share)
  while SSL handshaking, client: 107.189.19.172, server: 0.0.0.0:443
SSL_do_handshake() failed (... decryption failed or bad record mac)
  while SSL handshaking, client: 18.97.19.206, server: 0.0.0.0:443
```

`[crit]` 레벨이라 눈길이 갔지만 전부 스캐너 봇 노이즈였다. `bad key share`는 TLS 1.3의 key_share 그룹이 서로 안 맞을 때 흔히 나오고, 클라이언트 IP도 전형적인 스캔 대역이다. 인터넷에 노출된 서버의 nginx error.log는 평소에도 이런 기록으로 가득하다. 로그 레벨보다 타임스탬프를 먼저 봐야 했다.

그런데 타임스탬프를 보니 진짜 단서는 로그에 찍힌 내용이 아니라 **로그가 없다는 사실**이었다. 재부팅 시각 이후로는 봇 스캔 기록조차 끊겨 있었다. 외부 트래픽이 서버에 아예 도달하지 않고 있다는 뜻이다. `connect() failed ... upstream` 같은 502 계열 에러도 전무했으니 nginx–업스트림 구간도 무죄였다.

에러가 찍히는 것보다 아무것도 안 찍히는 게 더 중요한 신호일 수 있다.

## 4. 3차 오진 — `curl: (52) Empty reply from server`

로컬에서 nginx를 찔러봤다.

```bash
curl -I http://127.0.0.1
# curl: (52) Empty reply from server
```

역시 nginx가 이상하다고 판단했는데, 잘못된 건 테스트 쪽이었다. Host 헤더가 `127.0.0.1`이라 `server_name` 블록에 매칭되지 않고 캐치올로 떨어진 것이다.

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;
}
```

`return 444`는 응답 없이 연결을 끊는다. curl은 그걸 `(52) Empty reply`로 보고한다. 즉 설정대로 완벽히 동작한 결과를 장애 증상으로 오독했다. 가상호스트 환경에서 로컬 검증을 할 때는 Host 헤더를 반드시 지정해야 한다.

```bash
curl -Ik https://example.com --resolve example.com:443:127.0.0.1
# HTTP/2 301 → 정상
```

## 5. 결정타 — IP 3종 비교

여기서 서버 안이 아니라 서버 밖을 보기 시작했다.

```bash
curl -s ifconfig.me           # X.X.X.183      ← 아웃바운드 공인 IP
dig +short www.example.com    # X.X.X.184      ← DNS A 레코드
ip -4 addr show scope global  # 10.1.1.141/20
sudo ufw status verbose       # inactive       ← 서버 방화벽 무관
```

`ifconfig.me`와 `dig` 결과가 다르다. 나가는 IP와 들어오는 IP가 다르니 비대칭 라우팅이 확정됐다. 이 서버에서 무엇이 정상인지 확인하는 데 오래 걸렸는데, 실제로 답을 준 건 이 세 줄이었다.

```bash
ip route show default
# default via 10.1.1.1 dev enp4s0   ← .2여야 하는데 롤백됨
```

과거에 누군가 게이트웨이를 `.2`로 바꾼 이력이 있었다. 그런데 그게 `ip route` 런타임 명령이었거나, `50-cloud-init.yaml`을 고쳤지만 cloud-init을 비활성화하지 않은 상태였다. 어느 쪽이든 재부팅하면 원래 설정으로 되돌아간다.

## 6. 해결 — 여기서도 한 번 실패했다

### 시도 1: 우선순위 파일 분리 → 실패

`99-static.yaml`을 새로 만들어 `50-cloud-init.yaml`을 덮어쓰려 했다.

```text
Error: Conflicting default route declarations for IPv4
  (table: main, metric: default), first declared in enp4s0 but also in enp4s0
```

netplan은 여러 파일의 설정을 병합한다. 그런데 `routes`는 리스트라서 덮어쓰기가 아니라 합쳐진다. 결과적으로 `via 10.1.1.1`과 `via 10.1.1.2`가 같은 테이블·같은 메트릭에 공존하면서 충돌했다.

"숫자 큰 파일이 이긴다"는 netplan 우선순위 규칙은 **스칼라 값에만** 적용된다. 리스트형 필드에는 통하지 않는다. 알고 있다고 생각했던 규칙이 절반만 맞았던 셈이다.

### 시도 2: cloud-init 비활성화 + 단일 파일 수정 → 성공

```bash
# 1) 충돌 파일 제거
sudo rm /etc/netplan/99-static.yaml

# 2) cloud-init의 네트워크 관리 차단 (이걸 빼면 재부팅 시 원복)
echo 'network: {config: disabled}' \
  | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg

# 3) 원본 백업 후 수정
sudo cp /etc/netplan/50-cloud-init.yaml /root/50-cloud-init.yaml.bak
sudo nano /etc/netplan/50-cloud-init.yaml   # via: 10.1.1.1 → 10.1.1.2

# 4) 문법 검증 (출력이 없어야 정상)
sudo netplan generate

# 5) 적용 — apply 아님, try
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

pm2 list
# 6 processes online, ↺ 0                         ← 크래시 루프 없음
```

## 7. 부수적으로 건진 것들

- **`netplan try` vs `apply`**: SSH 원격 작업이면 반드시 `try`. 120초 안에 Enter로 확정하지 않으면 자동 롤백되어 접속 유실을 막아준다. 반대로 Enter를 안 누르면 적용도 안 된다. 이것 때문에 한 번 헛돌았다.
- **`netplan get routes`가 `null`인 건 정상**: 최상위 `network.routes`를 조회하는 명령이다. 인터페이스 하위 route는 `netplan get ethernets.enp4s0.routes`로 봐야 한다.
- **`WARNING:root:Cannot call Open vSwitch`**: OVS를 안 쓰는 환경에서 나오는 정상 메시지다. 무시해도 된다.
- **netplan 파일 권한**: `chmod 600`을 안 하면 경고가 뜬다.
- **`gateway4`와 `routes: to: default` 동시 선언 금지**: 위와 같은 충돌이 재발한다.
- **PM2 영속화**: `pm2 startup` 등록 후 `pm2 save`. 프로세스 구성을 바꿀 때마다 `save`를 다시 해야 한다. `pm2 resurrect`는 `~/.pm2/dump.pm2` 기준으로 복원한다.

## 8. 회고

증상은 위에서 보이는데 원인은 아래에 있었다. 웹 서버 → 앱 → DB 순으로 파고들었지만 정답은 그보다 아래인 L3 라우팅이었다. OSI 계층을 위에서만 훑는 습관이 이번 시간 낭비의 대부분을 만들었다.

"수동으로 고쳤다"는 영속성을 보장하지 않는다는 것도 다시 확인했다. `ip route`는 휘발성이고, cloud-init 관리 하의 파일 수정도 결국 휘발성이다. 재부팅으로 검증하지 않았다면 고쳤다고 말할 수 없다.

그리고 로그의 침묵도 신호다. 이번엔 찍힌 에러보다 안 찍힌 구간이 더 정확한 단서였다.

## 9. 트러블슈팅 체크리스트

다음에 같은 상황이 오면 이 순서로 본다.

```bash
# 1. 증상 특정 — 에러 종류가 곧 계층을 가리킨다
curl -v https://example.com
#   timeout → 방화벽/라우팅  |  refused → nginx 미기동
#   502     → 업스트림 다운   |  504     → 업스트림 지연

# 2. 로컬 검증 (Host 헤더 필수)
curl -Ik https://example.com --resolve example.com:443:127.0.0.1

# 3. 업스트림 리스닝 확인
ss -tlnp | grep -E ':(3000|3001|4000)'

# 4. 외부 도달성 — 이 3줄이 핵심
curl -s ifconfig.me
dig +short www.example.com
ip route show default

# 5. 요청 도달 여부
sudo tail -50 /var/log/nginx/access.log
sudo tcpdump -i enp4s0 -n 'tcp port 443'
```

후속 과제도 남았다. 게이트웨이 `10.1.1.1`(일반망)과 `10.1.1.2`(DMZ·NAT망)의 역할 구분을 팀 문서에 명시해 두는 것, 그리고 온프레미스 NAT/라우팅 구간이 재부팅 한 번에 서비스 전체를 내리는 SPOF라는 점이다. 후자는 클라우드로 옮기면 보안그룹과 라우팅 테이블로 코드화되면서 상당 부분 해소될 영역이다.

## 10. 참고 자료

- [`netplan` 설정 레퍼런스](https://netplan.readthedocs.io/en/stable/netplan-yaml/) — `routes`, 파일 병합 규칙
- [`netplan try`](https://netplan.readthedocs.io/en/stable/netplan-try/) — 자동 롤백 동작
- [cloud-init 네트워크 설정 비활성화](https://cloudinit.readthedocs.io/en/latest/reference/network-config.html#disabling-network-configuration) — `99-disable-network-config.cfg`
- [nginx `return 444`](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#return) — 응답 없이 연결 종료
- [PM2 startup / save](https://pm2.keymetrics.io/docs/usage/startup/) — 재부팅 후 프로세스 복원

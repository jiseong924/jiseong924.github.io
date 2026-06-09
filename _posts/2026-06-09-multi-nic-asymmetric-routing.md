---
layout: post
title: "멀티 NIC 서버 비대칭 라우팅 해결기: 내부망은 되는데 외부망만 안 됐던 이유"
date: 2026-06-09 18:19:00 +0900
categories: [troubleshooting]
tags: [network, linux, routing]
---

NIC 2개짜리 Ubuntu 서버에서 외부망 접속만 안 됐다. 내부망, 서버, 방화벽은 모두 정상이었다. 확인해 보니 원인은 라우팅에 있었다.

> **TL;DR**
>
> - 증상: 외부에서 TCP 연결은 되는데 HTTP 응답을 못 받고 timeout (내부망은 정상)
> - 원인: 내부망 default route의 metric이 외부망보다 낮아져, 외부로 갈 응답이 내부망 NIC로 나감 (비대칭 라우팅)
> - 해결: 외부망 default route metric을 더 낮게 고정
> - 진단 명령: `ip route get <외부_클라이언트_IP>`

## 1. 문제 상황

```text
내부망 NIC: eth0 / 10.0.1.10      (GW 10.0.1.1)
외부망 NIC: eth1 / 203.0.113.10   (GW 203.0.113.1)
Tomcat: 8180 / 외부 클라이언트: 198.51.100.20
```

외부에서 `curl http://203.0.113.10:8180`을 실행하면 TCP 연결은 되는데 응답 0바이트로 timeout이 발생했다. 포트가 닫혔다면 connection refused가 떴을 테니 단순 포트나 방화벽 문제는 아니다. 내부망(`10.0.1.10:8180`)은 정상이었다. 우선 연결은 되는데 응답만 안 온다는 점을 단서로 잡았다.

## 2. 원인 진단 과정

먼저 기본부터 점검했는데 전부 정상이었다.

| 확인             | 명령                         | 결과        |
| ---------------- | ---------------------------- | ----------- |
| 포트 listen      | `ss -lntp \| grep 8180`      | 정상        |
| 로컬/내부망 응답 | `curl http://10.0.1.10:8180` | 정상        |
| 방화벽           | `iptables -S`                | 8180 ACCEPT |

여기까지 정상이라 서버 자체보다 패킷 경로를 의심하는 방향으로 넘어갔다. tcpdump로 8180 포트를 떠봤다.

```bash
sudo tcpdump -i any -A tcp port 8180
```

서버는 `HTTP/1.1 200`을 만들어 보내는데 클라이언트가 ACK를 주지 않아 같은 응답을 계속 재전송하고 있었다. 서버는 응답을 보냈지만 클라이언트가 받지 못하는 경로 문제다. 그럼 응답이 어느 NIC로 나가는지 봐야 했다.

```bash
ip route get 198.51.100.20
# 198.51.100.20 via 10.0.1.1 dev eth0 src 10.0.1.10   ← 내부망으로 나감!
```

요청은 외부망 `203.0.113.10`(eth1)으로 들어왔는데 응답은 내부망 `10.0.1.10`(eth0)으로 나가고 있었다. 들어온 경로와 나간 경로가 다른 비대칭 라우팅 상태라, 클라이언트는 응답 패킷을 자기 것으로 인정하지 않고 폐기한다.

원인은 `ip route`의 metric이었다. default 경로가 2개인데 내부망(metric 100)이 외부망(metric 101)보다 낮았다. metric은 값이 낮을수록 우선이므로 외부로 나갈 트래픽이 내부망 게이트웨이를 타고 있었다.

이 서버는 NetworkManager로 관리되고 있었는데, NetworkManager는 metric을 따로 지정하지 않으면 NIC에 metric 100, 101을 자동으로 부여한다. 이 값이 내부망에 더 낮게 잡혀 있었던 것이다. 별도 세팅 없이 NIC를 여러 개 두면 이렇게 외부망 응답이 내부망으로 새는 문제가 생길 수 있다.

## 3. 해결 방법

NetworkManager에 metric을 고정한다. 재부팅이나 인터페이스 재연결로 활성화 순서가 바뀌어도 우선순위가 뒤집히지 않도록, 이 방향으로 고정하는 게 안전하다.

```bash
nmcli con show   # 연결 이름 확인
sudo nmcli con mod "외부망_연결이름" ipv4.route-metric 50
sudo nmcli con mod "내부망_연결이름" ipv4.route-metric 100
```

적용 후 `ip route get 198.51.100.20`을 다시 실행해 `dev eth1 src 203.0.113.10`으로 바뀌면 정상이다.

## 4. 원인 요약

멀티 NIC 서버에서 내부망 default route의 metric이 외부망보다 낮아져, 공인 IP로 들어온 요청의 응답이 내부망 NIC로 나갔다(비대칭 라우팅). 그래서 외부 클라이언트는 TCP 연결까지만 성공하고 HTTP 응답은 받지 못했다.

NIC가 둘 이상이면 패킷이 어느 경로로 나가는지는 자동으로 보장되지 않는다. 이 경우처럼 프로세스와 방화벽이 정상인데 응답만 안 온다면, 우선 `ip route get <클라이언트_IP>`로 응답이 나가는 NIC와 출발지 IP를 확인하는 게 빠르다. 향후 멀티 NIC를 새로 구성할 때 default route metric을 명시적으로 고정해 두면 같은 문제를 예방할 수 있다.

## 5. 참고 자료

- [`ip-route(8)`](https://man7.org/linux/man-pages/man8/ip-route.8.html) — `metric`, `ip route get`
- [`nmcli`](https://networkmanager.dev/docs/api/latest/nmcli.html) / [`ipv4.route-metric`](https://networkmanager.dev/docs/api/latest/nm-settings-nmcli.html) — route metric 고정
- [NetworkManager의 활성화 순서별 metric(100/101) 부여 규칙](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/managing-the-default-gateway-setting_configuring-and-managing-networking) — 24.8. How NetworkManager manages multiple default gateways 참고
- [`tcpdump(1)`](https://www.tcpdump.org/manpages/tcpdump.1.html)

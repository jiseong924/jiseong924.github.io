---
layout: post
title: "멀티 NIC 서버 비대칭 라우팅 해결기: 내부망은 되는데 외부망만 안 됐던 이유"
date: 2026-06-09 21:00:00 +0900
categories: [troubleshooting]
tags: [network, linux, routing]
---

NIC 2개짜리 Ubuntu 서버의 Tomcat이 며칠 잘 돌다 갑자기 **외부망에서만** 안 됐다. 내부망·서버·방화벽 다 정상. 원인은 애플리케이션이 아니라 **라우팅**이었다.

> **TL;DR**
> - **증상:** 외부에서 TCP 연결은 되는데 HTTP 응답을 못 받고 timeout (내부망은 정상)
> - **원인:** 내부망 default route의 metric이 외부망보다 낮아져, 외부로 갈 응답이 내부망 NIC로 새어 나감 (비대칭 라우팅)
> - **해결:** 외부망 default route metric을 더 낮게 고정
> - **진단 명령:** `ip route get <외부_클라이언트_IP>`
>
> *(IP는 모두 예시값)*

## 1. 문제 상황

```text
내부망 NIC: eth0 / 10.0.1.10      (GW 10.0.1.1)
외부망 NIC: eth1 / 203.0.113.10   (GW 203.0.113.1)
Tomcat: 8180 / 외부 클라이언트: 198.51.100.20
```

외부에서 `curl http://203.0.113.10:8180`을 치면 **TCP 연결은 되는데 응답 0바이트로 timeout.** 포트가 닫혔다면 connection refused가 떴을 테니, 단순 포트/방화벽 문제는 아니다. 내부망(`10.0.1.10:8180`)은 정상. "연결은 되는데 응답만 안 온다"가 핵심 단서였다.

## 2. 원인을 찾아간 과정

**① 기본 점검 — 전부 정상**

| 확인 | 명령 | 결과 |
|---|---|---|
| 포트 listen | `ss -lntp \| grep 8180` | 정상 |
| 로컬/내부망 응답 | `curl http://10.0.1.10:8180` | 정상 |
| 방화벽 | `iptables -S` | 8180 ACCEPT |

**② tcpdump — 응답이 새고 있다**

```bash
sudo tcpdump -i any -A tcp port 8180
```

서버는 `HTTP/1.1 200`을 만들어 보내는데, 클라이언트가 ACK를 안 주니 **같은 응답을 계속 재전송**하고 있었다. 즉 서버는 보냈지만 클라이언트가 못 받는 경로 문제.

**③ `ip route get` — 결정타**

```bash
ip route get 198.51.100.20
# 198.51.100.20 via 10.0.1.1 dev eth0 src 10.0.1.10   ← 내부망으로 나감!
```

요청은 외부망 `203.0.113.10`(eth1)으로 들어왔는데, **응답은 내부망 `10.0.1.10`(eth0)으로 나가고 있었다.** 들어온 길과 나간 길이 다른 **비대칭 라우팅**이라, 클라이언트는 응답 패킷을 자기 것으로 인정하지 않고 버린다.

원인은 `ip route`의 metric. default 경로가 2개인데 내부망(metric 100)이 외부망(metric 101)보다 낮았다. **metric은 낮을수록 우선**이므로, 외부로 나갈 트래픽이 내부망 게이트웨이를 탔다.

> **"처음엔 됐는데 왜?"** — 내부망 통신은 `10.0.0.0/16` 전용 경로를 타므로 default 우선순위와 무관하다. default 싸움은 외부 트래픽에서만 벌어진다.
>
> 이 서버는 NetworkManager로 관리되고 있었다. NetworkManager는 이더넷 NIC가 둘이면 **먼저 활성화된 연결에 metric 100, 다음 연결에 101**을 부여한다(배포판과 무관한 NetworkManager 공통 동작). 그래서 처음엔 외부망이 먼저 올라와 100(우선)을 받아 정상이었는데, 재부팅·DHCP 갱신·인터페이스 재연결로 **활성화 순서가 바뀌자** 내부망이 100을 받으며 우선순위가 통째로 뒤집힌 것.
>
> *(Ubuntu Server는 기본적으로 systemd-networkd를 쓴다. NetworkManager를 쓰지 않는 환경이라면 metric은 netplan/networkd 설정을 따른다.)*

## 3. 해결 방법

**임시** — metric을 즉시 되돌린다 (재부팅 시 사라짐).

```bash
sudo ip route replace default via 203.0.113.1 dev eth1 metric 50
ip route get 198.51.100.20   # → dev eth1 src 203.0.113.10 으로 바뀌면 OK
```

**영구** — NetworkManager에 고정한다.

```bash
nmcli con show   # 연결 이름 확인
sudo nmcli con mod "외부망_연결이름" ipv4.route-metric 50
sudo nmcli con mod "내부망_연결이름" ipv4.route-metric 100
```

## 4. 원인 요약

> 멀티 NIC 서버에서 내부망 default route의 metric이 외부망보다 낮아져, 공인 IP로 들어온 요청의 응답이 내부망 NIC로 나갔다(비대칭 라우팅). 외부 클라이언트는 TCP 연결까지만 성공하고 HTTP 응답은 받지 못했다.

NIC가 둘 이상이면 "패킷이 어느 길로 나가는가"는 자동으로 보장되지 않는다. 외부 접속 장애에서 프로세스·방화벽이 멀쩡한데 응답만 안 온다면, 가장 먼저 `ip route get <클라이언트_IP>`로 응답이 나가는 NIC와 출발지 IP를 확인하자.

## 5. 참고 자료

- [`ip-route(8)`](https://man7.org/linux/man-pages/man8/ip-route.8.html) — `metric`, `ip route get`
- [`nmcli`](https://networkmanager.dev/docs/api/latest/nmcli.html) / [`ipv4.route-metric`](https://networkmanager.dev/docs/api/latest/nm-settings-nmcli.html) — route metric 고정
- [NetworkManager의 활성화 순서별 metric(100/101) 부여 규칙](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/managing-the-default-gateway-setting_configuring-and-managing-networking) — 배포판 무관한 NetworkManager 데몬 동작 (문서 출처는 Red Hat이지만 동일 데몬을 쓰는 Ubuntu에도 적용)
- [`tcpdump(1)`](https://www.tcpdump.org/manpages/tcpdump.1.html)

---
layout: single
title: "PGA, PSA, PSC — 이름이 다 비슷한 GCP Private 삼형제 구분하기"
date: 2026-09-12 21:00:00 +0900
categories: [GCP, Networking]
tags: [gcp, vpc, networking, pca, private-service-connect]
author_profile: true
toc: true
---

PCA를 준비하면서 가장 헷갈렸던 개념 세 가지를 정리한다.

- Private Google Access (PGA)
- Private Service Access (PSA)
- Private Service Connect (PSC)

이름이 전부 "Private"으로 시작하고, 셋 다 "외부 IP 없이 접근"이라는 같은 문제를 푼다.
그래서 강의를 들을 때는 이해한 것 같았는데, 며칠 뒤에 누가 물어보면 한 마디도 못 했다.

이 글은 그 실패를 복기하면서 쓴다.

## 왜 헷갈렸나

셋을 "비슷한 기능 세 개"로 외우려고 했기 때문이다.

실제로는 **푸는 방식이 전혀 다르다.** 목적지가 다르고, IP가 어디서 나오는지가 다르고,
온프렘에서 쓸 수 있는지가 다르다. 이 축을 잡지 않고 기능 설명만 읽으면 계속 섞인다.

## 공통 전제: 왜 외부 IP를 없애려 하는가

VM에 외부 IP를 붙이면 인터넷에서 그 VM으로 **들어올 수 있다.** 공격 표면이 생긴다.

그런데 외부 IP를 떼면 이번엔 VM이 밖으로 나가지 못한다.
GCS에 파일을 못 올리고, BigQuery에 쿼리를 못 던진다.

이 문제를 푸는 방식이 세 가지로 갈린다.

## PGA — 경로만 사설로

서브넷 설정의 on/off 토글이다. 켜면 외부 IP 없는 VM이 구글 API에 접근할 수 있다.

```
Compute Engine → 서브넷 편집 → 비공개 Google 액세스: 사용
```

여기서 중요한 오해 하나. **목적지 주소는 여전히 구글의 공개 IP다.**
사설 IP가 생기는 게 아니라 **경로가 인터넷을 타지 않는 것**이다.
"Private"이라는 이름이 오해를 부르는 지점.

- 대상: `storage.googleapis.com`, `bigquery.googleapis.com` 등 구글 관리형 API
- 설정 단위: 서브넷
- 비용: 무료

가장 단순하고, 대부분의 경우 이걸로 충분하다.

## PSA — 관리형 인스턴스가 옆집에

Cloud SQL에 사설 IP를 주는 게 이것이다. Memorystore, AlloyDB, Filestore도 같다.

동작 방식이 독특하다.

1. 내 VPC에서 IP 대역을 통째로 예약한다 (보통 `/16` 또는 `/20`)
2. 구글이 그 대역을 가져가 **자기 쪽 VPC**에 인스턴스를 만든다
3. 두 VPC를 **피어링**으로 연결한다
4. 내 VM에서 `10.x.x.x`로 접속

즉 DB가 내 VPC 안에 있는 게 아니라, **옆집에 있는데 다리가 놓인 상태**다.

### PSA의 약점: 피어링은 전이되지 않는다

여기가 실무에서 물리는 지점이다.

```
온프렘 ←(VPN)→ 내 VPC ←(피어링)→ Cloud SQL
```

이 그림에서 **온프렘 → Cloud SQL은 기본적으로 통하지 않는다.**
VPC 피어링은 한 다리만 건너기 때문이다.

뚫으려면 Cloud Router에서 커스텀 경로 광고를 설정해야 한다.

## PSC — 서비스를 우리 집 주소로

내 서브넷 대역에서 IP를 하나 뽑아 서비스에 붙인다.

```
10.0.1.50  →  BigQuery
10.0.1.51  →  서드파티 SaaS
```

VM 입장에서는 그냥 옆에 있는 내부 IP다. Cloud DNS 비공개 영역으로 이름도 붙일 수 있다.

PSC만 할 수 있는 것 세 가지:

1. **온프렘에서 접근** — VPN/Interconnect로 들어온 서버가 그 내부 IP를 그대로 쓴다
2. **서드파티 서비스 연결** — 다른 회사가 만든 서비스를 사설로 붙인다
3. **세밀한 통제** — 특정 서비스 하나만 연다 (PGA는 구글 API 전체가 열린다)

PSA에 있던 전이 문제가 없다. 그래서 신규 서비스는 PSC 방향으로 가는 추세다.

## 한 장 정리

| | PGA | PSA | PSC |
|---|---|---|---|
| 방식 | 경로만 사설 | VPC 피어링 | 엔드포인트 |
| IP 출처 | 구글 공개 IP | 예약 대역 | 내 서브넷 |
| 대상 | 구글 API 전체 | 관리형 인스턴스 | 특정 서비스 |
| 온프렘 | 불가 | 추가 설정 필요 | 가능 |
| 설정 | 서브넷 토글 | 대역 예약 | 엔드포인트 생성 |
| 비용 | 무료 | 무료(대역 소모) | 엔드포인트 + 처리량 |

한 줄로 줄이면 이렇다.

> **PGA는 경로를 사설로, PSC는 주소를 사설로, PSA는 관리형 DB를 옆집에.**

## 시험에서 구분하는 법

지문의 단어 하나가 답을 가른다.

| 지문 | 답 |
|---|---|
| 외부 IP 없는 VM이 GCS 접근 | PGA |
| **Cloud SQL 사설 IP** | PSA |
| **IP 대역을 예약** | PSA |
| **온프렘 서버**가 구글 서비스 접근 | PSC |
| **내부 IP 주소**로 접근 | PSC |
| 서드파티 SaaS 사설 연결 | PSC |

"온프렘"과 "내부 IP"가 PSC 신호다. "대역 예약"이 나오면 PSA다.

## 함께 정리해두면 좋은 것

셋 다 "외부 IP 없이"로 시작하지만, 목적지에 따라 다른 답이 나온다.

| 목적지 | 답 |
|---|---|
| 일반 인터넷 (apt, pip, 외부 API) | Cloud NAT |
| 구글 API, VPC 안에서 | PGA |
| 관리형 DB 인스턴스 | PSA |
| 구글 API, 온프렘에서 / 내부 IP 필요 | PSC |
| 외부 IP 없이 SSH | IAP |

이 표까지 묶어두니 그제야 머릿속에서 정리가 됐다.

## 마무리

개념을 이해하는 것과 설명할 수 있는 것은 다른 능력이다.
강의를 들을 때는 다 알아들었는데 며칠 뒤 빈 화이트보드 앞에서는 한 마디도 안 나왔다.

그래서 앞으로는 배운 것을 글로 옮겨보려고 한다. 안 써지는 부분이 곧 구멍이다.

---

**참고**

- [Private Google Access](https://cloud.google.com/vpc/docs/private-google-access)
- [Private services access](https://cloud.google.com/vpc/docs/private-services-access)
- [Private Service Connect](https://cloud.google.com/vpc/docs/private-service-connect)

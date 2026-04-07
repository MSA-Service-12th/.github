# 🚚 Loopang

> **허브 기반 분산 배송 시스템 — Hub-and-Spoke MSA**
>
> 단일 배송자가 가질 수밖에 없는 한계를 17개의 허브로 분산하여 해결한,
> Spring Boot 기반 멀티 클라우드 마이크로서비스 시스템입니다.

<p align="center">
  <a href="https://loopang.site"><img alt="Live" src="https://img.shields.io/badge/live-loopang.site-blue" /></a>
  <img alt="Java" src="https://img.shields.io/badge/Java-21-orange" />
  <img alt="Spring Boot" src="https://img.shields.io/badge/Spring%20Boot-3.5.13-green" />
  <img alt="Spring Cloud" src="https://img.shields.io/badge/Spring%20Cloud-2025.0.1-green" />
  <img alt="PostgreSQL" src="https://img.shields.io/badge/PostgreSQL-17-336791" />
  <img alt="Kafka" src="https://img.shields.io/badge/Apache%20Kafka-SSL-black" />
  <img alt="AWS" src="https://img.shields.io/badge/AWS-EC2%20%2B%20ALB-orange" />
  <img alt="GCP" src="https://img.shields.io/badge/GCP-Kafka-4285F4" />
</p>

## ✨ 핵심 특징

- **17개 허브 + Hub-and-Spoke 네트워크**로 분산된 배송 거점
- **다익스트라 + 실시간 혼잡도 가중**으로 막힌 허브를 자동 우회
- **Outbox/Inbox 패턴**으로 도메인 이벤트의 트랜잭션 안전성 보장
- **AWS + GCP 멀티 클라우드**로 단일 장애점 격리
- **Spring Cloud 기반 9개 마이크로서비스**

---

## 🏗️ 시스템 아키텍처

<!-- ![architecture](docs/images/architecture.png) -->

```
                         Internet
                            │
                            ▼
                  Route53 (loopang.site)
                            │
                            ▼
                ALB (HTTPS:443, ACM)
                            │
                            ▼
            Spring Cloud Gateway (EC2 ASG)
              │  Keycloak JWT 검증 + Eureka 라우팅
              ▼
      ┌───────────────────────────────────┐
      │        9개 마이크로서비스          │
      │  user / hub / route / order /     │
      │  delivery / company / item /      │
      │  message / (gateway)              │
      └───────────────────────────────────┘
              │                  │
              ▼                  ▼
       PostgreSQL 17        Apache Kafka (GCP, SSL)
       (서비스별 DB)         Outbox/Inbox 패턴
```

**왜 멀티 클라우드인가?**
- AWS는 EC2/ALB/EBS 등 컴퓨팅 리소스에 강점
- GCP의 Kafka 클러스터를 분리하여 **AWS 단일 장애로부터 메시지 큐 격리**
- 서로 다른 클라우드 사이는 SSL/TLS로 암호화 통신

---

## 🧱 마이크로서비스 구성

| 서비스 | 도메인 | 포트 | 저장소 | 주요 이벤트 |
|---|---|---|---|---|
| [**gateway**](https://github.com/MSA-Service-12th/gateway) | API Gateway, JWT 인증, 라우팅 | 18080 | — | — |
| [**user-service**](https://github.com/MSA-Service-12th/user-service) | 사용자, 인증/인가, 배송담당자 | 8080 | PostgreSQL | `user-update-topic` 발행 |
| [**hub-service**](https://github.com/MSA-Service-12th/hub-service) | 허브, 허브재고 | 18100 | PostgreSQL | `hub-update-topic`, `dev-hub-stock-updated` 발행 / `dev-order-pending` 구독 |
| [**route-service**](https://github.com/MSA-Service-12th/route-service) | 허브 간 경로 계산 (다익스트라) | 18105 | PostgreSQL | — |
| [**order-service**](https://github.com/MSA-Service-12th/order-service) | 주문 | 18085 | PostgreSQL | `dev-order-pending`, `dev-order-accepted` 발행 / `dev-hub-stock-updated`, `dev-delivery-*` 구독 |
| [**delivery-service**](https://github.com/MSA-Service-12th/delivery-service) | 배송 | 18099 | PostgreSQL | `dev-delivery-created`, `dev-delivery-status-updated` 발행 / `dev-order-accepted` 구독 |
| [**company-service**](https://github.com/MSA-Service-12th/company-service) | 업체 | 18102 | PostgreSQL | `hub-update-topic` 구독 |
| [**item-service**](https://github.com/MSA-Service-12th/item-service) | 상품 | 8081 | PostgreSQL | — |
| [**message-service**](https://github.com/MSA-Service-12th/message-service) | Slack/Gemini AI 메시지 발송 | 8083 | PostgreSQL | `dev-delivery-message-created` 구독 |

### 인프라 모듈
| 모듈 | 역할 |
|---|---|
| [**common**](https://github.com/MSA-Service-12th/common) | 공통 라이브러리 (`com.loopang:common`) — Outbox/Inbox, BaseEntity, GlobalExceptionAdvice, MDC 트레이싱, Feign 헤더 전파 |
| [**eureka-server**](https://github.com/MSA-Service-12th/eureka-server) | 서비스 디스커버리 |
| [**config-server**](https://github.com/MSA-Service-12th/config-server) | 중앙 설정 관리 (Spring Cloud Config + Git 백엔드) |
| [**configs**](https://github.com/MSA-Service-12th/configs) | 환경별 설정 Git 저장소 |

---

## ⚙️ 기술 스택

### Backend
- **Java 21**, **Spring Boot 3.5.13**, **Spring Cloud 2025.0.1**
- **Spring MVC** (도메인 서비스), **Spring WebFlux** (Gateway)
- **JPA + QueryDSL**, **PostgreSQL 17**
- **Lombok**, Gradle 멀티 모듈

### Messaging & Async
- **Apache Kafka** (SSL/TLS), **Outbox/Inbox 패턴**
- 트랜잭션 ↔ 메시지 발행 원자성 보장
- `@IdempotentConsumer` 어노테이션으로 멱등성 자동 처리
- DLT(Dead Letter Topic) 자동 재시도

### Auth & Security
- **Keycloak** (OAuth2 / OIDC, JWT)
- Realm Roles 기반 권한 (`MASTER` / `HUB` / `DELIVERY` / `COMPANY`)
- Gateway 단일 진입점에서 JWT 검증 → 도메인 서비스로 `X-User-*` 헤더 전파

### Service Mesh
- **Spring Cloud Eureka** — 서비스 디스커버리
- **Spring Cloud Config Server** — 중앙 설정 (Git 백엔드)
- **Spring Cloud OpenFeign** — 동기 서비스 간 통신
- **Resilience4J** — 서킷 브레이커, 재시도, 벌크헤드, 레이트 리미터

### External APIs
- **Kakao Mobility Directions** — 업체 → 허브 last-mile 거리
- **TMap API** — 허브 ↔ 허브 실거리/시간
- **Google Gemini** — AI 메시지 생성
- **Slack** — 알림 발송

### Cloud & DevOps
- **AWS** — EC2 + Auto Scaling Group, ALB, ACM, Route53, ECR
- **GCP** — Kafka 클러스터 (3 broker, SSL/TLS)
- **GitHub Actions** — CI/CD 자동화
- **Blue/Green 배포** — ASG Instance Refresh 기반 무중단 배포
- **CloudFormation** — 인프라 IaC (게이트웨이)

---

## 🧮 핵심 알고리즘 — 혼잡도 가중 다익스트라

route-service는 단순 최단 거리가 아닌 **현실적으로 가장 빠른 경로**를 계산합니다.

```
cost(edge) = distance × (1 + 혼잡도)
```

| 허브 상태 | 혼잡도 | 실효 가중치 |
|---|---|---|
| 정상 | 0.0 | × 1.0 |
| 혼잡 (BUSY) | 0.8 | **× 1.8** |
| 만원 (FULL) | 2.0 | **× 3.0** |

- **PriorityQueue 기반 다익스트라**로 최소 비용 경로 탐색
- 가중치는 **경로 선택용**, 응답으로 나가는 거리/시간은 **원본 값** (사용자에게 정직한 정보)
- 허브 상태는 호출 단위 캐싱으로 중복 조회 차단

### 외부 API 3중 안전장치
| 구간 | 1차 | Fallback |
|---|---|---|
| 허브 ↔ 허브 | TMap API (1시간 캐싱) | Haversine × 1.3 |
| 업체 → 허브 | Kakao Mobility | Haversine × 1.3 |

→ 외부 API가 모두 죽어도 라우팅은 절대 멈추지 않습니다.

---

## 📡 이벤트 흐름 — Outbox / Inbox 패턴

### 발행 (Outbox)
```
@Transactional
public void createOrder(...) {
    Order order = orderRepository.save(...);
    Events.trigger(OutboxEvent.of(order));  // ← 한 줄
}
```

1. 도메인 서비스가 `Events.trigger()` 호출
2. 같은 트랜잭션 안에서 `p_outbox` 테이블에 PENDING 저장
3. 트랜잭션 커밋 후 `OutboxEventListener`가 Kafka로 발행
4. 성공 시 PROCESSED, 실패 시 FAILED + retry (최대 3회) → DLT

### 구독 (Inbox)
```
@KafkaListener(topics = "...")
@IdempotentConsumer(value = "order-accepted-group")
public void handle(ConsumerRecord<String, String> record) { ... }
```

1. `InboxAdvice`(AOP)가 `message_id` 헤더 추출
2. `p_inbox`에 저장 시도 → unique 위반 = 중복 메시지 → 스킵
3. 처리 성공 시 커밋, 실패 시 롤백
4. `InboxCleanupScheduler`가 매일 03:00 7일 이전 메시지 정리

### 전체 이벤트 토폴로지

```
order ──[dev-order-pending]──▶ hub
     ──[dev-order-accepted]──▶ delivery ──[dev-delivery-message-created]──▶ message
hub  ──[dev-hub-stock-updated]──▶ order
hub  ──[hub-update-topic]──▶ company
user ──[user-update-topic]──▶ company
```

---

## 🌐 라이브 데모

| 항목 | 링크 |
|---|---|
| 🌍 라이브 서비스 | https://loopang.site |
| 🎤 발표 자료 (Canva) | https://canva.link/hr2enth7mtwv4ge |
| 📥 소스코드 (Drive) | https://drive.google.com/drive/folders/1IHgI75XTCaiEMEdQ-G3TsfzHIDrIQflv |

---

## 🚀 로컬 실행 가이드

### Prerequisites
- Java 21
- Docker (PostgreSQL 컨테이너용)
- IntelliJ IDEA
- GCP Kafka 접속용 truststore 파일 (팀 슬랙에서 수령)

### Step 1 — 인프라 서비스 기동
```bash
# Eureka Server (port 18761)
# Config Server (port 18888)
```

### Step 2 — Docker PostgreSQL
```bash
# 서비스별 컨테이너 또는 단일 컨테이너 + 다중 DB
docker run -d --name postgres-shared \
  -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres \
  -p 5439:5432 postgres:17
```

### Step 3 — 각 서비스의 `.env` 파일 작성
모듈 root에 `.env.example`을 복사하여 `.env` 생성:
```
DB_URL=localhost:5439/<db_name>
BOOTSTRAP_SERVERS=<GCP Kafka>
TRUST_STORE_PASSWORD=<truststore 비번>
...
```

### Step 4 — IntelliJ Run Configuration
각 서비스의 Run/Debug Configuration에서:
- **Modify options → Working directory**
- 값: `$MODULE_WORKING_DIR$` 또는 모듈 절대경로

### Step 5 — Gateway 먼저, 그 다음 도메인 서비스 순으로 실행
```
1. eureka-server (18761)
2. config-server (18888)
3. gateway (18080)
4. hub / user / route / order / delivery / company / item / message
```

### Step 6 — Eureka 등록 확인
http://localhost:18761 에서 9개 서비스 등록 확인

---

## 📂 프로젝트 구조

```
loopang/
├── eureka-server/        # 서비스 디스커버리
├── config-server/        # 중앙 설정 관리
├── configs/              # 환경별 설정 (Git 저장소)
├── common/               # 공통 라이브러리 (com.loopang:common)
├── gateway/              # API Gateway (Spring Cloud Gateway, AWS 배포)
├── user-service/         # 사용자, 인증/인가
├── hub-service/          # 허브, 재고
├── route-service/        # 경로 계산 (다익스트라)
├── order-service/        # 주문
├── delivery-service/     # 배송
├── company-service/      # 업체
├── item-service/         # 상품
└── message-service/      # Slack / AI 메시지
```

---

## 👥 팀 — 스파르타 12조

| 이름 | 담당 |
|---|---|
| **이현규** | `common` 라이브러리 · `gateway` AWS 배포 · `user-service` · `hub-service` · `route-service` |
| **이현빈** | GCP **Kafka 클러스터 구축** · `order-service` · `delivery-service` 구현 |
| **이예원** | `item-service` · `message-service` (+ Gemini AI) · `config-server` · `common` 리팩터링 |
| **신단비** | `eureka-server` · `company-service` |
| **문연희** | `delivery-service` 설계 |
| 성명규 *(중도 이탈)* | `user-service` 설계 |

---

## 📜 License

본 프로젝트는 스파르타 코딩클럽 백엔드 트랙 12조의 학습용 프로젝트입니다.

---

<p align="center">
  <strong>Loopang — 스파르타 12조</strong>
</p>

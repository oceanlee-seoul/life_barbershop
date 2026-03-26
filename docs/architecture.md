# 시스템 아키텍처

> 최초 작성: 2026-03-26 | 엔지니어링 리뷰 기반

## 프로젝트 개요

네이버 예약(파트너 API)을 통해 바버샵 예약 내역을 조회·관리하는 내부 웹 도구.
주요 기능: 일별/달력/타임테이블 예약 조회, 예약 메모, 일별·월별 매출 통계.

---

## 기술 스택

| 영역 | 기술 |
|------|------|
| Frontend | Next.js (App Router) |
| Backend | Spring Boot (JPA, Spring Security, @Scheduled) |
| DB | PostgreSQL |
| 인증 | JWT (아이디 + 패스워드) |
| 인프라 | Oracle Cloud ARM VM (무료) + Docker Compose |
| 리버스 프록시 | Nginx + Let's Encrypt (Certbot) |

---

## 시스템 다이어그램

```
┌─────────────────────────────────────────────────────────────┐
│           Oracle Cloud ARM VM (4 OCPU, 24GB, 무료)          │
│                                                             │
│  ┌──────────┐   ┌────────────────┐   ┌──────────────────┐  │
│  │  Nginx   │   │   Next.js      │   │  Spring Boot     │  │
│  │ :80/:443 │──▶│  App Router    │──▶│  :8080           │  │
│  │ (HTTPS,  │   │  :3000         │   │                  │  │
│  │ 리버스   │   │                │   │  - JWT 인증      │  │
│  │ 프록시)  │   │  - 달력 뷰     │   │  - 예약 API      │  │
│  └──────────┘   │  - 타임테이블  │   │  - 메모 CRUD     │  │
│       ▲         │  - 매출 통계   │   │  - 매출 집계     │  │
│       │         └────────────────┘   │  - @Scheduled    │  │
│    [Internet]           │            │    (5분 동기화)  │  │
│                         └────────────┤                  │  │
│                                      └────────┬─────────┘  │
│                                               │             │
│                                      ┌────────▼─────────┐  │
│                                      │   PostgreSQL      │  │
│                                      │   :5432           │  │
│                                      │                   │  │
│                                      │  reservations     │  │
│                                      │  memos            │  │
│                                      │  sync_logs        │  │
│                                      └───────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                                ▲
                                │ 파트너 API
                      ┌─────────┴──────────┐
                      │   네이버 예약 API   │
                      └────────────────────┘
```

---

## 데이터 동기화 플로우

```
┌──────────────────────────────────────────────────────────────┐
│  트리거: Spring @Scheduled (5분 주기) 또는 수동 POST /api/sync │
│                                                              │
│  1. 네이버 파트너 API 호출 → 예약 목록 조회                  │
│  2. DB 기존 데이터와 diff                                    │
│     ├─ 신규 예약 → INSERT                                    │
│     ├─ 변경된 예약 → UPDATE                                  │
│     └─ 취소된 예약 → 상태 업데이트 (DELETE 아님)             │
│  3. sync_logs에 기록 (시각, 동기화 건수, 오류 여부)          │
│  4. 메모는 예약 ID 기준 로컬 DB에만 존재 → 동기화 시 보존    │
└──────────────────────────────────────────────────────────────┘
```

---

## DB 스키마 (초안)

```sql
-- 네이버에서 동기화된 예약 데이터
CREATE TABLE reservations (
    id              BIGSERIAL PRIMARY KEY,
    naver_id        VARCHAR(100) UNIQUE NOT NULL,  -- 네이버 예약 고유 ID
    customer_name   VARCHAR(100),
    customer_phone  VARCHAR(20),
    service_name    VARCHAR(200),
    reserved_at     TIMESTAMP NOT NULL,
    duration_min    INT,
    price           INT,
    status          VARCHAR(50),                   -- confirmed / cancelled / completed
    raw_data        JSONB,                         -- 원본 API 응답 전체 보관
    synced_at       TIMESTAMP NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);

-- 예약에 남기는 메모 (로컬 전용)
CREATE TABLE memos (
    id              BIGSERIAL PRIMARY KEY,
    reservation_id  BIGINT NOT NULL REFERENCES reservations(id),
    content         TEXT NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);

-- 동기화 이력
CREATE TABLE sync_logs (
    id              BIGSERIAL PRIMARY KEY,
    synced_at       TIMESTAMP NOT NULL,
    fetched_count   INT,
    inserted_count  INT,
    updated_count   INT,
    error_message   TEXT,
    duration_ms     INT
);
```

---

## 코드 구조

### Backend (Spring Boot)

```
backend/
└── src/main/java/com/lifebarbershop/
    ├── config/
    │   ├── SecurityConfig.java
    │   ├── JwtConfig.java
    │   └── SchedulerConfig.java
    ├── controller/
    │   ├── AuthController.java
    │   ├── ReservationController.java
    │   ├── MemoController.java
    │   ├── StatsController.java
    │   └── SyncController.java
    ├── service/
    │   ├── NaverApiService.java      # 네이버 파트너 API 연동 (인터페이스)
    │   ├── SyncService.java          # 동기화 로직
    │   ├── ReservationService.java
    │   ├── MemoService.java
    │   └── StatsService.java
    ├── repository/
    │   ├── ReservationRepository.java
    │   ├── MemoRepository.java
    │   └── SyncLogRepository.java
    ├── domain/
    │   ├── Reservation.java
    │   ├── Memo.java
    │   └── SyncLog.java
    ├── dto/
    │   ├── request/
    │   └── response/
    └── exception/
        └── GlobalExceptionHandler.java
```

### Frontend (Next.js App Router)

```
frontend/
└── app/
    ├── (auth)/
    │   └── login/page.tsx
    ├── (dashboard)/
    │   ├── layout.tsx              # 공통 레이아웃 + 인증 체크
    │   ├── page.tsx                # 오늘 예약 대시보드
    │   ├── calendar/page.tsx       # 달력 뷰
    │   ├── timetable/page.tsx      # 타임테이블 뷰
    │   └── stats/page.tsx          # 매출 통계
    └── components/
        ├── ReservationCard.tsx     # 예약 카드 (메모 포함)
        ├── CalendarGrid.tsx
        ├── TimeTable.tsx
        ├── MemoForm.tsx
        └── StatsChart.tsx
```

### Infra (Docker Compose)

```
infra/
├── docker-compose.yml
├── docker-compose.prod.yml
├── nginx/
│   ├── nginx.conf
│   └── certbot/                   # Let's Encrypt 인증서
└── Makefile                       # deploy, logs, restart 등 단축 명령어
```

---

## 아키텍처 결정사항 (ADR)

| # | 결정 | 이유 |
|---|------|------|
| 1 | DB 동기화 방식 채택 | 실시간 프록시 대비 조회 속도·안정성·집계 편의성 압도적으로 유리 |
| 2 | JWT 인증 | 단일 오너 내부 도구에 OAuth는 과잉. 간단하고 충분함 |
| 3 | Oracle Cloud ARM VM 1개 | AMD 1GB VM은 Spring Boot + 2개 컨테이너에 메모리 부족. ARM 24GB는 무료로 여유로움 |
| 4 | Docker Compose 수동 배포 | CI/CD 불필요. SSH + Makefile로 충분 |
| 5 | Next.js App Router | 신규 프로젝트에 Pages Router는 비추천. Server Components 활용 가능 |
| 6 | Nginx + Let's Encrypt 필수 | JWT 토큰을 HTTP로 전송 시 탈취 위험. HTTPS 필수 |

---

## 보안 체크리스트

- [ ] JWT Secret은 환경변수로 관리 (`.env`, `.gitignore`에 추가 필수)
- [ ] 네이버 파트너 API 키는 환경변수로 관리
- [ ] DB 패스워드는 환경변수로 관리
- [ ] Next.js에서 JWT는 `httpOnly` 쿠키로 저장 (localStorage 금지 — XSS 위험)
- [ ] Nginx에서 HTTPS 강제 리다이렉트 설정
- [ ] PostgreSQL은 외부 포트 노출 금지 (Docker 내부 네트워크만)

---

## 수동 배포 플로우

```bash
# Makefile 예시
deploy:
    ssh oracle-vm 'cd ~/life_barbershop && git pull && docker compose -f infra/docker-compose.prod.yml up -d --build'

logs:
    ssh oracle-vm 'docker compose -f ~/life_barbershop/infra/docker-compose.prod.yml logs -f'

restart:
    ssh oracle-vm 'docker compose -f ~/life_barbershop/infra/docker-compose.prod.yml restart'
```

---

## 향후 확장 고려사항

- **헤어제품 재고 관리**: 별도 `products`, `inventory` 테이블 추가로 확장 가능
- **푸시 알림**: 예약 변경 시 알림 (FCM 또는 카카오 알림톡)
- **PostgreSQL 백업**: `pg_dump` 크론잡 + Oracle Object Storage (무료 10GB)

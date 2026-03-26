# 구현 체크리스트

> 최초 작성: 2026-03-26
> 전략: 인프라 먼저 → 앱 스켈레톤 → API 연동

---

## 전체 진행 순서

```
Phase 1. 인프라 구축          ← 현재 단계
Phase 2. 앱 스켈레톤 & 배포
Phase 3. 기능 구현 (API 제외)
Phase 4. 네이버 API 연동
Phase 5. QA & 마무리
```

---

## Phase 1. 인프라 구축

### 1-1. Oracle Cloud ARM VM 생성

- [ ] Oracle Cloud 계정 생성 (Always Free 선택)
- [ ] Compute → Instances → Create Instance
  - Shape: `VM.Standard.A1.Flex`
  - OCPU: `4`, Memory: `24GB` (Always Free 한도 내)
  - OS: `Ubuntu 22.04 (aarch64)`
  - SSH 키 페어 생성 및 `.pem` 파일 로컬 저장
- [ ] 인스턴스 Public IP 확인 및 기록
- [ ] (선택) 고정 IP 설정: Networking → Reserved Public IPs → 할당

### 1-2. 도메인 & DNS 설정

- [ ] 도메인 준비 (보유 도메인 사용 또는 구매)
  - 무료 대안: [freedns.afraid.org](https://freedns.afraid.org) 또는 Cloudflare Free
- [ ] DNS A 레코드 추가: `도메인` → Oracle VM Public IP
- [ ] DNS 전파 확인: `nslookup 도메인` 또는 [dnschecker.org](https://dnschecker.org)

> 도메인이 없으면 IP 직접 접속도 가능하나 HTTPS(Let's Encrypt) 발급 불가.
> 이 경우 self-signed 인증서 또는 Cloudflare Tunnel 사용.

### 1-3. 서버 기본 설정

SSH 접속:
```bash
ssh -i ~/.ssh/oracle.pem ubuntu@<VM_PUBLIC_IP>
```

- [ ] 패키지 업데이트
  ```bash
  sudo apt update && sudo apt upgrade -y
  ```
- [ ] 타임존 설정
  ```bash
  sudo timedatectl set-timezone Asia/Seoul
  ```
- [ ] swap 설정 (메모리 여유 있지만 안전망용)
  ```bash
  sudo fallocate -l 2G /swapfile
  sudo chmod 600 /swapfile
  sudo mkswap /swapfile
  sudo swapon /swapfile
  echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
  ```
- [ ] 기본 패키지 설치
  ```bash
  sudo apt install -y git curl wget unzip make
  ```

### 1-4. 방화벽(Security List) 설정

Oracle Cloud는 두 곳을 모두 열어야 함.

**① Oracle Cloud 콘솔 — Security List:**
- [ ] Networking → Virtual Cloud Networks → Security Lists
  - Ingress Rule 추가:
    | 포트 | 프로토콜 | 용도 |
    |------|----------|------|
    | 22   | TCP | SSH |
    | 80   | TCP | HTTP (Let's Encrypt 인증용) |
    | 443  | TCP | HTTPS |

**② VM 내부 — iptables/ufw:**
- [ ] UFW 설정
  ```bash
  sudo ufw allow 22
  sudo ufw allow 80
  sudo ufw allow 443
  sudo ufw enable
  sudo ufw status
  ```
- [ ] Oracle 기본 iptables 규칙 확인 (Ubuntu 22.04는 기본 REJECT 규칙 있음)
  ```bash
  sudo iptables -L INPUT --line-numbers
  # REJECT 규칙이 있으면 80/443 허용 규칙을 그 앞에 삽입
  sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT
  sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT
  # 재부팅 후 유지
  sudo apt install -y iptables-persistent
  sudo netfilter-persistent save
  ```

### 1-5. Docker & Docker Compose 설치

- [ ] Docker 설치 (공식 스크립트)
  ```bash
  curl -fsSL https://get.docker.com | sh
  sudo usermod -aG docker ubuntu
  newgrp docker
  ```
- [ ] Docker 버전 확인
  ```bash
  docker --version
  docker compose version
  ```
- [ ] Docker 부팅 시 자동 시작
  ```bash
  sudo systemctl enable docker
  sudo systemctl start docker
  ```

### 1-6. 프로젝트 디렉토리 & 환경변수 설정

- [ ] 프로젝트 클론
  ```bash
  git clone <REPO_URL> ~/life_barbershop
  cd ~/life_barbershop
  ```
- [ ] `.env` 파일 생성 (절대 git에 커밋 금지)
  ```bash
  cp infra/.env.example infra/.env
  vi infra/.env
  ```
  ```env
  # DB
  POSTGRES_DB=barbershop
  POSTGRES_USER=barbershop
  POSTGRES_PASSWORD=<강력한_패스워드>

  # JWT
  JWT_SECRET=<최소_32자_랜덤_문자열>
  JWT_EXPIRATION_MS=86400000

  # 네이버 API (Phase 4에서 채움)
  NAVER_API_KEY=
  NAVER_API_SECRET=

  # 도메인
  DOMAIN=yourdomain.com
  ```
- [ ] `.env`를 `.gitignore`에 추가
  ```bash
  echo 'infra/.env' >> .gitignore
  ```

### 1-7. PostgreSQL 컨테이너 단독 기동 테스트

- [ ] `infra/docker-compose.db.yml` 작성 (DB만 먼저 테스트)
  ```yaml
  services:
    db:
      image: postgres:16-alpine
      container_name: barbershop_db
      restart: unless-stopped
      environment:
        POSTGRES_DB: ${POSTGRES_DB}
        POSTGRES_USER: ${POSTGRES_USER}
        POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      volumes:
        - postgres_data:/var/lib/postgresql/data
      networks:
        - barbershop_net

  volumes:
    postgres_data:

  networks:
    barbershop_net:
      driver: bridge
  ```
- [ ] DB 컨테이너 기동
  ```bash
  cd infra
  docker compose -f docker-compose.db.yml --env-file .env up -d
  docker compose -f docker-compose.db.yml logs db
  ```
- [ ] DB 접속 확인
  ```bash
  docker exec -it barbershop_db psql -U barbershop -d barbershop -c '\l'
  ```

### 1-8. Nginx 설정

- [ ] `infra/nginx/nginx.conf` 작성
  ```nginx
  events {}

  http {
    server {
      listen 80;
      server_name yourdomain.com;

      # Let's Encrypt 인증용 (초기에만)
      location /.well-known/acme-challenge/ {
        root /var/www/certbot;
      }

      # 이후 HTTPS 리다이렉트로 교체
      location / {
        return 301 https://$host$request_uri;
      }
    }

    server {
      listen 443 ssl;
      server_name yourdomain.com;

      ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

      # Frontend (Next.js)
      location / {
        proxy_pass         http://frontend:3000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection 'upgrade';
        proxy_set_header   Host $host;
        proxy_cache_bypass $http_upgrade;
      }

      # Backend API
      location /api/ {
        proxy_pass         http://backend:8080;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
      }
    }
  }
  ```

### 1-9. Let's Encrypt SSL 인증서 발급

- [ ] Certbot으로 인증서 발급 (Docker 사용)
  ```bash
  # 먼저 Nginx 컨테이너를 HTTP 모드(80만)로 기동
  docker run -d --name nginx_temp \
    -p 80:80 \
    -v $(pwd)/nginx/nginx.conf:/etc/nginx/nginx.conf \
    -v $(pwd)/certbot/www:/var/www/certbot \
    nginx:alpine

  # Certbot 실행
  docker run --rm \
    -v $(pwd)/certbot/conf:/etc/letsencrypt \
    -v $(pwd)/certbot/www:/var/www/certbot \
    certbot/certbot certonly --webroot \
    --webroot-path=/var/www/certbot \
    -d yourdomain.com \
    --email your@email.com \
    --agree-tos --non-interactive

  docker rm -f nginx_temp
  ```
- [ ] 인증서 발급 확인
  ```bash
  ls infra/certbot/conf/live/yourdomain.com/
  # fullchain.pem, privkey.pem 확인
  ```
- [ ] 인증서 자동 갱신 크론 등록 (90일마다 갱신 필요)
  ```bash
  # crontab -e
  0 3 * * * cd ~/life_barbershop/infra && docker run --rm \
    -v $(pwd)/certbot/conf:/etc/letsencrypt \
    -v $(pwd)/certbot/www:/var/www/certbot \
    certbot/certbot renew --quiet && \
    docker compose -f docker-compose.prod.yml restart nginx
  ```

### 1-10. 전체 Docker Compose (프로덕션) 작성

- [ ] `infra/docker-compose.prod.yml` 작성
  ```yaml
  services:
    nginx:
      image: nginx:alpine
      container_name: barbershop_nginx
      restart: unless-stopped
      ports:
        - "80:80"
        - "443:443"
      volumes:
        - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
        - ./certbot/conf:/etc/letsencrypt:ro
        - ./certbot/www:/var/www/certbot:ro
      depends_on:
        - frontend
        - backend
      networks:
        - barbershop_net

    frontend:
      build:
        context: ../frontend
        dockerfile: Dockerfile
      container_name: barbershop_frontend
      restart: unless-stopped
      environment:
        - NEXT_PUBLIC_API_URL=/api
      networks:
        - barbershop_net

    backend:
      build:
        context: ../backend
        dockerfile: Dockerfile
      container_name: barbershop_backend
      restart: unless-stopped
      environment:
        - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/${POSTGRES_DB}
        - SPRING_DATASOURCE_USERNAME=${POSTGRES_USER}
        - SPRING_DATASOURCE_PASSWORD=${POSTGRES_PASSWORD}
        - JWT_SECRET=${JWT_SECRET}
        - JWT_EXPIRATION_MS=${JWT_EXPIRATION_MS}
      depends_on:
        - db
      networks:
        - barbershop_net

    db:
      image: postgres:16-alpine
      container_name: barbershop_db
      restart: unless-stopped
      environment:
        POSTGRES_DB: ${POSTGRES_DB}
        POSTGRES_USER: ${POSTGRES_USER}
        POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      volumes:
        - postgres_data:/var/lib/postgresql/data
      networks:
        - barbershop_net
      # 외부 포트 노출 금지 (내부 네트워크만)

  volumes:
    postgres_data:

  networks:
    barbershop_net:
      driver: bridge
  ```
- [ ] `infra/.env.example` 작성 (빈 값으로 git 커밋)
  ```env
  POSTGRES_DB=barbershop
  POSTGRES_USER=barbershop
  POSTGRES_PASSWORD=
  JWT_SECRET=
  JWT_EXPIRATION_MS=86400000
  NAVER_API_KEY=
  NAVER_API_SECRET=
  DOMAIN=
  ```

### 1-11. Makefile 작성

- [ ] `infra/Makefile` 작성
  ```makefile
  COMPOSE = docker compose -f docker-compose.prod.yml --env-file .env

  deploy:
  	$(COMPOSE) up -d --build

  stop:
  	$(COMPOSE) down

  restart:
  	$(COMPOSE) restart

  logs:
  	$(COMPOSE) logs -f

  logs-backend:
  	$(COMPOSE) logs -f backend

  logs-frontend:
  	$(COMPOSE) logs -f frontend

  ps:
  	$(COMPOSE) ps

  db-shell:
  	docker exec -it barbershop_db psql -U $${POSTGRES_USER} -d $${POSTGRES_DB}

  db-backup:
  	docker exec barbershop_db pg_dump -U $${POSTGRES_USER} $${POSTGRES_DB} \
  	  > backups/backup_$$(date +%Y%m%d_%H%M%S).sql

  clean:
  	$(COMPOSE) down -v
  ```
- [ ] `infra/backups/` 디렉토리 생성 및 `.gitignore`에 백업 파일 제외
  ```
  infra/backups/*.sql
  ```

### 1-12. 로컬 개발 Docker Compose 작성

- [ ] `infra/docker-compose.dev.yml` 작성 (로컬에서 DB만 띄울 때)
  ```yaml
  services:
    db:
      image: postgres:16-alpine
      container_name: barbershop_db_dev
      ports:
        - "5432:5432"    # 로컬에서 직접 접속 가능
      environment:
        POSTGRES_DB: barbershop_dev
        POSTGRES_USER: barbershop
        POSTGRES_PASSWORD: devpassword
      volumes:
        - postgres_dev_data:/var/lib/postgresql/data

  volumes:
    postgres_dev_data:
  ```

### 1-13. 인프라 구축 최종 검증

- [ ] VM에서 `docker ps` → 4개 컨테이너(nginx, frontend, backend, db) 실행 확인
- [ ] `https://yourdomain.com` 브라우저 접속 확인
- [ ] HTTPS 인증서 유효 확인 (자물쇠 아이콘)
- [ ] `/api/health` 엔드포인트 응답 확인 (Backend 헬스체크)
- [ ] DB 접속 확인 (`make db-shell`)
- [ ] 재부팅 후 컨테이너 자동 복구 확인
  ```bash
  sudo reboot
  # 1분 후 SSH 재접속
  docker ps  # 모두 Up 상태인지 확인
  ```

---

## Phase 2. 앱 스켈레톤 & 배포

> 빈 껍데기 앱을 만들어서 인프라 위에 올리는 단계.
> 실제 기능 없이 "배포 파이프라인이 작동하는가"를 먼저 검증.

- [ ] **Backend**: Spring Boot 프로젝트 초기화
  - Spring Initializr: Web, JPA, PostgreSQL, Security, Lombok, Validation
  - `GET /api/health` → `{"status": "ok"}` 응답하는 헬스체크 엔드포인트만 구현
  - `Dockerfile` 작성 (멀티스테이지 빌드)

- [ ] **Frontend**: Next.js 프로젝트 초기화
  - `npx create-next-app@latest frontend --typescript --tailwind --app`
  - 기본 랜딩 페이지만 있는 상태로 배포
  - `Dockerfile` 작성

- [ ] **배포 테스트**: `make deploy` 한 번에 전체 스택 기동 확인

- [ ] **로컬 ↔ 서버 배포 스크립트** 작성
  ```bash
  # 루트 Makefile (로컬에서 실행)
  deploy:
  	ssh ubuntu@<VM_IP> 'cd ~/life_barbershop && git pull && cd infra && make deploy'
  ```

---

## Phase 3. 기능 구현 (API 연동 제외)

- [ ] JWT 로그인 구현 (Backend + Frontend)
- [ ] 예약 CRUD (Mock 데이터로 UI 먼저 완성)
- [ ] 달력 뷰 컴포넌트
- [ ] 타임테이블 뷰 컴포넌트
- [ ] 메모 CRUD
- [ ] 일별·월별 매출 통계 페이지

---

## Phase 4. 네이버 API 연동

- [ ] `NaverApiService` 구현 (파트너 API 스펙 기반)
- [ ] `SyncService` 구현 (@Scheduled 5분 주기)
- [ ] 수동 동기화 엔드포인트 (`POST /api/sync`)
- [ ] Mock 데이터 → 실제 API 데이터로 교체
- [ ] 동기화 로그 (`sync_logs`) 확인 UI

---

## Phase 5. QA & 마무리

- [ ] `/qa` 실행 — 전체 기능 QA 테스트
- [ ] 보안 체크리스트 최종 확인 (`docs/architecture.md` 참고)
- [ ] PostgreSQL 백업 크론 설정 (`make db-backup` 자동화)
- [ ] `docs/architecture.md` 최신화
- [ ] `README.md` 배포 방법 업데이트

---

## 참고 문서

- [시스템 아키텍처](./architecture.md)
- [gstack 스킬 목록](./gstack-skills.md)

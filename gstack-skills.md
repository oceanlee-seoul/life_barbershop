# gstack 기능 정리

| 스킬 | 설명 |
|------|------|
| `/browse` | 헤드리스 브라우저로 웹 페이지를 탐색·테스트하고 스크린샷을 찍음 (모든 웹 브라우징에 사용) |
| `/qa` | 웹 앱을 체계적으로 테스트하고 발견된 버그를 직접 수정까지 함 |
| `/qa-only` | 웹 앱을 테스트해 버그 리포트만 작성 (코드 수정 없음) |
| `/review` | PR 머지 전 코드 리뷰 — SQL 안전성, LLM 신뢰 경계 등 구조적 문제 분석 |
| `/ship` | 베이스 브랜치 병합 → 테스트 → 버전 범프 → PR 생성까지 배포 전 과정 자동화 |
| `/land-and-deploy` | PR 머지 후 CI/배포 완료 대기 및 프로덕션 상태 자동 검증 |
| `/canary` | 배포 후 라이브 앱을 지속 모니터링하며 오류·성능 이상을 감지 |
| `/benchmark` | 페이지 로드·Web Vitals·번들 크기 등 성능 회귀를 감지하고 추적 |
| `/design-review` | 실제 사이트를 시각적으로 감사해 디자인 문제를 찾고 코드까지 수정 |
| `/design-consultation` | 디자인 시스템(색상·폰트·레이아웃 등)을 설계하고 `DESIGN.md` 생성 |
| `/office-hours` | YC 스타일 질문으로 아이디어를 검증하거나 사이드 프로젝트를 브레인스토밍 |
| `/autoplan` | CEO·디자인·엔지니어링 리뷰를 자동으로 순차 실행해 완성된 플랜을 출력 |
| `/plan-ceo-review` | CEO 관점에서 플랜의 범위와 전략을 재검토하고 도전적 질문으로 개선 |
| `/plan-eng-review` | 엔지니어링 관점에서 아키텍처·데이터 흐름·엣지 케이스·테스트 커버리지를 검토 |
| `/plan-design-review` | 디자이너 관점에서 플랜의 각 설계 항목을 0~10점으로 평가하고 개선 |
| `/investigate` | 버그의 근본 원인을 체계적으로 분석하고 수정 (근본 원인 없이 패치 금지) |
| `/retro` | 커밋 이력과 작업 패턴을 분석해 주간 엔지니어링 회고 리포트 생성 |
| `/document-release` | 배포 후 README·CHANGELOG·ARCHITECTURE 등 문서를 코드에 맞게 업데이트 |
| `/cso` | 보안 최고 책임자 모드 — 시크릿 누출·의존성·CI/CD·OWASP Top 10 보안 감사 |
| `/codex` | OpenAI Codex CLI를 통한 코드 리뷰·도전적 분석·독립적 의견 제시 |
| `/setup-deploy` | 배포 플랫폼(Fly.io, Vercel 등)을 감지하고 `CLAUDE.md`에 설정을 저장 |
| `/setup-browser-cookies` | 실제 브라우저의 쿠키를 헤드리스 세션으로 가져와 인증된 페이지 테스트 가능하게 함 |
| `/careful` | `rm -rf`, `DROP TABLE`, 강제 푸시 등 위험한 명령 실행 전 경고를 표시 |
| `/freeze` | 수정 가능한 디렉토리를 특정 폴더로 제한해 실수로 다른 코드를 건드리지 않게 함 |
| `/guard` | `/careful` + `/freeze` 결합 — 최대 안전 모드 |
| `/unfreeze` | `/freeze`로 설정한 제한을 해제해 전체 디렉토리 편집을 다시 허용 |
| `/gstack-upgrade` | gstack을 최신 버전으로 업그레이드 |

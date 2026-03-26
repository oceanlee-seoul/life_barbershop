# gstack

Use the `/browse` skill from gstack for all web browsing. Never use `mcp__claude-in-chrome__*` tools.

Available skills:
- `/office-hours` — YC 스타일 아이디어 검증 및 브레인스토밍
- `/plan-ceo-review` — CEO 관점 플랜 전략 리뷰
- `/plan-eng-review` — 엔지니어링 관점 아키텍처 리뷰
- `/plan-design-review` — 디자이너 관점 플랜 리뷰
- `/design-consultation` — 디자인 시스템 설계 및 DESIGN.md 생성
- `/review` — PR 머지 전 코드 리뷰
- `/ship` — 배포 전 과정 자동화 (테스트 → 버전 범프 → PR)
- `/land-and-deploy` — PR 머지 후 배포 및 프로덕션 검증
- `/canary` — 배포 후 라이브 앱 지속 모니터링
- `/benchmark` — 성능 회귀 감지 및 추적
- `/browse` — 헤드리스 브라우저 웹 탐색·테스트 (모든 웹 브라우징에 사용)
- `/qa` — 웹 앱 체계적 테스트 및 버그 수정
- `/qa-only` — 웹 앱 테스트 후 버그 리포트만 작성 (수정 없음)
- `/design-review` — 실제 사이트 시각적 감사 및 디자인 수정
- `/setup-browser-cookies` — 실제 브라우저 쿠키를 헤드리스 세션으로 가져오기
- `/setup-deploy` — 배포 플랫폼 감지 및 설정 저장
- `/retro` — 주간 엔지니어링 회고 리포트 생성
- `/investigate` — 버그 근본 원인 체계적 분석 및 수정
- `/document-release` — 배포 후 문서 업데이트
- `/codex` — OpenAI Codex CLI를 통한 독립적 코드 리뷰
- `/cso` — 보안 감사 (시크릿 누출·의존성·OWASP Top 10)
- `/careful` — 위험한 명령 실행 전 경고 표시
- `/freeze` — 수정 가능한 디렉토리를 특정 폴더로 제한
- `/guard` — `/careful` + `/freeze` 결합 최대 안전 모드
- `/unfreeze` — `/freeze` 제한 해제
- `/gstack-upgrade` — gstack 최신 버전으로 업그레이드

If gstack skills aren't working, run the following to build the binary and register skills:

```bash
cd .claude/skills/gstack && ./setup
```

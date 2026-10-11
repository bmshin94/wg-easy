# wg-easy 심화 분석 & 수익화 전략 정리

> 작성일: 2026-10-11
> 작성: Claude Code (카리나 페르소나) / 요청자: mono7594@gmail.com

## 📌 저장소 정보

| 항목 | 값 |
|---|---|
| **분석 대상 (내 포크)** | https://github.com/bmshin94/wg-easy |
| **원본 업스트림** | https://github.com/wg-easy/wg-easy |
| **공식 문서** | https://wg-easy.github.io/wg-easy/latest |
| **도커 이미지** | `ghcr.io/wg-easy/wg-easy:15` |
| **버전** | 15.4.0 (`src/package.json`) |
| **라이선스** | AGPL-3.0-only |
| **규모** | `src/` 기준 310개 파일 (Vue 125 / TS 121 / JSON 43 / SQL 9) |
| **작업 브랜치** | `claude/github-project-analysis-oitpb9` |
| **관련 노트** | [`wg-easy-analysis-ko.md`](./wg-easy-analysis-ko.md), [`wg-easy-conversation-summary-ko.md`](./wg-easy-conversation-summary-ko.md) |

---

## 1. 프로젝트 정체 — 전수조사 결과

### 한 줄 정의
**WireGuard VPN 서버 + 웹 관리자 UI를 Docker 한 방으로 돌리는 올인원 오픈소스 셀프호스팅 애플리케이션.**

### 폴더별 역할

| 경로 | 역할 |
|---|---|
| `src/app/` | 프론트엔드 (Vue 125파일, 페이지 18개, Pinia 스토어) |
| `src/server/api/` | REST API 엔드포인트 (파일기반 라우팅, 총 43개) |
| `src/server/routes/` | 특수 라우트 (`/cnf/[oneTimeLink]`, `/metrics/json`, `/metrics/prometheus`) |
| `src/server/utils/` | 핵심 로직 21개 — `WireGuard.ts`(311줄), `firewall.ts`(377줄), `oauth.ts`, `password.ts`, `prometheus.ts`, `securityLogger.ts`, `trustedProxy.ts`, `cmd.ts` 등 |
| `src/server/database/` | SQLite + Drizzle ORM, 마이그레이션 9개, 레포지토리 7종 (client / user / interface / general / hooks / oneTimeLink / userConfig) |
| `src/cli/` | CLI 도구 (`admin/reset`, `clients/list`, `clients/qr`) |
| `src/i18n/locales/` | 다국어 26개 (`ko.json` 포함) |
| `src/test/unit/` | vitest 유닛테스트 7개 (awg, clientStatus, firewall, ip, password, prometheus, trustedProxy) |
| `docs/` | 공식 문서 사이트 소스 (zensical) |
| `.github/workflows/` | CI/CD 7개 (deploy, deploy-edge, deploy-pr, deploy-development, lint, codeql, stale) |
| `Dockerfile` | 3-stage 빌드 (Nuxt 빌드 → AmneziaWG 컴파일 → Alpine 런타임) |

### 기술 스택

```
프론트: Nuxt 4 · Vue 3 · TailwindCSS 3 · Pinia · Reka UI · ApexCharts
백엔드: Nitro(Node) · Drizzle ORM 0.45 · libSQL(SQLite)
인증:   argon2 · otpauth(TOTP 2FA) · openid-client(OIDC)
VPN:    wireguard-tools · wireguard-go · AmneziaWG(DPI 우회 난독화)
검증:   Zod 4 / 언어: TypeScript 6 / 패키지: pnpm 12.4.1
```

### 주요 기능 (코드로 확인)

- 클라이언트 CRUD + 활성/비활성 토글
- QR코드 즉시 발급 (`api/client/[clientId]/qrcode.svg.get.ts`)
- 설정파일 다운로드 (`configuration.get.ts`)
- 실시간 Tx/Rx 트래픽 차트 + 접속 상태
- One Time Link — 1회용 설정 공유 (`routes/cnf/[oneTimeLink].ts`)
- 클라이언트 만료일 (`expiresAt`) 기반 자동 비활성화
- 2FA(TOTP) + OIDC 로그인 (Google / GitHub / Authelia / Authentik)
- 클라이언트별 방화벽 격리 (`firewallIps`, iptables 기반)
- Prometheus / JSON 메트릭
- IPv6, CIDR, 다크모드, 26개 언어, Gravatar

### 어떨 때 쓰나

1. 개인 VPN 서버 (VPS·NAS에 올려 집/사무실 망 접속)
2. 재택근무 사내망 원격 접속
3. 공용 WiFi 트래픽 암호화
4. 해외에서 지역 제한 서비스 이용
5. 팀 단위 VPN 운영 (Role 기반 사용자별 발급)
6. 검열 환경 우회 (AmneziaWG)

### 나에게 주는 가치

| 관점 | 가치 |
|---|---|
| 비용 | VPS 월 5천원으로 상용 VPN(월 1.5만원) 대체 → 연 약 15만원 절감 |
| 학습 | Nuxt 4 + Nitro + Drizzle 실전 설계 교보재 |
| 포트폴리오 | AGPL 오픈소스 기여 이력 |
| 사업 | VPN SaaS / 설치대행 / MCP 연동 레이어 창업 기반 |
| 콘텐츠 | 유튜브·강의·전자책 소재 |

---

## 2. 쉬운 설명 — 동작 원리

### 왜 필요한가
순수 WireGuard는 `wg genkey`로 키 생성 → `wg0.conf` 수동 작성 → IP 수동 할당 → 방화벽 수동 설정을 **클라이언트마다 반복**해야 한다. wg-easy는 이 전 과정을 웹 버튼 클릭으로 자동화한다.

### 사용 흐름 (5분)

```
1. docker compose up -d
2. 브라우저로 https://서버:51821 접속
3. 초기 설정 (setup/1~4) — 관리자 계정 · 호스트 · DNS · CIDR
4. "Add Client" 클릭 → 이름 입력
5. QR코드 스캔 (폰 WireGuard 앱)
6. 연결 완료
```

### 내부 처리 파이프라인

```
[Add Client 클릭]
  → Vue 페이지 (src/app/pages/index.vue)
  → POST /api/client (src/server/api/client/index.post.ts)
  → Zod 입력 검증
  → SQLite 저장 (키페어 · IPv4/IPv6 · 이름 · 만료일)
  → WireGuard.saveConfig()
       ├─ #saveWireguardConfig()  : wg0.conf 자동 생성
       ├─ #syncWireguardConfig()  : wg syncconf 로 무중단 반영
       └─ #applyFirewallRules()   : iptables/ip6tables 규칙 재구성
  → QR코드 SVG 생성 → 화면 표시
```

### 3개 덩어리로 기억

| 덩어리 | 역할 | 비유 |
|---|---|---|
| 웹 UI (Vue) | 사용자 화면 | 운전석 |
| API 서버 (Nitro) | 명령 처리 · DB | 엔진 |
| WireGuard (리눅스) | 실제 VPN 터널 | 바퀴 |

Docker가 이 셋을 한 컨테이너로 묶어 어느 서버에서든 동일하게 동작시킨다.

---

## 3. 질문 7개 답변

### Q1. 설치 및 사용법

**필수 조건:** 공인 IP 있는 리눅스 서버, `51820/UDP` + `51821/TCP` 개방

```bash
# 1) 도커 설치
curl -sSL https://get.docker.com | sh
exit   # 재로그인

# 2) docker-compose.yml 배치 후 실행
docker compose up -d
docker compose logs -f
```

핵심 compose 설정 (레포 `docker-compose.yml` 기준):

```yaml
image: ghcr.io/wg-easy/wg-easy:15
volumes:
  - etc_wireguard:/etc/wireguard
  - /lib/modules:/lib/modules:ro
ports:
  - "51820:51820/udp"
  - "51821:51821/tcp"
cap_add: [NET_ADMIN, SYS_MODULE]     # 필수 권한
sysctls:
  - net.ipv4.ip_forward=1
  - net.ipv4.conf.all.src_valid_mark=1
  - net.ipv6.conf.all.forwarding=1
```

> HTTPS가 아니면 기본 거부됨. 테스트용만 `INSECURE=true`, 실서비스는 Caddy/Traefik 리버스 프록시 필수.

**선택 환경변수**

| 변수 | 기본값 | 설명 |
|---|---|---|
| `PORT` | 51821 | 웹UI 포트 |
| `HOST` | 0.0.0.0 | 바인딩 IP |
| `INSECURE` | false | HTTP 허용 |
| `DISABLE_IPV6` | false | IPv6 비활성 |
| `DISABLE_VERSION_CHECK` | false | 업데이트 확인 비활성 |
| `TRUSTED_PROXIES` | — | 프록시 IP/CIDR (CrowdSec 연동) |

**무인 설치 (Ansible 등):** `INIT_ENABLED=true` + `INIT_USERNAME` / `INIT_PASSWORD` / `INIT_HOST` / `INIT_PORT` / `INIT_DNS` / `INIT_IPV4_CIDR` / `INIT_IPV6_CIDR` / `INIT_ALLOWED_IPS`
(같은 그룹 변수는 세트로 지정해야 하며, 설정 완료 후 변수 제거 권장)

**개발 명령어**

```bash
pnpm dev                    # 개발서버 (docker watch)
pnpm cli:dev                # CLI 테스트
cd src && pnpm test:unit    # 유닛테스트
cd src && pnpm check:all    # typecheck + lint + format + build
```

### Q2. 플러그인 / 스킬 / MCP 중 무엇인가

**셋 다 아니다. 독립 실행형 웹 애플리케이션(self-hosted web app)이다.**

| 근거 | 확인 결과 |
|---|---|
| `.claude/` 디렉터리 | 없음 → 스킬 아님 |
| `plugin.json` | 없음 → 플러그인 아님 |
| `@modelcontextprotocol/sdk` 의존성 | 없음 → MCP 서버 아님 |
| `Dockerfile` + `CMD ["node","server/index.mjs"]` | 자체 서버 프로세스로 독립 구동 |

단, REST API 43개를 제공하므로 **이를 감싸는 MCP 서버를 별도로 만들면** AI가 자연어로 VPN을 관리할 수 있다 (→ 수익화 아이디어 2).

### Q3. API 토큰이 필요한가

**토큰 방식이 아니라 HTTP Basic 인증이다.** 웹UI 로그인 계정의 ID/PW를 그대로 사용한다.

```python
import requests
from requests.auth import HTTPBasicAuth

url = "https://example.com:51821/api/client"
r = requests.get(url, auth=HTTPBasicAuth('username', 'password'))
print(r.json())
```

**제약 3가지**
1. 별도 API 토큰 발급 기능이 없다
2. **2FA를 켜면 API를 쓸 수 없다** (공식 문서 명시 — API 사용 시 2FA 해제 필요)
3. API는 아직 unstable — 예고 없이 변경될 수 있다는 경고가 문서에 있음

**외부 키 필요 여부**

| 기능 | 키/토큰 |
|---|---|
| VPN 기본 기능 | 불필요 (완전 셀프호스팅) |
| OIDC 로그인 | Client ID + Secret 필요 |
| Prometheus 메트릭 | 불필요 |
| Gravatar | 불필요 |

**주요 엔드포인트 (파일기반 라우팅)**

| 파일 | 엔드포인트 | 메서드 |
|---|---|---|
| `api/client/index.get.ts` | `/api/client` | GET |
| `api/client/index.post.ts` | `/api/client` | POST |
| `api/client/[clientId]/index.delete.ts` | `/api/client/{id}` | DELETE |
| `api/client/[clientId]/qrcode.svg.get.ts` | `/api/client/{id}/qrcode.svg` | GET |
| `api/client/[clientId]/generateOneTimeLink.post.ts` | `/api/client/{id}/generateOneTimeLink` | POST |
| `routes/metrics/json.get.ts` | `/metrics/json` | GET |

### Q4. AI 에이전트 구축에 도움이 되는가

**된다. 세 방향.**

**(1) 에이전트가 조작하는 도구(Tool)로서** — API 43개가 그대로 에이전트의 액션이 된다.

```
"신입 3명 VPN 계정 만들어서 1회용 링크 전달해줘"
 → POST /api/client ×3
 → POST /api/client/{id}/generateOneTimeLink ×3
 → 메신저 전송
```

**(2) 아키텍처 교보재로서**

| 패턴 | 파일 | 배울 점 |
|---|---|---|
| 파일기반 라우팅 | `server/api/**` | 엔드포인트 = 파일 구조 |
| Zod 입력검증 | 각 `*.post.ts` | 에이전트 툴 스키마 설계와 동형 |
| 레포지토리 패턴 | `database/repositories/` | DB 로직 분리 |
| 핸들러 추상화 | `utils/handler.ts` | 인증/권한 공통 처리 |
| 역할기반 권한 | `shared/utils/permissions` | 멀티테넌트 설계 |
| 외부 명령 실행 | `utils/cmd.ts` | 에이전트의 안전한 셸 실행 패턴 |
| 보안 로깅 | `utils/securityLogger.ts` | 감사(audit) 로그 설계 |

**(3) 에이전트 인프라로서** — 에이전트를 VPN 내부에 배치해 사내 리소스에 안전하게 접근시키고, `firewallIps`로 접근 범위를 클라이언트 단위로 제한한다.

### Q5. 수익화 아이디어

→ 4장에서 상세 정리.

### Q6. React나 PHP로 만들 수 있는가

**React (Next.js) — 난이도 ★★☆☆☆, 2~3주**

| 현재 | React 대응 |
|---|---|
| Vue 컴포넌트 125개 | React 컴포넌트 재작성 |
| Pinia | Zustand / TanStack Query |
| `@nuxtjs/i18n` | `next-intl` |
| `vue3-apexcharts` | `react-apexcharts` (동일 라이브러리) |
| Reka UI | Radix UI |

**재사용 가능 (작업량 약 60% 절감):** `server/utils/` 순수 TS 로직 21개, Drizzle 스키마 + 마이그레이션 9개, Zod 스키마, Dockerfile 구조, 26개 언어 JSON.

**PHP (Laravel) — 난이도 ★★★★☆, 2~3개월**

가능한 근거: WireGuard 제어는 결국 `wg` / `wg-quick` 셸 호출이므로 PHP `exec()`로도 구현 가능. SQLite 기본 지원. Laravel Fortify(2FA) / Socialite(OIDC) 보유.

어려운 점: 요청-응답 모델이라 상주 모니터링에 Queue/Horizon·Supervisor 필요, 실시간 차트에 Reverb 추가 필요, TS 로직 121개 파일 전면 포팅(재사용 0%), 컨테이너 구성 복잡화(PHP-FPM + Nginx + Supervisor).

| 항목 | Nuxt(현재) | Next(React) | Laravel(PHP) |
|---|---|---|---|
| 로직 재사용 | 100% | 60% | 0% |
| 작업기간 | — | 2~3주 | 2~3개월 |
| 실시간 처리 | 네이티브 | 네이티브 | 별도 구성 |
| 추천도 | ★★★★★ | ★★★★ | ★★ |

### Q7. 유튜브 강의 영상 제작이 가능한가

**가능하다.** AGPL-3.0은 사용·설명·교육을 자유롭게 허용한다.
주의: ① 코드를 수정해 네트워크 서비스로 배포하면 소스 공개 의무 발생(강의는 해당 없음) ② "WireGuard"는 Jason A. Donenfeld의 등록상표 → 영상에 비공식/비제휴 고지 ③ 실제 IP·키 모자이크 ④ 교육 목적·본인 서버 한정 고지 ⑤ HTTPS 미적용 운영 금지 경고 포함.

**10부작 커리큘럼 제안**

| # | 주제 | 길이 |
|---|---|---|
| 1 | VPN 개념 + WireGuard vs OpenVPN | 10분 |
| 2 | VPS로 나만의 VPN 서버 구축 | 15분 |
| 3 | docker-compose 해부 (cap_add·sysctls 원리) | 20분 |
| 4 | QR 1분 연결 + 공용WiFi 보안 실험 | 12분 |
| 5 | Caddy 리버스 프록시로 HTTPS | 18분 |
| 6 | 2FA + Google OIDC 연동 | 20분 |
| 7 | 클라이언트별 방화벽 네트워크 격리 | 18분 |
| 8 | Prometheus + Grafana 대시보드 | 25분 |
| 9 | API 자동화 (Python) | 20분 |
| 10 | MCP 서버 직접 만들어 AI에게 VPN 관리 시키기 | 30분 |

**수익 모델:** 애드센스 + 멤버십 + VPS 제휴링크(가입당 $10~25) + 강의/전자책 + 설치대행 의뢰 유입.

---

## 4. 수익화 아이디어 상세

### 아이디어 1 — 관리형 VPN SaaS (B2B)

| 항목 | 내용 |
|---|---|
| 타겟 | 직원 5~50명 중소기업, 스타트업, 병원·학원 |
| 문제 | 전담 IT인력 없음 + 상용 VPN 어플라이언스 고비용 |
| 솔루션 | wg-easy 기반 멀티테넌트 + 관리 대시보드 + 지원 |
| 가격 | 기본 월 9.9만원(10명) / 프로 월 29만원(50명) / 엔터프라이즈 별도 |
| 원가 | VPS 월 1만원 수준 → 마진 80%+ |
| 구현 | 기존 `user` 테이블 + `Role` 권한 체계 활용, 조직(org) 테이블 추가 |

**AGPL 리스크 대응:** ① 코드 미수정 + 운영 서비스만 과금 ② 수정분 공개 + 운영/지원으로 과금 ③ 부가 서비스를 별도 시스템으로 분리 설계.

### 아이디어 2 — AI 에이전트 / MCP 연동 레이어 (최우선 추천)

```
"민수씨 계정 만료 연장하고 재무팀 서버 접근 권한 줘"
 → POST /api/client/{id}  (expiresAt 수정)
 → POST /api/client/{id}  (firewallIps 추가)
 → "3개월 연장, 10.8.0.50 접근 허용 완료"
```

- 난이도 ★★☆☆☆ (API 43개 래핑)
- MCP 생태계 초기 → 선점 효과
- 수익: MCP 서버 오픈소스 공개 + 호스팅/엔터프라이즈 유료(월 5~20만원)
- 확장: 동일 패턴을 다른 오픈소스에 적용 → "MCP 래핑 전문 스튜디오"

### 아이디어 3 — 설치 대행 + 유지보수

| 상품 | 가격 | 소요 |
|---|---|---|
| 기본 설치 (VPS + wg-easy + HTTPS) | 15만원 | 1시간 |
| 프리미엄 (2FA + OIDC + 방화벽 + 모니터링) | 40만원 | 3시간 |
| 월 유지보수 (업데이트·백업·장애대응) | 월 5만원 | — |
| 긴급 출동 | 건당 10만원 | — |

유입: 크몽·숨고·당근 재능, 유튜브 더보기란. 설치 자동화 스크립트 준비 시 1시간 → 10분으로 단축.

### 아이디어 4 — 교육 콘텐츠 퍼널

```
유튜브 10부작 무료 공개 → 인프런/클래스101 강의(59,000원)
→ 전자책(29,000원) → 유료 커뮤니티·멘토링(월 29,000원)
```

| 수익원 | 월 예상 |
|---|---|
| 애드센스 (10만뷰) | 30~80만원 |
| 강의 (월 30건) | 약 170만원 |
| VPS 제휴 (월 20건 × $20) | 약 55만원 |
| 멤버십 (100명) | 290만원 |

> 수치는 가정 기반 추정치이며 실제 전환율에 따라 크게 달라진다.

### 아이디어 5 — 엔터프라이즈 확장 애드온

| 기능 | 현재 상태 | 애드온 아이디어 |
|---|---|---|
| 감사 로그 | 보안 로그만 | 전체 감사 + 컴플라이언스 리포트 |
| 트래픽 쿼터 | 없음 | 사용량 제한 + 과금 연동 |
| 다중 인터페이스 | 단일 | 멀티 리전 / 멀티 서버 |
| SSO | OIDC 기본 | SAML, LDAP/AD 연동 |
| 백업·HA | 없음 | 자동 백업 + 이중화 |
| 청구 | 없음 | Stripe / 토스페이먼츠 연동 |

타겟: VPN 재판매 사업자. 라이선스 연 300~1,000만원 수준.

### 아이디어 6~8 (보너스)

6. **가족 VPN 구독** — 자녀 안전 인터넷 + 광고차단(AdGuard 연동), 월 9,900원
7. **게이밍 VPN** — 핑 최적화 + 지역 서버 선택, 월 4,900원
8. **개발자 툴킷 판매** — Next.js 포팅 버전 + Terraform/Ansible 자동화 템플릿, Gumroad 99,000원

### 추천 로드맵

```
1개월   유튜브 10부작 제작 → 신뢰 자산 (리스크 0)
2개월   설치 대행 시작 (크몽·숨고) → 즉시 현금흐름
3~4개월 MCP 서버 개발 → 깃허브 공개 → 기술 브랜딩
5~6개월 축적된 고객 데이터로 SaaS MVP 출시
```

"콘텐츠 → 서비스 → 제품" 순서가 리스크를 최소화한다.

---

## 5. 참고 링크

- 포크 저장소: https://github.com/bmshin94/wg-easy
- 원본 저장소: https://github.com/wg-easy/wg-easy
- 공식 문서: https://wg-easy.github.io/wg-easy/latest
- 시작하기: https://wg-easy.github.io/wg-easy/latest/getting-started/
- 기본 설치: https://wg-easy.github.io/wg-easy/latest/examples/tutorials/basic-installation/
- API 문서: https://wg-easy.github.io/wg-easy/latest/advanced/api/
- 선택 설정: https://wg-easy.github.io/wg-easy/latest/advanced/config/optional-config/
- 무인 설치: https://wg-easy.github.io/wg-easy/latest/advanced/config/unattended-setup/
- Prometheus: https://wg-easy.github.io/wg-easy/latest/advanced/metrics/prometheus/
- 마이그레이션(14→15): https://wg-easy.github.io/wg-easy/latest/advanced/migrate/

# wg-easy 전수조사 및 질문 답변 정리

- 작성일: 2026-10-11
- 분석 대상 저장소 (포크): https://github.com/bmshin94/wg-easy
- 원본 저장소: https://github.com/wg-easy/wg-easy
- 공식 문서: https://wg-easy.github.io/wg-easy/latest
- 분석 기준: `src/package.json` 버전 15.4.0, 추적 파일 379개, 라이선스 AGPL-3.0-only
- 이전 분석 노트: [`notes/wg-easy-analysis-ko.md`](./wg-easy-analysis-ko.md)

---

## 대화 흐름

| 순서 | 요청 | 결과 |
|---|---|---|
| 1 | 전수조사해서 무엇·언제·나에게 어떤 도움인지 분석 | 1장 |
| 2 | 1번을 더 쉽게 | 2장 |
| 3 | 설치법 / 플러그인·스킬·MCP 여부 / API 토큰 / AI 에이전트 / 수익화 / React·PHP / 유튜브 | 3장 |
| 4 | 수익화 아이디어 상세 | 4장 |
| 5 | 이 파일 저장 후 머지 | 이 문서 |

---

## 1. 전수조사: 무엇이고, 언제 쓰고, 어떤 도움이 되나

### 한 줄 요약
리눅스 서버 한 대에 **WireGuard VPN 서버**와 **웹 관리 화면**을 도커 컨테이너 하나로 띄워서, 클릭 몇 번으로 VPN 사용자(클라이언트)를 만들고 QR 코드로 나눠 주는 **셀프호스팅 프로그램**.

### 포크 상태
- 원본 `wg-easy/wg-easy`를 포크한 저장소
- 원본 대비 추가된 것은 문서뿐: `CLAUDE.md` (Claude Code 대화 말투 설정), `notes/` (분석 노트)
- 실제 프로그램 코드는 원본과 동일

### 기술 스택
| 영역 | 사용 기술 |
|---|---|
| 웹 UI | Vue 3 + Nuxt 4 + Tailwind CSS + Pinia + ApexCharts |
| 서버 | Nuxt 서버(Nitro/h3), TypeScript |
| DB | SQLite(libsql) + Drizzle ORM, 마이그레이션 9개 |
| 보안 | Argon2 비밀번호 해시, TOTP 2단계 인증, OpenID Connect(OAuth) 로그인 |
| VPN | `wireguard-tools`, `wireguard-go`, AmneziaWG(실험 기능) |
| 배포 | 3단계 Docker 빌드 → `ghcr.io/wg-easy/wg-easy:15` |
| 테스트 | Vitest 단위 테스트 7개 (ip, firewall, password, prometheus 등) |

### 폴더별 역할
| 경로 | 내용 |
|---|---|
| `src/app/pages/` | 화면: 로그인, 설정 마법사(`setup`), 클라이언트 목록·상세, 관리자(`admin`), 내 정보(`me`) |
| `src/app/components/` | 화면 부품: 클라이언트 카드, 폼, 헤더, 관리자 패널 등 |
| `src/app/stores/` | 화면 상태 관리(Pinia): 로그인, 클라이언트, 토스트 알림 |
| `src/server/api/client/` | 클라이언트 API: 목록·생성·수정·삭제·활성화·비활성화·QR·설정 파일·일회용 링크 |
| `src/server/api/admin/` | 관리자 API: 일반 설정, 인터페이스(CIDR·재시작), 훅(PreUp/PostUp 등), 클라이언트 기본값 |
| `src/server/api/auth/` | 로그인: 비밀번호, 2FA 확인, OAuth 연결·해제 |
| `src/server/api/setup/` | 첫 설치 마법사 단계별 API, v14 → v15 마이그레이션 |
| `src/server/routes/cnf/[oneTimeLink].ts` | 일회용 설정 파일 다운로드 링크 |
| `src/server/routes/metrics/` | Prometheus·JSON 지표 |
| `src/server/utils/WireGuard.ts` | 핵심 엔진: DB → `wg0.conf` 작성 → `wg syncconf`로 무중단 반영 → 방화벽 갱신, 만료 확인 크론 |
| `src/server/utils/wgHelper.ts` | 서버·클라이언트 `.conf` 텍스트 생성, 키 생성 |
| `src/server/utils/firewall.ts` | 클라이언트별 접근 허용 대상 필터링(iptables) |
| `src/server/database/` | 테이블 정의와 저장소 7종: user, client, interface, general, hooks, oneTimeLink, userConfig |
| `src/cli/` | 터미널 도구: 관리자 비밀번호 초기화, 클라이언트 목록, QR 출력 |
| `src/i18n/locales/` | 번역 26개 (한국어 `ko.json` 포함) |
| `docs/content/` | 공식 문서 원본: 설치, Caddy·Traefik·Podman·AdGuard 튜토리얼, API, 2FA, 마이그레이션 |
| `Dockerfile` | UI 빌드 → AmneziaWG 컴파일 → Alpine 실행 이미지 |
| `docker-compose.yml` | 설치용 파일. UDP 51820(VPN), TCP 51821(웹 UI) |
| `.github/workflows/` | 배포(정식·엣지·개발·PR), 린트, CodeQL 보안 검사, 오래된 이슈 정리 |

### 동작 흐름
1. 관리자가 웹 화면에서 "새 클라이언트" 클릭
2. 서버가 키 쌍과 내부 IP를 만들어 SQLite에 저장 (`POST /api/client` → `Database.clients.create`)
3. `WireGuard.saveConfig()`가 `/etc/wireguard/wg0.conf`를 다시 쓰고 `wg syncconf`로 즉시 반영 (재시작 없음)
4. 사용자는 휴대폰으로 QR 스캔하거나 PC에서 `.conf` 파일을 가져와 접속
5. 1분 주기 크론이 만료된 클라이언트를 비활성화

### 언제 쓰나
- 밖에서 집·회사 내부망(NAS, 홈서버, 사내 DB, 관리 페이지)에 안전하게 접속할 때
- 카페·호텔 공용 와이파이에서 트래픽을 암호화할 때
- 팀원·가족에게 VPN 접속권을 발급하고 회수·만료 관리를 할 때
- 여러 서버를 하나의 사설망으로 묶을 때

### 나에게 주는 도움
- 개인 서버 서비스(대시보드, n8n, DB, 로컬 LLM 등)를 인터넷에 공개하지 않고 VPN 안에서만 사용
- AdGuard Home과 묶어 밖에서도 광고 차단 DNS 사용 (공식 튜토리얼 있음)
- Tailscale·상용 VPN과 달리 외부 회사에 의존하지 않고 내 서버에서 무료 운영
- Nuxt 4 풀스택, Drizzle ORM, OAuth/2FA, 도커 멀티스테이지 빌드, GitHub Actions를 실제 프로젝트로 배우는 교재

---

## 2. 더 쉽게 설명

### 비유
- **인터넷** = 사람이 많이 다니는 큰길. 누가 엿볼 수 있음
- **VPN(WireGuard)** = 내 폰에서 우리 집 서버까지 이어진 **비밀 지하 통로**
- **wg-easy** = 그 통로의 **출입증 자동 발급기**. 버튼을 누르면 출입증(QR 코드)이 나옴
- **관리 화면** = 발급기 관리실. 누가 출입증을 가졌는지, 지금 통로에 누가 있는지, 언제 만료되는지 확인

### 하루 사용 장면
1. 아침: 관리 화면에서 "회사폰" 출입증 생성 → 폰 WireGuard 앱으로 QR 스캔
2. 점심: 카페 와이파이에서 VPN을 켜고 집 NAS 사진 확인
3. 저녁: 친구에게 일회용 링크로 출입증 전달, 만료일 7일 설정 → 7일 뒤 자동 차단
4. 관리 화면 그래프로 누가 얼마나 썼는지 확인

### 꼭 기억할 3가지
1. 24시간 켜진 리눅스 서버(클라우드 VPS나 홈서버)가 필요하다
2. UDP 51820(통로), TCP 51821(관리 화면) 포트를 쓴다
3. 관리 화면은 HTTPS(리버스 프록시)로 보호하고 2FA를 켠다

---

## 3. 질문별 답변

### 3-1. 설치 및 사용법

**준비물**: 공인 IP 또는 도메인이 있는 리눅스 서버(x86_64/arm64), 도커

```bash
# 1) 도커 설치 (root)
curl -sSL https://get.docker.com | sh

# 2) compose 파일 받기
sudo mkdir -p /etc/docker/containers/wg-easy
sudo curl -o /etc/docker/containers/wg-easy/docker-compose.yml \
  https://raw.githubusercontent.com/wg-easy/wg-easy/master/docker-compose.yml

# 3) 실행
cd /etc/docker/containers/wg-easy
sudo docker compose up -d
```

- **방화벽**: UDP 51820 개방 (클라우드는 보안 그룹에서도 개방)
- **관리 화면 접속**
  - 권장: Caddy 또는 Traefik 리버스 프록시로 `https://vpn.내도메인`
  - 내부망 테스트용: `INSECURE=true` 설정 후 `http://서버IP:51821` (인터넷 노출 금지)
- **첫 실행**: 설정 마법사에서 관리자 계정 생성 → 서버 주소(도메인/IP)와 포트 입력
- **사용**: "새 클라이언트" → 폰은 QR 스캔, PC는 `.conf` 파일 가져오기
- **업데이트**: `sudo docker compose pull && sudo docker compose up -d`
- **무인 설치**: `INIT_ENABLED=true`, `INIT_USERNAME`, `INIT_PASSWORD`, `INIT_HOST`, `INIT_PORT` 등 환경 변수
- **비밀번호 분실**: `docker exec -it wg-easy cli` → 관리자 비밀번호 초기화
- **주요 환경 변수**: `PORT`, `HOST`, `INSECURE`, `DISABLE_IPV6`, `DISABLE_VERSION_CHECK`, `TRUSTED_PROXIES`

### 3-2. 플러그인? 스킬? MCP?
**셋 다 아님.** 혼자 돌아가는 **셀프호스팅 웹 애플리케이션(도커 앱)**.

| 구분 | 의미 | wg-easy 해당 여부 |
|---|---|---|
| 플러그인 | 다른 프로그램(Claude Code, 워드프레스 등)에 끼워 넣는 확장 | 아님 |
| 스킬 | Claude가 읽는 작업 지침 묶음(`SKILL.md`) | 아님 |
| MCP | AI가 외부 도구를 호출하게 해 주는 서버 규격 | 아님 |

- 포크에 있는 `CLAUDE.md`는 Claude Code 대화 말투 설정 문서일 뿐 프로그램 기능과 무관
- 다만 REST API(`/api/...`)가 있어서 **MCP 서버를 직접 만들어 붙이는 것은 가능**

### 3-3. API 토큰을 써야 하나?
**필요 없음.**
- VPN 기능은 외부 유료 API를 쓰지 않음 (OpenAI·Claude 토큰 불필요, 비용 0원)
- wg-easy 자체 API는 관리자 아이디·비밀번호로 **Basic Auth** 인증 (`src/server/utils/session.ts`)
  - 2FA를 켠 계정은 API 사용 불가 (공식 문서 명시)
  - API는 "아직 불안정, 예고 없이 바뀔 수 있음" 단계
- 선택 사항
  - Prometheus 지표 Bearer 비밀번호
  - OAuth 로그인을 쓸 때 Google·GitHub 등의 Client ID/Secret

```python
import requests
r = requests.get("https://vpn.example.com/api/client", auth=("admin", "비밀번호"))
print(r.json())
```

엔드포인트는 파일 이름이 곧 주소: `src/server/api/client/index.post.ts` → `POST /api/client`

### 3-4. AI 에이전트 구축에 도움이 되나?
**직접 AI 기능은 없지만 두 가지로 도움이 됨.**

1. **안전한 인프라**: 에이전트 서버, 로컬 LLM(Ollama), 벡터 DB, n8n, 사내 DB를 VPN 사설망 안에서만 연결 → API 포트를 인터넷에 열지 않아도 됨
2. **에이전트의 도구로 연결 (MCP)**: wg-easy API를 감싼 MCP 서버를 만들면 Claude가 자연어로 VPN 관리
   - "민수 계정 만들고 30일 뒤 만료" → `POST /api/client` (`name`, `expiresAt`)
   - "지금 접속 중인 사람?" → `GET /api/client`
   - "퇴사자 차단" → `POST /api/client/{id}/disable`
   - "일회용 링크 보내 줘" → `POST /api/client/{id}/generateOneTimeLink`
   - 주의: 삭제·비활성화 같은 위험 작업은 사람 확인 단계를 두기

### 3-5. 수익화할 만한 아이디어가 있나?
있음. 요약하면 **설치·운영 대행 / AI(MCP) 관리 도구 / 강의 콘텐츠 / 설치 템플릿 / 관리형 호스팅**. 자세한 내용은 4장.

### 3-6. React나 PHP로 만들 수 있나?
**만들 수 있음.** 방식별 난이도:

| 방식 | 난이도 | 설명 |
|---|---|---|
| React 프런트엔드만 새로 제작 | 중 | wg-easy를 백엔드로 두고 `/api`를 호출하는 나만의 대시보드. 가장 현실적 |
| React + Node 전체 재작성 | 상 | 키 생성 → `.conf` 생성 → `wg syncconf` 실행 → iptables 규칙 로직 재구현 |
| PHP(Laravel) 재작성 | 상 | `wg` 명령 실행 + MySQL. 웹 서버에 root/NET_ADMIN 권한이 필요해 보안 설계가 까다로움. 별도 권한 데몬 분리 권장 |

- 원본 코드를 복사·수정해서 서비스로 제공하면 **AGPL-3.0** 적용 → 사용자에게 소스 공개 의무
- 처음부터 새로 작성하면 이 부담이 줄어듦
- "WireGuard"는 등록 상표라 제품 이름에 쓰지 않기

### 3-7. 유튜브 강의 영상으로 만들 수 있나?
**가능.** 화면 결과가 바로 보여서(QR 스캔 → 접속) 영상에 잘 맞음.

추천 시리즈:
1. VPN이 뭐고 왜 직접 만들까? (상용 VPN·Tailscale 비교)
2. 무료·저가 클라우드 서버 만들기
3. 도커 + wg-easy 10분 설치
4. 폰·PC 연결, QR 실습
5. Caddy HTTPS + 2FA로 보안 강화
6. AdGuard Home 조합 광고 차단 VPN
7. 홈서버·NAS 원격 접속
8. Claude + MCP로 말로 VPN 관리하기 (차별화 포인트)

주의: 불법 우회·저작권 회피 용도는 다루지 않고 보안·원격 접속 중심으로 구성.

---

## 4. 수익화 아이디어 상세

> 가격·기간은 예시 가정이며 시장 조사 결과가 아님. 법률·세무는 전문가 확인 필요.

### 아이디어 1. 소규모 사업장 VPN 설치·운영 대행
- **대상**: 직원 5~30명 사무실, 병원·학원, 원격 근무 스타트업
- **제공**: 서버 구축 + HTTPS + 2FA, 계정 발급·회수·만료, 월간 점검·백업, 접속 리포트(Prometheus + Grafana)
- **가격 구조**: 설치비 1회 + 월 관리비
- **장점**: 원본을 수정하지 않고 쓰므로 AGPL 부담이 거의 없음, 초기 비용 낮음
- **시작**: 재능 마켓 등록, 지인 사무실 무료 세팅으로 사례·후기 확보

### 아이디어 2. AI로 관리하는 VPN — MCP 서버
- **도구 구성**: `create_client`, `list_clients`, `list_connected`, `disable_client`, `send_one_time_link`, `traffic_report`
- **수익 구조**: 코어는 오픈소스로 공개(홍보·신뢰), 유료로 멀티 서버 관리, Slack/디스코드 승인 흐름, 감사 리포트, 기업 설치 지원
- **장점**: AI 에이전트·MCP 포트폴리오로도 활용 가능

### 아이디어 3. 교육 콘텐츠
- **흐름**: 무료 유튜브(3-7 시리즈) → 유료 강의(초급·중급·고급) → 1:1 컨설팅
- **부가 수익**: 클라우드 제휴 링크, 설정 템플릿 묶음, 전자책
- **장점**: 아이디어 1·2의 고객 유입 통로 역할

### 아이디어 4. 원클릭 설치 템플릿 판매
- "wg-easy + Caddy + AdGuard Home + 자동 백업·업데이트" compose 템플릿
- 클라우드 마켓플레이스 이미지, 설치 스크립트
- 한국어 운영 매뉴얼(PDF) 묶음

### 아이디어 5. 관리형 VPN 호스팅 (SaaS)
- 고객별 인스턴스 + 월 구독, 웹에서 가입·결제·자동 생성
- **주의**
  - AGPL: 수정한 코드로 서비스하면 소스 공개 의무
  - "WireGuard" 상표 사용 금지
  - 트래픽 중계에 따른 법적 책임, 이용 약관, 로그 정책
  - 24시간 장애 대응 부담
- 1~3번으로 수요를 확인한 뒤 도전 권장

### 추천 로드맵
| 단계 | 할 일 | 기간 |
|---|---|---|
| 1 | 직접 설치, 홈서버·AdGuard 연동 | 1주 |
| 2 | 유튜브 1~4편 제작 | 2~3주 |
| 3 | MCP 서버 MVP + GitHub 공개 + 유튜브 8편 | 2주 |
| 4 | 설치 대행 상품 등록, 첫 고객 사례 | 1개월 |
| 5 | 유료 강의·템플릿 판매 | 이후 |

**핵심 전략**: 콘텐츠로 신뢰 → 오픈소스 도구로 차별화 → 서비스로 수익

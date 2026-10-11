# wg-easy 분석 및 대화 정리

- 작성일: 2026-10-12
- 분석 대상 저장소: https://github.com/bmshin94/wg-easy
- 원본 저장소: https://github.com/wg-easy/wg-easy
- 공식 문서: https://wg-easy.github.io/wg-easy/latest
- 분석 기준 버전: v15.4.0 (2026-08-14), 포크 최신 커밋 `efa0e72`

---

## 0. 대화 흐름 요약

| 순서 | 질문 | 결과 |
|---|---|---|
| 1 | 내 깃허브 연결돼? | `bmshin94` 계정으로 연결 확인. 저장소 목록 조회 및 push 권한 있음 |
| 2 | nautilus_trader 레포 있어? | 최근 200개 목록에는 없음. 직접 연결 시도는 사용자가 취소 |
| 3 | wg-easy 분석 + 질문 5단계 | 아래 1~4장에 정리, 이 파일이 5단계 결과물 |

---

## 1. 전수조사 결과: 무엇이고 언제 쓰나

### 한 줄 요약
리눅스 서버에 **WireGuard VPN**을 설치하고, **웹 관리 화면**에서 클릭만으로 VPN 사용자를 관리하게 해 주는 셀프호스팅 프로그램. 도커 이미지 다운로드 2,800만 회 이상.

### 저장소 상태
- 원본을 포크한 저장소. 추가 커밋은 PR #1 (`CLAUDE.md` 추가) 하나뿐이고 코드는 원본과 동일
- 파일 수 약 378개, 라이선스 **AGPL-3.0-only**

### 주요 기능 (코드로 확인)
- 클라이언트 추가·수정·삭제·활성화·비활성화 (`src/server/api/client/*`)
- 휴대폰용 QR 코드, PC용 `.conf` 파일 다운로드
- 일회용 다운로드 링크 (`src/server/routes/cnf/[oneTimeLink].ts`)
- 클라이언트 만료일, 접속 상태, 송수신 그래프
- 2단계 인증(TOTP), Argon2 비밀번호 해시, OAuth/OIDC 로그인 (Google, GitHub 등)
- 클라이언트별 방화벽 필터링 (`src/server/utils/firewall.ts`)
- Prometheus·JSON 지표 (`/metrics/prometheus`, `/metrics/json`)
- AmneziaWG 실험 지원 (DPI 우회용 트래픽 위장)
- 26개 언어(한국어 포함), 다크 모드, 모바일 UI, IPv6, CLI

### 폴더 구조
| 경로 | 역할 |
|---|---|
| `src/app/` | 웹 UI (Vue + Nuxt 4 + Tailwind). 로그인, 설정 마법사, 클라이언트 목록, 관리자 화면 |
| `src/server/api/` | REST API. 파일 이름이 곧 주소와 메서드 (`client/index.get.ts` → `GET /api/client`) |
| `src/server/utils/WireGuard.ts` | 핵심 엔진. DB → `wg0.conf` 생성 → `wg syncconf` → 방화벽 재구성 |
| `src/server/utils/wgHelper.ts` | 서버·클라이언트 설정 파일 텍스트 생성 |
| `src/server/database/` | SQLite + Drizzle ORM. 사용자·클라이언트·인터페이스·훅·일회용 링크 테이블, 마이그레이션 9개 |
| `src/cli/` | 관리자 비밀번호 초기화, 클라이언트 목록, QR 출력 |
| `src/i18n/` | 번역 파일 26개 |
| `docs/` | 공식 문서 원본 (설치, Caddy·Traefik, AdGuard, API, 마이그레이션) |
| `Dockerfile` | 3단계 빌드: UI 빌드 → AmneziaWG 컴파일 → Alpine 실행 이미지 |
| `docker-compose.yml` | 설치용 파일. UDP 51820(VPN), TCP 51821(웹 UI) |
| `.github/workflows/` | 빌드·배포·린트·CodeQL 등 7개 |

### 동작 흐름
1. 웹에서 클라이언트 추가
2. 서버가 키 쌍과 IP를 생성해 SQLite에 저장
3. `/etc/wireguard/wg0.conf`를 다시 쓰고 WireGuard에 즉시 반영 (재시작 없음)
4. 사용자는 QR을 스캔하거나 설정 파일을 받아 연결

### 언제 쓰나
- 밖에서 집·회사 내부망(NAS, 홈서버, 사내 DB)에 안전하게 접속
- 공용 와이파이에서 트래픽 암호화
- 팀원에게 VPN 계정 발급·회수
- 여러 서버를 하나의 사설망으로 묶기

### 나에게 주는 도움
- 셀프호스팅 서비스(Coolify, AdGuardHome, 대시보드 등)를 공개 인터넷에 노출하지 않고 VPN 안에서만 사용
- AdGuard Home과 조합해 외부에서도 광고 차단 DNS 사용 (공식 튜토리얼 있음)
- Tailscale 대비: 외부 서비스 의존 없이 100% 내 서버에서 무료 운영
- Nuxt 4 풀스택, Drizzle ORM, OAuth/2FA, 도커 멀티스테이지 빌드 학습 교재

---

## 2. 더 쉽게: 비유로 이해하기

- **인터넷** = 사람 많은 큰길
- **VPN(WireGuard)** = 우리 집까지 이어진 비밀 지하 통로
- **wg-easy** = 통로 출입증(QR 코드)을 버튼 하나로 뽑아 주는 자동 발급기

**사용 장면**
1. 여행 중 호텔에서 집 NAS 사진 보기
2. 카페 와이파이에서 안전하게 인터넷 쓰기
3. 친구·팀원에게 QR 링크로 접속권 주고, 만료일이 지나면 자동 차단

**꼭 기억할 것**
1. 24시간 켜진 리눅스 서버가 필요하다
2. UDP 51820, TCP 51821 포트를 쓴다
3. 관리 화면은 HTTPS(리버스 프록시)로 보호한다

---

## 3. 질문별 답변

### 3-1. 설치 및 사용법
준비물: 공인 IP나 도메인이 있는 리눅스 서버(x86_64/arm64), 도커

```bash
# 도커 설치
curl -sSL https://get.docker.com | sh

# compose 파일 받기
sudo mkdir -p /etc/docker/containers/wg-easy
sudo curl -o /etc/docker/containers/wg-easy/docker-compose.yml \
  https://raw.githubusercontent.com/wg-easy/wg-easy/master/docker-compose.yml

# 실행
cd /etc/docker/containers/wg-easy
sudo docker compose up -d
```

- 방화벽: UDP 51820 개방 (클라우드는 보안그룹도)
- 관리 화면 접속
  - 추천: Caddy/Traefik 리버스 프록시로 `https://vpn.도메인`
  - 내부망 전용: `INSECURE=true` 설정 후 `http://서버IP:51821` (외부 노출 금지)
- 사용: 설정 마법사 → 새 클라이언트 → 폰은 QR 스캔, PC는 `.conf` 가져오기
- 업데이트: `docker compose pull && docker compose up -d`
- 무인 설치: `INIT_ENABLED`, `INIT_USERNAME`, `INIT_PASSWORD`, `INIT_HOST`, `INIT_PORT`
- 비밀번호 분실: `docker exec -it wg-easy cli` → 관리자 초기화

### 3-2. 플러그인? 스킬? MCP?
셋 다 아님. **독립 실행형 셀프호스팅 웹 애플리케이션**. 이 포크의 `CLAUDE.md`는 Claude Code 작업용 말투 설정일 뿐 기능과 무관. REST API가 있어 MCP 서버를 직접 만들어 붙일 수는 있음.

### 3-3. API 토큰이 필요한가?
필요 없음.
- VPN 기능은 외부 API를 쓰지 않음 (AI 토큰 불필요, 무료)
- wg-easy 자체 API(`/api/...`)는 관리자 아이디·비번 **Basic Auth** 사용. 2FA를 켠 계정은 API 사용 불가. API는 아직 불안정 단계라고 문서에 명시
- 선택 사항: Prometheus 지표 비밀번호(Bearer), OAuth 로그인 시 각 서비스의 Client ID/Secret

```python
import requests
r = requests.get("https://vpn.example.com/api/client", auth=("admin", "password"))
print(r.json())
```

### 3-4. AI 에이전트 구축에 도움이 되나?
- **인프라로 활용:** 에이전트, 로컬 LLM(Ollama), 벡터 DB, n8n 등을 VPN 사설망 안에서만 연결
- **MCP 서버로 확장:** wg-easy API를 감싸 Claude가 자연어로 VPN을 관리
  - 예: "철수 계정 만들고 30일 뒤 만료", "지금 접속 중인 사람은?", "퇴사자 비활성화"
  - 사용 API: `GET/POST /api/client`, `POST /api/client/{id}/disable`, `POST /api/client/{id}/generateOneTimeLink`

### 3-5. 수익화 아이디어 (요약)
1. 소규모 사업장 VPN 설치·운영 대행
2. wg-easy용 MCP 서버·관리 도구 (오픈소스 + 유료 지원)
3. 유튜브 → 유료 강의
4. 원클릭 설치 템플릿·스크립트 판매
5. 관리형 VPN 호스팅 (난이도 높음)

### 3-6. React나 PHP로 만들 수 있나?
| 방식 | 난이도 | 설명 |
|---|---|---|
| React 프런트만 새로 제작 | 중 | wg-easy를 백엔드로 두고 `/api`를 호출하는 대시보드. 가장 현실적 |
| React + Node 전체 재작성 | 상 | 키 생성 → `.conf` 생성 → `wg syncconf` 실행 로직을 재구현 |
| PHP(Laravel) 재작성 | 상 | `wg` 명령 실행 + MySQL. 웹서버에 root/NET_ADMIN 권한이 필요해 보안 설계가 까다로움 |

원본 코드를 복사해 쓰면 AGPL이 적용됨. 새로 작성하면 부담이 줄어듦.

### 3-7. 유튜브 강의로 제작 가능한가?
가능. 추천 시리즈:
1. VPN이란? 직접 만들면 좋은 이유 (Tailscale·상용 VPN 비교)
2. 무료 클라우드 서버 만들기
3. 도커 + wg-easy 10분 설치
4. 폰·PC 연결과 QR 실습
5. Caddy HTTPS + 2FA 보안 강화
6. AdGuard Home 조합 광고 차단 VPN
7. 홈서버·NAS 원격 접속
8. Claude + MCP로 말로 VPN 관리하기 (차별화 포인트)

불법 우회 용도는 다루지 않고 보안·원격 접속 중심으로 구성.

---

## 4. 수익화 아이디어 상세

> 금액·기간은 예시 가정이며 시장 조사 결과가 아님. 법률·세무 판단은 전문가와 확인 필요.

### 아이디어 1. 소규모 사업장 VPN 설치·운영 대행
- 대상: 직원 5~30명 사무실, 병원, 원격 근무 스타트업
- 제공: 설치 + HTTPS + 2FA, 계정 발급·회수·만료, 월 점검·백업, 접속 리포트
- 가격 구조: 설치비 1회 + 월 관리비
- 원본을 수정하지 않으므로 AGPL 부담이 거의 없음
- 시작: 재능 마켓 등록, 지인 사무실 무료 세팅으로 사례 확보

### 아이디어 2. AI로 관리하는 VPN — MCP 서버
- 도구: `create_client`, `list_connected`, `disable_client`, `send_one_time_link`, `traffic_report`
- 수익: 코어는 오픈소스, 유료로 멀티 서버 관리·Slack 승인 흐름·감사 리포트·기업 설치 지원
- 기존에 모아 둔 AI 에이전트·MCP 관련 저장소와 방향이 맞아 포트폴리오로도 유리

### 아이디어 3. 교육 콘텐츠
- 무료 유튜브 → 유료 강의(초급·중급·고급) → 1:1 컨설팅
- 부가 수익: 제휴 링크, 설정 템플릿 묶음

### 아이디어 4. 원클릭 설치 템플릿
- 클라우드 마켓플레이스 이미지, "wg-easy + Caddy + AdGuard + 자동 백업·업데이트" compose 템플릿, 한국어 운영 매뉴얼

### 아이디어 5. 관리형 VPN 호스팅 (SaaS)
- 고객별 인스턴스 + 월 구독
- 주의: AGPL(수정 시 소스 공개), "WireGuard" 상표 사용 금지, 트래픽 중계에 따른 법적 책임·약관·로그 정책, 장애 대응 부담
- 1~3번으로 수요 확인 후 도전 권장

### 추천 로드맵
| 단계 | 할 일 | 기간 |
|---|---|---|
| 1 | 직접 설치, 홈서버·AdGuard 연동 | 1주 |
| 2 | 유튜브 1~4편 제작 | 2~3주 |
| 3 | MCP 서버 MVP + GitHub 공개 + 유튜브 8편 | 2주 |
| 4 | 설치 대행 상품 등록, 첫 고객 사례 | 1개월 |
| 5 | 유료 강의·템플릿 판매 | 이후 |

핵심 전략: 콘텐츠(신뢰) → 오픈소스 도구(차별화) → 서비스(수익)

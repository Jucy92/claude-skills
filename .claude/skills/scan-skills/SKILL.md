---
name: scan-skills
description: 프로젝트 전체 구조를 스캔하여 KEYWORD_INDEX.md(경량 인덱스), scan-backend.md, scan-frontend.md, PROJECT_SCAN.md를 생성/갱신하고, verify-impact 파생 스킬을 생성합니다. 또한 기능 추가/구현 질문 시 navigate 모드로 키워드 기반 자동 참조를 수행합니다.
disable-model-invocation: true
argument-hint: "[선택사항: update(갱신만) | reset(전체 재생성) | navigate(질문 탐색)]"
---

# 프로젝트 구조 스캔 & 네비게이터 스킬

## 목적

두 가지 모드로 동작합니다:

1. **scan 모드** — 프로젝트 전체 스캔 → 4개 지식 파일 생성/갱신 + verify-impact 파생 스킬 생성
2. **navigate 모드** — 기능 질문 시 KEYWORD_INDEX.md 기반 선택적 파일 읽기 → 현재 구조 기반 정확한 답변

---

## 모드 1: scan 모드

### 트리거

- `/scan-skills` 명령어
- "구조 파악해줘", "전체 분석해줘", "이 프로젝트 어떻게 돼?" 요청
- `.claude/PROJECT_SCAN.md` 없을 때 (자동 제안)
- 새 도메인/엔티티 추가 후 갱신 필요 시

### Step 1: 모드 결정

```bash
ls .claude/PROJECT_SCAN.md 2>/dev/null && echo "EXISTS" || echo "NEW"
```

- **EXISTS + 인수 없음**: 갱신 모드 (변경된 도메인만 업데이트)
- **EXISTS + `reset`**: 전체 재생성 모드
- **NEW**: 최초 생성 모드

### Step 2: 프로젝트 타입 감지

```bash
ls backend/build.gradle 2>/dev/null && echo "BACKEND: SPRING_BOOT"
ls frontend/package.json 2>/dev/null && echo "FRONTEND: VUE/NODE"
```

### Step 3: 백엔드 도메인 스캔

**엔티티 필드 추출:**
```bash
grep -rn "private " backend/src/main/java/com/nearsplit/domain/*/entity/*.java
```

**서비스 메서드 추출:**
```bash
grep -rn "public " backend/src/main/java/com/nearsplit/domain/*/service/*Service.java
```

**API 엔드포인트 추출:**
```bash
grep -rn "@RequestMapping\|@GetMapping\|@PostMapping\|@PutMapping\|@PatchMapping\|@DeleteMapping" \
  backend/src/main/java/com/nearsplit/domain/*/controller/*Controller.java
```

**기술 스택 확인:**
```bash
grep -n "postgresql\|h2\|redis\|kafka" backend/src/main/resources/application.yml
grep -n "implementation\|runtimeOnly" backend/build.gradle | grep -i "postgres\|redis\|kafka"
```

### Step 4: 프론트엔드 스캔

```bash
grep -n "path:\|component:" frontend/src/router/index.js
grep -rn "import.*from.*api\|await.*(" frontend/src/views/*.vue | head -50
grep -rn "export\|axios\." frontend/src/api/*.js | head -50
```

### Step 5: 4개 지식 파일 생성/갱신

수집된 데이터를 기반으로 다음 파일들을 생성합니다:

#### 5a. `.claude/KEYWORD_INDEX.md` (경량, @auto-load)

```markdown
# NearSplit 키워드 인덱스
> scan-skills 자동 생성. 직접 편집 금지.

## 백엔드 도메인 키워드 맵
| 키워드 | 도메인 | scan-backend.md 섹션 |
|--------|--------|---------------------|
| 위치,거리,좌표,GPS,latitude,longitude,지도,반경,근처,PostGIS | split_group,user | #split_group, #user |
| 결제,payment,토스,toss,카드,취소,승인,환불,주문 | payment | #payment |
| 채팅,chat,WebSocket,STOMP,실시간,메시지 | chat | #chat |
| 인증,JWT,로그인,로그아웃,토큰,회원가입,쿠키,refresh | user | #user |
| 알림,notification,읽음,미읽음,푸시 | notification | #notification |
| 상품,product,검색,등록,쿠팡,외부API | product | #product |
| 그룹,소분,참여,승인,거절,모집,호스트,방장 | split_group | #split_group |

## 프론트엔드 키워드 맵
| 키워드 | 뷰 | scan-frontend.md 섹션 |
|--------|-----|----------------------|
| 홈,대시보드 | HomeView | #home |
| 그룹목록,GroupList | GroupListView | #group |
| 채팅화면,ChatView | ChatView | #chat |
| 결제화면,Checkout,PaymentSuccess | CheckoutView,PaymentSuccessView | #payment |
| 로그인,LoginView | LoginView | #auth |
| 프로필,ProfileView | ProfileView | #user |

## DB/인프라 키워드
| 키워드 | 현재 상태 |
|--------|----------|
| DB,데이터베이스,PostgreSQL | PostgreSQL 사용 중 (H2 주석처리됨) |
| H2 | 주석처리, 테스트 시에만 사용 |
| Redis | 설정만 존재, 구현 미완 |
| Kafka | 설정만 존재, 구현 미완 |
| PostGIS | 미적용 (latitude/longitude Double 타입으로만 저장) |
```

#### 5b. `.claude/scan-backend.md` (도메인 상세)

도메인별 섹션 구조:
- `## #meta` — 기술 스택, DB, 외부 API
- `## #user` — User 엔티티 필드, 서비스 메서드, API 엔드포인트
- `## #split_group` — SplitGroup/Participant 엔티티, 서비스, API (위치 필드 포함)
- `## #product` — Product 엔티티, 서비스, API
- `## #chat` — ChatMessage 엔티티, 서비스, WebSocket
- `## #notification` — Notification 엔티티, 서비스, API
- `## #payment` — Payment 엔티티, 서비스, API, Toss 연동

#### 5c. `.claude/scan-frontend.md` (프론트 상세)

섹션 구조:
- `## #auth` — LoginView, RegisterView, auth.js
- `## #user` — ProfileView, user.js
- `## #group` — GroupListView, GroupDetailView, GroupCreateView, group.js
- `## #chat` — ChatView, chat.js (WebSocket 포함)
- `## #product` — ProductListView, ProductCreateView, product.js
- `## #payment` — CheckoutView, PaymentSuccessView, PaymentFailView, payment.js
- `## #home` — HomeView

#### 5d. `.claude/PROJECT_SCAN.md` (전체 참조용, 기존 형식 유지)

### Step 6: verify-impact/SKILL.md 생성/갱신

PROJECT_SCAN.md의 도메인 정보를 반영하여 `.claude/skills/verify-impact/SKILL.md`를 생성합니다.

### Step 7: 연관 파일 편집 (기존과 동일)

7a. `manage-skills/SKILL.md` — verify-impact 행 추가 (이미 존재하면 스킵)
7b. `verify-implementation/SKILL.md` — verify-impact 행 추가 (이미 존재하면 스킵)
7c. `CLAUDE.md` — 아래 항목들 누락 시 추가 (이미 존재하면 스킵)
   - `@.claude/KEYWORD_INDEX.md` @auto-load 지시
   - Skills 테이블에 scan-skills, verify-impact 행
   - ⚡ 프로젝트 질문 처리 규칙 섹션 (⛔ FORBIDDEN 규칙 포함):
     ```
     > ⛔ **FORBIDDEN**: scan 파일 확인 전 소스 파일 직접 Read/Glob/Grep 절대 금지
     > ⛔ **FORBIDDEN**: 전체 `PROJECT_SCAN.md` 읽기 금지 (선택적 섹션만 읽기)
     > ⛔ **FORBIDDEN**: scan 파일 없이 임의로 추측하여 가이드 제공 금지
     > ✅ **REQUIRED**: 모든 구조 관련 질문은 반드시 KEYWORD_INDEX → scan-*.md 해당 섹션 순서로
     > ✅ **REQUIRED**: scan 파일 읽은 후에만 추가 정보 필요 시 소스 파일 접근 허용
     ```

### Step 8: 완료 보고서 출력

```markdown
## scan-skills 완료 보고서

### 생성/갱신된 파일
- `.claude/KEYWORD_INDEX.md`: [생성 / 갱신]
- `.claude/scan-backend.md`: [생성 / 갱신]
- `.claude/scan-frontend.md`: [생성 / 갱신]
- `.claude/PROJECT_SCAN.md`: [생성 / 갱신]
- `.claude/skills/verify-impact/SKILL.md`: [생성 / 갱신]

### 스캔 결과
- 도메인: N개
- API 엔드포인트: N개
- 프론트 페이지: N개
```

---

## 모드 2: navigate 모드

### 트리거 (자동 감지)

다음 패턴의 질문이 들어올 때 자동 활성화:
- "어떻게 구현해?", "어떻게 추가해?", "어디에 코드 작성해?"
- "~~ 기능 추가", "~~ 연동", "~~ 가이드"
- 특정 기능/도메인 관련 구현 질문

> **전제**: KEYWORD_INDEX.md가 CLAUDE.md @auto-load로 이미 컨텍스트에 있음

### 워크플로우

```
Step 1: 질문에서 키워드 추출
  예) "위치기반 거리 측정 구현" → [위치, 거리, 측정]

Step 2: KEYWORD_INDEX.md 키워드 매핑 확인 (이미 컨텍스트에 있음)
  → "위치" 매칭 → #split_group, #user 섹션
  → .claude/scan-backend.md #split_group 읽기

Step 3: 해당 섹션 정보 확인 후 답변
  → latitude/longitude Double 타입 확인
  → PostGIS 미적용 상태 확인
  → 현재 구조 기반 정확한 가이드 제공

Step 4 (키워드 미매칭):
  → 유사 키워드 검색 (동의어 포함)
  → 관련 scan-*.md 섹션 전체 스캔
  → #meta 섹션에서 기술 스택 확인

Step 5 (진짜 신규 기능):
  → "현재 프로젝트에 미구현 영역"임을 명시
  → 웹검색으로 구현 방법 조사
  → 기존 구조(패키지 구조, 패턴)에 맞는 방법 제안
```

### 퍼지 매칭 예시

| 질문 키워드 | KEYWORD_INDEX 매칭 | 읽는 섹션 |
|------------|-------------------|----------|
| 위치, GPS, 거리, 좌표, 반경 | 위치,거리,좌표,GPS | #split_group, #user |
| 결제, 취소, 토스, 카드 | 결제,payment,토스 | #payment |
| Redis, 캐싱, 캐시 | Redis → #meta | #meta (미구현 확인) |
| 블록체인, AI | 미매칭 → Step 5 | 웹검색 |

### navigate 모드 주의사항

> ⛔ navigate 규칙은 `CLAUDE.md` ⚡ 프로젝트 질문 처리 규칙 섹션에서 관리됨 (중복 정의 금지)
> scan-skills가 최초 실행될 때 Step 7c에서 CLAUDE.md에 해당 규칙을 추가함
- **현재 상태 기반** — 실제 구현 상태를 명시 (예: "PostGIS ✅ 적용 완료")

---

## 등록된 파생 스킬

| 스킬 | 설명 |
|------|------|
| `verify-impact` | 파일 수정 시 PROJECT_SCAN.md 기반으로 사이드 이펙트 추적 |

---

## Related Files

| File | Purpose |
|------|---------|
| `.claude/KEYWORD_INDEX.md` | @auto-load 경량 인덱스 (이 스킬이 생성) |
| `.claude/scan-backend.md` | 백엔드 도메인 상세 (이 스킬이 생성) |
| `.claude/scan-frontend.md` | 프론트엔드 상세 (이 스킬이 생성) |
| `.claude/PROJECT_SCAN.md` | 전체 지식 맵 (이 스킬이 생성/관리) |
| `.claude/skills/verify-impact/SKILL.md` | 파생 스킬 |

## 예외사항

1. **빈 프로젝트** — 파일 없어도 오류 없이 뼈대 파일 생성 후 종료
2. **키워드 미매칭** — navigate Step 4~5 워크플로우 진행
3. **외부 패키지** — `node_modules`, `.gradle` 탐색 범위에서 자동 제외
4. **테스트 파일** — `src/test/` 하위는 도메인 분석 시 제외
5. **KEYWORD_INDEX.md 없음** — navigate 모드 불가, scan 모드 먼저 실행 안내

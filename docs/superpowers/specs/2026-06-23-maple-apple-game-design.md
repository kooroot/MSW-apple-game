# 메이플 사과게임 (Maple Apple Game) — Phase 1 설계 스펙

- **작성일**: 2026-06-23
- **개정**: 2026-06-23 (코드리뷰 8건 반영 — 파일구조 / 랭킹 advanced / 랭크 시도권 영속 / 서버 시간검증 / PuzzleCore 결정성 / 모바일 보드·입력 / 페이로드 검증)
- **플랫폼**: MapleStory Worlds (MSW) — LocalWorkspace `world-dev`
- **레퍼런스 게임**: apple.oshizi.com (합이 10이 되도록 사과를 드래그로 묶어 제거하는 2분 퍼즐)
- **상태**: 설계 리뷰 대기 (이 문서 검토 후 writing-plans 단계로)

---

## 1. 개요 & 목표

격자에 1~9 숫자가 적힌 메이플풍 아이콘을 **드래그 사각형으로 선택 → 합이 정확히 10이면 제거 + 제거 개수만큼 득점 → 제한시간(120초) 내 최고 점수**를 겨루는 퍼즐 게임을 MSW에 런칭한다.

MSW의 차별점인 **소셜/경쟁(글로벌 리더보드)** 을 핵심 가치로 삼는다.

## 2. 범위 (Scope)

### Phase 1 (이번 스펙)
- 공용 **결정적** 퍼즐 코어 엔진 (클라·서버 동일 코드 공유)
- **랭크게임** 모드 (데일리 시드, 하루 1판, 서버 영속 시도권)
- **싱글플레이** 모드 (랜덤 보드, 무제한 연습 — **개인 최고 기록만**)
- 리더보드 UI: 랭크 탭(일간/주간/올타임) + 싱글 탭(개인 베스트) + 멀티 탭('준비중' 잠금)
- 서버 시드 발급 + **세션 토큰 + 서버 시간 권위** + 리플레이 점수 검증 (치팅 방지)
- 메이플풍 테마 + 모바일 우선 UI

### Phase 2 (다음, 별도 스펙)
- **멀티플레이** 모드 (방 공유 보드, 실시간 경쟁) → 멀티 탭 활성화
- Phase 1의 PuzzleCore / SeededRng / replay / 서버 검증을 그대로 재사용

### 명시적 비범위 (YAGNI)
- 아이템/상점/재화, 아바타 꾸미기, 친구 시스템, 시즌 보상 자동지급 — Phase 1 제외
- 난이도 선택, 보드 크기 가변 — 17×10 / 120초 고정
- **싱글 글로벌 랭킹** — 랜덤 보드는 플레이어 간 비교 불가(운+무제한 시도로 불공정·악용) → Phase 1 비범위. 싱글 탭은 개인 베스트만.

## 3. 모드 정의

| 모드 | 보드 | 횟수 | 리더보드 | Phase |
|------|------|------|----------|-------|
| **랭크게임** | 데일리 시드 (전원 동일 보드) | **하루 1판** (서버 영속 시도권) | 🏆 랭크 탭: 일간 / 주간 / 올타임 (글로벌) | 1 |
| **싱글플레이** | 랜덤 (서버 발급 시드) | 무제한 | 🎯 싱글 탭: **개인 최고 기록**(글로벌 보드 아님) | 1 |
| **멀티플레이** | 랜덤 (방 공유 시드) | 무제한 | ⚔️ 멀티 탭: 승/레이팅 | 2 |

- **랭크 시즌 정책**: 같은 "하루 1판" 데일리 점수를 ranking-advanced의 3개 사이클 보드(일/주/올타임)에 **동시 기록(팬아웃)**. 일간=오늘, 주간=그 주 베스트, 올타임=역대 베스트는 패키지의 `CycleEnum` + `force=false`(최고점 유지)로 자동 버킷팅(§7 참조).
- **싱글 리더보드 정정**: 보드가 매판 랜덤 + 무제한 시도 → 글로벌 비교는 운/시도수에 좌우되어 불공정하고 악용 가능. 따라서 **글로벌 보드를 두지 않고** 개인 최고 점수만 UserDataStorage에 저장해 싱글 탭에 표시한다. (사용자 합의 원칙 "랭킹은 전원 동일 퍼즐이어야 공정"의 직접 귀결.)

## 4. 아키텍처 개요

**UI 게임 + 얇은 호스트맵** 구조. 캐릭터 이동/물리 없음 → DefaultPlayer는 숨김. 전체 흐름이 UI(타이틀 → 모드선택 → 플레이 → 결과 → 리더보드).

```
[클라이언트 — UI/입력/로컬 플레이]            [서버 — 권위/검증/저장]
 BoardUI(.ui) ── GameSession ─┐              ┌─ SeedService       (시드+토큰 발급, 서버시간)
 LeaderboardUI ───────────────┼─── RPC ────┼─ ScoreService      (리플레이+페이로드 검증)
 (둘 다 PuzzleCore 사용)         │              ├─ RankAttemptStore (랭크 1일 1판 영속)
            └─ PuzzleCore / SeededRng ─┘      └─ LeaderboardStore (ranking-advanced 래핑)
                (결정적, 클라·서버 동일 코드)
```

- **클라**: 반응성 위해 보드를 로컬에서 즉시 굴림. 각 수(手)를 로그로 기록.
- **서버**: 같은 시드+수로그로 **재시뮬레이션(replay)** 해 점수를 검증한 것만 리더보드에 기록.
- UI는 **클라이언트 전용**(서버에서 UI 객체 nil). 점수 권위 로직은 **서버 ExecSpace**.
- 권위의 단일 원천 = **서버가 보관하는 세션 레코드 + 서버 시계**. 클라이언트 타임스탬프는 보조 정합성 검사에만 사용.

## 5. 모듈 (경계/책임 분리)

| 모듈 | 종류/위치 | 책임 | 의존 |
|------|----------|------|------|
| **SeededRng** | `@Logic` 공유 (Core) | 결정적 32비트 정수 PRNG(xorshift32). `math.random`/`_UtilLogic:RandomInteger*` 미사용(시드 재현 불가·플랫폼 비결정성). §9 | 없음 |
| **PuzzleCore** | `@Logic` 공유 (Core, 순수 로직) | `GenerateBoard(seed)→grid`, `ApplyMove(grid,rect)→{valid,clearedCount}`, `Replay(seed,moves)→{score,ok}`. UI·네트워크 무관. 클라·서버 **동일 인스턴스 코드** | SeededRng |
| **GameSession** | `@Component` 클라(호스트맵 엔티티), ClientOnly | 한 판 진행: 시드+토큰 요청 → 보드 구성 → 입력/드래그 → 로컬 점수·타이머 → 수로그 기록 → 종료 시 제출 | PuzzleCore, BoardController |
| **BoardController** | 클라 `.ui`(`ui/AppleGame/`)+컨트롤러 `.mlua` | 17×10 격자 렌더(아이콘+숫자), 드래그 사각형 오버레이+히트테스트(§11), 제거 연출 | msw-ui-system |
| **Hud/Result/Leaderboard Controller** | 클라 컨트롤러 `.mlua` | HUD(점수·시간바·포기), 결과 팝업, 리더보드 3탭 | msw-ui-system |
| **SeedService** | `@Logic` 서버, ServerOnly | 시드 발급(랭크=데일리/싱글=랜덤). **세션 토큰 발급 + 서버 시간 바인딩**(§10). 랭크는 RankAttemptStore로 1일 1판 소비 | RankAttemptStore |
| **ScoreService** | `@Logic` 서버 | `SubmitRun(token,moves,claimedScore)` → 토큰·서버시간·**페이로드 검증(§12)** → `PuzzleCore.Replay` 일치 시 기록 | PuzzleCore, LeaderboardStore, RankAttemptStore |
| **RankAttemptStore** | `@Logic` 서버 | `_DataStorageService` UserDataStorage 래핑. 키 `rankAttempt:{KST date}`. 발급 시점 소비(§8) | _DataStorageService |
| **LeaderboardStore** | `@Logic` 서버 | **ranking-advanced** 패키지(`PlayerRanking`) 래핑. 랭크 3사이클 보드 팬아웃 + 싱글 개인베스트(UserDataStorage) | ranking-advanced, _DataStorageService |
| **AssetCatalog** | 데이터 `@Logic`/테이블 | 숫자(1~9) → 메이플 아이콘 RUID 매핑 + 효과음 | msw-search / msw-painter |

## 6. 데이터 흐름 (한 판)

1. 플레이 시작 → `RPC(@ExecSpace("Server")) SeedService:RequestSeed(modeKey)` — `modeKey`는 문자열("ranked"/"single"); 엔진 enum은 RPC로 못 넘김(§ msw-scripting 6).
2. 서버:
   - 랭크면 RankAttemptStore에서 오늘(KST) 시도권 확인 → 이미 소비 시 거부. 미소비면 **발급 시점에 소비 기록(consumed)** 후 진행(§8).
   - 세션 토큰 생성 + 서버 레코드 `{userId, mode, seed, issuedAtServerTime, expiresAt, used=false}` 저장 → `{seed, token}` 반환(§10).
3. 클라가 `PuzzleCore:GenerateBoard(seed)`로 보드 생성 → 로컬 플레이, 각 수를 `{rect={c1,r1,c2,r2}, t}`(선택 사각형 + 상대 타임스탬프 초)로 로그.
4. 종료(120초 경과/포기) → `RPC(@ExecSpace("Server")) ScoreService:SubmitRun(token, moves, claimedScore)`.
5. 서버: 토큰 레코드 조회 → `senderUserId == record.userId` & `not used`(원자적 used=true) & `serverNow <= expiresAt` → 페이로드 검증(§12) → `Replay(seed,moves).score == claimedScore` 통과 시 기록:
   - 랭크: LeaderboardStore가 3사이클 보드에 팬아웃 + RankAttempt 레코드에 점수 갱신.
   - 싱글: 개인 베스트가 갱신되면 UserDataStorage 업데이트.
   → `{accepted, rank?, bestUpdated?}` 반환.
6. 클라(@ExecSpace("Client")): 결과 + 내 순위/베스트 표시 → 리더보드 화면.

## 7. 랭킹 패키지 결정 & 점수/타이머 규칙

### 7.1 패키지: ranking-advanced (확정)
- **ranking-basic은 단일 랭킹·시즌 미지원** → 우리 요구(일/주/올타임)와 불일치. **ranking-advanced** 사용.
- 의존성: `PlayerDBManager`, `PlayerRanking` (필요 시 player-data) 선설치. `WorldConfig.PlayerEntityAuthorityCheck = true`.
- **RankingConfigDataSet 3개**:

  | Id | Key | CycleEnum | ViewCount | 용도 |
  |----|-----|-----------|-----------|------|
  | 1 | `ranked_daily` | Day | 100 | 일간 |
  | 2 | `ranked_weekly` | Week | 100 | 주간 |
  | 3 | `ranked_alltime` | (무주기/대형 주기) | 100 | 올타임 |

  - `ReleaseBaseTime`은 **KST 자정 기준**으로 설정 → 데일리 시드 회전(§8)과 보드 회전 경계를 일치시킨다.
- **팬아웃 기록**: 검증 통과 시 `PlayerRanking:SetScoreAndWait(id, score, tag, force=false)`를 id=1,2,3에 호출. `force=false` → **최고점만 유지** → 주간=주 베스트, 올타임=역대 베스트가 패키지의 `CycleIndex` 버킷팅으로 자동 반영(수동 키 관리 아님). 이것이 코드리뷰 #5의 "주간/올타임 저장 방법"의 답.

### 7.2 ranking-advanced 제약 (UI·데이터에 반영)
- **비실시간**: 인스턴스별 `WorldInstanceSharedMemory` 갱신 주기(`RefreshIntervalMinutes`) 때문에 표시 순위가 지연될 수 있음. 패키지 문서: "정확한 *순위 기반* 보상엔 부적합, *점수 기반* 보상만 권장." → (a) Phase 1 **순위 기반 보상 없음**, (b) UI 카피는 실시간 단언 금지("갱신 지연 가능"), (c) '내 순위'는 근사값.
- **용량/포맷**: SharedMemory ≤ **80,000 byte**; `Tag`에 **`|` 금지**(구분자) → 표시명/extra에서 `|` 정제; `ViewCount` ≤ **1000**(top-100 OK).

### 7.3 점수·타이머
- 점수 = 제거한 아이콘 개수 누적. **선택 영역 잔여 칸 값의 합이 정확히 10일 때만** 제거, 점수 += 제거 칸 수.
- 이미 제거된 칸은 **0**으로 계산(여전히 합 10 필요) — 원작 규칙. 전부 제거된 칸만 선택하면 합 0 → 무효.
- 타이머 120초 클라 구동. **권위는 서버 시간**(§10): `issuedAtServerTime`→제출 수신 시각 차이가 `120 + grace` 이내여야 기록. 클라 `t[]`는 보조(단조성·창 내) 검사용.

## 8. 보드 생성 · 데일리 시드 · 랭크 시도권 영속 (코드리뷰 #2)

### 8.1 보드 생성 순서 (결정성 핵심)
- 17열 × 10행 = 170칸. **행 우선(row-major)** 채움: `for row=0..9 do for col=0..16 do grid[row][col] = rng:RangeInt(1,9) end end`. 정확히 170회 호출, 이 순서 고정(클라·서버 동일).

### 8.2 데일리 시드
- 서버 권위 **벽시계(KST)** 기반 `dayIndex` 산출 → `dailySeed = Mix32(dayIndex)`(§9 Mix32). 전원 동일, 하루 1회 회전.
- `dayIndex` 경계 = ranking-advanced `ReleaseBaseTime`(KST 자정)과 동일 기준 → 퍼즐과 일간 보드가 같은 순간에 회전.
- **미해결로 표시(plan-time 확인)**: 서버 벽시계 KST 날짜를 주는 정확한 API. `_UtilLogic.ServerElapsedSeconds`는 월드 인스턴스 수명 경과초라 **벽시계 아님**. writing-plans에서 `Environment/NativeScripts/Service/`·`Misc/`의 DateTime/시간 API를 `.d.mlua`로 확인(추측 금지, msw-scripting §1.3). ranking-advanced가 내부적으로 같은 시간원으로 사이클을 굴리므로, 가능하면 그 시간원에 정렬.

### 8.3 랭크 1일 1판 — 영속 모델 (토큰만으로 불충분)
- 토큰은 휘발성(세션) → 재접속/다른 방 입장으로 우회 가능. 따라서 **per-user 영속 레코드**로 보장.
- `RankAttemptStore` ← `_DataStorageService` **UserDataStorage**(유저별). 키 `rankAttempt:{KSTdate}`, 값 `{consumed=true, issuedAtServerTime, seed, token, score=nil, submittedAt=nil}`.
- **발급 시점 소비(consume at issuance)** — 제출 시점 아님:
  - 근거: 데일리 보드는 하루 종일 전원 **동일**이라 "리롤"이 불가능. 제한의 본질은 *그 고정 보드에 대한 채점 시도 횟수*. 제출 시 소비하면 나쁜 판을 버리고(미제출) 재시도 → 사실상 무제한이 되어 1일 1판이 무력화. 그러므로 발급 시 소비해야 공정(코드리뷰 #2의 권고와 일치).
  - 트레이드오프(정직히 명시): 발급 후 플레이 중 크래시/이탈 시 그날 시도권 소멸. Wordle류 데일리 챌린지 표준 모델 — 수용.
- 제출 성공 시 같은 레코드에 `score/submittedAt` 갱신. (시도 1회라 베스트 비교 불필요; 시즌 보드의 최고점 유지는 §7.1 `force=false`가 담당.)
- **크레딧 주의**: UserDataStorage 읽기/쓰기는 과금. 랭크 발급당 1 read+1 write, 제출당 1 write. **OnUpdate 금지**(msw-scripting §12).

### 8.4 싱글 시드 / 솔버블
- 싱글 시드: 시작마다 서버가 랜덤 32비트 발급 → 검증 가능하면서 매판 다름.
- 보드 솔버블 보장은 Phase 1 비범위(균등 난수면 사실상 다수 해 존재). 필요 시 후속.

## 9. SeededRng / PuzzleCore 결정성 명세 (코드리뷰 #6)

리플레이 검증이 게임의 신뢰축이므로 PRNG·보드·수 적용을 **모호함 없이** 고정한다. mlua는 Lua 5.3(정수 타입 + 비트연산 `& | << >>`)이라 비트 단위로 재현 가능.

### 9.1 SeededRng — xorshift32 (정수만, 부동소수 금지)
- 상태 `s`는 32비트로 마스킹된 정수. **시드 정규화**: `s = seed & 0xFFFFFFFF; if s == 0 then s = 0x9E3779B9 end`(0 상태 방지).
- `Next()`:
  ```
  s = (s ~ ((s << 13) & 0xFFFFFFFF)) & 0xFFFFFFFF
  s = s ~ (s >> 17)            -- Lua 5.3 >> 는 논리 시프트, 상위는 마스킹으로 0
  s = (s ~ ((s << 5)  & 0xFFFFFFFF)) & 0xFFFFFFFF
  return s
  ```
  (`~`는 mlua 비트 XOR. 좌시프트마다 `& 0xFFFFFFFF`로 32비트 유지.)
- `RangeInt(lo, hi)` = `lo + (Next() % (hi - lo + 1))`. 보드는 `RangeInt(1,9)`.
- `Mix32(x)`(데일리 dayIndex 해시) = `Next()`를 `x` 시드로 1회 적용한 값. 즉 시드 = `Mix32(dayIndex)`.

### 9.2 좌표·rect 규약
- 셀 좌표 `(col, row)`, `col∈[0,16]`, `row∈[0,9]`. 원점 = 좌상단, row 0 = 최상단.
- rect = `{c1,r1,c2,r2}` → **정규화**: `cLo=min(c1,c2), cHi=max(c1,c2), rLo=min(r1,r2), rHi=max(r1,r2)`.
- **양끝 포함(inclusive)**: 셀 (col,row)가 `cLo≤col≤cHi` 및 `rLo≤row≤rHi`이면 선택. 잔여 칸 값 합산.

### 9.3 골든 테스트 벡터 (필수)
- 고정 시드(예 `0x12345678`)에 대한 **전체 170칸 보드**(또는 첫 행 + 체크섬)와, 정해진 `moves` → 기대 `score`를 **고정값으로 커밋**. 구현 시 PuzzleCore 1회 실행해 산출·동결.
- 클라·서버는 **동일한 단일 Core `.mlua`** 를 공유(코드 중복 금지) → 결정성의 단일 원천. 테스트가 양측 보드/리플레이가 골든과 일치함을 어서션(§13).

## 10. 세션 토큰 + 서버 시간 권위 (코드리뷰 #3)

- SeedService가 발급하는 토큰은 **서버 보관 레코드**에 바인딩: `{userId, mode, seed, issuedAtServerTime, expiresAt = issuedAt + 120 + grace, used=false}`. 토큰 자체는 불투명 랜덤 문자열(검증은 서버 레코드 대조 → 별도 서명 암호화 불필요).
- 레코드 저장소: SeedService(@Logic, 서버) 인메모리 맵(세션 수명). 랭크 1일 보장은 §8.3 UserDataStorage가 별도 담당(인메모리는 재시작 시 소실되므로 영속 한계가 있어 1판 보장에는 못 씀).
- `SubmitRun` 검증 순서:
  1. 토큰 레코드 존재? (없으면 거부)
  2. `senderUserId == record.userId` (서버 부여, 클라 위변조 불가 — msw-scripting senderUserId)
  3. `not record.used` → **원자적으로 used=true**(재제출/중복 방지)
  4. `serverNow <= record.expiresAt` (**서버 시계** 기준 — 클라 `t[]` 아님)
  5. `record.mode == 제출 모드`
  6. 랭크면 §8.3 RankAttempt(오늘) consumed 레코드와 정합
- 클라 `t[]`는 §12의 보조 정합성(단조성·창 내)으로만.

## 11. 모바일 UI / 입력 명세 (코드리뷰 #7)

17×10 드래그 퍼즐은 셀 크기·방향·세이프에어리어·터치→셀 변환·선택 오버레이가 정해져야 UIBuilder 계획이 안정된다.

- **기준 해상도/방향**: 1920×1080(msw-ui-system §9 표준), **가로(landscape) 기본**. 근거: 17열 보드는 가로가 셀 폭을 키워 드래그 정밀도가 높음. (세로-fit도 가능하나 셀 ~63px로 작아짐 → §16에서 확정 대상으로 표시.)
- **레이아웃 수치(가로 기준)**: 상단 HUD ~140px, 여백 후 보드 가용 ≈ 1700×920. 정사각 셀 = `min(1700/17, 920/10) = min(100, 92) = 92px`. 보드 1564×920 중앙 정렬. (셀은 버튼이 아닌 드래그 타깃이지만 92px로 88×88 권고 이상 확보.)
- **세이프에어리어**: msw-ui-system §9 인셋 준수 — HUD/보드를 세이프 영역 안에, 노치 아래 핵심 컨트롤 배치 금지.
- **터치→셀 변환**: 포인터 화면좌표 → 보드 패널 로컬좌표 `local` → `col = clamp(floor(local.x / cellW), 0, 16)`, `row = clamp(floor(local.y / cellH), 0, 9)`. (UI y축 방향은 컨트롤러에서 보정해 row0=최상단 유지.)
- **입력(드래그)**: 보드 영역 전체를 덮는 **UI 터치 리시버**로 press/move/release를 받아 모바일 터치와 PC 좌클릭-드래그를 동일 처리.
  - press → `downCell` 기록; move → `currentCell` 갱신, 선택 오버레이 rect = `[downCell..currentCell]` 양끝 포함으로 스냅, 실시간 합 표시; release → `ApplyMove` 커밋(유효=제거+득점+성공음 / 무효=취소 피드백).
  - **plan-time 확인**: 정확한 이벤트/컴포넌트명(UITouchReceive 계열 vs `ScreenTouchEvent`+`IsPointerOverUI`)을 `.d.mlua`로 확정(추측 금지, msw-scripting §10). 메커니즘(press/move/release→셀 매핑)은 고정.
- **선택 오버레이**: 반투명 `SpriteGUIRenderer` 1개를 스냅된 셀 rect 경계로 매 move 리사이즈/이동 + 커서 근처 실시간 합 Text.
- 모든 `.ui`는 `ui/AppleGame/` 아래 두고 **UIBuilder로만** 작성(직접 JSON 편집 금지). 런타임 UI는 클라 전용.

## 12. 제출 페이로드 검증 (코드리뷰 #8)

`SubmitRun(token, moves, claimedScore)` — 리플레이 전/중 강제 검사. 위반 시 기록 거부 + 로그.

- **크기/개수**: `#moves > MAX_MOVES`(기본 256; 보드 170칸·클리어당 최소 2칸 → 유효 클리어 ≤85, 256은 안전 여유) 거부. 직렬화 바이트 상한 초과 거부.
- **좌표 범위**: 모든 rect `c∈[0,16]`, `r∈[0,9]` 아니면 거부.
- **타임스탬프**: 단조 비감소, 첫 값 ≥0, 마지막 ≤ `120 + grace`(초) 아니면 거부.
- **무효/노옵 수**: 정상 클라는 유효 클리어만 로깅. 리플레이 중 현재 격자에서 합10 유효 클리어를 못 만드는 수가 하나라도 있으면 **판 전체 거부**(빈 선택·전부-이미제거 선택 포함).
- **점수 정합**: `Replay(seed,moves).score == claimedScore` 정확히 일치, 아니면 거부 + 로그.
- **속도(소프트)**: 타임스탬프 분당 수 → 인간 가능 상한 초과 시 플래그/로그(비례적 완화, §13.x 치팅).
- **타입(msw-scripting §6)**: `moves`는 `table`(허용). 각 항목 `{rect={c1,r1,c2,r2}, t=number}`. `any`/엔진 enum RPC 금지 — 모드는 문자열 키.

## 13. 치팅 방지 모델 (한계 명시)

- **확실히 차단**: 위조 점수(점수만 조작 제출) → 리플레이 불일치로 거부. 토큰 미보유/만료/재사용/타인 토큰 → §10에서 거부. 랭크 2판째 → §8.3 영속 시도권으로 거부.
- **잔존 위험(솔직히 명시)**: 오프라인 솔버로 최적 수열을 계산해 제출하면 그 수열은 보드상 유효 → 리플레이 통과. 완전 차단은 서버 권위 타이밍(Phase 2)이라야 가능.
- **Phase 1 비례적 완화**: 랭크 1일 1판(영속) + 서버 시간 창 + 타임스탬프 단조/속도 상한 + 비현실적 점수 캡 + 의심 기록 로깅(사후 점검). 땜빵이 아니라 명시적·검증 가능한 한계로 둔다.

## 14. MSW 파일 구조 (코드리뷰 #1 — catch-all 금지)

`Scripts/`·`UI/` 같은 catch-all 폴더는 금지(msw-scripting §1.2). **기능 폴더 `AppleGame`** 아래로, `.ui`는 `ui/` 트리(msw-ui-system).

```
ui/
  AppleGame/
    AppleBoard.ui  AppleHud.ui  AppleResult.ui  AppleLeaderboard.ui
RootDesk/MyDesk/
  AppleGame/
    Core/    SeededRng.mlua  PuzzleCore.mlua                 (@Logic, 클라·서버 공유)
    Client/  GameSession.mlua  BoardController.mlua
             HudController.mlua  ResultController.mlua  LeaderboardController.mlua
    Server/  SeedService.mlua  ScoreService.mlua
             RankAttemptStore.mlua  LeaderboardStore.mlua
    Data/    AssetCatalog.mlua
  Models/
    AppleGame/   Background.model  등
map/
  AppleGame.map   (UI 호스트, DefaultPlayer 숨김)
```
> 구현/계획 단계 로드 순서: `msw-packages`(ranking-advanced + PlayerDBManager/PlayerRanking 의존성 확인·설치) → `msw-ui-system`(UIBuilder) → `msw-search`/`msw-painter`(에셋) → `msw-scripting`(로직). `.map`/`.model`/`.ui` 작성 전 `builder-protocol.md` 선독.

## 15. 테스트 전략

- **PuzzleCore 단위**(순수라 용이): 골든 벡터 어서션 — 같은 시드 → 클라·서버 동일 보드, `ApplyMove` 합10/이미제거 처리, `Replay` 점수 일치. `maker_play` + `maker_execute_script`로 실행, `maker_logs` 확인(msw-scripting 루프).
- **결정성 회귀**: §9.3 골든 벡터(보드 체크섬 + moves→score)를 고정해 변경 시 깨지도록.
- **검증 경로**: 위조 점수/만료 토큰/재사용 토큰/랭크 2판/범위초과 좌표/비단조 타임스탬프/무효 수 각각 거부되는지.
- **통합**: 모의 한 판 제출 → 3사이클 팬아웃 기록·근사 순위 검증.
- **수동**: 모바일/PC 드래그 UX, 세이프에어리어, 탭 전환, 잠금 탭.

## 16. 확정 기본값 (리뷰 시 조정 가능)

- 데일리 시드/사이클 기준 시간대: **KST**, 경계 = KST 자정(ranking `ReleaseBaseTime`과 일치).
- 화면 방향: **가로(landscape)** 기본 (세로-fit는 셀 축소 감수 시 대안 — 목업 단계 확정 대상).
- 랭킹 노출: **top-100**(ViewCount 100), 동점 시 **먼저 달성 우선**.
- 싱글 탭: **개인 최고점만**(글로벌 보드 없음 — §3 정정).
- 토큰 grace: 기본 **+5초**(네트워크 지연 여유). MAX_MOVES = **256**. ranking `RefreshIntervalMinutes` = **5분**(비실시간 표시 허용).

## 17. 결정 로그

- 최종 목표 = 솔로/랭킹 + 실시간 경쟁 멀티(둘 다). Phase 1은 랭크+싱글, 멀티는 Phase 2.
- 랭킹은 전원 동일 퍼즐이어야 공정 → 랭크 = 데일리 시드 + 하루 1판.
- **(정정)** 싱글 = 랜덤 무제한 → 글로벌 비교 불공정 → 개인 베스트만(글로벌 보드 비범위).
- 보드 17×10 / 120초 고정. 테마 메이플풍. 모바일 우선(가로) + PC.
- 권위 = 결정적 시드 + **세션 토큰 + 서버 시계** + 서버 리플레이; 시드는 서버 발급.
- **랭킹 패키지 = ranking-advanced**(basic은 단일·시즌 미지원). 일/주/올타임 = CycleEnum 3 config + `force=false` 팬아웃. 비실시간 → 순위 기반 보상 없음.
- 랭크 1일 1판 = **UserDataStorage 영속 시도권, 발급 시점 소비**(토큰만으로는 우회 가능).
- 치팅 방지의 솔버 잔존 위험은 명시하고 비례적으로 완화.
- 미해결(plan-time): 서버 벽시계 KST 날짜 API, UI 드래그 정확한 이벤트/컴포넌트명 — `.d.mlua`로 확정(추측 금지).

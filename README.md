# MSW Apple Game

[MapleStory Worlds(MSW)](https://maplestoryworlds.nexon.com/)에서 동작하는 **사과게임** 구현 프로젝트.

격자에 1~9 숫자가 적힌 메이플풍 아이콘을 **드래그 사각형으로 선택 → 합이 정확히 10이면 제거 + 제거 개수만큼 득점 → 120초 안에 최고 점수**를 겨루는 퍼즐 게임. (레퍼런스: apple.oshizi.com)

## 현재 상태 — 퍼블리시 완료

### Phase 1 — 싱글 / 랭크
- [x] 결정적 퍼즐 엔진 (`SeededRng` / `PuzzleCore`, 클라·서버 공유 + 골든벡터 회귀 테스트)
- [x] 서버 권위 (시드 발급·세션 토큰·서버 시계·리플레이 검증)
- [x] 랭킹 (ranking-advanced: 일 / 주 / 올타임 시즌 회전)
- [x] 클라이언트 게임 + UI (보드 드래그 / HUD / 결과 / 리더보드)
- [x] **싱글** (무제한 연습) + **오늘의 랭크** (일일 시드, 1일 1판)

### Phase 2 — 멀티플레이
- [x] 매칭 큐 + 매치 FSM (대기→매칭→카운트다운→레이스→결과)
- [x] 공유 시드 보드 실시간 경쟁 + 서버 권위 **ELO 레이팅** + 멀티 리더보드
- [x] **랜덤 매치** (2인 필수, 상대 못 찾으면 15초 후 취소)
- [x] **방 코드** — 4자리 코드로 방 생성 / 참가하는 사설 매치
  - 뎁스 내비게이션: `멀티플레이` → `랜덤 매치` / `방 코드` → `생성` / `참가` → `방 코드 입력`

### 폴리시
- [x] **타일 스킨** 12종 선택 (메인 화면 전용 셀렉터, 스킨별 숫자 가독성 보정)
- [x] **안티봇** — 서버 보드 스트리밍(시드 비노출) + 서버 시계 기반 입력 속도 휴리스틱
- [x] 퍼블리시된 라이브 버그 수정 + 타이틀 메뉴 정리

> ⚠️ **멀티 매칭 실측 한계** — Maker 에디터에는 로컬 플레이어가 1명뿐이라 실제 2인 매칭/방 참가는 **퍼블리시된 빌드**에서만 검증됩니다. 큐 타임아웃·교차 가드·UI 흐름 등 단일 플레이어로 검증 가능한 경로는 Maker에서 확인됨.

## 구조

```
RootDesk/MyDesk/AppleGame/   게임 스크립트 (mlua)
  Core/                      결정적 퍼즐 엔진 + 공용 자산 카탈로그 (클라·서버 공유)
  Client/                    UI 컨트롤러 (Title/Hud/Board/Result/Leaderboard/
                             Matchmaking/Room/MatchHud/MatchResult/SkinSelect) + GameSession
  Server/                    서버 권위 로직 (Match/Score/Leaderboard/Seed Service)
  Data/                      설정 / 데이터셋
RootDesk/MyDesk/RankingAdvanced/   랭킹 패키지 (일·주·올타임)
map/                         MSW 맵 (map01 — RectTile, UI 퍼즐)
ui/                          MSW UI (.ui)
docs/superpowers/            설계 스펙 & 구현 계획
```

## 빌드 / 실행

MapleStory Worlds **Maker** 에디터가 필요합니다. 이 워크스페이스를 Maker로 열고 **Refresh → Play**.

테스트는 MSW의 네이티브 루프(`maker_play` → `maker_execute_script` → `maker_logs`)로 실행합니다.

## 커밋 정책

MSW 엔진 정의(`Environment/`, `Global/`)와 로컬 에이전트/MCP 설정(`.mcp.json`, `.codex/`, `.claude/` 등)은 **서드파티 IP이거나 API 키를 포함**하므로 커밋하지 않습니다(`.gitignore` 참조). 클론 후 MSW Maker에서 열면 엔진 정의는 자동 제공됩니다.

## 설계 핵심

- **결정적 퍼즐 코어** — xorshift32 정수 PRNG로 시드→보드를 재현. 클라·서버가 동일 코드를 공유해 `Replay(seed, moves)`로 점수를 재검증.
- **서버 권위 + 치팅 방지** — 시드는 서버 발급, 세션 토큰 + 서버 시계로 시간 검증, 리플레이로 점수 위조 차단. 보드는 시드 대신 인코딩된 내용을 스트리밍해 오프라인 솔버를 차단하고, 입력 속도 휴리스틱으로 봇을 완화. 관측 가능한 보드의 한계는 명시하고 비례적으로 대응.
- **랭크 1일 1판** — UserDataStorage 영속 시도권(발급 시점 소비). 일/주/올타임 시즌은 ranking-advanced의 CycleEnum로 회전.
- **멀티플레이** — 논리적 매치 레지스트리 + 타깃 `@Client` RPC 팬아웃(`@Multicast` 미지원 빌드). 큐와 방을 상호배제(한 유저가 큐·방 동시 진입 시 이중 매칭되지 않도록 서버에서 교차 가드). MSW에는 스크립트 가능한 친구/파티 API가 없어 "친구와 하기"는 방 코드 로비로 구현.

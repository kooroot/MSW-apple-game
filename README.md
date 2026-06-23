# MSW Apple Game

[MapleStory Worlds(MSW)](https://maplestoryworlds.nexon.com/)에서 동작하는 **사과게임** 구현 프로젝트.

격자에 1~9 숫자가 적힌 메이플풍 아이콘을 **드래그 사각형으로 선택 → 합이 정확히 10이면 제거 + 제거 개수만큼 득점 → 120초 안에 최고 점수**를 겨루는 퍼즐 게임. (레퍼런스: apple.oshizi.com)

## 현재 상태 (Phase 1)

- [x] 설계 스펙 확정 — [`docs/superpowers/specs/2026-06-23-maple-apple-game-design.md`](docs/superpowers/specs/2026-06-23-maple-apple-game-design.md)
- [x] 구현 계획 Plan 1 (결정적 퍼즐 엔진) — [`docs/superpowers/plans/2026-06-23-apple-game-1-puzzle-engine.md`](docs/superpowers/plans/2026-06-23-apple-game-1-puzzle-engine.md)
- [ ] Plan 1 실행 — `SeededRng` / `PuzzleCore` + 골든벡터 회귀 테스트
- [ ] Plan 2 서버 권위 (시드·토큰·서버시계·리플레이 검증)
- [ ] Plan 3 랭킹 (ranking-advanced: 일/주/올타임)
- [ ] Plan 4 클라이언트 게임 + UI (보드 드래그 / HUD / 결과 / 리더보드)

## 구조

```
RootDesk/MyDesk/AppleGame/   게임 스크립트 (mlua)
  Core/                      결정적 퍼즐 엔진 (클라·서버 공유)
  Client/ Server/ Data/      Phase 진행에 따라 추가
map/                         MSW 맵
ui/                          MSW UI (.ui, Plan 4에서 추가)
docs/superpowers/            설계 스펙 & 구현 계획
```

## 빌드 / 실행

MapleStory Worlds **Maker** 에디터가 필요합니다. 이 워크스페이스를 Maker로 열고 **Refresh → Play**.

테스트는 MSW의 네이티브 루프(`maker_play` → `maker_execute_script` → `maker_logs`)로 실행합니다. 자세한 절차는 Plan 1 문서를 참고하세요.

## 커밋 정책

MSW 엔진 정의(`Environment/`, `Global/`)와 로컬 에이전트/MCP 설정(`.mcp.json`, `.codex/`, `.claude/` 등)은 **서드파티 IP이거나 API 키를 포함**하므로 커밋하지 않습니다(`.gitignore` 참조). 클론 후 MSW Maker에서 열면 엔진 정의는 자동 제공됩니다.

## 설계 핵심

- **결정적 퍼즐 코어** — xorshift32 정수 PRNG로 시드→보드를 재현. 클라·서버가 동일 코드를 공유해 `Replay(seed, moves)`로 점수를 재검증.
- **서버 권위 + 치팅 방지** — 시드는 서버 발급, 세션 토큰 + 서버 시계로 시간 검증, 리플레이로 점수 위조 차단. 한계(오프라인 솔버)는 명시하고 비례적으로 완화.
- **랭크 1일 1판** — UserDataStorage 영속 시도권(발급 시점 소비). 일/주/올타임 시즌은 ranking-advanced의 CycleEnum로 회전.

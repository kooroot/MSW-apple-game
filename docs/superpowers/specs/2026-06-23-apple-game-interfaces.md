# Apple Game — Cross-Subsystem Interface Contract (Plans 2–4)

> The authoritative API surface each subsystem **produces** and **consumes**, so the three plans compose without drift. Plan authors copy the relevant signatures verbatim into their tasks' `Interfaces` blocks. All names are final; change here first if a plan needs to deviate. Builds on the verified APIs in `2026-06-23-apple-game-api-gates.md` and the shipped Core (`AppleGame/Core`: `_SeededRng`, `_PuzzleCore`).

## Conventions
- mlua: `integer` ≠ `number`; `@Logic` accessed as `_<ExactScriptName>`; method doc comment first line inside body; no enums across RPC (pass integer/string); allowed RPC param types only.
- Folder layout: `RootDesk/MyDesk/AppleGame/{Core,Server,Client,Data}/`, `.ui` in `ui/AppleGame/`, `.map` in `map/`.
- Side rules: all persistence/ranking/seed = **ServerOnly**; all UI/input = **ClientOnly**; cross via `@ExecSpace("Server")` (client→server) and `@ExecSpace("Client")` (server→client, UserId appended at call site).
- `mode`: pass as **string** `"ranked"` | `"single"` across RPC (no enums).

## Shared move/run data shapes (Core, already shipped)
- Board: `_PuzzleCore:GenerateBoard(integer seed) -> grid[r][c]` (0-indexed, 17×10, values 1..9).
- `_PuzzleCore:ApplyMove(grid, rect{c1,r1,c2,r2}) -> {valid:boolean, clearedCount:integer}`.
- `_PuzzleCore:Replay(integer seed, table moves) -> {score:integer, ok:boolean}`; `moves` = array of `{rect={c1,r1,c2,r2}, t=number}` (t = client seconds-since-start, used only for time-window sanity, not for scoring).
- Seed derivation (server): `dailySeed = _SeededRng:Mix32(dateKey)` where `dateKey = year*10000 + month*100 + day` (KST).

---

## Plan 2 — Server Authority (produces)

### `_SeedService` (`@Logic`, Server/SeedService.mlua)
Issues seeds + session tokens; consumes the daily ranked attempt at **issuance** (anti-retry).
- `@ExecSpace("Server") method void RequestSeed(string mode)` — client entry. `mode="ranked"`: compute KST `dateKey`, CAS-consume `rankAttempt:{dateKey}`; if already consumed reply `alreadyPlayed=true`. `mode="single"`: random seed, no attempt consumption. Generates a session token, stores server-side session, then replies to the caller via the Client RPC below.
- `@ExecSpace("Client") method void ReceiveSeed(string mode, integer seed, string token, integer dateKey, boolean alreadyPlayed, integer personalBest)` — server→caller. (`targetUserId` appended at call site; do not declare it.)
- Server-side session record (in-memory table, keyed by token): `{userId, seed, dateKey, mode, issuedAtServerElapsed, consumed=false}`. Token is single-use for submit.
- Random single-mode seed: derive from server entropy that the client cannot predict (e.g. `_SeededRng:Mix32` of a server-side counter + `DateTime.UtcNow` ms); never trust a client-supplied seed.

### `_ScoreService` (`@Logic`, Server/ScoreService.mlua)
Validates and records a completed run.
- `@ExecSpace("Server") method void SubmitRun(string token, table moves, integer claimedScore, number clientElapsed)` — validates: (1) token exists, belongs to `senderUserId`, not yet consumed; (2) `clientElapsed` and server elapsed since issuance both within the 120s game window + slack (reject absurd timing); (3) `local r = _PuzzleCore:Replay(session.seed, moves)`; accept iff `r.ok and r.score == claimedScore`. On accept: mark token consumed, update `personalBest` (keep-max), and for `mode=="ranked"` call `_LeaderboardService:SubmitScore(senderUserId, r.score)` (Plan 3). Reply via Client RPC.
- `@ExecSpace("Client") method void ReceiveSubmitResult(boolean accepted, integer finalScore, integer personalBest, string reason)` — server→caller. `reason` ∈ `"ok"|"bad_token"|"timing"|"replay_mismatch"|"already"`.
- `@ExecSpace("Server") method void RequestPersonalBest()` — client entry for the Leaderboard "Single" tab. Reads `_RankAttemptStore:GetPersonalBest(senderUserId)` and replies via the Client RPC below. No token needed (read-only, idempotent).
- `@ExecSpace("Client") method void ReceivePersonalBest(integer personalBest)` — server→caller; delivered to `_LeaderboardController` via the same server→client bridge as `ReceiveSubmitResult`. (`targetUserId` appended at call site, not declared.)

### `_RankAttemptStore` (folded into SeedService or standalone, Server/)
- `method boolean TryConsumeDaily(string userId, integer dateKey)` — ServerOnly; `UpdateAndWait("rankAttempt:"..dateKey, "", "1")` on the user's UserDataStorage; returns true if newly consumed, false if already set. (First-write fallback: GetAndWait → if NotFound, SetAndWait.)
- `method integer GetPersonalBest(string userId)` / `method void SetPersonalBestIfHigher(string userId, integer score)` — UserDataStorage `personalBest` (string-serialized integer).

**Time source:** `DateTime.UtcNow + TimeSpan(9,0,0)` → `.Year/.Month/.Day` (GATE A). Server elapsed for the window check: store `DateTime.UtcNow` at issuance and diff at submit.

---

## Plan 3 — Ranking Integration (produces; consumes ScoreService)

### Package install (ranking-advanced)
- Copy `Util/` + `RankingAdvanced/Core/` into `RootDesk/MyDesk/RankingAdvanced/`; add `PlayerDBManager` + `PlayerRanking` to `DefaultPlayer` (Global — **user does this in Maker**, AI cannot edit Global); set `WorldConfig.PlayerEntityAuthorityCheck=true`; author `RankingConfigDataSet.csv` with 3 rows: Day(CycleEnum=1), Week(CycleEnum=2), AllTime(CycleEnum blank/None=0). Each row sets `ViewCount` (≤1000), `RefreshIntervalMinutes` (≥30), `ReleaseBaseTime` (KST boundary anchor).
- Board ids (final): `BOARD_DAY=1`, `BOARD_WEEK=2`, `BOARD_ALLTIME=3` (match CSV `Id`).

### `_LeaderboardService` (`@Logic`, Server/LeaderboardService.mlua)
- `@ExecSpace("ServerOnly") method void SubmitScore(string userId, integer score)` — fan out to all 3 boards: for each board, `cycleIndex = configData:GetCycleIndex(_DateTimeLogic:GetTimeElapsed())`, then `playerRanking:SetScoreAndWait(boardId, cycleIndex, score, tag, extra, false)` (force=false keep-max). `tag` = player display name (≤55 bytes, **no `|`**); `extra` = "" (or per-player metadata). Requires `PlayerDBManager.IsLoadSuccess`; skip/queue if not loaded yet.
- `@ExecSpace("Server") method void RequestLeaderboard(integer boardId, integer pageNo)` — read snapshot: `snap = _RankingDataStorageLogic:GetRankingSnapshot(boardId)`; `validPageNo = snap:GetValidPageNo(pageNo, PAGE_SIZE)`; `page = snap:GetRankingDataList(validPageNo, PAGE_SIZE)`; `mine = snap:GetRankingDataByProfileCode(myProfileCode)`. Reply via Client RPC. Handle `snap==nil` (InstanceRoom / not built yet) → empty + flag.
- `@ExecSpace("Client") method void ReceiveLeaderboard(integer boardId, table entries, integer myRank, integer myScore, boolean stale)` — `entries` = array of `{rank:integer, name:string, score:integer}`. `stale=true` always conveyed so the client can show "갱신 주기 약 30분".
- `PAGE_SIZE` final: `20`.

**Staleness (accepted):** public leaderboard is a ≥30-min snapshot; the player's own just-submitted score is shown immediately from ScoreService's `personalBest`, and the board labels its refresh cadence. Not real-time by design.

**Client tab layout:** the client renders **5 tabs** — Day/Week/AllTime (ranking boards 1/2/3 above) + Single (player's personal best, fetched via `_ScoreService:RequestPersonalBest`) + Multi (locked static placeholder, Phase 2). Boards 4 and 5 are **client-side UI sentinels** (`BOARD_SINGLE=4`, `BOARD_MULTI=5`) and are **never** passed to `RequestLeaderboard` (which only accepts 1/2/3).

---

## Plan 4 — Client Game (consumes all above)

### `_GameSession` / `BoardController` (`@Component`, ClientOnly, on the AppleGame map entity)
- On map enter: call `_SeedService:RequestSeed(mode)`; on `ReceiveSeed`, `grid = _PuzzleCore:GenerateBoard(seed)`, render board, start 120s timer, begin recording `moves`.
- Drag selection (GATE D): full-screen `UITouchReceiveComponent` panel; `UITouchDownEvent`→anchor, `UITouchDragEvent`→current corner (draw selection rect overlay), `UITouchUpEvent`→finalize. Map screen→cell via `_UILogic:ScreenToLocalUIPosition(TouchPoint, gridTransform)` ÷ cell size. Gate `TouchId==1` (mobile single-finger) / `-1` (PC). On a valid clear, call `_PuzzleCore:ApplyMove` locally for instant feedback AND append `{rect, t}` to `moves`.
- On timer end: `_ScoreService:SubmitRun(token, moves, clientScore, clientElapsed)`; on `ReceiveSubmitResult`, show Result UI (accepted/score/personalBest, or rejection reason).
- Leaderboard UI: **5-tab** popup — Day/Week/AllTime (call `_LeaderboardService:RequestLeaderboard(boardId, page)`, render `ReceiveLeaderboard` entries + stale label), Single (call `_ScoreService:RequestPersonalBest()`, show one "내 최고 점수: N" line from `ReceivePersonalBest`, no list, no stale label), Multi (locked static placeholder for Phase 2 — lock icon + "준비 중", no data call). Client-only UI sentinels: `BOARD_SINGLE=4`, `BOARD_MULTI=5` — never passed to `RequestLeaderboard`.

### UI (built via UIBuilder, `ui/AppleGame/`)
- HUD (score, 120s timer bar, mode), board grid panel + touch panel + selection overlay, Result popup, Leaderboard popup (5 tabs). DefaultPlayer avatar hidden (renderer off; entity remains for ranking components).

**Gates to confirm live in Plan 4:** UITouchDrag PC threshold, RaycastTarget requirement on the touch panel, midnight rollover, weekly cycle start day.

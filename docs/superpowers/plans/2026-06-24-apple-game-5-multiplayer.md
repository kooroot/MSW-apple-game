# Apple Game — Phase 2: Multiplayer (Private Grouped Matches)

> SDD plan. Base commit `4a16566` (Phase 1 complete + Gaps A/B). Authored 2026-06-24.
> Methodology: subagent-driven-development (implementer → review-package → task reviewer → fix → ledger per task → final whole-branch review). Ledger: `.git/sdd/progress.md`.

## 1. Context

Phase 1 shipped a solo Fruit Box clone on `map01` (RectTile, `TileMapMode=1`): drag a rectangle over a 17×10 grid of 1–9 apples summing to 10; score = cells cleared; 120s timer; ranked (daily seed, 1 play/day KST) + single modes; server-authoritative anti-cheat (token + time window + payload validation + `PuzzleCore.Replay` rescore); a RankingAdvanced leaderboard (boards 1/2/3 = day/week/all-time).

Phase 2 adds **private grouped multiplayer matches** (user scope decision: "인스턴스 룸 사설 매치"): N players are matched onto a **shared seeded board**, race for 120s, and the highest server-validated score wins; results adjust an **ELO rating** persisted on a new leaderboard board (id 4, "멀티 레이팅"). The design doc's Phase 2 line ("멀티 탭: 승/레이팅") is realized as: a Multiplayer entry on the title, a matchmaking lobby, a live scoreboard during the race, a win/lose/draw result with rating delta, and the leaderboard's **Multi tab flipped from locked placeholder to the live rating board**.

### 1.1 MSW platform constraints that shape this design (empirically probed in Maker)

- **No `@Multicast`** in this build. Server→all-clients = `@ExecSpace("Client")` called without `targetUserId`; **scoped** broadcast = loop the roster and call with `targetUserId` as the last call-site arg. Client→server = `@ExecSpace("Server")` (implicit `senderUserId`).
- **`@Sync`/`@TargetUserSync` replicate primitives only** — tables do NOT replicate. Send collections via RPC (see §2.4 marshalling rule).
- **Instance rooms are publish-time only.** `RoomService:GetOrCreateInstanceRoom` returns nil in Maker (`[LEA-3002]` "no instance-map designated"); `MoveUserToStaticRoom` does not move the lone local player. Physical room isolation requires an editor-designated `MapComponent.InstanceMap=true` map + a published multi-user world. Therefore the Maker-verifiable build groups players **logically** on `map01` and isolates them by **roster-scoped RPC** (every outbound RPC loops only the match roster; every inbound RPC verifies `senderUserId ∈ rosterSet`). The physical-instance-room path is implemented behind a `MatchVenue` seam, gated off (`USE_INSTANCE_ROOMS=false`) and unreachable in Maker by design.
- **`@Logic` lifecycle runs on BOTH server and client.** A script-level `@ExecSpace` does NOT stop `OnBeginPlay`/`OnUpdate` from running on the wrong side. Server-authoritative FSM ticks MUST guard `if not self:IsServer() then return end`. (`IsServer()`/`IsClient()` are **methods** — colon-call.)
- **Maker has exactly one local player.** With `MATCH_SIZE=1` a lone player forms an instant 1-player match ("a match of one") that exercises ~95% of the logic. True 2-player head-to-head, cross-client scoped delivery, the instance-room venue, and real-leaver forfeit are **publish-only** (documented, not Maker-verifiable).

## 2. Architecture

### 2.1 New modules

| Module | Type | Path | Responsibility |
|---|---|---|---|
| `MatchService` | `@Logic` (server-authoritative; client-side thin RPC bridge) | `RootDesk/MyDesk/AppleGame/Server/MatchService.mlua` | Match registry + matchmaking queue + FSM + scoped RPC fanout + settlement (ELO, rating persist). |
| `MatchVenue` | `@Logic` | `RootDesk/MyDesk/AppleGame/Server/MatchVenue.mlua` | Isolation seam. `LogicalVenue` (default) vs `InstanceRoomVenue` (gated). `MatchService` never names `RoomService` directly. |
| `MatchRatingStore` | `@Logic` | `RootDesk/MyDesk/AppleGame/Server/MatchRatingStore.mlua` | Per-user `multiRating` (default 1000) + `multiWins/multiLosses/multiDraws` in UserDataStorage, RankAttemptStore-style fail-closed. |
| `MatchmakingController` | `@Logic` `ClientOnly` | `RootDesk/MyDesk/AppleGame/Client/MatchmakingController.mlua` | Lobby: Searching state, match-found + countdown overlay, Cancel. |
| `MatchHudController` | `@Logic` `ClientOnly` | `RootDesk/MyDesk/AppleGame/Client/MatchHudController.mlua` | Compact live scoreboard strip during the race. |
| `MatchResultController` | `@Logic` `ClientOnly` | `RootDesk/MyDesk/AppleGame/Client/MatchResultController.mlua` | WIN/LOSE/DRAW banner, rating delta, standings, back-to-lobby. |
| `AppleMatchmaking.ui` | `.ui` | `ui/AppleMatchmaking.ui` | Matchmaking lobby UI (UIBuilder). |
| `AppleMatchHud.ui` | `.ui` | `ui/AppleMatchHud.ui` | Live scoreboard strip UI (UIBuilder). |
| `AppleMatchResult.ui` | `.ui` | `ui/AppleMatchResult.ui` | Match result UI (UIBuilder). |
| `MatchTest` | `@Logic` `ServerOnly` (test harness) | `RootDesk/MyDesk/AppleGame/Server/MatchTest.mlua` | ELO table tests + registry-injection settlement tests (callable via `maker_execute_script`). |

### 2.2 Modified modules (exact seams in §3 task specs)

`SeedService` (multi seed + shared-token issuance), `ScoreService` (extract server-internal `ValidateRun`), `LeaderboardService` (board-4 rating + `SubmitRating`), `RankingConfigDataSet.csv` (append board 4), `GameSession` (+`multi` mode), `HudController` (+`멀티` label), `TitleController` (+Multiplayer button), `LeaderboardController` (+Multi-tab flip), `AppleTitle.ui` (+button). **`map01.map` needs NO edit** (GameSession already registered). `ResultController` is **unchanged** (multi uses its own result UI).

### 2.3 RPC contract (the wire boundary)

Inbound (client→server, `@ExecSpace("Server")` on `MatchService`, implicit `senderUserId`):
- `EnqueueMatch(string mode)` — join the queue.
- `CancelQueue()` — leave the queue.
- `ReportLiveScore(string matchId, integer score)` — throttled cosmetic live score during racing.
- `ReportFinished(string matchId, string token, table moves, integer claimedScore, number clientElapsed)` — submit the full run at timer expiry.

Outbound (server→roster, `@ExecSpace("Client")` on `MatchService`, **scoped** = looped per roster userId with `targetUserId` as the last call-site arg):
- `OnMatchFound(string matchId, table rosterNames, integer mySlot)`
- `OnCountdown(string matchId, integer n)`
- `OnMatchStart(string matchId, integer seed, string token, table rosterNames)`
- `OnLiveScoreboard(string matchId, table names, table scores, table userIds)`
- `OnMatchResult(string matchId, string winnerName, string outcomeForMe, integer myRatingDelta, integer myNewRating, table names, table scores)`

Each client-side `@Client` handler bridges to the matching controller (guarded with `isvalid`/`~= nil`), exactly like Phase 1's `ReceiveLeaderboard → LeaderboardController`.

Server-internal (`ServerOnly`, no RPC): `SeedService:NewMatchSeed`, `SeedService:IssueSharedFor`, `ScoreService:ValidateRun`, `LeaderboardService:SubmitRating`, `MatchRatingStore:ApplyMatchResult`, `MatchVenue:Allocate/Place/Release`.

### 2.4 Cross-boundary marshalling rule (Global Constraint)

Only `string`/`integer`/`number`/`boolean`/`table`/`Vector*`/`Color`/`Entity`/`Component` cross an RPC; `any` and engine enums do NOT. **Collections are sent as parallel flat arrays of primitives** (e.g. `names: table<string>`, `scores: table<integer>`, `userIds: table<string>`) — never nested tables, never delimiter-joined strings (player names may contain `:`/`|`). Per-recipient derived values (`mySlot`, `outcomeForMe`, `myRatingDelta`, `myNewRating`) are **precomputed server-side per recipient** so no enum/derivation crosses the boundary.

### 2.5 Server FSM (authoritative, in `MatchService._T`)

`IDLE → QUEUED → MATCHED → COUNTDOWN → RACING → FINISHED → RESULTS → IDLE`

Driven by a 1 Hz server tick (TimerService repeat, guarded `if not self:IsServer() then return end`):
1. **QUEUED**: `EnqueueMatch` adds `senderUserId` to `_T.queue`. Tick pops up to `MATCH_SIZE` queued users into a new match (in Maker `MATCH_SIZE=1` → instant 1-player match).
2. **MATCHED**: mint `matchId` (`'m_'..seq`), `sharedSeed = SeedService:NewMatchSeed(matchId)` once, `MatchVenue:AllocateVenue` + `PlaceUsers`, `SeedService:IssueSharedFor(matchId, userId, sharedSeed)` per member, scoped `OnMatchFound`.
3. **COUNTDOWN**: 3-second countdown, scoped `OnCountdown(3/2/1)`.
4. **RACING**: scoped `OnMatchStart(seed, perPlayerToken, rosterNames)`; clients build the board + start a local 120s timer. `ReportLiveScore` updates `match.scores[userId]` (cosmetic) and fans `OnLiveScoreboard`. A server deadline (`RACE_DEADLINE = 125s`) force-finishes non-reporting clients with their last validated score (0 if none).
5. **FINISHED**: when all finished (or deadline), for each finished run call `ScoreService:ValidateRun`; winner = highest validated `finalScore`; equal-top = draw for all tied; forfeit/no-validated-run = 0. Compute ELO deltas, `MatchRatingStore:ApplyMatchResult`, `LeaderboardService:SubmitRating`. Scoped `OnMatchResult` with per-recipient `outcomeForMe`/`myRatingDelta`/`myNewRating`. Set `settled=true` BEFORE the yielding persist calls to block re-entry.
6. **RESULTS → IDLE**: `MatchVenue:ReleaseVenue`; clear `_T.matches[matchId]` and every `_T.userToMatch[userId]` for the roster.

`MatchService` subscribes to `UserLeaveEvent` (the `PlayerDBManager.HandleUserLeaveEvent` pattern): a leaver mid-match is marked finished with score 0 (forfeit); if one remains they auto-win; `_T.queue`/`_T.userToMatch` are cleaned on leave/cancel/settle. Because a `@Logic` survives map transitions and gets no `OnMapLeave`, **all per-match state cleanup is explicit**.

### 2.6 State shape

```
_T = {
  queue        = { [userId]=true },
  matches      = { [matchId]=MatchRecord },
  userToMatch  = { [userId]=matchId },
  nextMatchSeq = integer,
}
MatchRecord = {
  id, roster (array of userId), rosterSet ({[userId]=slot}),
  sharedSeed:integer, dateKey:integer, state:string,
  scores ({[userId]=liveScore}),         -- cosmetic, untrusted
  finalScores ({[userId]=validatedScore}),-- authoritative (Replay)
  finished ({[userId]=bool}), tokens ({[userId]=token}),
  names ({[userId]=displayName}),
  winnerUserId, settled:boolean, venueHandle,
  createdAt, countdownEndsAt, deadlineAt,
}
```
All match truth is in-memory `@Logic` state (non-synced, server-instance), exactly like `SeedService._T.sessions` / `RankAttemptStore._T.inFlight`. Single-threaded `@Logic` ⇒ RPC handlers are mutually atomic; do not `AndWait` between a check and its mutation. Yielding writes (`MatchRatingStore`, `SubmitRating`) run only after `settled=true`.

### 2.7 Rating (ELO)

Base 1000, `K=32`. For an N-player match, pairwise vs every other roster member, summed and divided by `(N-1)`:
`E_ij = 1/(1+10^((R_j - R_i)/400))`; `S_ij = 1.0` win / `0.5` draw / `0.0` loss; `delta_i = round( K/(N-1) * Σ_j (S_ij - E_ij) )`. **N=1 ⇒ delta=0** (solo-practice semantics; keeps the Maker 1-player match rating-neutral). `round(x) = x>=0 and floor(x+0.5) or ceil(x-0.5)`.
**Persist:** `newRating = max(1, old + delta)` — **floor 1, NOT 0** (`PlayerRanking.SetScoreAndWait` rejects `score<1`). `LeaderboardService:SubmitRating` writes board 4 with **`force=TRUE`** (rating can DECREASE — unlike the keep-max day/week/all-time boards). Wins/losses/draws are counters in `MatchRatingStore` for display; board 4 ranks by rating.

### 2.8 Venue seam

`MatchVenue` is the only module that may name `RoomService`/`DynamicMapService`. Interface: `AllocateVenue(matchId, roster)->handle`, `PlaceUsers(handle, roster)->bool`, `ReleaseVenue(handle)`. `USE_INSTANCE_ROOMS=false` (Maker default) selects `LogicalVenue` (handle=matchId, PlaceUsers no-op true, isolation = roster filter, everyone stays on `map01`). `USE_INSTANCE_ROOMS=true` (post-publish) selects `InstanceRoomVenue` (`CreateInstanceRoom` against an editor-designated `InstanceMap=true` map + `MoveUsers`); unreachable in Maker by the probed constraints. The release delta is this one file + the flag + the editor InstanceMap flag — `MatchService`/`SeedService`/`ScoreService`/UI are byte-identical across venues.

## 3. Global Constraints (reviewers: copy verbatim into the attention lens)

1. **Phase 1 must not regress.** Single/ranked submit (`ScoreService.SubmitFor`), the daily 1-play lock, and the existing day/week/all-time boards must behave byte-identically. The `ValidateRun` extraction must preserve: every return shape including `personalBest` on all five rejection paths + success; the `MarkConsumed`-before-`Replay` ordering; the single-mode PB write and ranked-mode leaderboard fanout. `ServerAuthTest` must still pass.
2. **Rating floor is `max(1, ...)`, never `max(0, ...)`** — `PlayerRanking.SetScoreAndWait` rejects `score<1` (returns false, silent write-fail).
3. **Board 4 writes use `force=TRUE`** (rating can decrease). Copying the `SubmitScore` `force=false` keep-max pattern would freeze ratings at peak — that is a bug.
4. **Shared match seed uses a DISTINCT salt** from the ranked daily seed (`SeededRng:Mix32(dateKey)`) and from the token salts (`0x5A5A5A5A`/`0xA5A5A5A5`). Reusing the daily seed would let players farm the daily-board pattern.
5. **Scoped RPC is the isolation mechanism** (LogicalVenue): every outbound `@Client` call loops only `match.roster` with `targetUserId`; every inbound `@Server` handler verifies `senderUserId ∈ match.rosterSet` (or `∈ _T.queue` for queue ops) before mutating. No un-scoped fanout.
6. **`MatchService` server logic must guard `if not self:IsServer() then return end`** in `OnBeginPlay`/`OnUpdate`/the FSM tick/`UserLeaveEvent` handler — a `@Logic` lifecycle runs on both sides.
7. **Collections cross RPC as parallel flat arrays of primitives** (§2.4) — no nested tables, no delimiter-joined strings. Per-recipient derived values precomputed server-side.
8. **`N=1` match ⇒ rating delta 0** (no opponent). Settlement, persistence, board write, and result UI still run (winner = self).
9. **Client sentinel `BOARD_MULTI=5` stays the tab identity.** `BOARD_MULTI_RATING=4` is used ONLY as the `RequestLeaderboard` argument and as the normalize source in `OnReceiveLeaderboard`; it is NEVER assigned to `currentTab` and NEVER a `SelectTab` branch. The new server board id 4 numerically equals the client's `BOARD_SINGLE=4` sentinel — this is safe because single-mode uses `RequestPersonalBest`/`ReceivePersonalBest` (a separate bridge) and never reaches `OnReceiveLeaderboard`; document it in a comment.
10. **`.ui` only via UIBuilder** (no raw JSON edit/grep). After any `.ui` build, inject the controller's property-default UUIDs (binding injection) so bindings resolve. `.mlua` + Maker `refresh` pairing required.
11. **`SpawnService` parent ≠ nil** (not used here, but if any spawn is added, pass `self.Entity.CurrentMap`). World coordinates are world units (1 unit = 100 px); `.ui` uses anchoredPosition.
12. **mlua specifics:** method doc comment is the FIRST line inside the body; `integer` ≠ `number`; `@Logic` accessed as `_<ExactScriptName>`; `pcall` is available; reserved RPC param names (`self`/`senderUserId`/`targetUserId`/`messageOwnerEntity`) must not be redeclared.

## 4. Tasks

> Dependency order. Each task: one implementer subagent, then review-package + task reviewer, fix Critical/Important, ledger line, then next. Foundation seams (1–5) first so the engine (6–7) and clients (8–13) have stable interfaces. Verify Phase-1 non-regression right after Task 1.

### Task 1 — Extract `ScoreService:ValidateRun` (behavior-preserving refactor)

**Files:** `RootDesk/MyDesk/AppleGame/Server/ScoreService.mlua` (read `Server/ServerAuthTest.mlua` for the regression contract).

**Spec:** Extract a new `@ExecSpace("ServerOnly") method table ValidateRun(string userId, string token, table moves, integer claimedScore, number clientElapsed)` containing the current `SubmitFor` validation body (token existence + ownership; single-use check + `MarkConsumed`; server time window; `ValidatePayload`; `PuzzleCore:Replay` rescore with `r.score == claimedScore`). It returns `{ accepted = boolean, finalScore = integer, reason = string }` (no `personalBest`, no side effects). **Preserve the `MarkConsumed`-before-`Replay` ordering.** `SubmitFor` becomes a thin wrapper: call `ValidateRun`; on every return re-attach `personalBest = _RankAttemptStore:GetPersonalBest(userId)`; on `accepted` apply the existing single-mode PB write (`SetPersonalBestIfHigher`) and ranked-mode `LeaderboardService` fanout, returning `{accepted, finalScore, personalBest, reason}` exactly as before. The reason strings (`bad_token`/`already`/`timing`/`<pay.reason>`/`replay_mismatch`/`ok`) and values must be unchanged for single/ranked.

**Verify:** `npx`-style type-check N/A (mlua) → rely on LSP diagnose (auto) = 0 errors; Maker `refresh` → build logs 0 errors; run `ServerAuthTest` via `maker_execute_script` (server_main) and confirm the same pass output as Phase 1 (tampered score → `replay_mismatch`; reused token → `already`; foreign token → `bad_token`; valid single → PB write; valid ranked → leaderboard fanout). **Main session runs this Maker check before Task 2.**

### Task 2 — `SeedService`: multi seed + shared-token issuance

**Files:** `RootDesk/MyDesk/AppleGame/Server/SeedService.mlua`.

**Spec:** Add `@ExecSpace("ServerOnly") method integer NewMatchSeed(string matchId)` — fold entropy via the existing `MixEntropy` with a **distinct salt** (e.g. `0x3C3C3C3C` — confirm it is not already used; pick another distinct constant if so), seeded by `matchId`, returning a 32-bit seed. Add `@ExecSpace("ServerOnly") method table IssueSharedFor(string matchId, string userId, integer sharedSeed)` — mint a fresh token via `NewToken`, store `_T.sessions[token] = { userId=userId, seed=sharedSeed, dateKey=<KstDateKey>, mode="multi", matchId=matchId, issuedAt=<UtcNow>, consumed=false }`, and return `{ token=token, seed=sharedSeed }`. **No `TryConsumeDaily`** (multi is unlimited). Do not alter `IssueFor`/`NewSingleSeed`/the daily lock. `ValidateRun` consumes a multi token exactly like a single/ranked token (single-use).

**Verify:** LSP 0 errors; build 0 errors; `maker_execute_script` (server_main): `NewMatchSeed("m_test")` returns an integer; two `IssueSharedFor("m_test", "u1"/"u2", seed)` calls return distinct tokens with the same seed; the seed differs from `_SeedService` daily `Mix32(dateKey)`.

### Task 3 — `LeaderboardService` rating board + CSV row

**Files:** `RootDesk/MyDesk/AppleGame/Server/LeaderboardService.mlua`, `RootDesk/MyDesk/RankingAdvanced/RankingConfigDataSet.csv`.

**Spec:** Add `property integer BOARD_MULTI_RATING = 4`. Relax the `RequestLeaderboard` whitelist guard (currently rejects boardId ∉ {1,2,3}) to also accept `BOARD_MULTI_RATING`. Add `@ExecSpace("ServerOnly") method void SubmitRating(string userId, integer rating)` modeled on `SubmitScore` but writing ONLY board 4: resolve `configData = _RankingConfigDataSetLogic:GetData(4)`, `cycleIndex = configData:GetCycleIndex(elapsed)` (returns 1 for blank cycle), `playerRanking:SetScoreAndWait(4, cycleIndex, rating, tag, extra, true)` — **`force=true`**. Append to the CSV exactly one row (preserve the file's existing line-ending/BOM): `4,multi_rating,멀티 레이팅,,100,30,,Index,,2026-01-01,` (blank CycleEnum + blank MaxUserDataCount, matching board 3). Do NOT touch `.userdataset` (metadata only). No RankingAdvanced code change (board registration is generic).

**Verify:** LSP 0; build 0; Maker `refresh`; `maker_execute_script` (server_main): `SubmitRating("u1", 1500)` then `RequestLeaderboard(4,1)` returns a non-empty snapshot (force a snapshot with `TryForceUpdateSharedMemory(4)` if needed); confirm `force=true` lowers a value (submit 1016 then 1000 → board shows 1000); `RequestLeaderboard(6)` still rejected.

### Task 4 — `MatchRatingStore` (@Logic)

**Files:** `RootDesk/MyDesk/AppleGame/Server/MatchRatingStore.mlua`.

**Spec:** New `@Logic`. Per-user UserDataStorage keys: `multiRating` (default **1000** when missing), `multiWins`, `multiLosses`, `multiDraws` (default 0). Mirror `RankAttemptStore`'s fail-closed read pattern (`GetAndWait` → ERR_OK + ""/numeric → `tonumber` floored; treat errors as the default and log). Methods: `@ExecSpace("ServerOnly") method integer GetRating(string userId)`; `@ExecSpace("ServerOnly") method table ApplyMatchResult(string userId, integer delta, string outcome)` → `newRating = max(1, oldRating + delta)`, `SetAndWait("multiRating", tostring(newRating))`, increment the `multiWins/multiLosses/multiDraws` key matching `outcome` ("win"/"lose"/"draw"), return `{ rating=newRating, wins, losses, draws }`. Read→compute→Set is serialized (one match settles at a time per instance — same trade-off as `RankAttemptStore.SetPersonalBestIfHigher`).

**Verify:** LSP 0; build 0; `maker_execute_script` (server_main): `GetRating("u_new")` returns 1000; `ApplyMatchResult("u_new", 16, "win")` returns rating 1016, wins 1; `ApplyMatchResult("u_new", -2000, "lose")` floors rating at 1 (not 0) and persists.

### Task 5 — `MatchVenue` (@Logic) seam

**Files:** `RootDesk/MyDesk/AppleGame/Server/MatchVenue.mlua`.

**Spec:** New `@Logic`. `property boolean USE_INSTANCE_ROOMS = false`. Methods (all `ServerOnly`): `AllocateVenue(string matchId, table roster) -> string` (handle), `PlaceUsers(string handle, table roster) -> boolean`, `ReleaseVenue(string handle)`. When `USE_INSTANCE_ROOMS=false` (`LogicalVenue`): `AllocateVenue` returns `matchId`; `PlaceUsers` is a no-op returning `true`; `ReleaseVenue` is a no-op. When `true` (`InstanceRoomVenue`): `AllocateVenue` calls `_RoomService:CreateInstanceRoom(...)` against an `InstanceMap=true` map and returns the room key; `PlaceUsers` calls `MoveUsers`; `ReleaseVenue` destroys/empties the room. This branch is **gated and unreachable in Maker** (`[LEA-3002]`); guard each `RoomService` call with `pcall` and `log_warning` on failure so a misconfigured flag fails soft. `MatchVenue` is the ONLY module naming `RoomService`/`DynamicMapService`.

**Verify:** LSP 0; build 0; `maker_execute_script` (server_main): with `USE_INSTANCE_ROOMS=false`, `AllocateVenue("m1", {"u1"})` returns "m1", `PlaceUsers("m1", {"u1"})` returns true, `ReleaseVenue("m1")` no error.

### Task 6 — `MatchService` core: registry, queue, formation, racing, RPC fanout

**Files:** `RootDesk/MyDesk/AppleGame/Server/MatchService.mlua`.

**Spec:** New `@Logic`. Establish `_T` per §2.6 in `OnBeginPlay` (server only). `property integer MATCH_SIZE = 1`, `property number RACE_DEADLINE = 125`, `property number COUNTDOWN = 3`. Start a 1 Hz server tick (`_TimerService:SetTimerRepeat`, guarded `if not self:IsServer() then return end`) that advances the FSM (§2.5) through QUEUED→MATCHED→COUNTDOWN→RACING and enforces `RACE_DEADLINE`. Implement inbound `@Server` `EnqueueMatch(mode)`, `CancelQueue()`, `ReportLiveScore(matchId, score)` (verify `senderUserId ∈ rosterSet`; clamp/monotonic; store `match.scores`; fan `OnLiveScoreboard`). Implement outbound scoped `@Client` `OnMatchFound`/`OnCountdown`/`OnMatchStart`/`OnLiveScoreboard` (loop roster, `targetUserId` last arg; each bridges to `_MatchmakingController`/`_MatchHudController` guarded). Match formation: mint `matchId`, `sharedSeed = _SeedService:NewMatchSeed(matchId)` once, `_MatchVenue:AllocateVenue`+`PlaceUsers`, `_SeedService:IssueSharedFor` per member (store `tokens[userId]`), capture display names (`PlayerComponent`/`UserService`). **Leave settlement to Task 7** — for now `ReportFinished` may be a stub that marks finished and logs (Task 7 fills settlement). Apply Global Constraints 4/5/6/7.

**Verify:** LSP 0; build 0; `maker_execute_script` (server_main) drives a synthetic flow: inject a userId into the queue / call `EnqueueMatch` path → confirm logs `[Match] queued`, `[Match] formed matchId=... roster=1`, `[Match] sharedSeed=...`, countdown ticks, `OnMatchStart` dispatched. (Full play-mode bridge verification happens after Task 8.)

### Task 7 — `MatchService` settlement: ELO, rating persist, result fanout, forfeit

**Files:** `RootDesk/MyDesk/AppleGame/Server/MatchService.mlua`.

**Spec:** Add a pure `method integer ComputeDelta(integer myRating, table oppRatings, integer myScore, table oppScores)` implementing §2.7 (N=1 → 0; pairwise summed /(N-1); symmetric round). Implement `ReportFinished(matchId, token, moves, claimedScore, clientElapsed)` (`@Server`, verify roster membership) → store the submitted run; mark `finished[userId]`. When all finished or `RACE_DEADLINE` hit, run settlement: for each finished run `local v = _ScoreService:ValidateRun(userId, token, moves, claimed, elapsed)`; `finalScores[userId] = v.accepted and v.finalScore or 0`; winner = max finalScore (equal-top ⇒ draw for all tied; all-zero/forfeit handled); set `settled=true` BEFORE yielding writes; for each member compute `delta` (ELO vs others), `MatchRatingStore:ApplyMatchResult(userId, delta, outcome)`, `LeaderboardService:SubmitRating(userId, newRating)`; scoped `OnMatchResult(matchId, winnerName, outcomeForMe, myRatingDelta, myNewRating, names, scores)` per recipient (each bridges to `_MatchResultController`); `MatchVenue:ReleaseVenue`; clear `_T.matches[matchId]` + `_T.userToMatch` for roster. Subscribe `UserLeaveEvent` in `OnBeginPlay` (server only): a leaver mid-match → finished, score 0, forfeit; auto-resolve if one remains; also drop from `_T.queue`. `settled` guard prevents double-settlement.

**Verify:** LSP 0; build 0; `MatchTest` (Task 13) covers ELO + settlement deterministically. `maker_execute_script` (server_main): inject a synthetic 2-player match into `_T.matches` with `finalScores {A=50,B=30}` and call the settle entry → winner A, outcomes win/lose, deltas +16/−16 at 1000/1000, board-4 writes happen, `_T.matches[matchId]` cleared.

### Task 8 — `GameSession` multi mode + `HudController` label

**Files:** `RootDesk/MyDesk/AppleGame/Client/GameSession.mlua`, `RootDesk/MyDesk/AppleGame/Client/HudController.mlua`.

**Spec:** In `GameSession`: refactor the board-build/timer-start sequence from `OnReceiveSeed` (grid = `GenerateBoard(seed)`, `RenderBoard`, `BindInput` closure → `OnValidClear`, timer init) into a private helper `StartBoard(seed, token)` that `OnReceiveSeed` calls (preserving single/ranked behavior). Add `property string matchId = ""` and `method void StartMatchSession(string matchId, integer seed, string token)` that sets `self.mode="multi"`, `self.matchId=matchId`, `_HudController:SetMode("multi")`, then `StartBoard(seed, token)`. In `OnValidClear` after the score/HUD update, add `if self.mode == "multi" then _MatchService:ReportLiveScore(self.matchId, self.score) end` **throttled** (only on change + min ~250ms via an elapsed check). In `EndSession`, branch: `if self.mode == "multi" then _MatchService:ReportFinished(self.matchId, self.token, self.moves, self.score, clientElapsed) else _ScoreService:SubmitRun(self.token, self.moves, self.score, clientElapsed) end`. Reuse the existing `active`/`started` guard against double-init. In `HudController:SetMode`, add `elseif mode == "multi" then self.modeText.Text = "멀티"`.

**Verify:** LSP 0; build 0; full play-mode bridge test deferred to Task 9 (needs matchmaking entry). Confirm single/ranked still start a board and submit (quick `play` → single run → result).

### Task 9 — `MatchmakingController` + `AppleMatchmaking.ui`

**Files:** `RootDesk/MyDesk/AppleGame/Client/MatchmakingController.mlua`, `ui/AppleMatchmaking.ui`.

**Spec:** Build `AppleMatchmaking.ui` (UIBuilder): full-screen Dimmer + centered Panel with a Title ("멀티플레이"), a Searching label ("상대를 찾는 중..." + "x/N"), a Cancel button, and a match-found/countdown overlay (roster names + a large countdown number). New `@Logic` `ClientOnly` `MatchmakingController`: `Show()` (call from TitleController) → enable group + `_MatchService:EnqueueMatch("multi")`; `Hide()`; Cancel → `_MatchService:CancelQueue()` + Hide; bridge methods `OnMatchFound(rosterNames, mySlot)`, `OnCountdown(n)` (render the number), and on match start hand off to gameplay (the `OnMatchStart` bridge in `MatchService` calls `GameSession:StartMatchSession` on the map component + hides matchmaking). Inject all `.ui` UUID bindings into the controller's property defaults.

**Verify:** LSP 0; build 0; Maker `refresh`; `play`; click the (Task-12) Multiplayer entry → Searching shows → 1-player match forms → countdown 3-2-1 → board renders from the shared seed (log seed + a board checksum). Capture via `maker_logs` (build then runtime). Main session drives `maker_play`/`maker_mouse_input`/`maker_logs`.

### Task 10 — `MatchHudController` + `AppleMatchHud.ui`

**Files:** `RootDesk/MyDesk/AppleGame/Client/MatchHudController.mlua`, `ui/AppleMatchHud.ui`.

**Spec:** Build `AppleMatchHud.ui` (UIBuilder): a compact scoreboard strip (top-right) with a hidden `RowTemplate` (name + score). New `@Logic` `ClientOnly` `MatchHudController`: `Show()`/`Hide()`; `RenderScoreboard(names, scores, userIds)` clones `RowTemplate` per entry sorted desc, highlights the local player's row (accent), via the same clone pattern as `LeaderboardController.OnReceiveLeaderboard`. The `MatchService:OnLiveScoreboard` bridge calls it. Show on match start, hide on result. Inject bindings.

**Verify:** LSP 0; build 0; `play` a 1-player match → each valid clear updates the single scoreboard row live; confirm via runtime logs + on-screen.

### Task 11 — `MatchResultController` + `AppleMatchResult.ui`

**Files:** `RootDesk/MyDesk/AppleGame/Client/MatchResultController.mlua`, `ui/AppleMatchResult.ui`.

**Spec:** Build `AppleMatchResult.ui` (UIBuilder): Dimmer + Panel with a WIN/LOSE/DRAW banner, a rating line ("1000 → 1016 (+16)"), a final standings list (RowTemplate), and a "로비로" button. New `@Logic` `ClientOnly` `MatchResultController`: `ShowResult(outcomeForMe, winnerName, myRatingDelta, myNewRating, names, scores)` (bridged from `MatchService:OnMatchResult`); the lobby button returns to title (show `TitleController`, hide result + match HUD). Inject bindings. **`ResultController` is untouched.**

**Verify:** LSP 0; build 0; `play` a 1-player match to timer expiry → result shows WIN, rating "1000 → 1000 (+0)" (N=1 delta 0), standings 1 row; lobby button returns to title.

### Task 12 — Wire entry points: `TitleController` button + `LeaderboardController` Multi-tab flip + `AppleTitle.ui`

**Files:** `RootDesk/MyDesk/AppleGame/Client/TitleController.mlua`, `RootDesk/MyDesk/AppleGame/Client/LeaderboardController.mlua`, `ui/AppleTitle.ui`.

**Spec:** `AppleTitle.ui` (UIBuilder): add a `BtnMulti` button (reflow the 3-button vertical stack cleanly, e.g. Ranked / Multi / Single, 460×120, ~150px spacing), label "멀티플레이". `TitleController`: add `property ButtonComponent btnMulti = "<new-uuid>"` (injected), connect it in `OnBeginPlay`, add `method void OnMulti()` → `_MatchmakingController:Show()` (NOT `StartMode`); disconnect in `OnEndPlay`; `Hide()` the title when entering matchmaking. `LeaderboardController`: add `property integer BOARD_MULTI_RATING = 4`; flip the `SelectTab` `BOARD_MULTI` branch from `ShowMultiPanel()` to `ShowRankingPanel()` + `if _LeaderboardService ~= nil then _LeaderboardService:RequestLeaderboard(self.BOARD_MULTI_RATING, 1) end`; in `OnReceiveLeaderboard`, add `if boardId == self.BOARD_MULTI_RATING then boardId = self.BOARD_MULTI end` **before** the `if boardId ~= self.currentTab then return end` guard; repurpose `multiLocked` as the empty-state label (shown when board 4 returns no rows). Add the Constraint-9 comment about the 4↔BOARD_SINGLE coincidence. Inject the new title button UUID.

**Verify:** LSP 0; build 0; `play`: title shows 3 buttons; Multiplayer → matchmaking; open Leaderboard → Multi tab now shows the live rating board (after a `SubmitRating` + `TryForceUpdateSharedMemory(4)`), not the locked placeholder; other tabs unchanged.

### Task 13 — `MatchTest` harness (ELO + settlement, no 2nd client)

**Files:** `RootDesk/MyDesk/AppleGame/Server/MatchTest.mlua`.

**Spec:** New `ServerOnly` `@Logic` mirroring `PuzzleCoreTest`/`ServerAuthTest` style, with a `RunAll()` callable via `maker_execute_script`. Table-driven `ComputeDelta` tests: (1000 vs 1000, win) → +16; (1000 vs 1000, lose) → −16; draw → 0; (1200 vs 1000, win) → +8; N=1 → 0; an N=3 mixed case → per-pair sum /(N-1) with correct rounding. Settlement tests via direct registry injection: build a `MatchRecord` with two fake roster userIds + `finalScores`, call the settle entry, assert winner/outcomes, `MatchRatingStore` deltas (floor ≥ 1 on a large negative), board-4 write, and `_T.matches` cleanup. Assert with `log`/`log_error` PASS/FAIL lines.

**Verify:** `maker_execute_script` (server_main) `_MatchTest:RunAll()` → all PASS in runtime logs.

## 5. Integration verification (main session, after all tasks)

Single-player Maker run (`MATCH_SIZE=1`, `USE_INSTANCE_ROOMS=false`), via `maker_play` + `maker_mouse_input`/`maker_keyboard_input` + `maker_logs` (build first, then runtime) + `maker_execute_script`:
1. Build clean (0 errors) after `refresh`.
2. Title → Multiplayer → Searching → instant 1-player match → countdown → board renders from the shared seed (log seed + checksum).
3. Clears push `ReportLiveScore`; MatchHud row updates live.
4. Timer expiry → `ReportFinished` → `ValidateRun` accepts → winner=me, delta=0, `multiRating` persisted (1000).
5. Leaderboard → Multi tab → `RequestLeaderboard(4)` renders (force a snapshot via `TryForceUpdateSharedMemory(4)`).
6. `_MatchTest:RunAll()` all PASS.
7. Regression: single + ranked still play and submit; daily lock still triggers "오늘은 끝!".

**Publish-only (documented, NOT Maker-verifiable):** real 2+ player same-seed race; scoped-RPC isolation across concurrent real matches; live scoreboard showing OTHER players; real winner/rating between distinct humans; `InstanceRoomVenue` (`[LEA-3002]`/`MoveUser` no-op confirmed); real-leaver forfeit.

## 6. Final review & finish

Whole-branch review (most capable model) against §3 Global Constraints + this plan. Fix Critical/Important in one batched fix wave. Then secret-scan and commit per the session security rules (explicit safe paths only; never `git add -A`; HEREDOC trailers; scan `git diff --cached` for the project API key and any `bearer <token>` literal — the staged diff must contain neither). **Push is user-gated** (public repo `kooroot/MSW-apple-game`).

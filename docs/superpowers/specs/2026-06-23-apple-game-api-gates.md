# Apple Game — API Gate Findings (Plans 2–4 de-risk)

> Read-only research pass verifying the blocking APIs for the server-authority, ranking, and client plans **before** authoring them. Every claim is cited to `Environment/NativeScripts/**/*.d.mlua` (local engine defs) or to `MSW-Git/MSWPackages` source. Date: 2026-06-23.

Status: **all four gates PASS** (feasible), with three corrections to prior assumptions (all in GATE C).

---

## GATE A — Server wall-clock / KST date  ✅ (Plan 2)

**Use `DateTime.UtcNow`** — `static readonly property DateTime UtcNow` (`Misc/DateTime.d.mlua:44-45`). It is a `@Misc` value type with **no `@ExecSpace`**, so it is safe inside `@ExecSpace("ServerOnly")` code and reads the **server** system clock (client cannot influence it).

KST derivation (pure arithmetic, no timezone API):
```lua
local kst = DateTime.UtcNow + TimeSpan(9, 0, 0)        -- UTC+9; TimeSpan(hours,min,sec), TimeSpan.d.mlua:60
local dailyKey = kst.Year * 10000 + kst.Month * 100 + kst.Day   -- e.g. 20260623
```
- `DateTime` fields (all `int32`): `Year`(:49) `Month`(:38) `Day`(:8) `Hour`(:20) `Minute`(:34) `Second`(:42) `Millisecond`(:27); `DayOfWeek`→`DayOfWeekType` Sun=0..Sat=6 (:12). Operators `+`/`-` with `TimeSpan` and comparisons declared (`DateTime.d.mlua:88`).

**Rejected:** `_UtilLogic.ServerElapsedSeconds`/`.ElapsedSeconds` (instance-lifetime monotonic, NOT wall-clock — resets on world reboot); `_UtilLogic:GetUTCNow()` (deprecated, returns string); `_UtilLogic:GetLocalTimeFrom()` (`ClientOnly` + uses client timezone → tamperable). There is **no dedicated TimeService**.

**Gaps:** `DateTime.Elapsed` epoch is unspecified — derive the date from named fields, not from `Elapsed`. Verify midnight rollover (`UTC 16:00 + 9h` → next KST day) with a runtime `log()` during Plan 2.

**Design implication:** the daily seed and the 1-play-per-day boundary both key off `dailyKey` computed server-side. The puzzle seed for a given KST day = a deterministic function of `dailyKey` (feed it through `_SeededRng:Mix32`).

---

## GATE B — DataStorage (per-player persistence)  ✅ (Plan 2)

**`_DataStorageService`** — every method `@ExecSpace("ServerOnly")` (`Service/DataStorageService.d.mlua`). Acquiring a storage object is free (0 credits):
- `GetUserDataStorage(string userId) → UserDataStorage` (:127) — per-player
- `GetGlobalDataStorage(string name) → GlobalDataStorage` (:97) — world-wide
- `GetSortableDataStorage(string name) → SortableDataStorage` (:112) — integer values, sorted reads

**UserDataStorage** (`Misc/UserDataStorage.d.mlua`, all ServerOnly, values are **strings** — serialize yourself):
- Read: `GetAndWait(key) → (int32 errCode, string value)` (:53). Multi-return, no handle. `errCode 1000002 = NotFound`.
- Write: `SetAndWait(key, value) → int32` (:83).
- **CAS: `UpdateAndWait(key, expectedValue, newValue) → (int32, string)` (:123)** — the atomic primitive for "set the rank-attempt flag only if not already set," avoiding a check-then-write race across two server instances.
- Batch: `BatchGetAndWait(List<string>) → (int32, DataStorageItemPages)` (:13), `BatchSetAndWait(Dictionary) → (int32, List<string>)` (:33). Cheaper for ≥2 keys.
- `GlobalDataStorage` has identical method surface (`Name` instead of `UserId`).

**SortableDataStorage** (`Misc/SortableDataStorage.d.mlua`): `IncreaseAndWait(key, delta) → (int32, integer)` (:93), `GetSortedAndWait(SortDirection, min, max)` (:83), `SetByInfoAndWait(DataStorageKeyInfo, score)`. This is the substrate ranking-advanced is built on, and the fallback if we ever build custom ranking.

**Caveats (credit-billed):** never call in `OnUpdate`/sub-1s timers; compare-before-write (identical write still costs); use Batch for multiple keys; value target ≤4KB (1 credit/4KB chunk; hard max 300KB); `Transact*` = 2× credits, ≤20 keys; **reading a missing key still costs**; key 1–100 bytes UTF-8; **Maker and release environments do NOT share storage**.

**Our keys (per-player UserDataStorage):** `rankAttempt:{YYYY-MM-DD}` (~24 B) and `personalBest` — both well under limits. `errCode` is `int32`; declare holding vars as `integer`.

**Gaps:** CAS-mismatch error code is undocumented (treat any non-zero as failure, re-read, retry). For first-ever write of a key, prefer `GetAndWait` then `SetAndWait` (dirty-checked) over `UpdateAndWait` (empty-expected behavior undocumented).

---

## GATE C — ranking-advanced package  ✅ with 3 CORRECTIONS (Plan 3)

Source: `MSW-Git/MSWPackages/ranking-advanced-package/MyDesk/RankingAdvanced/Core/*` + `Util/*` + README + `RankingConfigDataSet.csv`.

### Correction 1 — score-submit signature (was wrong)
```lua
@ExecSpace("ServerOnly")
method boolean PlayerRanking:SetScoreAndWait(integer id, integer cycleIndex, integer score, string tag, string extra, boolean force)
```
- **6 params, not 4.** Prior belief `(id, score, tag, force)` was missing `cycleIndex` (2nd) and `extra` (5th). No default for `force` (mlua has no optional params — always pass it).
- Compute `cycleIndex` yourself: `configData:GetCycleIndex(curElapsed)`, `curElapsed = _DateTimeLogic:GetTimeElapsed()`.
- `id` = the board's integer Id from the config dataset. `extra` = arbitrary per-player string stored in `UserRankingData.Extra` (NOT shown in the shared snapshot; read server-side via `GetUserData`). May be `""`.
- **force=false keeps the higher score** (`if not force and score <= userData.Score then return false`). force=true overwrites unconditionally. Confirmed.
- Guards: returns false if `self.Entity.PlayerDBManager.IsLoadSuccess` is false, or if `score < 1`.

### Correction 2 — CycleEnum has no "all-time" (was wrong)
`CycleEnum`: None=0, **Day=1, Week=2, Month=3, Year=4**. There is **no all-time constant.** A permanent board = `CycleEnum=None` (blank in CSV) → `GetCycleIndex` always returns 1. **A `None` board cannot distribute rewards** (validation). So our three boards = **Day(1)**, **Week(2)**, and **all-time via None(0)/blank**.

### Correction 3 — refresh floor is 30 minutes (sharper than "eventually consistent")
`RefreshIntervalMinutes >= 30` enforced in `RankingConfigData.Load()`. The leaderboard the player sees can be **up to 30+ minutes stale**. (See design implication below.)

### Confirmed
- **NON-real-time** by design: a leader instance reads `SortableDataStorage`, builds the ranked list, serializes to `WorldInstanceSharedMemory`; other instances serve that cached snapshot. A 60s timer checks each board's `NextUpdateTime`; actual rebuild per `RefreshIntervalMinutes`.
- **80,000-byte** SharedMemory cap per board (README). `ViewCount ≤ 1000`. Tag delimiter is `"|"` → **Tag must not contain `|`**, **MaxTagLen=55 bytes**. Tie-handling includes up to +10 entries past the cutoff.
- **"Not suitable for accurate rank-based rewards"** (README) — score-based rewards only; snapshot may be stale across instances.
- **InstanceRoom:** can submit scores but `GetRankingSnapshot` returns nothing (no leaderboard view in instance rooms).

### Read/query API (all from the in-memory snapshot, not live storage)
```lua
local snap = _RankingDataStorageLogic:GetRankingSnapshot(id)         -- RankingSnapshotData | nil
local page = snap:GetRankingDataList(validPageNo, pageSize)          -- table<RankingData{ProfileCode,Score,Tag}>
local mine = snap:GetRankingDataByProfileCode(profileCode)           -- RankingData | nil
local validPageNo = snap:GetValidPageNo(pageNo, pageSize)
```

### Dependencies & install
- On `DefaultPlayer`: **`PlayerDBManager`** (load/save lifecycle, exposes `IsLoadSuccess`) + **`PlayerRanking`**.
- Global Logic singletons: `_RankingDataStorageLogic`, `_RankingConfigDataSetLogic`, `_DateTimeLogic`, `_Util`.
- Ships its own `MyDesk/Util/` (`AdminLogic`, `Base64Logic`, `CycleEnum`, `DataLoadLogic`, `DateTimeLogic`, `DayOfWeekEnum`, `ServerTimeOffsetChangedEvent`, `Util`) — install alongside Core. **No external package dependency.**
- `WorldConfig.PlayerEntityAuthorityCheck = true` — security-recommended (README), not a hard runtime requirement.
- `RankingConfigDataSet.csv` header: `Id, Key, Name, CycleEnum, ViewCount, RefreshIntervalMinutes, HasReward, RankModeEnum, MaxUserDataCount, ReleaseBaseTime, Disable`. `ReleaseBaseTime` is the cycle boundary reference date used by `GetCycleIndex`.
- Install order: copy `Util/` → copy `RankingAdvanced/Core/` → add `PlayerDBManager`+`PlayerRanking` to DefaultPlayer → configure CSV → set WorldConfig flag.

**Gaps:** some files (`RankingConfigData`, `RankingSnapshotData`, `RankingConfigDataSetLogic`) were summarized by the fetch, not returned byte-for-byte — method names corroborated by usage in `RankingViewLogic.mlua`; `GetCycleIndex` exact formula and the weekly cycle start-day are high-confidence but not byte-verified. Confirm against the actual installed source during Plan 3.

---

## GATE D — UI drag-rectangle input  ✅ (Plan 4)

**Use `UITouchReceiveComponent`** on a full-screen UI panel over the board. It emits the `UITouch*` event family; each event carries `TouchId` (`int32`: left-mouse `-1`, mid `-2`, right `-3`, mobile touch `≥1`) and `TouchPoint` (`Vector2`, **screen coords**). Component is a pure emitter (no settable props) and only fires when the pointer is over its UI — so no `IsPointerOverUI` filter needed.

Events (`Environment/NativeScripts/Event/UITouch*Event.d.mlua`):
- **`UITouchDownEvent`** → anchor corner (`dragStart`).
- **`UITouchDragEvent`** (extra field `TouchDelta:Vector2`) → current corner each frame while held.
- **`UITouchUpEvent`** → finalize selection.
- (`UITouchBeginDragEvent`/`UITouchEndDragEvent`/`UITouchEnter`/`UITouchExit` available, optional.)

Wiring (must be `@ExecSpace("ClientOnly")`; connect on the **Entity**, not the component):
```lua
self.downHandler = self.Entity:ConnectEvent(UITouchDownEvent, self.OnTouchDown)
self.dragHandler = self.Entity:ConnectEvent(UITouchDragEvent, self.OnDrag)
self.upHandler   = self.Entity:ConnectEvent(UITouchUpEvent,   self.OnTouchUp)
```
Screen→cell mapping (`Logic/UILogic.d.mlua`, ClientOnly):
- `_UILogic:ScreenToLocalUIPosition(TouchPoint, gridPanelUITransform)` (:27) — local space of the grid panel; divide by cell size to get `(row,col)`.
- `_UILogic:ScreenToUIPosition(TouchPoint)` (:32); `_UILogic.ScreenWidth`/`.ScreenHeight` (:8/:12) for normalization.

**Rejected:** `ScreenTouchEvent`/`ScreenTouchReleaseEvent` (`_InputService`) — **no drag event** in that family; mid-drag tracking would need `MouseMoveEvent` (delta only) or per-frame `GetCursorPosition()` polling. UITouch is cleaner and cross-platform.

**Gaps to verify in Plan 4 (live test):** PC drag firing threshold (slow drags may drop first pixels); whether `UITouchDragEvent` starts immediately after Down or only after BeginDrag (treat Down as the definitive anchor); gate on `TouchId == 1` for single-finger on mobile; whether the panel needs `RaycastTarget=true` for events to fire.

---

## Cross-gate design implications (feed into the plans)

1. **Server-authoritative daily flow (Plan 2):** server computes `dailyKey` from `DateTime.UtcNow + 9h`; daily seed = `_SeededRng:Mix32(dailyKey)`. Rank attempt = `UpdateAndWait("rankAttempt:"..dateStr, "", "1")` CAS — consume **at seed issuance**, not at submit, so a disconnect can't be exploited for a retry. Submission validates session token + server elapsed time window + replays moves via `_PuzzleCore:Replay(seed, moves)`; accept only if replayed score == claimed score.

2. **Leaderboard staleness is real (≥30 min) (Plan 3 + UX):** the **public** leaderboard list is a stale snapshot (≥30-min rebuild). Mitigation: after a ranked submit, show the player **their own** score/rank read server-side immediately (`GetUserData`/`GetRankingDataByProfileCode` on the freshest available snapshot) and label the public board "갱신 주기 약 30분" so the staleness is expected, not a bug. This fits a daily-seed puzzle (not live PvP) — recommend accepting it rather than building custom real-time ranking.

3. **Three boards = Day(1) + Week(2) + all-time(None/blank)** (Plan 3), each a CSV row with its own Id/Key. `tag` carries display name (≤55 B, no `|`); `extra` can carry anything per-player. Submit fans out one `SetScoreAndWait` per board with `force=false` (keep-max) and the per-board `cycleIndex`.

4. **DefaultPlayer hosts ranking components (Plan 3/4):** even though the apple game is a UI puzzle with the player avatar hidden, the `DefaultPlayer` entity still exists and must carry `PlayerDBManager`+`PlayerRanking`. Hiding the avatar (renderer) does not remove the entity, so this composes — confirm during Plan 3.

5. **Everything that persists or ranks is ServerOnly; all UI/input is ClientOnly** — the client→server submit and server→client result/leaderboard cross via `@ExecSpace("Server")`/`("Client")` RPCs (allowed param types only; no enums across the boundary — pass cycle/board as integers/strings).

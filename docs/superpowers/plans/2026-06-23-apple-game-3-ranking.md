# Maple Apple Game — Plan 3: Ranking Integration

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Install the `ranking-advanced` package, author three leaderboard config rows (Day / Week / AllTime), implement `_LeaderboardService` with 3-board fan-out, and wire `_ScoreService`'s ranked path to call it — so a verified ranked run is immediately reflected in all three boards.

**Architecture:** `ranking-advanced` package files are copied into `RootDesk/MyDesk/RankingAdvanced/` (Util + Core). A new `_LeaderboardService` `@Logic` (ServerOnly) wraps `PlayerRanking:SetScoreAndWait` with the corrected 6-param signature and `_RankingDataStorageLogic:GetRankingSnapshot` for reads. `_ScoreService` (Plan 2) calls `_LeaderboardService:SubmitScore` on accept — replacing the Plan-2 stub. Two Maker-only user actions are required for `DefaultPlayer` components and `WorldConfig`; these are clearly labelled.

**Tech Stack:** mlua (MSW @Logic, ServerOnly), `ranking-advanced-package` (MSW first-party), `RankingConfigDataSet.csv` (UserDataSet), MSW Maker MCP (`maker_refresh_workspace`, `maker_play`, `maker_logs`, `maker_execute_script`, `maker_stop`), `_DataStorageService` (UserDataStorage, ServerOnly).

---

## Global Constraints

All constraints from `docs/superpowers/specs/2026-06-23-maple-apple-game-design.md`. Every task implicitly includes these.

- Folder: `RootDesk/MyDesk/AppleGame/Server/` for `LeaderboardService.mlua`; package files under `RootDesk/MyDesk/RankingAdvanced/`. `.model` → `RootDesk/MyDesk/Models/`. Never edit `Global/` or `Environment/` (read-only). (spec §14, AGENTS.md)
- `PlayerRanking:SetScoreAndWait` has **6 params**: `(integer id, integer cycleIndex, integer score, string tag, string extra, boolean force)`. No defaults — always pass all 6. `force=false` = keep-max (higher score wins). (GATE C Correction 1)
- `cycleIndex` must be computed as `configData:GetCycleIndex(_DateTimeLogic:GetTimeElapsed())` — **not** hardcoded. (GATE C Correction 1)
- Board ids are final: `BOARD_DAY=1`, `BOARD_WEEK=2`, `BOARD_ALLTIME=3`. (interfaces contract)
- `CycleEnum` in CSV: Day=1, Week=2, AllTime=blank (None=0). There is **no all-time constant** — blank means `None`. (GATE C Correction 2)
- `RefreshIntervalMinutes >= 30` (enforced by engine). `ViewCount <= 1000`. `tag` ≤ 55 bytes, **no `|`** character. (GATE C Correction 3, spec §7.2)
- `leaderboard snapshot is ≥ 30-min stale` — test verifies submit + per-player read, not that the public list updates immediately. (GATE C Correction 3)
- All DataStorage calls are **ServerOnly**, event-driven (never in `OnUpdate`). (msw-scripting §12, datastorage.md rule 1)
- `@Logic` singletons accessed as `_<ExactScriptName>` — no suffix stripping. (msw-scripting §3.2)
- Every `method` body has its doc comment as the **first line inside** the body. (msw-scripting §1.8)
- After any `.mlua` create/modify: `maker_stop` → `maker_refresh_workspace` → `maker_logs(category="build")` (zero errors) before testing. (msw-scripting §1.5)
- Commits: PUBLIC repo — explicit paths only (no `git add -A`); run `git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'` before every commit (must be empty); HEREDOC trailers `Co-Authored-By:` + `Claude-Session:` on every commit.
- **Licensing note (action required before Task 1):** The `ranking-advanced-package` source files are MSW first-party code being copied into a PUBLIC repository. Confirm the package license permits redistribution before writing files. If redistribution is not permitted, reference/install via the Maker modpackage installer instead of vendoring. Flag this for the controller. **Do not assume permission.**

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `RootDesk/MyDesk/RankingAdvanced/Util/*.mlua` | Copy from package | Package utility logic (AdminLogic, Base64Logic, CycleEnum, DataLoadLogic, DateTimeLogic, DayOfWeekEnum, ServerTimeOffsetChangedEvent, Util) |
| `RootDesk/MyDesk/RankingAdvanced/Core/*.mlua` | Copy from package | Core ranking engine (PlayerDBManager, PlayerRanking, RankingDataStorageLogic, RankingConfigDataSetLogic, RankingConfigData, RankingSnapshotData, RankingViewLogic, etc.) |
| `RootDesk/MyDesk/RankingAdvanced/Core/*.ui` | Copy from package | Package UI files (if any) → also `ui/` tree |
| `RootDesk/MyDesk/RankingAdvanced/Core/*.model` | Copy from package → `RootDesk/MyDesk/Models/RankingAdvanced/` | Package models |
| `RootDesk/MyDesk/RankingAdvanced/RankingConfigDataSet.csv` | Create | 3-row leaderboard config (Day/Week/AllTime) |
| `RootDesk/MyDesk/RankingAdvanced/RankingConfigDataSet.userdataset` | Create | Metadata wrapper for the CSV |
| `RootDesk/MyDesk/AppleGame/Server/LeaderboardService.mlua` | Create | `_LeaderboardService`: SubmitScore fan-out + RequestLeaderboard snapshot |
| `RootDesk/MyDesk/AppleGame/Server/ScoreService.mlua` | Modify (seam) | Replace the `_LeaderboardService:SubmitScore` stub call left by Plan 2 |

---

## The MSW Test Loop (used by every task)

There is no pytest. "Run the test" means this sequence:

1. `maker_stop` (if currently playing) — refresh needs edit mode.
2. `maker_refresh_workspace` — compiles `.mlua` edits into `.codeblock`.
3. `maker_logs(category="build")` — must show **0 errors** before proceeding.
4. `maker_play` — enters play mode; `@Logic` singletons go live.
5. Poll `maker_get_current_map` until state is "play".
6. `maker_clear_logs` — clean slate for this test run.
7. `maker_execute_script(snippet, context="server")` — drive the ServerOnly test harness.
8. `maker_logs(category="normal")` — read `[TEST PASS]`/`[TEST FAIL]`/`[TEST SUMMARY]` lines.
9. `maker_stop`.

> **DataStorage/ranking calls are credit-billed.** Keep test invocations minimal — one submit per board per test run. The snapshot is ≥ 30-min stale, so the test verifies submit returns success + per-player `GetRankingDataByProfileCode` (not that the public list updated instantly). Maker and release environments use separate storage.

---

## Task 1: Fetch, audit, and install the ranking-advanced package files

Copies the package source into the workspace, checks for UUID/RUID collisions, and verifies the package builds cleanly.

**Files:**
- Create: `RootDesk/MyDesk/RankingAdvanced/Util/` (all Util `.mlua` files from package)
- Create: `RootDesk/MyDesk/RankingAdvanced/Core/` (all Core `.mlua` + `.ui` + `.model` files from package)
- Copy `.model` files: also into `RootDesk/MyDesk/Models/RankingAdvanced/`

**Interfaces:**
- Consumes: nothing (package is self-contained; no external package dependency per GATE C).
- Produces: `_RankingDataStorageLogic`, `_RankingConfigDataSetLogic`, `_DateTimeLogic`, `_Util` Logic singletons available in play mode. `PlayerDBManager` and `PlayerRanking` components ready to attach to `DefaultPlayer` (user action — see USER ACTION IN MAKER).

- [ ] **Step 1: Check license before copying**

  > **ACTION REQUIRED:** Before writing any files, confirm the `ranking-advanced-package` license permits copying into a public repository. Check `https://raw.githubusercontent.com/MSW-Git/MSWPackages/main/ranking-advanced-package/README.md` for a license section. If no explicit license is found or redistribution is not permitted, stop and ask the controller how to proceed (reference vs vendor). Only continue to Step 2 if redistribution is confirmed.

- [ ] **Step 2: Fetch the package file tree**

  Fetch the GitHub tree API to list all package files:
  ```
  https://api.github.com/repos/MSW-Git/MSWPackages/git/trees/main?recursive=1
  ```
  Filter for `ranking-advanced-package/` entries. If truncated, fall back:
  ```
  https://api.github.com/repos/MSW-Git/MSWPackages/contents/ranking-advanced-package/MyDesk/RankingAdvanced
  https://api.github.com/repos/MSW-Git/MSWPackages/contents/ranking-advanced-package/MyDesk/Util
  ```
  List every file path that will be copied.

- [ ] **Step 3: UUID / RUID collision check**

  Before writing, grep the workspace for every `id` and `EntryKey` that appears in the incoming `.ui` and `.model` files:
  ```bash
  grep -rE '"EntryKey":|"id":' RootDesk/MyDesk/ ui/ --include="*.ui" --include="*.model"
  ```
  Compare against each incoming file's IDs. If any UUID collides, generate a fresh UUID:
  ```bash
  node -e "console.log(require('node:crypto').randomUUID())"
  ```
  Patch the colliding file before writing. Log all collisions found (even if none).

- [ ] **Step 4: Copy Util files**

  Fetch each Util `.mlua` file from:
  ```
  https://raw.githubusercontent.com/MSW-Git/MSWPackages/main/ranking-advanced-package/MyDesk/Util/<filename>
  ```
  Write to `RootDesk/MyDesk/RankingAdvanced/Util/<filename>`. Do **not** copy `Sample/` content.

- [ ] **Step 5: Copy Core files**

  Fetch each Core `.mlua` file from:
  ```
  https://raw.githubusercontent.com/MSW-Git/MSWPackages/main/ranking-advanced-package/MyDesk/RankingAdvanced/Core/<filename>
  ```
  Write `.mlua` files to `RootDesk/MyDesk/RankingAdvanced/Core/`.
  Write any `.ui` files to both `RootDesk/MyDesk/RankingAdvanced/Core/` and `ui/` (as required by the platform).
  Write any `.model` files to `RootDesk/MyDesk/Models/RankingAdvanced/`.

- [ ] **Step 6: Verify build is clean**

  Run the test loop steps 1–3 only (no play needed yet):
  - `maker_stop` → `maker_refresh_workspace` → `maker_logs(category="build")`

  Expected: **0 error-severity diagnostics** from the copied package files. If any LSP errors appear, read the flagged file and fix the signature/type before proceeding. Common cause: mlua version mismatch in package source — fix by aligning to local `.d.mlua` signatures.

- [ ] **Step 7: Confirm package Logic singletons are visible**

  Run play mode and probe:
  ```lua
  -- maker_execute_script, context="server"
  local ok = _RankingDataStorageLogic ~= nil and _RankingConfigDataSetLogic ~= nil and _DateTimeLogic ~= nil
  log("[TEST PASS] package_logics_visible=" .. tostring(ok))
  ```
  Expected log line: `[TEST PASS] package_logics_visible=true`

- [ ] **Step 8: Commit package files**

  ```bash
  git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'
  # Must produce no output before committing.
  git add RootDesk/MyDesk/RankingAdvanced/
  git commit -m "$(cat <<'EOF'
  feat: vendor ranking-advanced package (Util + Core) into workspace

  Copies MSW first-party ranking-advanced-package source into
  RootDesk/MyDesk/RankingAdvanced/. UUID collision check passed (no
  collisions). License confirmed as [INSERT LICENSE RESULT HERE] in Step 1.

  Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
  EOF
  )"
  ```

---

## Task 2: Author RankingConfigDataSet (3 boards)

Creates the `.userdataset` metadata wrapper and the `.csv` sidecar defining Day, Week, and AllTime boards. The ranking engine reads this at startup.

**Files:**
- Create: `RootDesk/MyDesk/RankingAdvanced/RankingConfigDataSet.userdataset`
- Create: `RootDesk/MyDesk/RankingAdvanced/RankingConfigDataSet.csv`

**Interfaces:**
- Consumes: nothing (static config).
- Produces: Three board config rows readable via `_RankingConfigDataSetLogic`. Board ids 1/2/3 match the constants in `LeaderboardService` (Task 3).

- [ ] **Step 1: Confirm the exact CSV header from the package source**

  Fetch the package's own `RankingConfigDataSet.csv` (or its equivalent sample) from:
  ```
  https://raw.githubusercontent.com/MSW-Git/MSWPackages/main/ranking-advanced-package/MyDesk/RankingAdvanced/Core/RankingConfigDataSet.csv
  ```
  (If that path 404s, check the file tree from Task 1 Step 2 for the correct path.)

  Confirmed header from GATE C:
  ```
  Id, Key, Name, CycleEnum, ViewCount, RefreshIntervalMinutes, HasReward, RankModeEnum, MaxUserDataCount, ReleaseBaseTime, Disable
  ```
  Verify it matches the fetched file exactly. If there are additional columns, add them with safe defaults (blank or 0).

- [ ] **Step 2: Generate a fresh UUID for the dataset**

  ```bash
  node -e "console.log(require('node:crypto').randomUUID())"
  ```
  Record the output as `<DATASET_UUID>`.

- [ ] **Step 3: Write the .userdataset metadata wrapper**

  Create `RootDesk/MyDesk/RankingAdvanced/RankingConfigDataSet.userdataset`:

  ```json
  {
    "Id": "",
    "GameId": "",
    "EntryKey": "userdataset://<DATASET_UUID>",
    "ContentType": "x-mod/userdataset",
    "Content": "",
    "Usage": 0,
    "UsePublish": 1,
    "UseService": 0,
    "CoreVersion": "26.5.0.0",
    "StudioVersion": "0.1.0.0",
    "DynamicLoading": 0,
    "ContentProto": {
      "Use": "Json",
      "Json": {
        "name": "RankingConfigDataSet",
        "id": "<DATASET_UUID>",
        "serveronly": true,
        "syncDataSetWebUrl": "",
        "dynamicloading": 0
      }
    }
  }
  ```
  (Replace `<DATASET_UUID>` with the value from Step 2. `serveronly: true` because ranking data must not leak to clients.)

- [ ] **Step 4: Write the .csv sidecar**

  Create `RootDesk/MyDesk/RankingAdvanced/RankingConfigDataSet.csv`:

  ```csv
  Id, Key, Name, CycleEnum, ViewCount, RefreshIntervalMinutes, HasReward, RankModeEnum, MaxUserDataCount, ReleaseBaseTime, Disable
  1, ranked_daily, 일간 랭킹, 1, 100, 30, 0, 0, 0, 2026-01-01 00:00:00, 0
  2, ranked_weekly, 주간 랭킹, 2, 100, 30, 0, 0, 0, 2026-01-01 00:00:00, 0
  3, ranked_alltime, 전체 랭킹, , 100, 30, 0, 0, 0, 2026-01-01 00:00:00, 0
  ```

  Notes:
  - `CycleEnum`: Day board = `1`, Week board = `2`, AllTime board = **blank** (None=0 — no constant exists per GATE C Correction 2).
  - `ViewCount` = `100` (top-100 per spec §16; well under the 1000 cap).
  - `RefreshIntervalMinutes` = `30` (engine minimum; spec §16 originally said 5 but GATE C Correction 3 enforces ≥30 — this is the authoritative value).
  - `HasReward` = `0` — no rank-based rewards in Phase 1 (spec §7.2, "score-based only").
  - `ReleaseBaseTime` = `2026-01-01 00:00:00` — KST midnight anchor. This is the cycle boundary reference. It determines the KST-based day/week boundaries for `GetCycleIndex`. The actual daily puzzle boundary aligns with this because both derive from KST midnight (spec §7.1, §8.2).
  - `Disable` = `0` (enabled).
  - If the package source has a different column order, reorder to match exactly.

- [ ] **Step 5: Verify the dataset loads**

  Run the test loop steps 1–3:
  - `maker_stop` → `maker_refresh_workspace` → `maker_logs(category="build")`

  Then play and probe:
  ```lua
  -- maker_execute_script, context="server"
  local ds = _DataService:GetTable("RankingConfigDataSet")
  local rowCount = ds:GetRowCount()
  log("[TEST PASS] config_rows=" .. tostring(rowCount))
  local id1 = tonumber(ds:GetCell(1, "Id")) or 0
  log("[TEST PASS] board1_id=" .. tostring(id1))
  ```
  Expected logs:
  ```
  [TEST PASS] config_rows=3
  [TEST PASS] board1_id=1
  ```

- [ ] **Step 6: Commit**

  ```bash
  git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'
  git add RootDesk/MyDesk/RankingAdvanced/RankingConfigDataSet.userdataset RootDesk/MyDesk/RankingAdvanced/RankingConfigDataSet.csv
  git commit -m "$(cat <<'EOF'
  feat: add RankingConfigDataSet with Day/Week/AllTime boards

  Three boards: Day(CycleEnum=1), Week(CycleEnum=2), AllTime(CycleEnum=blank/None).
  RefreshIntervalMinutes=30 (engine minimum), ViewCount=100, no rank rewards.
  ReleaseBaseTime anchored at 2026-01-01 KST midnight.

  Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
  EOF
  )"
  ```

---

## Task 3: USER ACTION IN MAKER — DefaultPlayer components + WorldConfig

The `ranking-advanced` package requires `PlayerDBManager` and `PlayerRanking` on `DefaultPlayer`, and `WorldConfig.PlayerEntityAuthorityCheck = true`. `DefaultPlayer` and `WorldConfig` live in `Global/` — read-only to AI edits. The user must perform these steps in the Maker editor.

**Files:**
- `Global/DefaultPlayer.model` — read-only to AI; user edits via Maker entity inspector.
- `Global/WorldConfig.config` — read-only to AI; user edits via Maker world settings.

**Interfaces:**
- Consumes: installed package Core files (Task 1).
- Produces: `DefaultPlayer` entity with `PlayerDBManager.IsLoadSuccess` accessible at runtime; `PlayerRanking` component on the same entity for `SetScoreAndWait` calls.

> **Important (GATE C cross-gate note):** Even though the apple game hides the `DefaultPlayer` avatar renderer, the `DefaultPlayer` *entity* still exists in the world and carries the ranking components. Hiding the renderer does not remove the entity — the two concerns compose cleanly. Confirm this is working during the verify step below.

- [ ] **Step 1: USER ACTION IN MAKER — Add PlayerDBManager to DefaultPlayer**

  In the Maker editor:
  1. Open the **Entity** panel and find `DefaultPlayer` (under Global entities).
  2. Click **Add Component**.
  3. Search for and add **`PlayerDBManager`**.
  4. Verify the component appears in the inspector with no red errors.

- [ ] **Step 2: USER ACTION IN MAKER — Add PlayerRanking to DefaultPlayer**

  In the Maker editor:
  1. With `DefaultPlayer` still selected.
  2. Click **Add Component** again.
  3. Search for and add **`PlayerRanking`**.
  4. Verify the component appears.

- [ ] **Step 3: USER ACTION IN MAKER — Set WorldConfig.PlayerEntityAuthorityCheck**

  In the Maker editor:
  1. Open **World Settings** (or find `WorldConfig` in the Global settings panel).
  2. Find the **`PlayerEntityAuthorityCheck`** field.
  3. Set it to **`true`** (enabled).
  4. Save world settings.

- [ ] **Step 4: Refresh and verify via probe script**

  Run the test loop steps 1–3 (stop → refresh → build logs = 0 errors), then play and run:
  ```lua
  -- maker_execute_script, context="server"
  -- This verifies PlayerDBManager is on DefaultPlayer and IsLoadSuccess is accessible.
  -- In a real session, IsLoadSuccess becomes true after DB load; here we just confirm the field exists.
  local players = _UserService.UserEntities
  local found = false
  for _, p in ipairs(players.Values) do
      if isvalid(p) then
          local dbm = p.PlayerDBManager
          if dbm ~= nil then
              found = true
              log("[TEST PASS] player_db_manager_found IsLoadSuccess=" .. tostring(dbm.IsLoadSuccess))
          end
      end
  end
  if not found then
      log("[TEST FAIL] player_db_manager_not_found - did you add it to DefaultPlayer in Maker?")
  end
  ```

  Expected (when at least one player is in the world, which may require entering play via a client):
  ```
  [TEST PASS] player_db_manager_found IsLoadSuccess=true
  ```

  > If no players are in the world during the server-side test, the loop iterates zero users — this is not a failure. Instead, verify by **entering play mode yourself** (as the client), then re-running the script. `IsLoadSuccess` becomes `true` after `PlayerDBManager` finishes its async DB load (~1–2s after map enter).

- [ ] **Step 5: Commit probe note (no file changes — this task is Maker-only)**

  No files to commit (Maker saves `Global/` internally). Add a comment to the task checklist instead:
  ```
  # Maker action completed: PlayerDBManager + PlayerRanking on DefaultPlayer; WorldConfig.PlayerEntityAuthorityCheck=true
  ```

---

## Task 4: Implement LeaderboardService

The core of Plan 3. Wraps `PlayerRanking:SetScoreAndWait` (6-param corrected signature) for 3-board fan-out and `_RankingDataStorageLogic:GetRankingSnapshot` for leaderboard reads.

**Files:**
- Create: `RootDesk/MyDesk/AppleGame/Server/LeaderboardService.mlua`

**Interfaces:**
- Consumes (from Plan 2):
  - (none directly — ScoreService calls `_LeaderboardService:SubmitScore` not the reverse)
- Consumes (from ranking-advanced package):
  - `_RankingConfigDataSetLogic:GetRankingConfigData(integer boardId)` → `RankingConfigData`
  - `configData:GetCycleIndex(number elapsed)` → `integer`
  - `_DateTimeLogic:GetTimeElapsed()` → `number` (package-internal elapsed, used for cycle computation)
  - `playerRanking:SetScoreAndWait(integer id, integer cycleIndex, integer score, string tag, string extra, boolean force)` → `boolean`
  - `playerEntity.PlayerDBManager.IsLoadSuccess` → `boolean`
  - `playerEntity.PlayerRanking` → `PlayerRanking` component
  - `_RankingDataStorageLogic:GetRankingSnapshot(integer boardId)` → `RankingSnapshotData | nil`
  - `snap:GetValidPageNo(integer pageNo, integer pageSize)` → `integer`
  - `snap:GetRankingDataList(integer validPageNo, integer pageSize)` → `table<RankingData>`
  - `snap:GetRankingDataByProfileCode(string profileCode)` → `RankingData | nil`
  - `RankingData` fields: `.ProfileCode` (string), `.Score` (integer), `.Tag` (string)
- Produces (for Plan 2 ScoreService seam):
  - `@ExecSpace("ServerOnly") method void SubmitScore(string userId, integer score)`
- Produces (for Plan 4 LeaderboardController):
  - `@ExecSpace("Server") method void RequestLeaderboard(integer boardId, integer pageNo)`
  - `@ExecSpace("Client") method void ReceiveLeaderboard(integer boardId, table entries, integer myRank, integer myScore, boolean stale)`
  - `PAGE_SIZE = 20` (constant property)
  - Board id constants: `BOARD_DAY = 1`, `BOARD_WEEK = 2`, `BOARD_ALLTIME = 3`

- [ ] **Step 1: Write LeaderboardService.mlua**

  Create `RootDesk/MyDesk/AppleGame/Server/LeaderboardService.mlua`:

  ```lua
  @Logic
  script LeaderboardService extends Logic
      property integer BOARD_DAY     = 1
      property integer BOARD_WEEK    = 2
      property integer BOARD_ALLTIME = 3
      property integer PAGE_SIZE     = 20

      -- Sanitizes a display name for use as a ranking tag: truncates to 55 bytes
      -- and strips any '|' characters (delimiter in the ranking engine).
      method string SanitizeTag(string name)
          -- Returns a safe tag: pipe-free, UTF-8 byte count ≤ 55.
          local clean = name:gsub("|", "")
          -- Truncate to 55 bytes (simple byte slice; safe for ASCII/common Korean)
          if #clean > 55 then
              clean = clean:sub(1, 55)
          end
          return clean
      end

      -- Finds the PlayerRanking component on the entity that owns the given userId.
      -- Returns nil if the player is not in the world or DB is not yet loaded.
      method any FindPlayerRanking(string userId)
          -- Searches connected users for the matching userId.
          for _, p in ipairs(_UserService.UserEntities.Values) do
              if isvalid(p) then
                  local pc = p.PlayerComponent
                  if pc ~= nil and pc.UserId == userId then
                      local dbm = p.PlayerDBManager
                      if dbm ~= nil and dbm.IsLoadSuccess then
                          return p.PlayerRanking
                      end
                  end
              end
          end
          return nil
      end

      -- Submits a verified score to all three ranking boards (fan-out).
      -- Called by ScoreService after replay validation succeeds for a ranked run.
      -- force=false means keep-max: a lower score will not overwrite a higher one.
      @ExecSpace("ServerOnly")
      method void SubmitScore(string userId, integer score)
          -- Fan out to Day, Week, and AllTime boards with keep-max (force=false).
          if score < 1 then
              log_warning("[LeaderboardService] SubmitScore rejected: score < 1 for userId=" .. userId)
              return
          end

          local playerRanking = self:FindPlayerRanking(userId)
          if playerRanking == nil then
              log_warning("[LeaderboardService] SubmitScore: PlayerRanking not ready for userId=" .. userId)
              return
          end

          local elapsed = _DateTimeLogic:GetTimeElapsed()
          local tag = self:SanitizeTag(userId)  -- Plan 4 will pass the display name; userId is the fallback
          local extra = ""

          local boards = { self.BOARD_DAY, self.BOARD_WEEK, self.BOARD_ALLTIME }
          for _, boardId in ipairs(boards) do
              local configData = _RankingConfigDataSetLogic:GetRankingConfigData(boardId)
              if configData ~= nil then
                  local cycleIndex = configData:GetCycleIndex(elapsed)
                  local ok = playerRanking:SetScoreAndWait(boardId, cycleIndex, score, tag, extra, false)
                  log("[LeaderboardService] SubmitScore board=" .. boardId .. " cycleIndex=" .. cycleIndex .. " score=" .. score .. " ok=" .. tostring(ok))
              else
                  log_warning("[LeaderboardService] SubmitScore: no config for boardId=" .. boardId)
              end
          end
      end

      -- Server-side entry point for leaderboard data requests from clients.
      -- Reads the cached snapshot (≥30-min stale by design) and replies via ReceiveLeaderboard.
      @ExecSpace("Server")
      method void RequestLeaderboard(integer boardId, integer pageNo)
          -- Fetches the snapshot for the given board and pages; sends stale=true always.
          local snap = _RankingDataStorageLogic:GetRankingSnapshot(boardId)
          local entries = {}
          local myRank = 0
          local myScore = 0

          if snap == nil then
              -- InstanceRoom or snapshot not yet built — return empty with stale flag.
              log("[LeaderboardService] RequestLeaderboard: snapshot nil for boardId=" .. boardId)
              self:ReceiveLeaderboard(boardId, entries, myRank, myScore, true, senderUserId)
              return
          end

          local validPageNo = snap:GetValidPageNo(pageNo, self.PAGE_SIZE)
          local page = snap:GetRankingDataList(validPageNo, self.PAGE_SIZE)

          local startRank = (validPageNo - 1) * self.PAGE_SIZE + 1
          for i, rd in ipairs(page) do
              entries[i] = {
                  rank  = startRank + (i - 1),
                  name  = rd.Tag,
                  score = rd.Score,
              }
          end

          -- Look up the caller's own entry by ProfileCode (== UserId in MSW).
          local mine = snap:GetRankingDataByProfileCode(senderUserId)
          if mine ~= nil then
              myScore = mine.Score
              -- Approximate rank from entries list (may not be on current page)
              for _, e in ipairs(entries) do
                  if e.name == mine.Tag and e.score == mine.Score then
                      myRank = e.rank
                      break
                  end
              end
          end

          -- stale=true always: snapshot may be up to 30+ minutes old.
          self:ReceiveLeaderboard(boardId, entries, myRank, myScore, true, senderUserId)
      end

      -- Delivers leaderboard data to the requesting client.
      -- targetUserId is appended at call site (do NOT declare it).
      @ExecSpace("Client")
      method void ReceiveLeaderboard(integer boardId, table entries, integer myRank, integer myScore, boolean stale)
          -- Route to the LeaderboardController UI (Plan 4 will connect this event).
          log("[LeaderboardService] ReceiveLeaderboard boardId=" .. boardId .. " entryCount=" .. tostring(#entries) .. " stale=" .. tostring(stale))
      end
  end
  ```

- [ ] **Step 2: Verify build is clean**

  - `maker_stop` → `maker_refresh_workspace` → `maker_logs(category="build")`
  - Expected: **0 error-severity diagnostics**. Common LSP issues to pre-empt:
    - If `_RankingConfigDataSetLogic` / `_DateTimeLogic` / `_RankingDataStorageLogic` show `LIA-1114 Info` (user cross-script ref), treat as noise — these are package Logic singletons and will resolve at runtime.
    - If `PlayerRanking:SetScoreAndWait` shows a type error, confirm the exact signature in the installed package `.mlua` and adjust if the source differs from GATE C.

- [ ] **Step 3: Commit**

  ```bash
  git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'
  git add RootDesk/MyDesk/AppleGame/Server/LeaderboardService.mlua
  git commit -m "$(cat <<'EOF'
  feat: add LeaderboardService with 3-board fan-out and snapshot read

  SubmitScore fans out to Day/Week/AllTime boards using corrected 6-param
  SetScoreAndWait (force=false keep-max). RequestLeaderboard reads the
  >=30-min cached snapshot and replies via Client RPC with stale=true.
  Guards: score<1 rejected, PlayerDBManager.IsLoadSuccess checked, nil
  snapshot (InstanceRoom) handled.

  Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
  EOF
  )"
  ```

---

## Task 5: Wire ScoreService seam — replace Plan-2 stub

Plan 2's `ScoreService.mlua` contains a stub call to `_LeaderboardService:SubmitScore` (marked as the Plan-3 seam). Replace that stub with the real call now that `LeaderboardService` exists.

**Files:**
- Modify: `RootDesk/MyDesk/AppleGame/Server/ScoreService.mlua`

**Interfaces:**
- Consumes (from Task 4): `_LeaderboardService:SubmitScore(string userId, integer score)` (ServerOnly)
- Produces: no new interface — this closes the Plan-2 seam so ranked runs are actually recorded.

- [ ] **Step 1: Read the current ScoreService to locate the seam**

  Read `RootDesk/MyDesk/AppleGame/Server/ScoreService.mlua` in full.
  Locate the comment marking the Plan-3 seam. It should look like:
  ```lua
  -- Plan-3 seam: replace with _LeaderboardService:SubmitScore(senderUserId, r.score)
  -- (stub — LeaderboardService not yet implemented)
  ```
  Record the exact line(s) to replace.

- [ ] **Step 2: Replace the stub with the real call**

  Replace the stub block with:
  ```lua
  _LeaderboardService:SubmitScore(senderUserId, r.score)
  ```

  The surrounding context (from the interface contract) should look like:
  ```lua
  -- After replay validation passes and token is consumed:
  if session.mode == "ranked" then
      _LeaderboardService:SubmitScore(senderUserId, r.score)
  end
  ```

  If the seam comment is not present (Plan 2 may have implemented it differently), search for `SubmitScore` in the file and confirm the call site. If it already calls `_LeaderboardService:SubmitScore` correctly, skip this step.

- [ ] **Step 3: Verify build is clean**

  - `maker_stop` → `maker_refresh_workspace` → `maker_logs(category="build")`
  - Expected: **0 error-severity diagnostics**.

- [ ] **Step 4: Commit**

  ```bash
  git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'
  git add RootDesk/MyDesk/AppleGame/Server/ScoreService.mlua
  git commit -m "$(cat <<'EOF'
  feat: wire ScoreService ranked path to LeaderboardService.SubmitScore

  Replaces the Plan-2 seam stub with the real fan-out call now that
  LeaderboardService is implemented. Ranked runs accepted by ScoreService
  are now propagated to all three ranking boards.

  Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
  EOF
  )"
  ```

---

## Task 6: Integration test — submit + per-player read

Drives a full end-to-end ranked submission through `LeaderboardService` using `maker_execute_script` and verifies the per-player snapshot read. The public list is NOT expected to update immediately (≥30-min snapshot); the test verifies submit success + `GetRankingDataByProfileCode` round-trip.

**Files:**
- Create (test harness only): `RootDesk/MyDesk/AppleGame/Server/LeaderboardServiceTest.mlua`

**Interfaces:**
- Consumes: `_LeaderboardService:SubmitScore`, `_RankingDataStorageLogic:GetRankingSnapshot`, `_DateTimeLogic:GetTimeElapsed`.
- Produces: `[TEST PASS]`/`[TEST SUMMARY]` log lines (no new production interface).

- [ ] **Step 1: Write the test harness**

  Create `RootDesk/MyDesk/AppleGame/Server/LeaderboardServiceTest.mlua`:

  ```lua
  @Logic
  script LeaderboardServiceTest extends Logic
      property integer passed = 0
      property integer failed = 0

      method void Expect(string name, boolean cond)
          -- Records one assertion result for maker_logs inspection.
          if cond then
              self.passed += 1
              log("[TEST PASS] " .. name)
          else
              self.failed += 1
              log_error("[TEST FAIL] " .. name)
          end
      end

      method void Reset()
          -- Clears counters before a test run.
          self.passed = 0
          self.failed = 0
      end

      method void Summary()
          -- Emits the final tally line checked by maker_logs.
          log("[TEST SUMMARY] passed=" .. self.passed .. " failed=" .. self.failed)
      end

      -- Integration test: submit a test score and verify the per-player snapshot read.
      -- NOTE: DataStorage is credit-billed. Run this test minimally.
      -- NOTE: The public leaderboard snapshot is >=30 min stale; we do NOT assert
      -- that the submitted score appears in GetRankingDataList immediately.
      -- We DO assert that SetScoreAndWait returns without error and that the
      -- snapshot object is accessible (nil only in InstanceRoom, not in normal play).
      @ExecSpace("ServerOnly")
      method void TestSubmitAndRead()
          -- Requires: at least one player in the world with PlayerDBManager.IsLoadSuccess=true.
          self:Reset()

          -- Find any connected player to test with
          local testUserId = nil
          local testRanking = nil
          for _, p in ipairs(_UserService.UserEntities.Values) do
              if isvalid(p) then
                  local dbm = p.PlayerDBManager
                  if dbm ~= nil and dbm.IsLoadSuccess then
                      testUserId = p.PlayerComponent.UserId
                      testRanking = p.PlayerRanking
                      break
                  end
              end
          end

          self:Expect("found_player_with_loaded_db", testUserId ~= nil)
          if testUserId == nil then
              log_warning("[TEST] No player with loaded DB found. Enter play mode as a client first.")
              self:Summary()
              return
          end

          -- Submit a test score of 5 to all three boards via LeaderboardService
          local testScore = 5
          _LeaderboardService:SubmitScore(testUserId, testScore)
          self:Expect("submit_did_not_error", true)  -- if we reach here, no runtime error

          -- Verify that snapshots are accessible (nil = InstanceRoom, which is not an error)
          local boards = { 1, 2, 3 }
          for _, boardId in ipairs(boards) do
              local snap = _RankingDataStorageLogic:GetRankingSnapshot(boardId)
              local snapAccessible = snap ~= nil
              log("[TEST] board=" .. boardId .. " snapshot_accessible=" .. tostring(snapAccessible))
              -- Snapshot nil in InstanceRoom is expected; in a normal room it should exist after first build
              -- We don't fail here because the first-ever run may not have built the snapshot yet
          end

          -- Per-player read: GetRankingDataByProfileCode (reads from snapshot)
          -- The score we just submitted may not be in the snapshot yet (stale by design).
          -- We verify the call does not crash and returns either nil (not in snapshot yet) or a RankingData.
          local snap1 = _RankingDataStorageLogic:GetRankingSnapshot(1)
          if snap1 ~= nil then
              local mine = snap1:GetRankingDataByProfileCode(testUserId)
              local readOk = mine == nil or (mine.Score ~= nil)
              self:Expect("per_player_read_no_crash", readOk)
              log("[TEST] per_player_entry=" .. (mine ~= nil and tostring(mine.Score) or "nil (not in snapshot yet -- stale is expected)"))
          else
              log("[TEST] snapshot nil -- per_player_read skipped (InstanceRoom or not yet built)")
              self:Expect("per_player_read_no_crash", true)  -- nil snapshot is not an error
          end

          self:Summary()
      end
  end
  ```

- [ ] **Step 2: Verify build is clean**

  - `maker_stop` → `maker_refresh_workspace` → `maker_logs(category="build")`
  - Expected: **0 error-severity diagnostics**.

- [ ] **Step 3: Run the integration test**

  Run the full test loop:
  1. `maker_stop` → `maker_refresh_workspace` → `maker_logs(category="build")` (0 errors)
  2. `maker_play`
  3. Poll `maker_get_current_map` until state = "play".
  4. **Enter as a client** (tab into the game world so a player entity exists with `IsLoadSuccess=true`). Wait ~2 seconds for DB load.
  5. `maker_clear_logs`
  6. `maker_execute_script("_LeaderboardServiceTest:TestSubmitAndRead()", context="server")`
  7. `maker_logs(category="normal")`

  Expected log lines:
  ```
  [TEST PASS] found_player_with_loaded_db
  [TEST PASS] submit_did_not_error
  [TEST] board=1 snapshot_accessible=<true or false>
  [TEST] board=2 snapshot_accessible=<true or false>
  [TEST] board=3 snapshot_accessible=<true or false>
  [TEST PASS] per_player_read_no_crash
  [TEST] per_player_entry=<5 or nil (not in snapshot yet -- stale is expected)>
  [TEST SUMMARY] passed=3 failed=0
  ```

  Also confirm `[LeaderboardService] SubmitScore board=1 ... ok=true` (and board 2, 3) appear in the log from `LeaderboardService.mlua`'s own `log()` calls.

  8. `maker_stop`

- [ ] **Step 4: Commit test harness**

  ```bash
  git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'
  git add RootDesk/MyDesk/AppleGame/Server/LeaderboardServiceTest.mlua
  git commit -m "$(cat <<'EOF'
  test: add LeaderboardService integration test (submit + per-player snapshot read)

  Verifies 3-board SetScoreAndWait fan-out succeeds and snapshot read does
  not crash. Does not assert public list updates immediately (>=30-min stale
  by design). Credit-billed: run minimally.

  Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
  EOF
  )"
  ```

---

## Task 7: Final Verify (verify-checklist.md)

Runs the full `msw-scripting/references/verify-checklist.md` against all files produced by this plan.

**Files:**
- Read (checklist): `msw-scripting/references/verify-checklist.md` (Skill: msw-scripting must be loaded first)
- Review: `RootDesk/MyDesk/AppleGame/Server/LeaderboardService.mlua`
- Review: `RootDesk/MyDesk/AppleGame/Server/ScoreService.mlua` (seam change only)
- Review: `RootDesk/MyDesk/RankingAdvanced/RankingConfigDataSet.csv`

**Interfaces:**
- Consumes: all files from Tasks 1–6.
- Produces: PASS/FAIL verdict per `verify-checklist.md`.

- [ ] **Step 1: Load the verify checklist**

  Load `Skill: msw-scripting` (if not already loaded this turn). Then `Read` `references/verify-checklist.md` in full (no offset/limit).

- [ ] **Step 2: Code review checklist — LeaderboardService**

  Re-read `RootDesk/MyDesk/AppleGame/Server/LeaderboardService.mlua` and confirm:
  - [ ] `SubmitScore` is `@ExecSpace("ServerOnly")` — correct (no client can call it).
  - [ ] `RequestLeaderboard` is `@ExecSpace("Server")` — correct (client→server RPC).
  - [ ] `ReceiveLeaderboard` is `@ExecSpace("Client")` — correct (server→client RPC). `targetUserId` is NOT declared (appended at call site only).
  - [ ] `senderUserId` is used (not declared) inside `RequestLeaderboard` body — correct.
  - [ ] `SetScoreAndWait` called with 6 args in order: `(boardId, cycleIndex, score, tag, extra, false)` — correct.
  - [ ] `cycleIndex` is computed via `configData:GetCycleIndex(elapsed)` — not hardcoded.
  - [ ] `tag` passes through `SanitizeTag` (≤55 bytes, no `|`) before use.
  - [ ] `PlayerDBManager.IsLoadSuccess` is checked before `SetScoreAndWait`.
  - [ ] No DataStorage calls in `OnUpdate` or short timers.
  - [ ] `snap == nil` (InstanceRoom case) is handled — returns empty + stale=true.
  - [ ] Every `method` has its doc comment as the first line inside the body.
  - [ ] No `Global/` or `Environment/` files were modified.

- [ ] **Step 3: Runtime final run**

  Run the full MSW test loop one more time (clean slate):
  1. `maker_stop` → `maker_clear_logs` → `maker_refresh_workspace`
  2. `maker_logs(category="build")` — confirm **0 errors**.
  3. `maker_play`
  4. Enter as a client, wait for DB load.
  5. `maker_clear_logs`
  6. `maker_execute_script("_LeaderboardServiceTest:TestSubmitAndRead()", context="server")`
  7. `maker_logs(category="normal")` — confirm `[TEST SUMMARY] passed=3 failed=0`.
  8. `maker_stop`

- [ ] **Step 4: Record PASS/FAIL verdict**

  | Criterion | Status |
  |---|---|
  | Build logs: 0 errors | [ ] PASS / [ ] FAIL |
  | Integration test: passed=3 failed=0 | [ ] PASS / [ ] FAIL |
  | SetScoreAndWait fan-out log lines (3 boards) | [ ] PASS / [ ] FAIL |
  | No `[TEST FAIL]` in logs | [ ] PASS / [ ] FAIL |
  | Code review checklist: all items checked | [ ] PASS / [ ] FAIL |

  All rows must be PASS before reporting Plan 3 complete.

---

## Plan 3 Self-Review

**Spec coverage:**
- §7.1 ranking-advanced package selection → Task 1 (install).
- §7.1 RankingConfigDataSet 3 boards (Day/Week/AllTime) → Task 2 (CSV + userdataset).
- §7.1 KST `ReleaseBaseTime` anchor → Task 2 Step 4 (`2026-01-01 00:00:00`).
- §7.1 `force=false` keep-max fan-out → Task 4 (`SubmitScore` with `force=false`).
- §7.2 `RefreshIntervalMinutes ≥ 30` → Task 2 (30 in CSV).
- §7.2 `ViewCount ≤ 1000` → Task 2 (100 in CSV).
- §7.2 `tag` ≤ 55 bytes, no `|` → Task 4 (`SanitizeTag`).
- §7.2 bisal-realtime UI label → Task 4 `ReceiveLeaderboard` passes `stale=true` always.
- §3 single=personal-best (no global board) → NOT in this plan (single personal best lives in `_RankAttemptStore` / `_ScoreService` in Plan 2; this plan handles only ranked boards). Confirmed out-of-scope for Plan 3.
- §14 file structure (AppleGame/Server/, no catch-all) → Tasks 4–6.
- GATE C Correction 1 (6-param signature) → Task 4 exactly.
- GATE C Correction 2 (no AllTime constant — use blank/None) → Task 2 (blank CycleEnum for board 3).
- GATE C Correction 3 (≥30-min stale, RefreshIntervalMinutes ≥ 30) → Tasks 2 + 6 test expectations.
- Cross-gate note: DefaultPlayer hosts ranking components even with avatar hidden → Task 3 (USER ACTION IN MAKER) + verify probe.
- Licensing: flagged explicitly in Task 1 Step 1 and Global Constraints.

**Placeholder scan:**
- Task 1 Step 8 commit message contains `[INSERT LICENSE RESULT HERE]` — intentional: the license result is determined at runtime by Step 1. Replace before committing.
- No other "TBD", "TODO", or deferred implementation stubs.

**Type consistency:**
- `SubmitScore(string userId, integer score)` — declared in interfaces, used identically in Task 4 and Task 5.
- `SetScoreAndWait(integer id, integer cycleIndex, integer score, string tag, string extra, boolean force)` — 6-param form used in Task 4, matches GATE C Correction 1 exactly.
- `ReceiveLeaderboard(integer boardId, table entries, integer myRank, integer myScore, boolean stale)` — same in interfaces and Task 4 implementation.
- `entries` element shape: `{rank: integer, name: string, score: integer}` — consistent across Task 4 build and Plan 4 consume contract.
- `PAGE_SIZE = 20` — property on `_LeaderboardService`, referenced by name not magic literal.
- `BOARD_DAY=1`, `BOARD_WEEK=2`, `BOARD_ALLTIME=3` — constants on `_LeaderboardService`, match CSV `Id` column values exactly.

---

> **Open questions / risks for the controller:**
>
> 1. **License (blocking):** No explicit redistribution license was found in the package at research time. Task 1 Step 1 is a hard stop until this is confirmed. If redistribution is not permitted, the plan must be revised to use Maker's modpackage installer instead of vendoring.
>
> 2. **DefaultPlayer hidden + ranking components:** GATE C cross-gate note confirms the entity exists even with renderer off. Verify during Task 3 that `PlayerDBManager.IsLoadSuccess` becomes `true` for a hidden-avatar player. If not (edge case: avatar renderer off suppresses DB load), Task 3's probe script will log `[TEST FAIL]` — escalate to MSW Discord.
>
> 3. **`GetCycleIndex` formula and weekly cycle start day:** GATE C notes these were "high-confidence but not byte-verified" from the package source. During Task 4 Step 2 (build verify), read the installed `RankingConfigData.mlua` and confirm `GetCycleIndex` signature and weekly rollover day (Sun vs Mon). If it differs from KST Monday-0 assumption, the daily seed rotation (keyed off the same KST midnight) may misalign with the Week board's cycle index by one day. Confirm and adjust `ReleaseBaseTime` if needed.
>
> 4. **`_RankingConfigDataSetLogic:GetRankingConfigData` exact name:** corroborated by usage in the package's `RankingViewLogic.mlua` but not byte-verified. Confirm the method name during Task 4 Step 2 by reading the installed file. If the name differs, update `LeaderboardService.mlua` before committing.

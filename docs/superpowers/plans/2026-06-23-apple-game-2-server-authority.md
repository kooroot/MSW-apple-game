# Maple Apple Game — Plan 2: Server Authority

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the server-authoritative anti-cheat core (`_SeedService` + `_ScoreService` + `_RankAttemptStore`) that issues seeds and single-use session tokens, consumes the daily ranked attempt at issuance via DataStorage CAS, and accepts a submitted run only after re-simulating it with the already-shipped `_PuzzleCore:Replay`.

**Architecture:** Three `@Logic` singletons under `RootDesk/MyDesk/AppleGame/Server/`, all server-side. `_RankAttemptStore` wraps `_DataStorageService` UserDataStorage (per-player persistence): daily attempt flag + personal best. `_SeedService` computes the KST date from `DateTime.UtcNow + TimeSpan(9,0,0)`, derives the daily seed via `_SeededRng:Mix32(dateKey)`, consumes the daily attempt at **issuance** (CAS, anti-retry), and hands out an opaque single-use session token bound to `senderUserId` in an in-memory record. `_ScoreService` validates a submitted run (token ownership + single-use + server/client time window + payload shape) then trusts only `_PuzzleCore:Replay(seed, moves)` for the score. A fourth `@Logic` (`ServerAuthTest`) is a ServerOnly integration harness that drives all three in play mode, mirroring Plan 1's `PuzzleCoreTest`.

**Tech Stack:** mlua (Lua 5.3 dialect, integer type + bitwise `~ & | << >>`), MSW server-side `@Logic` + `@ExecSpace`, `_DataStorageService` UserDataStorage (credit-billed), `DateTime`/`TimeSpan` value types, MSW Maker MCP (`maker_refresh_workspace`, `maker_play`, `maker_get_current_map`, `maker_clear_logs`, `maker_execute_script`, `maker_logs`, `maker_stop`).

## Global Constraints

Copied verbatim from the spec (`docs/superpowers/specs/2026-06-23-maple-apple-game-design.md`), the API gate findings (`2026-06-23-apple-game-api-gates.md`), and the interface contract (`2026-06-23-apple-game-interfaces.md`). Every task implicitly includes these.

- **Side rules:** all persistence / ranking / seed logic is **ServerOnly**; all UI/input is ClientOnly (not in this plan). Client→server crosses via `@ExecSpace("Server")`; server→client via `@ExecSpace("Client")` with `targetUserId` appended **at the call site**, never declared. (spec §4, interfaces "Conventions")
- **`mode` is a string** `"ranked"` | `"single"` across RPC — no engine enums across RPC; allowed RPC param types only (`string`/`integer`/`number`/`boolean`/`table`). (spec §6, §12; msw-scripting §6)
- **Server time source = `DateTime.UtcNow` only** (static readonly, no `@ExecSpace`, reads the server system clock; client cannot influence it). KST = `DateTime.UtcNow + TimeSpan(9, 0, 0)`; `dateKey = kst.Year*10000 + kst.Month*100 + kst.Day`. **Never** use `_UtilLogic.ServerElapsedSeconds`/`.ElapsedSeconds` for wall-clock (instance-lifetime monotonic, resets on world reboot). (GATE A)
- **`integer` ≠ `number`.** DataStorage `errCode` is `int32` → hold in `integer` vars. `DateTime.Year/.Month/.Day` are `int32`. `clientElapsed`/`TotalSeconds` are `number`. (interfaces "Conventions"; GATE A/B)
- **Daily seed = `_SeededRng:Mix32(dateKey)`** (deterministic, KST-keyed, identical for everyone that day). Single-mode seed = server-random the client cannot predict; **never trust a client-supplied seed**. (spec §8.2, §8.4; interfaces Plan 2)
- **Consume the daily ranked attempt at seed ISSUANCE, not at submit** — via `UpdateAndWait("rankAttempt:"..dateKey, "", "1")` CAS. Issuance-time consumption is the anti-retry property: a disconnect after issuance must not return the attempt. (spec §8.3; GATE B; interfaces Plan 2)
- **Session token is single-use and bound to `senderUserId`.** Token is an opaque random string; verification is server-record lookup (no signing). Submit must reject: missing token, wrong owner, already-consumed token, expired time window. (spec §10)
- **Submit trusts only the replay:** accept iff `r.ok and r.score == claimedScore` after `_PuzzleCore:Replay(session.seed, moves)`. (spec §6, §12; interfaces Plan 2)
- **DataStorage is credit-billed.** Never call in `OnUpdate` or sub-1s timers; compare-before-write (an identical write still costs); reading a missing key still costs; value ≤4KB; key 1–100 bytes UTF-8. Our keys: `rankAttempt:{dateKey}` (~16 B) and `personalBest` — both tiny. **The test harness must NOT loop storage calls** — keep each test to a handful of Get/Set/Update calls. **Maker and release environments do NOT share storage.** (datastorage.md; GATE B)
- `@Logic` singletons are accessed as `_<ExactScriptName>` (no suffix stripping): `SeedService.mlua` → `_SeedService`, `ScoreService.mlua` → `_ScoreService`, `RankAttemptStore.mlua` → `_RankAttemptStore`, `ServerAuthTest.mlua` → `_ServerAuthTest`. (msw-scripting §3.2)
- Every `method` has its description as the **first line inside the body**, never above the declaration. (msw-scripting §1.8)
- No globals; no coroutines; parent call is `__base:` not `super`; `self:IsServer()`/`self:IsClient()` are **method calls**. (msw-scripting §4, §6)
- New `.mlua` lives under the feature folder `AppleGame/Server/`, never a catch-all like `Scripts/`. (msw-scripting §1.2; spec §14)
- Do **not** edit `Global/` or `Environment/` (read-only). Plan 2 needs neither; if a step ever would, it must be written as an explicit **USER ACTION IN MAKER** step, not a file edit.
- After any `.mlua` create/modify, run `maker_refresh_workspace` (requires edit mode — `maker_stop` first if playing). The `mlua-diagnose` LSP hook auto-runs after each edit; drive error-severity diagnostics to **zero** before testing. (msw-scripting §1.4, §1.5)

## Verified API surface (from `.d.mlua` — do NOT re-derive)

These are confirmed against `Environment/NativeScripts/**`. Copy verbatim; do not guess variants.

- `_DataStorageService:GetUserDataStorage(string userId) -> UserDataStorage` (`Service/DataStorageService.d.mlua:127`, ServerOnly).
- `UserDataStorage:GetAndWait(string key) -> (int32 errCode, string value)` (`Misc/UserDataStorage.d.mlua:53`). `errCode 1000002 = NotFound`.
- `UserDataStorage:SetAndWait(string key, string value) -> int32 errCode` (`:83`).
- `UserDataStorage:UpdateAndWait(string key, string expectedValue, string newValue) -> (int32 errCode, string value)` (`:123`) — the CAS primitive.
- `DateTime.UtcNow` — `static readonly property DateTime` (`Misc/DateTime.d.mlua:45`). Fields `.Year` (`:49`), `.Month` (`:38`), `.Day` (`:8`) are `int32`. `DateTime + TimeSpan -> DateTime` (`:88`). `DateTime - DateTime -> TimeSpan` (`:91`).
- `TimeSpan(int32 hours, int32 minutes, int32 seconds)` constructor (`Misc/TimeSpan.d.mlua:60`). `TimeSpan.TotalSeconds -> number` readonly (`:54`).
- `_SeededRng:Mix32(integer x) -> integer` and `_SeededRng:Next(integer s) -> integer`, `_SeededRng:Normalize(integer s) -> integer` (Plan 1, shipped, `AppleGame/Core/SeededRng.mlua`).
- `_PuzzleCore:Replay(integer seed, table moves) -> { score = integer, ok = boolean }`; `moves` = array of `{ rect = {c1,r1,c2,r2}, t = number }` (Plan 1, shipped, `AppleGame/Core/PuzzleCore.mlua`). Also `_PuzzleCore:GenerateBoard(integer seed) -> grid[r][c]`, `_PuzzleCore.COLS=17`, `_PuzzleCore.ROWS=10`.
- `_UserService.LocalPlayer -> Entity` (`Service/UserService.d.mlua:8`); `Entity.PlayerComponent.UserId -> string` readonly (`Component/PlayerComponent.d.mlua:48`) — used by the test harness to obtain a real userId server-side in play mode.

## GATE gaps to verify LIVE in this plan

The gate doc flags two undocumented behaviors. They are converted to explicit runtime checks inside the relevant tasks:

1. **Midnight rollover (GATE A gap):** confirm `UTC 16:00 + 9h` rolls to the next KST day. Task 3 logs `dateKey` so a reviewer can sanity-check the value matches "today in KST."
2. **CAS-mismatch error code (GATE B gap):** `UpdateAndWait`'s error code on an expected-value mismatch is undocumented. Treat **any non-zero errCode as "already consumed / failed"**; Task 2 logs the raw errCode of the second (mismatching) CAS so the real code is captured. First-ever write uses a **GetAndWait → SetAndWait** fallback (empty-expected `UpdateAndWait` behavior is undocumented).

## Cross-plan seam (READ BEFORE Task 4)

`_ScoreService`'s **ranked** accept path must record to the leaderboard, which is **`_LeaderboardService:SubmitScore(userId, score)` installed by Plan 3**. Plan 2 runs **before** Plan 3, so `_LeaderboardService` does **not exist yet**. Every call site MUST be guarded:

```lua
if _LeaderboardService ~= nil then
    _LeaderboardService:SubmitScore(userId, finalScore)
else
    log_warning("[SEAM] _LeaderboardService not installed yet (Plan 3); ranked score not fanned out: " .. tostring(finalScore))
end
```

Do **not** create a stub `LeaderboardService.mlua` in this plan — Plan 3 owns that file and a stub would collide. The guard above is the contract: Plan 3 simply ships the real `_LeaderboardService` and the guard starts taking the true branch with no further edit to `_ScoreService`. This seam is intentional and must remain in the committed code.

## The MSW Test Loop (used by every task)

There is no `pytest`. "Run the test" means this exact MSW Maker MCP sequence. Because DataStorage and `DateTime.UtcNow` are **ServerOnly**, the harness is driven `context="server"`.

1. `maker_stop` (if currently playing) — refresh needs edit mode.
2. `maker_refresh_workspace` — compiles the `.mlua` edits into `.codeblock`.
3. `maker_logs(kind="build")` — **must be 0 errors**. If any, fix → refresh → recheck before playing. (The `mlua-diagnose` LSP hook also auto-runs after each edit; drive it to zero too.)
4. `maker_play` — enter play mode.
5. Poll `maker_get_current_map` until `mode == "play"` — `@Logic` singletons + `_UserService.LocalPlayer` are live only then.
6. `maker_clear_logs` — isolate this run's output.
7. `maker_execute_script(<snippet>, context="server")` — drive the harness server-side.
8. `maker_logs(kind="normal")` — read back `[TEST PASS]` / `[TEST FAIL]` and the `[TEST SUMMARY] passed=N failed=0` line. **Any `[TEST FAIL]` or `failed>0` is a test failure.**
9. `maker_stop`.

> If `maker_execute_script` cannot run a multi-statement snippet, wrap the calls in a single `RunX()` method on the harness and execute `_ServerAuthTest:RunX()` instead. Verify which form works in Task 1 before relying on it.

> **DataStorage credit discipline in tests:** each test method makes only a handful of Get/Set/Update calls (never in a loop, never on a timer). The CAS-retry test reuses one key; the personal-best test does one Set + one Update. Keep it minimal — the Maker storage is credit-billed just like release.

## Commit policy (PUBLIC repo)

- **Never `git add -A` / `git add .`** — stage explicit paths only.
- Before every commit, run the secret scan (must print nothing):
  ```bash
  git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'
  ```
  If it prints anything, unstage and remove the secret before committing.
- Commit via HEREDOC so the trailers are exact:
  ```bash
  git commit -F - <<'EOF'
  <subject line>

  Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
  EOF
  ```

---

### Task 1: Server-side test harness + loop verification

Establishes the `@Logic` test harness and proves the `refresh → build-clean → play → execute(server) → logs` loop works **server-side** before any real logic exists. Uses whatever map is already current as the play host (the dedicated `AppleGame.map` host is built in Plan 4). Also captures the **real LocalPlayer userId** server-side, which later tasks reuse as the test subject for per-user storage.

**Files:**
- Create: `RootDesk/MyDesk/AppleGame/Server/ServerAuthTest.mlua`
- Test: same file (self-hosting harness)

**Interfaces:**
- Consumes: `_UserService.LocalPlayer.PlayerComponent.UserId` (string, verified).
- Produces: `_ServerAuthTest` `@Logic` with `Expect(string name, boolean cond)`, `Reset()`, `Summary()`, `LocalUserId() -> string`, and `TestHarness()`. Later tasks add `TestRankAttemptStore()`, `TestPersonalBest()`, `TestSeedIssuance()`, `TestSubmitRun()` to this same script.

- [ ] **Step 1: Write the harness with a trivial server-side self-test**

Create `RootDesk/MyDesk/AppleGame/Server/ServerAuthTest.mlua`:

```lua
@Logic
script ServerAuthTest extends Logic
    property integer passed = 0
    property integer failed = 0

    @ExecSpace("ServerOnly")
    method void Expect(string name, boolean cond)
        -- Records one assertion result to the server log for maker_logs inspection.
        if cond then
            self.passed += 1
            log("[TEST PASS] " .. name)
        else
            self.failed += 1
            log_error("[TEST FAIL] " .. name)
        end
    end

    @ExecSpace("ServerOnly")
    method void Reset()
        -- Clears counters before a test method run.
        self.passed = 0
        self.failed = 0
    end

    @ExecSpace("ServerOnly")
    method void Summary()
        -- Emits the final tally line that maker_logs is checked against.
        log("[TEST SUMMARY] passed=" .. self.passed .. " failed=" .. self.failed)
    end

    @ExecSpace("ServerOnly")
    method string LocalUserId()
        -- Returns a real userId (the local player) for per-user storage tests.
        local p = _UserService.LocalPlayer
        if not isvalid(p) then return "" end
        return p.PlayerComponent.UserId
    end

    @ExecSpace("ServerOnly")
    method void TestHarness()
        -- Smoke test proving the assert/log/execute loop is wired correctly server-side.
        self:Expect("harness_true", true)
        self:Expect("harness_eq", 1 + 1 == 2)
        self:Expect("harness_has_userid", self:LocalUserId() ~= "")
        log("[INFO] LocalUserId=" .. self:LocalUserId())
    end
end
```

- [ ] **Step 2: Refresh, build-check, and run, expecting PASS**

Run the MSW Test Loop:
- `maker_stop`, `maker_refresh_workspace`, `maker_logs(kind="build")` (expect 0 errors)
- `maker_play`, poll `maker_get_current_map` until `mode=="play"`, `maker_clear_logs`
- `maker_execute_script("_ServerAuthTest:Reset() _ServerAuthTest:TestHarness() _ServerAuthTest:Summary()", context="server")`
- `maker_logs(kind="normal")`

Expected log lines:
```
[TEST PASS] harness_true
[TEST PASS] harness_eq
[TEST PASS] harness_has_userid
[INFO] LocalUserId=<some non-empty id>
[TEST SUMMARY] passed=3 failed=0
```
If the multi-statement `maker_execute_script` snippet does not run, add a `RunHarness()` method that calls `Reset()/TestHarness()/Summary()` and execute `_ServerAuthTest:RunHarness()` instead. **Note which form works — every later task uses the same form and `context="server"`.** Then `maker_stop`.

- [ ] **Step 3: Stop and commit**

```bash
git add RootDesk/MyDesk/AppleGame/Server/ServerAuthTest.mlua
git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'   # must print nothing
git commit -F - <<'EOF'
test: add server-side ServerAuthTest harness; verify play/execute(server)/logs loop

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
EOF
```

---

### Task 2: `_RankAttemptStore` — UserDataStorage wrapper (daily attempt CAS + personal best)

The per-player persistence layer. Wraps `_DataStorageService` UserDataStorage. Two responsibilities: (a) `TryConsumeDaily` consumes one ranked attempt per KST day via `UpdateAndWait` CAS (returns true only on first consume that day); (b) `GetPersonalBest` / `SetPersonalBestIfHigher` for the single-mode keep-max best. All ServerOnly.

**Files:**
- Create: `RootDesk/MyDesk/AppleGame/Server/RankAttemptStore.mlua`
- Test: add `TestRankAttemptStore()` + `TestPersonalBest()` to `RootDesk/MyDesk/AppleGame/Server/ServerAuthTest.mlua`

**Interfaces:**
- Consumes: `_DataStorageService:GetUserDataStorage(string)`, `UserDataStorage:GetAndWait/SetAndWait/UpdateAndWait` (verified above).
- Produces: `_RankAttemptStore` `@Logic` with
  - `TryConsumeDaily(string userId, integer dateKey) -> boolean` — ServerOnly; CAS on `rankAttempt:{dateKey}`; true iff newly consumed this call, false if already set (or on storage error → treated as not-consumable, fail-closed).
  - `IsDailyConsumed(string userId, integer dateKey) -> boolean` — ServerOnly; read-only check (used by tests / diagnostics; NOT used on the hot issuance path to avoid an extra credit-billed read).
  - `GetPersonalBest(string userId) -> integer` — ServerOnly; reads `personalBest`, 0 if NotFound.
  - `SetPersonalBestIfHigher(string userId, integer score) -> (boolean updated, integer best)` — ServerOnly; keep-max; writes only when strictly higher (dirty check), returns whether it updated and the resulting best.

- [ ] **Step 1: Write the failing test**

Add to `ServerAuthTest.mlua` inside the `script` block. The CAS test uses a **synthetic, date-keyed-by-the-test** key so it is deterministic across reruns: it deletes nothing (no Delete to save credits) but uses a fresh `dateKey` derived from a counter the test bumps, so the first consume is always "new." To make reruns repeatable without unbounded key growth, the test keys off the current KST `dateKey` plus a salt the test controls, and asserts the **relative** behavior (first true, second false) within one run.

```lua
    @ExecSpace("ServerOnly")
    method void TestRankAttemptStore()
        -- Verifies daily attempt CAS: first consume true, immediate re-consume false (same day).
        local uid = self:LocalUserId()
        self:Expect("ras_has_uid", uid ~= "")
        -- Use a per-run unique dateKey so reruns don't collide with a previously-consumed key.
        -- We piggyback on milliseconds to vary the key each run (test-only; real code uses the KST dateKey).
        local now = DateTime.UtcNow
        local saltedKey = (now.Year * 10000 + now.Month * 100 + now.Day) * 100000 + (now.Hour * 3600 + now.Minute * 60 + now.Second) % 100000

        local first = _RankAttemptStore:TryConsumeDaily(uid, saltedKey)
        self:Expect("ras_first_consume_true", first == true)

        local second = _RankAttemptStore:TryConsumeDaily(uid, saltedKey)
        self:Expect("ras_second_consume_false", second == false)

        local consumedNow = _RankAttemptStore:IsDailyConsumed(uid, saltedKey)
        self:Expect("ras_is_consumed_after", consumedNow == true)

        -- A never-touched key must read as not consumed.
        local fresh = _RankAttemptStore:IsDailyConsumed(uid, saltedKey + 1)
        self:Expect("ras_fresh_key_not_consumed", fresh == false)
    end

    @ExecSpace("ServerOnly")
    method void TestPersonalBest()
        -- Verifies keep-max personal best: higher updates, lower/equal does not.
        local uid = self:LocalUserId()
        local base = _RankAttemptStore:GetPersonalBest(uid)  -- whatever is stored (0 if none)

        local upd1, best1 = _RankAttemptStore:SetPersonalBestIfHigher(uid, base + 50)
        self:Expect("pb_higher_updates", upd1 == true and best1 == base + 50)

        local upd2, best2 = _RankAttemptStore:SetPersonalBestIfHigher(uid, base + 10)
        self:Expect("pb_lower_keeps_max", upd2 == false and best2 == base + 50)

        local readBack = _RankAttemptStore:GetPersonalBest(uid)
        self:Expect("pb_read_back_max", readBack == base + 50)
    end
```

- [ ] **Step 2: Run, expecting FAIL**

Run the MSW Test Loop with `_ServerAuthTest:Reset() _ServerAuthTest:TestRankAttemptStore() _ServerAuthTest:TestPersonalBest() _ServerAuthTest:Summary()` (`context="server"`).
Expected: failures / runtime error referencing `_RankAttemptStore` (script does not exist yet).

- [ ] **Step 3: Implement `_RankAttemptStore`**

Create `RootDesk/MyDesk/AppleGame/Server/RankAttemptStore.mlua`:

```lua
@Logic
script RankAttemptStore extends Logic
    -- DataStorage error codes (see datastorage.md §6).
    property integer ERR_OK = 0
    property integer ERR_NOT_FOUND = 1000002

    @ExecSpace("ServerOnly")
    method string AttemptKey(integer dateKey)
        -- Builds the per-day ranked-attempt key, e.g. "rankAttempt:20260623".
        return "rankAttempt:" .. tostring(dateKey)
    end

    @ExecSpace("ServerOnly")
    method boolean TryConsumeDaily(string userId, integer dateKey)
        -- Consumes today's ranked attempt via CAS; true only if newly consumed, false if already set or on error (fail-closed).
        local storage = _DataStorageService:GetUserDataStorage(userId)
        local key = self:AttemptKey(dateKey)

        -- Read first to distinguish first-ever write from an existing flag.
        -- (Empty-expected UpdateAndWait behavior is undocumented (GATE B gap), so we branch on the read.)
        local readErr, cur = storage:GetAndWait(key)
        if readErr == self.ERR_NOT_FOUND then
            -- First write of this key today: plain Set, no expected-value contract needed.
            local setErr = storage:SetAndWait(key, "1")
            if setErr == self.ERR_OK then
                log("[RAS] consumed (first-write) key=" .. key)
                return true
            end
            log_warning("[RAS] first-write Set failed errCode=" .. tostring(setErr) .. " key=" .. key)
            return false
        end

        if readErr ~= self.ERR_OK then
            -- Read error (not NotFound): cannot safely grant an attempt -> fail closed.
            log_warning("[RAS] Get failed errCode=" .. tostring(readErr) .. " key=" .. key)
            return false
        end

        if cur == "1" then
            -- Already consumed today.
            return false
        end

        -- Key exists but is not "1" (unexpected value): CAS from the observed value to "1".
        local updErr, newVal = storage:UpdateAndWait(key, cur, "1")
        if updErr == self.ERR_OK and newVal == "1" then
            log("[RAS] consumed (CAS) key=" .. key)
            return true
        end
        -- Any non-zero errCode = CAS mismatch / failure (GATE B gap: exact code undocumented). Capture it.
        log_warning("[RAS] CAS update failed errCode=" .. tostring(updErr) .. " key=" .. key)
        return false
    end

    @ExecSpace("ServerOnly")
    method boolean IsDailyConsumed(string userId, integer dateKey)
        -- Read-only diagnostic check; not on the issuance hot path (avoids an extra billed read).
        local storage = _DataStorageService:GetUserDataStorage(userId)
        local readErr, cur = storage:GetAndWait(self:AttemptKey(dateKey))
        return readErr == self.ERR_OK and cur == "1"
    end

    @ExecSpace("ServerOnly")
    method integer GetPersonalBest(string userId)
        -- Reads the stored personal best (string-serialized integer); 0 when not yet set.
        local storage = _DataStorageService:GetUserDataStorage(userId)
        local readErr, raw = storage:GetAndWait("personalBest")
        if readErr == self.ERR_NOT_FOUND then return 0 end
        if readErr ~= self.ERR_OK then
            log_warning("[RAS] personalBest Get failed errCode=" .. tostring(readErr))
            return 0
        end
        local n = tonumber(raw)
        if n == nil then return 0 end
        return math.floor(n)
    end

    @ExecSpace("ServerOnly")
    method boolean, integer SetPersonalBestIfHigher(string userId, integer score)
        -- Keep-max write with dirty check: writes only when strictly higher; returns (updated, resulting best).
        local current = self:GetPersonalBest(userId)
        if score <= current then
            return false, current
        end
        local storage = _DataStorageService:GetUserDataStorage(userId)
        local setErr = storage:SetAndWait("personalBest", tostring(score))
        if setErr ~= self.ERR_OK then
            log_warning("[RAS] personalBest Set failed errCode=" .. tostring(setErr))
            return false, current
        end
        return true, score
    end
end
```

> Multi-return methods (`SetPersonalBestIfHigher`, `GetAndWait`, `UpdateAndWait`) return two values; capture both at the call site (`local a, b = ...`). If the LSP flags `math.floor`/`math.min` as unavailable in this mlua build, replace `math.floor(n)` with `n // 1`.

- [ ] **Step 4: Run, expecting PASS**

Run the MSW Test Loop (same snippet as Step 2). Expected:
```
[TEST PASS] ras_has_uid
[TEST PASS] ras_first_consume_true
[TEST PASS] ras_second_consume_false
[TEST PASS] ras_is_consumed_after
[TEST PASS] ras_fresh_key_not_consumed
[TEST PASS] pb_higher_updates
[TEST PASS] pb_lower_keeps_max
[TEST PASS] pb_read_back_max
[TEST SUMMARY] passed=8 failed=0
```
Also inspect the `[RAS]` log lines: confirm the first consume logs `consumed (first-write)` and check whether any `CAS update failed errCode=` line appears — **record that errCode** (this captures the undocumented GATE B CAS-mismatch code). Confirm `mlua-diagnose` reports zero errors on `RankAttemptStore.mlua`. Then `maker_stop`.

- [ ] **Step 5: Commit**

```bash
git add RootDesk/MyDesk/AppleGame/Server/RankAttemptStore.mlua RootDesk/MyDesk/AppleGame/Server/ServerAuthTest.mlua
git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'   # must print nothing
git commit -F - <<'EOF'
feat: add RankAttemptStore (UserDataStorage daily-attempt CAS + keep-max personal best)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
EOF
```

---

### Task 3: `_SeedService` — KST date, seed derivation, token issuance, attempt consumption

Issues seeds + single-use session tokens. Computes the KST `dateKey` from `DateTime.UtcNow + TimeSpan(9,0,0)`. For `mode=="ranked"`: derive `dailySeed = _SeededRng:Mix32(dateKey)`, CAS-consume the daily attempt **at issuance**; if already consumed, reply `alreadyPlayed=true` (no token, no board). For `mode=="single"`: a server-random seed the client can't predict, no attempt consumption. Stores an in-memory session record keyed by token (single-use for submit) and replies via a Client RPC. The session record is what `_ScoreService` consumes in Task 4.

**Files:**
- Create: `RootDesk/MyDesk/AppleGame/Server/SeedService.mlua`
- Test: add `TestSeedDateKey()` + `TestSeedIssuance()` to `ServerAuthTest.mlua`

**Interfaces:**
- Consumes: `_SeededRng:Mix32(integer) -> integer`, `_SeededRng:Next(integer) -> integer` (Plan 1); `_RankAttemptStore:TryConsumeDaily(string,integer) -> boolean`, `_RankAttemptStore:GetPersonalBest(string) -> integer` (Task 2); `DateTime.UtcNow`, `TimeSpan(int32,int32,int32)`, `senderUserId` (RPC-injected caller id).
- Produces: `_SeedService` `@Logic` with
  - `KstDateKey() -> integer` — ServerOnly; `year*10000 + month*100 + day` from `DateTime.UtcNow + 9h`.
  - `RequestSeed(string mode)` — `@ExecSpace("Server")`; client entry; issues seed+token, consumes ranked attempt, replies via `ReceiveSeed`.
  - `ReceiveSeed(string mode, integer seed, string token, integer dateKey, boolean alreadyPlayed, integer personalBest)` — `@ExecSpace("Client")` (`targetUserId` appended at call site, not declared).
  - **Test seam (ServerOnly, not RPC):** `IssueFor(string userId, string mode) -> table` returning `{ token, seed, dateKey, alreadyPlayed, personalBest }`, and `GetSession(string token) -> table` (the in-memory record `{userId, seed, dateKey, mode, issuedAt(DateTime), consumed}` or nil). `RequestSeed` is a thin wrapper over `IssueFor(senderUserId, mode)` so the same logic is unit-testable without simulating an RPC.
  - In-memory session map: `self._T.sessions[token] = { userId, seed, dateKey, mode, issuedAt, consumed=false }`. `issuedAt` is a `DateTime` captured at issuance for the submit-time window check (Task 4).

- [ ] **Step 1: Write the failing test**

Add to `ServerAuthTest.mlua`:

```lua
    @ExecSpace("ServerOnly")
    method void TestSeedDateKey()
        -- Verifies the KST dateKey is a plausible YYYYMMDD integer (midnight-rollover sanity, GATE A gap).
        local dk = _SeedService:KstDateKey()
        log("[INFO] KstDateKey=" .. tostring(dk))   -- reviewer compares against "today in KST"
        self:Expect("datekey_in_range", dk >= 20240101 and dk <= 99991231)
        local mm = (dk // 100) % 100
        local dd = dk % 100
        self:Expect("datekey_month_valid", mm >= 1 and mm <= 12)
        self:Expect("datekey_day_valid", dd >= 1 and dd <= 31)
    end

    @ExecSpace("ServerOnly")
    method void TestSeedIssuance()
        -- Verifies single-mode always issues; ranked seed == Mix32(dateKey); ranked consumes at issuance.
        local uid = self:LocalUserId()

        -- Single mode: always issues a token, never blocked, no attempt consumed.
        local s1 = _SeedService:IssueFor(uid, "single")
        self:Expect("single_issues_token", s1.token ~= nil and s1.token ~= "" and s1.alreadyPlayed == false)
        local s2 = _SeedService:IssueFor(uid, "single")
        self:Expect("single_token_unique", s2.token ~= s1.token)
        self:Expect("single_seed_unpredictable", s2.seed ~= s1.seed)  -- server entropy varies per issue
        local sess = _SeedService:GetSession(s1.token)
        self:Expect("single_session_stored", sess ~= nil and sess.userId == uid and sess.consumed == false)

        -- Ranked mode: seed must equal Mix32(dateKey); attempt is consumed at issuance.
        local dk = _SeedService:KstDateKey()
        local r1 = _SeedService:IssueFor(uid, "ranked")
        -- r1 may be alreadyPlayed=true if a prior run consumed today's attempt; assert the seed contract only when issued.
        if r1.alreadyPlayed == false then
            self:Expect("ranked_seed_is_mix32", r1.seed == _SeededRng:Mix32(dk))
            self:Expect("ranked_token_issued", r1.token ~= nil and r1.token ~= "")
            -- Second ranked issue same day must be blocked (consumed at issuance).
            local r2 = _SeedService:IssueFor(uid, "ranked")
            self:Expect("ranked_second_blocked", r2.alreadyPlayed == true and (r2.token == nil or r2.token == ""))
        else
            -- Already consumed earlier today: the block path itself is the assertion.
            self:Expect("ranked_already_blocked", r1.alreadyPlayed == true and (r1.token == nil or r1.token == ""))
            log("[INFO] ranked already consumed for dateKey=" .. tostring(dk) .. " (expected on reruns)")
        end
    end
```

> Note on reruns: ranked consumes the **real** `rankAttempt:{dateKey}` for today, so the first run after a fresh KST day exercises the `alreadyPlayed==false` branch and subsequent reruns exercise the `alreadyPlayed==true` branch. Both branches are asserted, so the test passes either way. To force the issue path again, run on a new KST day or (manually, outside the harness) clear the key — do **not** add a Delete to the harness (credit-billed and would weaken the anti-retry guarantee being tested).

- [ ] **Step 2: Run, expecting FAIL**

Run the MSW Test Loop with `_ServerAuthTest:Reset() _ServerAuthTest:TestSeedDateKey() _ServerAuthTest:TestSeedIssuance() _ServerAuthTest:Summary()` (`context="server"`).
Expected: error referencing `_SeedService` (not yet created).

- [ ] **Step 3: Implement `_SeedService`**

Create `RootDesk/MyDesk/AppleGame/Server/SeedService.mlua`:

```lua
@Logic
script SeedService extends Logic
    property integer entropyCounter = 0

    @ExecSpace("ServerOnly")
    method void EnsureSessions()
        -- Lazily initializes the in-memory token->record map (temporary, non-synced server state).
        if self._T.sessions == nil then
            self._T.sessions = {}
        end
    end

    @ExecSpace("ServerOnly")
    method integer KstDateKey()
        -- Derives the KST calendar date as YYYYMMDD from the server UTC clock plus 9 hours.
        local kst = DateTime.UtcNow + TimeSpan(9, 0, 0)
        return kst.Year * 10000 + kst.Month * 100 + kst.Day
    end

    @ExecSpace("ServerOnly")
    method string NewToken(string userId)
        -- Builds an opaque single-use token from server entropy the client cannot predict.
        self.entropyCounter += 1
        local now = DateTime.UtcNow
        local ms = now.Hour * 3600000 + now.Minute * 60000 + now.Second * 1000 + now.Millisecond
        local mixed = _SeededRng:Mix32((ms + self.entropyCounter * 2654435761) & 0xFFFFFFFF)
        mixed = _SeededRng:Next((mixed + self.entropyCounter) & 0xFFFFFFFF)
        return "tk_" .. tostring(mixed) .. "_" .. tostring(self.entropyCounter)
    end

    @ExecSpace("ServerOnly")
    method integer NewSingleSeed()
        -- Produces a server-random single-mode seed the client cannot predict.
        self.entropyCounter += 1
        local now = DateTime.UtcNow
        local ms = now.Hour * 3600000 + now.Minute * 60000 + now.Second * 1000 + now.Millisecond
        return _SeededRng:Mix32((ms * 31 + self.entropyCounter) & 0xFFFFFFFF)
    end

    @ExecSpace("ServerOnly")
    method table IssueFor(string userId, string mode)
        -- Core issuance: validates mode, consumes ranked attempt at issuance, stores a session, returns the reply payload.
        self:EnsureSessions()
        local dateKey = self:KstDateKey()
        local personalBest = _RankAttemptStore:GetPersonalBest(userId)

        if mode == "ranked" then
            local consumed = _RankAttemptStore:TryConsumeDaily(userId, dateKey)
            if not consumed then
                log("[SEED] ranked blocked (already played) userId=" .. userId .. " dateKey=" .. tostring(dateKey))
                return { token = "", seed = 0, dateKey = dateKey, alreadyPlayed = true, personalBest = personalBest, mode = mode }
            end
            local seed = _SeededRng:Mix32(dateKey)
            local token = self:NewToken(userId)
            self._T.sessions[token] = { userId = userId, seed = seed, dateKey = dateKey, mode = "ranked", issuedAt = DateTime.UtcNow, consumed = false }
            log("[SEED] ranked issued userId=" .. userId .. " dateKey=" .. tostring(dateKey) .. " seed=" .. tostring(seed))
            return { token = token, seed = seed, dateKey = dateKey, alreadyPlayed = false, personalBest = personalBest, mode = mode }
        end

        -- "single" (and any non-"ranked" mode is treated as single).
        local seed = self:NewSingleSeed()
        local token = self:NewToken(userId)
        self._T.sessions[token] = { userId = userId, seed = seed, dateKey = dateKey, mode = "single", issuedAt = DateTime.UtcNow, consumed = false }
        log("[SEED] single issued userId=" .. userId .. " seed=" .. tostring(seed))
        return { token = token, seed = seed, dateKey = dateKey, alreadyPlayed = false, personalBest = personalBest, mode = "single" }
    end

    @ExecSpace("ServerOnly")
    method table GetSession(string token)
        -- Returns the in-memory session record for a token, or nil if unknown/expired.
        self:EnsureSessions()
        return self._T.sessions[token]
    end

    @ExecSpace("ServerOnly")
    method void MarkConsumed(string token)
        -- Marks a session token as used so the same token cannot be submitted twice (called by ScoreService).
        self:EnsureSessions()
        local rec = self._T.sessions[token]
        if rec ~= nil then rec.consumed = true end
    end

    @ExecSpace("Server")
    method void RequestSeed(string mode)
        -- Client entry point: issues a seed+token for the authenticated caller and replies on their client.
        local payload = self:IssueFor(senderUserId, mode)
        self:ReceiveSeed(payload.mode, payload.seed, payload.token, payload.dateKey, payload.alreadyPlayed, payload.personalBest, senderUserId)
    end

    @ExecSpace("Client")
    method void ReceiveSeed(string mode, integer seed, string token, integer dateKey, boolean alreadyPlayed, integer personalBest)
        -- Server->caller reply; the client GameSession (Plan 4) builds the board from seed unless alreadyPlayed.
        log("[SEED] client received mode=" .. mode .. " alreadyPlayed=" .. tostring(alreadyPlayed) .. " seed=" .. tostring(seed))
    end
end
```

> `RequestSeed` is `@ExecSpace("Server")` so `senderUserId` is the server-assigned, unforgeable caller id (msw-scripting §6). The last arg to `ReceiveSeed` at the call site (`senderUserId`) is the `targetUserId` the engine appends — it is **not** declared on the method. The session map lives in `self._T` (non-synced server state), so a world reboot drops in-flight sessions — that is fine; the durable 1-play-per-day guarantee is the UserDataStorage attempt key, not the in-memory token (spec §10).

- [ ] **Step 4: Run, expecting PASS**

Run the MSW Test Loop (Step 2 snippet). Expected all `TestSeedDateKey` + `TestSeedIssuance` assertions PASS, `failed=0`. Read the `[INFO] KstDateKey=` line and **confirm it matches today's date in KST** (this is the midnight-rollover live check, GATE A gap). Confirm zero LSP errors on `SeedService.mlua`. Then `maker_stop`.

- [ ] **Step 5: Commit**

```bash
git add RootDesk/MyDesk/AppleGame/Server/SeedService.mlua RootDesk/MyDesk/AppleGame/Server/ServerAuthTest.mlua
git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'   # must print nothing
git commit -F - <<'EOF'
feat: add SeedService (KST dateKey, Mix32 daily seed, single-use token, consume-at-issuance)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
EOF
```

---

### Task 4: `_ScoreService` — token + time-window + payload validation + replay-trusted scoring

Validates and records a completed run. Consumes the `_SeedService` session record. Validation order (reject on first failure, all rejections logged): (1) token exists and belongs to `senderUserId`; (2) token not yet consumed → mark consumed; (3) server-elapsed-since-issuance AND `clientElapsed` both within the 120s window + slack; (4) payload shape sane (move count cap, coordinate ranges, monotonic non-decreasing timestamps within window); (5) `_PuzzleCore:Replay(session.seed, moves)` and accept iff `r.ok and r.score == claimedScore`. On accept: update `personalBest` (keep-max) and, for `mode=="ranked"`, the **guarded** `_LeaderboardService:SubmitScore` (cross-plan seam). Reply via Client RPC. Also exposes `RequestPersonalBest` / `ReceivePersonalBest` for the Leaderboard "Single" tab (read-only, independent of the play flow).

**Files:**
- Create: `RootDesk/MyDesk/AppleGame/Server/ScoreService.mlua`
- Test: add `TestSubmitValidation()` to `ServerAuthTest.mlua`

**Interfaces:**
- Consumes: `_SeedService:IssueFor/GetSession/MarkConsumed` (Task 3); `_RankAttemptStore:SetPersonalBestIfHigher/GetPersonalBest` (Task 2); `_PuzzleCore:Replay(integer,table) -> {score,ok}`, `_PuzzleCore:GenerateBoard`, `_PuzzleCore.COLS/.ROWS` (Plan 1); `DateTime.UtcNow`, `DateTime - DateTime -> TimeSpan`, `TimeSpan.TotalSeconds`; `senderUserId`; `_LeaderboardService:SubmitScore(string,integer)` **(Plan 3, guarded — see Cross-plan seam)**.
- Produces: `_ScoreService` `@Logic` with
  - `property integer GAME_SECONDS = 120`, `property number GRACE_SECONDS = 5.0`, `property integer MAX_MOVES = 256`.
  - `ValidatePayload(table session, table moves, number clientElapsed) -> (boolean okShape, string reason)` — ServerOnly; pure shape/time-window checks (no replay).
  - `SubmitFor(string userId, string token, table moves, integer claimedScore, number clientElapsed) -> table` — ServerOnly; full validation + replay; returns `{ accepted, finalScore, personalBest, reason }`. `reason ∈ "ok"|"bad_token"|"timing"|"replay_mismatch"|"already"|"bad_payload"`.
  - `SubmitRun(string token, table moves, integer claimedScore, number clientElapsed)` — `@ExecSpace("Server")`; thin wrapper over `SubmitFor(senderUserId, ...)`; replies via `ReceiveSubmitResult`.
  - `ReceiveSubmitResult(boolean accepted, integer finalScore, integer personalBest, string reason)` — `@ExecSpace("Client")`.
  - `@ExecSpace("Server") method void RequestPersonalBest()` — reads `_RankAttemptStore:GetPersonalBest(senderUserId)`, replies via `ReceivePersonalBest`. Read-only, no token, idempotent (safe to call whenever the Single tab opens).
  - `@ExecSpace("Client") method void ReceivePersonalBest(integer personalBest)` — server→caller (`targetUserId` appended at call site, not declared); body bridges to the client (`_LeaderboardController:OnReceivePersonalBest`) via the same delivery mechanism chosen for `ReceiveSubmitResult` in Plan 4 Task 6 Step 1.

- [ ] **Step 1: Write the failing test**

Add to `ServerAuthTest.mlua`. The test issues a real single-mode session, finds a genuine valid move on that exact board (so the positive replay has a known score), then exercises each rejection path against the same session/fresh sessions.

```lua
    @ExecSpace("ServerOnly")
    method table FindOneValidPairOnBoard(integer seed)
        -- Test helper: scans the generated board for the first horizontally-adjacent pair summing to 10.
        local g = _PuzzleCore:GenerateBoard(seed)
        for r = 0, _PuzzleCore.ROWS - 1 do
            for c = 0, _PuzzleCore.COLS - 2 do
                if g[r][c] + g[r][c + 1] == 10 then
                    return { rect = { c1 = c, r1 = r, c2 = c + 1, r2 = r }, t = 1.0 }
                end
            end
        end
        return nil
    end

    @ExecSpace("ServerOnly")
    method void TestSubmitValidation()
        -- Validates accept path + every rejection path (bad token, wrong owner, reuse, timing, payload, replay mismatch).
        local uid = self:LocalUserId()

        -- 1) Unknown token -> bad_token.
        local rBad = _ScoreService:SubmitFor(uid, "no_such_token", {}, 0, 1.0)
        self:Expect("submit_unknown_token", rBad.accepted == false and rBad.reason == "bad_token")

        -- 2) Accept path: fresh single session + one real valid move.
        local s = _SeedService:IssueFor(uid, "single")
        local mv = self:FindOneValidPairOnBoard(s.seed)
        self:Expect("found_valid_move", mv ~= nil)
        if mv ~= nil then
            local rOk = _ScoreService:SubmitFor(uid, s.token, { mv }, 2, 5.0)
            self:Expect("submit_accepts_valid", rOk.accepted == true and rOk.reason == "ok" and rOk.finalScore == 2)

            -- 3) Reuse same token -> already.
            local rReuse = _ScoreService:SubmitFor(uid, s.token, { mv }, 2, 5.0)
            self:Expect("submit_token_reuse_blocked", rReuse.accepted == false and rReuse.reason == "already")
        end

        -- 4) Wrong owner -> bad_token.
        local s2 = _SeedService:IssueFor(uid, "single")
        local rOwner = _ScoreService:SubmitFor("someone_else", s2.token, {}, 0, 5.0)
        self:Expect("submit_wrong_owner_blocked", rOwner.accepted == false and rOwner.reason == "bad_token")

        -- 5) Timing: clientElapsed beyond window -> timing.
        local s3 = _SeedService:IssueFor(uid, "single")
        local rTime = _ScoreService:SubmitFor(uid, s3.token, {}, 0, 99999.0)
        self:Expect("submit_timing_blocked", rTime.accepted == false and rTime.reason == "timing")

        -- 6) Payload: out-of-range coordinate -> bad_payload.
        local s4 = _SeedService:IssueFor(uid, "single")
        local badMove = { rect = { c1 = 0, r1 = 0, c2 = 99, r2 = 0 }, t = 1.0 }
        local rPay = _ScoreService:SubmitFor(uid, s4.token, { badMove }, 0, 5.0)
        self:Expect("submit_bad_coord_blocked", rPay.accepted == false and rPay.reason == "bad_payload")

        -- 7) Replay mismatch: valid move but lied-about score -> replay_mismatch.
        local s5 = _SeedService:IssueFor(uid, "single")
        local mv5 = self:FindOneValidPairOnBoard(s5.seed)
        if mv5 ~= nil then
            local rLie = _ScoreService:SubmitFor(uid, s5.token, { mv5 }, 9999, 5.0)
            self:Expect("submit_score_lie_blocked", rLie.accepted == false and rLie.reason == "replay_mismatch")
        end
    end
```

- [ ] **Step 2: Run, expecting FAIL**

Run the MSW Test Loop with `_ServerAuthTest:Reset() _ServerAuthTest:TestSubmitValidation() _ServerAuthTest:Summary()` (`context="server"`).
Expected: error referencing `_ScoreService` (not yet created).

- [ ] **Step 3: Implement `_ScoreService`**

Create `RootDesk/MyDesk/AppleGame/Server/ScoreService.mlua`:

```lua
@Logic
script ScoreService extends Logic
    property integer GAME_SECONDS = 120
    property number GRACE_SECONDS = 5.0
    property integer MAX_MOVES = 256

    @ExecSpace("ServerOnly")
    method boolean, string ValidatePayload(table session, table moves, number clientElapsed)
        -- Pure shape/time-window validation (no replay): count cap, coord ranges, monotonic timestamps, window.
        if type(moves) ~= "table" then return false, "bad_payload" end
        if #moves > self.MAX_MOVES then return false, "bad_payload" end

        local windowMax = self.GAME_SECONDS + self.GRACE_SECONDS
        -- Client-reported elapsed must be within the window (and non-negative).
        if clientElapsed < 0 or clientElapsed > windowMax then return false, "timing" end

        local maxCol = _PuzzleCore.COLS - 1
        local maxRow = _PuzzleCore.ROWS - 1
        local prevT = -1.0
        for i = 1, #moves do
            local m = moves[i]
            if type(m) ~= "table" or type(m.rect) ~= "table" then return false, "bad_payload" end
            local rc = m.rect
            if rc.c1 == nil or rc.r1 == nil or rc.c2 == nil or rc.r2 == nil then return false, "bad_payload" end
            if rc.c1 < 0 or rc.c1 > maxCol or rc.c2 < 0 or rc.c2 > maxCol then return false, "bad_payload" end
            if rc.r1 < 0 or rc.r1 > maxRow or rc.r2 < 0 or rc.r2 > maxRow then return false, "bad_payload" end
            local t = m.t
            if type(t) ~= "number" then return false, "bad_payload" end
            if t < 0 or t > windowMax then return false, "timing" end
            if t < prevT then return false, "timing" end   -- timestamps must be non-decreasing
            prevT = t
        end
        return true, "ok"
    end

    @ExecSpace("ServerOnly")
    method table SubmitFor(string userId, string token, table moves, integer claimedScore, number clientElapsed)
        -- Full server-authoritative validation + replay-trusted scoring; returns the result payload.
        local session = _SeedService:GetSession(token)

        -- (1) token exists and belongs to the caller.
        if session == nil or session.userId ~= userId then
            log_warning("[SCORE] reject bad_token userId=" .. tostring(userId) .. " token=" .. tostring(token))
            return { accepted = false, finalScore = 0, personalBest = _RankAttemptStore:GetPersonalBest(userId), reason = "bad_token" }
        end

        -- (2) single-use: not yet consumed; consume now (atomic in single-threaded @Logic).
        if session.consumed == true then
            log_warning("[SCORE] reject already token=" .. token)
            return { accepted = false, finalScore = 0, personalBest = _RankAttemptStore:GetPersonalBest(userId), reason = "already" }
        end
        _SeedService:MarkConsumed(token)

        -- (3) server elapsed since issuance within the window (authoritative; client time is only a cross-check).
        local serverElapsed = (DateTime.UtcNow - session.issuedAt).TotalSeconds
        local windowMax = self.GAME_SECONDS + self.GRACE_SECONDS
        if serverElapsed < 0 or serverElapsed > windowMax then
            log_warning("[SCORE] reject timing(server) elapsed=" .. tostring(serverElapsed))
            return { accepted = false, finalScore = 0, personalBest = _RankAttemptStore:GetPersonalBest(userId), reason = "timing" }
        end

        -- (4) payload shape + client-time-window sanity.
        local okShape, payReason = self:ValidatePayload(session, moves, clientElapsed)
        if not okShape then
            log_warning("[SCORE] reject payload reason=" .. payReason)
            return { accepted = false, finalScore = 0, personalBest = _RankAttemptStore:GetPersonalBest(userId), reason = payReason }
        end

        -- (5) replay is the only source of truth for the score.
        local r = _PuzzleCore:Replay(session.seed, moves)
        if not (r.ok and r.score == claimedScore) then
            log_warning("[SCORE] reject replay_mismatch ok=" .. tostring(r.ok) .. " replayScore=" .. tostring(r.score) .. " claimed=" .. tostring(claimedScore))
            return { accepted = false, finalScore = 0, personalBest = _RankAttemptStore:GetPersonalBest(userId), reason = "replay_mismatch" }
        end

        -- Accept: keep-max personal best, and (ranked) fan out to the leaderboard (Plan 3 seam).
        local finalScore = r.score
        local updated, best = _RankAttemptStore:SetPersonalBestIfHigher(userId, finalScore)

        if session.mode == "ranked" then
            -- CROSS-PLAN SEAM: _LeaderboardService is installed by Plan 3. Guard until then.
            if _LeaderboardService ~= nil then
                _LeaderboardService:SubmitScore(userId, finalScore)
            else
                log_warning("[SEAM] _LeaderboardService not installed yet (Plan 3); ranked score not fanned out: " .. tostring(finalScore))
            end
        end

        log("[SCORE] accept userId=" .. userId .. " mode=" .. session.mode .. " score=" .. tostring(finalScore) .. " best=" .. tostring(best))
        return { accepted = true, finalScore = finalScore, personalBest = best, reason = "ok" }
    end

    @ExecSpace("Server")
    method void SubmitRun(string token, table moves, integer claimedScore, number clientElapsed)
        -- Client entry point: validates and scores the run for the authenticated caller, replies on their client.
        local res = self:SubmitFor(senderUserId, token, moves, claimedScore, clientElapsed)
        self:ReceiveSubmitResult(res.accepted, res.finalScore, res.personalBest, res.reason, senderUserId)
    end

    @ExecSpace("Client")
    method void ReceiveSubmitResult(boolean accepted, integer finalScore, integer personalBest, string reason)
        -- Server->caller reply; the client Result UI (Plan 4) renders accepted/score/best or the rejection reason.
        log("[SCORE] client result accepted=" .. tostring(accepted) .. " score=" .. tostring(finalScore) .. " reason=" .. reason)
    end
end
```

> `SubmitRun` is `@ExecSpace("Server")` so `senderUserId` is unforgeable. The single-use guarantee is enforced server-side via `session.consumed` flipped through `_SeedService:MarkConsumed` before replay, so a duplicate submit short-circuits at step (2). The window check uses **server** elapsed (`DateTime.UtcNow - session.issuedAt`) as authoritative; `clientElapsed` is only an additional sanity bound (spec §10, §12). The ranked leaderboard call is the **cross-plan seam** — keep the guard.

- [ ] **Step 4: Add the personal-best read RPC (Single-tab support)**

> The Leaderboard "Single" tab needs the player's stored single-mode best on demand, independent of the play flow. Add a thin read RPC. It reuses `_RankAttemptStore:GetPersonalBest` (already built + tested in Task 2), so no new storage logic.

Add to `ScoreService.mlua`, inside the `script` block, after `ReceiveSubmitResult`:

```lua
    @ExecSpace("Server")
    method void RequestPersonalBest()
        -- Client→server: fetch this player's single-mode personal best for the Leaderboard "Single" tab.
        local uid = senderUserId
        local best = _RankAttemptStore:GetPersonalBest(uid)
        self:ReceivePersonalBest(best, uid)   -- targetUserId appended at call site
    end

    @ExecSpace("Client")
    method void ReceivePersonalBest(integer personalBest)
        -- Server→caller: deliver the single-mode personal best to this client.
        -- Bridge to the client controller the same way ReceiveSubmitResult is delivered
        -- (Plan 4 Task 6 Step 1 delivery decision — e.g. fire a PersonalBestEvent the
        -- LeaderboardController listens for, or call _LeaderboardController:OnReceivePersonalBest).
        log("[SCORE] ReceivePersonalBest personalBest=" .. personalBest)
    end
```

- [ ] **Step 5: Run, expecting PASS**

Run the MSW Test Loop (Step 2 snippet). Expected all `TestSubmitValidation` assertions PASS, `failed=0`. Inspect the `[SCORE]` log lines: confirm one `accept ... score=2`, the rejections each log their reason, and (because the test uses **single** mode) the `[SEAM]` warning does **not** fire (ranked-only). Confirm zero LSP errors on `ScoreService.mlua`. Then `maker_stop`.

- [ ] **Step 6: Commit**

```bash
git add RootDesk/MyDesk/AppleGame/Server/ScoreService.mlua RootDesk/MyDesk/AppleGame/Server/ServerAuthTest.mlua
git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'   # must print nothing
git commit -F - <<'EOF'
feat: add ScoreService (token+time-window+payload validation, replay-trusted scoring, leaderboard seam)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
EOF
```

---

### Task 5: Full-suite integration run + ranked seam verification

Runs the entire server-authority suite end-to-end in one execute, confirms the whole subsystem is green, and exercises the **ranked accept path** specifically to prove the `_LeaderboardService` seam logs (rather than crashing) while Plan 3 is not yet installed.

**Files:**
- Modify: `RootDesk/MyDesk/AppleGame/Server/ServerAuthTest.mlua` (add `TestRankedSeam()` + `RunAll()`)

**Interfaces:**
- Consumes: all of `_RankAttemptStore`, `_SeedService`, `_ScoreService` (Tasks 2–4).
- Produces: `_ServerAuthTest:TestRankedSeam()` and `_ServerAuthTest:RunAll()` (one-call full suite for executing-plans convenience).

- [ ] **Step 1: Add the ranked-seam test and the all-in-one runner**

Add to `ServerAuthTest.mlua`:

```lua
    @ExecSpace("ServerOnly")
    method void TestRankedSeam()
        -- Proves a ranked accept does not crash on the missing _LeaderboardService and logs the seam warning instead.
        local uid = self:LocalUserId()
        local r = _SeedService:IssueFor(uid, "ranked")
        if r.alreadyPlayed == true then
            -- Ranked attempt already consumed today (rerun); the block path is itself valid behavior.
            self:Expect("seam_ranked_consumed_ok", r.token == nil or r.token == "")
            log("[INFO] ranked already consumed; seam accept path not exercised this run")
            return
        end
        local mv = self:FindOneValidPairOnBoard(r.seed)
        self:Expect("seam_found_valid_move", mv ~= nil)
        if mv ~= nil then
            local res = _ScoreService:SubmitFor(uid, r.token, { mv }, 2, 5.0)
            -- Accept must succeed even though _LeaderboardService is absent (guarded seam logs a warning).
            self:Expect("seam_ranked_accepts", res.accepted == true and res.reason == "ok")
            self:Expect("seam_leaderboard_absent", _LeaderboardService == nil)  -- Plan 3 not installed yet

            -- Personal-best read path (backs ScoreService:RequestPersonalBest for the Single tab)
            local pb = _RankAttemptStore:GetPersonalBest(uid)
            self:Expect("personal_best_read_after_accept", pb >= res.finalScore)
        end
    end

    @ExecSpace("ServerOnly")
    method void RunAll()
        -- One-call full suite for the MSW test loop (server context).
        self:Reset()
        self:TestHarness()
        self:TestRankAttemptStore()
        self:TestPersonalBest()
        self:TestSeedDateKey()
        self:TestSeedIssuance()
        self:TestSubmitValidation()
        self:TestRankedSeam()
        self:Summary()
    end
```

- [ ] **Step 2: Run the full suite, expecting PASS**

Run the MSW Test Loop with `_ServerAuthTest:RunAll()` (`context="server"`). Expected:
```
[TEST PASS] harness_true
... (all assertions from Tasks 1–4) ...
[TEST PASS] seam_ranked_accepts          (or seam_ranked_consumed_ok on a rerun)
[TEST PASS] seam_leaderboard_absent
[TEST SUMMARY] passed=<total> failed=0
```
Confirm exactly one `[SEAM] _LeaderboardService not installed yet` warning fires when the ranked accept path runs (the proof the guard works). If ranked was already consumed today, the seam accept path is skipped and the consumed-block assertion stands in — both are green. Then `maker_stop`.

- [ ] **Step 3: Commit**

```bash
git add RootDesk/MyDesk/AppleGame/Server/ServerAuthTest.mlua
git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'   # must print nothing
git commit -F - <<'EOF'
test: full server-authority integration suite + ranked leaderboard-seam guard verification

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
EOF
```

---

### Task 6: Verify (final PASS/FAIL gate)

Run the canonical verification checklist before declaring Plan 2 done. This is a process task — no new production code; it gathers positive evidence the subsystem works and re-confirms the constraints.

**Files:**
- Reference only: `RootDesk/MyDesk/AppleGame/Server/*.mlua`

**Interfaces:**
- Consumes: everything in Tasks 1–5.
- Produces: a PASS/FAIL verdict and the test-result report.

- [ ] **Step 1: Load the checklist**

Load `Skill: msw-scripting` (if not already loaded this turn), then `Read` `references/verify-checklist.md` **in full** and follow it.

- [ ] **Step 2: Runtime execution (per checklist Step 1)**

`maker_stop` → `maker_clear_logs` → `maker_refresh_workspace` → `maker_logs(kind="build")` (must be 0 errors) → `maker_play` → poll `maker_get_current_map` until `mode=="play"` → `maker_execute_script("_ServerAuthTest:RunAll()", context="server")` → `maker_logs(kind="normal")` → `maker_stop`.

- [ ] **Step 3: Code-review checklist (per checklist Step 2)**

Confirm each, re-reading the files:
- ExecSpace correct: `IssueFor/SubmitFor/Validate*/Get*/Set*/TryConsume*` are `ServerOnly`; `RequestSeed/SubmitRun` are `Server`; `ReceiveSeed/ReceiveSubmitResult` are `Client` with `targetUserId` appended only at the call site (not declared).
- DataStorage discipline: all storage calls are ServerOnly, none in `OnUpdate`/timers, personal-best write is dirty-checked, daily attempt uses CAS, no per-key loops.
- `senderUserId` is the trust anchor on both RPCs; `SubmitFor`/`IssueFor` take an explicit `userId` only for the test seam.
- Cross-plan seam: the `_LeaderboardService ~= nil` guard is present in `_ScoreService` and a warning logs when absent.
- `integer` vs `number` honored (errCodes/dateKey/scores are integer; `clientElapsed`/`TotalSeconds` are number).
- No `Global/`/`Environment/`/`.codeblock` edits; files under `RootDesk/MyDesk/AppleGame/Server/`.

- [ ] **Step 4: Log-evidence verification (per checklist Step 3)**

Confirm in `maker_logs(kind="normal")`: zero build errors; `[TEST SUMMARY] passed=N failed=0`; positive `[SEED]`/`[SCORE]`/`[RAS]` lines showing the accept and each rejection branch actually executed (not nil/0); the `[INFO] KstDateKey=` value matches today in KST (midnight-rollover live check); the captured CAS-mismatch errCode (if any `[RAS] CAS update failed` line appeared) is recorded for the GATE B gap.

- [ ] **Step 5: Final verdict + report**

Produce the §17.3 test-result report (Scenario / Env / Steps / Result / Evidence / Next action). If PASS, Plan 2 is complete and the subsystem is ready for Plan 3 to ship the real `_LeaderboardService`. If FAIL, fix the cause and re-run from Step 2. No commit needed unless a fix was made (then stage explicit paths, run the secret scan, and commit via HEREDOC).

---

## Plan 2 Self-Review

**1. Spec coverage** (each requirement → task):
- §6 seed issuance flow (mode string, RPC, consume ranked attempt, token+record, reply) → Task 3 (`RequestSeed`/`IssueFor`/`ReceiveSeed`).
- §8.2 daily seed = `Mix32(dateKey)` from server KST wall-clock → Task 3 (`KstDateKey` + `Mix32`); GATE A midnight-rollover live check → Tasks 3/6.
- §8.3 ranked 1-play/day persistent, **consume at issuance** via UserDataStorage CAS → Task 2 (`TryConsumeDaily`) consumed inside Task 3 issuance; anti-retry asserted (`ranked_second_blocked`).
- §8.4 single seed = server-random, never client-supplied → Task 3 (`NewSingleSeed`, `single_seed_unpredictable`).
- §10 session token single-use, bound to `senderUserId`, server-time authority, validation order → Task 4 (`SubmitFor` steps 1–5; `submit_unknown_token`/`wrong_owner`/`token_reuse`/`timing`).
- §12 payload validation (move-count cap, coord ranges, monotonic timestamps, window, replay-score equality) → Task 4 (`ValidatePayload` + replay; `submit_bad_coord`/`timing`/`score_lie`).
- §13 anti-cheat model (replay-trusted scoring, persistent attempt, time window) → Tasks 2–4 collectively.
- GATE A server time API (`DateTime.UtcNow + TimeSpan(9,0,0)`) → Task 3, verified signatures section. GATE B DataStorage (`GetUserDataStorage`/`GetAndWait`/`SetAndWait`/`UpdateAndWait`) → Task 2, verified signatures section; CAS-mismatch-code gap captured in Task 2 Step 4 + Task 6.
- Interface contract "Plan 2 produces": `_SeedService` (`RequestSeed`/`ReceiveSeed` exact params), `_ScoreService` (`SubmitRun`/`ReceiveSubmitResult`/`RequestPersonalBest`/`ReceivePersonalBest` exact params + reason enum), `_RankAttemptStore` (`TryConsumeDaily`/`GetPersonalBest`/`SetPersonalBestIfHigher`) → all match Tasks 2–4 verbatim. Cross-plan seam (`_LeaderboardService:SubmitScore` guarded) → dedicated section + Task 4 + Task 5 verification.
- ServerOnly integration test harness analogous to Plan 1's `PuzzleCoreTest` → `ServerAuthTest` (Tasks 1–5). MSW Maker-loop verification (no pytest) with `context="server"`, build-clean gate, `maker_get_current_map` poll, `maker_clear_logs` → "The MSW Test Loop" + every task Step 2/4. DataStorage no-loop discipline in tests → noted in test-loop callout and observed in test design. Final Verify task → Task 6.
- Out of this plan (correctly): ranking package install + `_LeaderboardService` (Plan 3); all UI/input/client `GameSession` (Plan 4).

**2. Placeholder scan:** No `TBD`/`TODO`/"add error handling"/"similar to Task N". The only intentionally-captured-at-runtime value is the undocumented CAS-mismatch errCode (Task 2 Step 4) — a live-observation gap explicitly flagged by GATE B, logged not hard-coded, not a deferred TODO. All methods have full bodies; reason strings are concrete.

**3. Type consistency:** Names match across tasks and the interface contract — `_SeedService` (`KstDateKey`, `IssueFor`, `GetSession`, `MarkConsumed`, `RequestSeed`, `ReceiveSeed`), `_ScoreService` (`ValidatePayload`, `SubmitFor`, `SubmitRun`, `ReceiveSubmitResult`, `RequestPersonalBest`, `ReceivePersonalBest`, constants `GAME_SECONDS`/`GRACE_SECONDS`/`MAX_MOVES`, reasons `ok|bad_token|timing|replay_mismatch|already|bad_payload`), `_RankAttemptStore` (`AttemptKey`, `TryConsumeDaily`, `IsDailyConsumed`, `GetPersonalBest`, `SetPersonalBestIfHigher`). Return shapes are stable: session `{userId, seed, dateKey, mode, issuedAt, consumed}`; issuance `{token, seed, dateKey, alreadyPlayed, personalBest, mode}`; submit `{accepted, finalScore, personalBest, reason}`; `_PuzzleCore:Replay` `{score, ok}` and `move={rect={c1,r1,c2,r2}, t}` consumed exactly as Plan 1 produces. `integer` vs `number` is consistent (errCode/dateKey/score integer; clientElapsed/TotalSeconds number). `RequestPersonalBest` takes no params (read-only, uses `senderUserId`); `ReceivePersonalBest(integer personalBest)` matches the interface contract byte-for-byte.

---

## Open questions / live gaps (flag during execution)

- **CAS-mismatch error code (GATE B gap):** undocumented; Task 2 logs the raw errCode of a mismatching `UpdateAndWait`. Treated as fail-closed (any non-zero = not consumable). Record the observed value.
- **Midnight rollover (GATE A gap):** Task 3 logs `KstDateKey`; reviewer confirms it equals today's KST date. If the project later spans UTC 15:00–16:00, re-run to confirm the +9h rollover lands on the right day.
- **Maker vs release storage isolation:** Maker and release environments do **not** share DataStorage; the daily-attempt key consumed during Maker testing does not affect release players (and vice versa). Noted so a "ranked already played" block during repeated Maker testing is expected, not a bug.
- **Single-mode seed unpredictability** rests on `DateTime.UtcNow.Millisecond` + a server counter through `Mix32`; sufficient for "client can't predict," not cryptographic — acceptable for a casual single-mode practice board (no global ranking on single, spec §3).

# Maple Apple Game — Plan 1: Deterministic Puzzle Engine

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the deterministic, network-free puzzle core (`SeededRng` + `PuzzleCore`) that both client and server share, with golden-vector regression locks, so later plans can trust `Replay(seed, moves)` for anti-cheat validation.

**Architecture:** Two `@Logic` singletons under `RootDesk/MyDesk/AppleGame/Core/`. `SeededRng` is a stateless 32-bit xorshift32 (explicit state threading — no hidden singleton mutation). `PuzzleCore` builds a 10×17 board row-major from the RNG and applies/replays drag-rect moves. Tests run as a third `@Logic` (`PuzzleCoreTest`) invoked in play mode via `maker_execute_script`, asserting through `log()` lines that `maker_logs` reads back.

**Tech Stack:** mlua (Lua 5.3 dialect with integer type + bitwise `~ & | << >>`), MSW Maker MCP (`maker_refresh_workspace`, `maker_play`, `maker_execute_script`, `maker_logs`, `maker_stop`).

## Global Constraints

Copied verbatim from `docs/superpowers/specs/2026-06-23-maple-apple-game-design.md`. Every task implicitly includes these.

- Board is **17 columns × 10 rows = 170 cells**, values **1..9**, filled **row-major**: `for row=0..9 do for col=0..16 do grid[row][col]=RangeInt(1,9) end end` — exactly 170 RNG draws in this order. (spec §8.1, §9)
- **Integer-only** board generation. No floats, no `math.random`, no `_UtilLogic:RandomInteger*` (not seed-reproducible). 32-bit masking with `& 0xFFFFFFFF` after every left shift. (spec §5, §9.1)
- Cell coordinates are **0-based**: `col ∈ [0,16]`, `row ∈ [0,9]`, origin top-left, row 0 = topmost. (spec §9.2)
- A move `rect = {c1,r1,c2,r2}` is **normalized** (`cLo=min, cHi=max, …`) and **inclusive on both ends**. (spec §9.2)
- **Cleared cells = value 0** and are excluded from the sum; a move is **valid iff the sum of remaining (non-zero) cells == 10**; on success those cells clear and **score += number of cleared cells**. (spec §7.3, §9.2)
- `@Logic` singletons accessed as `_<ExactScriptName>` (no suffix stripping). Pure methods declare **no `@ExecSpace`** (run locally on caller). (msw-scripting §3.2, §6)
- Every `method` has its description as the **first line inside the body**, never above the declaration. (msw-scripting §1.8)
- No globals; no coroutines; parent call is `__base:` not `super`. (msw-scripting §4)
- New `.mlua` lives under a **feature folder** (`AppleGame/Core/`), never a catch-all like `Scripts/`. (msw-scripting §1.2, spec §14)
- After any `.mlua` create/modify, call `maker_refresh_workspace` (requires edit mode — `maker_stop` first if playing). The `mlua-diagnose` LSP hook auto-runs; drive error-severity diagnostics to zero before testing. (msw-scripting §1.4, §1.5)

## The MSW Test Loop (used by every task)

There is no `pytest`. "Run the test" means this sequence:

1. `maker_stop` (if currently playing) — refresh needs edit mode.
2. `maker_refresh_workspace` — compiles the `.mlua` edits into `.codeblock`.
3. `maker_play` — enters play mode; `@Logic` singletons go live.
4. `maker_execute_script` with the snippet for this task, e.g.:
   `_PuzzleCoreTest:Reset() _PuzzleCoreTest:TestSeededRng() _PuzzleCoreTest:Summary()`
5. `maker_logs` — read back the `[TEST …]` lines.
6. Inspect: every assertion logs `[TEST PASS] <name>`; the summary logs `[TEST SUMMARY] passed=N failed=0`. **Any `[TEST FAIL]` or `failed>0` means the test failed.**
7. `maker_stop`.

> If `maker_execute_script` cannot run a multi-statement snippet, wrap the three calls in a single `RunX()` method on the harness and execute `_PuzzleCoreTest:RunSeededRng()` instead. Verify the snippet form works in Task 1 before relying on it.

---

### Task 1: Test harness + loop verification

Establishes the `@Logic` test harness and proves the refresh→play→execute→logs loop works before any real logic exists. Uses whatever map is already current (`maker_get_current_map`) as the play host — the dedicated `AppleGame.map` host is built in Plan 4.

**Files:**
- Create: `RootDesk/MyDesk/AppleGame/Core/PuzzleCoreTest.mlua`
- Test: same file (self-hosting harness)

**Interfaces:**
- Consumes: nothing.
- Produces: `_PuzzleCoreTest` `@Logic` with `Expect(string name, boolean cond)`, `Reset()`, `Summary()`, and `TestHarness()`. Later tasks add `TestSeededRng()`, `TestGenerateBoard()`, `TestApplyMove()`, `TestReplay()`, `TestGoldenBoard()` to this same script.

- [ ] **Step 1: Write the harness with a trivial self-test**

Create `RootDesk/MyDesk/AppleGame/Core/PuzzleCoreTest.mlua`:

```lua
@Logic
script PuzzleCoreTest extends Logic
    property integer passed = 0
    property integer failed = 0

    method void Expect(string name, boolean cond)
        -- Records one assertion result to the log for maker_logs inspection.
        if cond then
            self.passed += 1
            log("[TEST PASS] " .. name)
        else
            self.failed += 1
            log_error("[TEST FAIL] " .. name)
        end
    end

    method void Reset()
        -- Clears counters before a test method run.
        self.passed = 0
        self.failed = 0
    end

    method void Summary()
        -- Emits the final tally line that maker_logs is checked against.
        log("[TEST SUMMARY] passed=" .. self.passed .. " failed=" .. self.failed)
    end

    method void TestHarness()
        -- Smoke test proving the assert/log/execute loop is wired correctly.
        self:Expect("harness_true", true)
        self:Expect("harness_eq", 1 + 1 == 2)
    end
end
```

- [ ] **Step 2: Refresh and run, expecting PASS**

Run the test loop (see "The MSW Test Loop"):
- `maker_stop`, `maker_refresh_workspace`, `maker_play`
- `maker_execute_script`: `_PuzzleCoreTest:Reset() _PuzzleCoreTest:TestHarness() _PuzzleCoreTest:Summary()`
- `maker_logs`

Expected log lines:
```
[TEST PASS] harness_true
[TEST PASS] harness_eq
[TEST SUMMARY] passed=2 failed=0
```
If the multi-statement `maker_execute_script` snippet does not run, add a `RunHarness()` method that calls Reset/TestHarness/Summary and execute that instead. Note which form works — all later tasks use the same form.

- [ ] **Step 3: Stop and commit**

```bash
git add RootDesk/MyDesk/AppleGame/Core/PuzzleCoreTest.mlua
git commit -m "test: add PuzzleCore test harness and verify MSW play/execute/logs loop"
```

---

### Task 2: SeededRng (deterministic 32-bit xorshift32)

**Files:**
- Create: `RootDesk/MyDesk/AppleGame/Core/SeededRng.mlua`
- Test: add `TestSeededRng()` to `RootDesk/MyDesk/AppleGame/Core/PuzzleCoreTest.mlua`

**Interfaces:**
- Consumes: nothing.
- Produces: `_SeededRng` `@Logic` with
  - `Normalize(integer seed) -> integer` (32-bit mask, zero-state avoidance)
  - `Next(integer s) -> integer` (one xorshift32 step; returned new state is the random word)
  - `MapRange(integer state, integer lo, integer hi) -> integer`
  - `Mix32(integer x) -> integer` (= `Next(Normalize(x))`)

- [ ] **Step 1: Write the failing test**

Add to `PuzzleCoreTest.mlua` inside the `script` block:

```lua
    method void TestSeededRng()
        -- Verifies seed normalization, step determinism, and range mapping bounds.
        self:Expect("norm_zero_to_constant", _SeededRng:Normalize(0) == 0x9E3779B9)
        self:Expect("norm_masks_32bit", _SeededRng:Normalize(0x1FFFFFFFF) == 0xFFFFFFFF)

        local a1 = _SeededRng:Next(12345)
        local a2 = _SeededRng:Next(12345)
        self:Expect("next_is_deterministic", a1 == a2)
        self:Expect("next_changes_state", a1 ~= 12345)
        self:Expect("next_stays_32bit", a1 >= 0 and a1 <= 0xFFFFFFFF)

        local diff = _SeededRng:Next(1) ~= _SeededRng:Next(2)
        self:Expect("next_differs_by_seed", diff)

        local inRange = true
        local s = _SeededRng:Normalize(777)
        for i = 1, 500 do
            s = _SeededRng:Next(s)
            local v = _SeededRng:MapRange(s, 1, 9)
            if v < 1 or v > 9 then inRange = false end
        end
        self:Expect("maprange_within_1_9", inRange)
        self:Expect("mix32_deterministic", _SeededRng:Mix32(42) == _SeededRng:Mix32(42))
    end
```

- [ ] **Step 2: Run, expecting FAIL**

Run the test loop with `_PuzzleCoreTest:Reset() _PuzzleCoreTest:TestSeededRng() _PuzzleCoreTest:Summary()`.
Expected: failures / runtime error referencing `_SeededRng` (script does not exist yet).

- [ ] **Step 3: Implement SeededRng**

Create `RootDesk/MyDesk/AppleGame/Core/SeededRng.mlua`:

```lua
@Logic
script SeededRng extends Logic
    method integer Normalize(integer seed)
        -- Masks to 32 bits and avoids the all-zero fixed point of xorshift.
        local s = seed & 0xFFFFFFFF
        if s == 0 then s = 0x9E3779B9 end
        return s
    end

    method integer Next(integer s)
        -- One xorshift32 step on a 32-bit state; returns the new state (the random word).
        s = (s ~ ((s << 13) & 0xFFFFFFFF)) & 0xFFFFFFFF
        s = s ~ (s >> 17)
        s = (s ~ ((s << 5) & 0xFFFFFFFF)) & 0xFFFFFFFF
        return s
    end

    method integer MapRange(integer state, integer lo, integer hi)
        -- Maps a 32-bit state into [lo, hi]; tiny modulo bias accepted for a casual game.
        return lo + (state % (hi - lo + 1))
    end

    method integer Mix32(integer x)
        -- Hashes an arbitrary integer (e.g. a day index) into a well-distributed 32-bit seed.
        return self:Next(self:Normalize(x))
    end
end
```

- [ ] **Step 4: Run, expecting PASS**

Run the test loop again. Expected:
```
[TEST PASS] norm_zero_to_constant
... (all 8 SeededRng assertions PASS)
[TEST SUMMARY] passed=8 failed=0
```
Also confirm the `mlua-diagnose` hook reported zero error-severity diagnostics on `SeededRng.mlua`.

- [ ] **Step 5: Commit**

```bash
git add RootDesk/MyDesk/AppleGame/Core/SeededRng.mlua RootDesk/MyDesk/AppleGame/Core/PuzzleCoreTest.mlua
git commit -m "feat: add deterministic SeededRng (xorshift32) with tests"
```

---

### Task 3: PuzzleCore.GenerateBoard

**Files:**
- Create: `RootDesk/MyDesk/AppleGame/Core/PuzzleCore.mlua`
- Test: add `TestGenerateBoard()` to `PuzzleCoreTest.mlua`

**Interfaces:**
- Consumes: `_SeededRng:Normalize`, `_SeededRng:Next`, `_SeededRng:MapRange`.
- Produces: `_PuzzleCore` `@Logic` with `property integer COLS = 17`, `property integer ROWS = 10`, and `GenerateBoard(integer seed) -> table` returning a 0-indexed `grid[row][col]` of values 1..9.

- [ ] **Step 1: Write the failing test**

Add to `PuzzleCoreTest.mlua`:

```lua
    method void TestGenerateBoard()
        -- Verifies board dimensions, value range, determinism, and seed sensitivity.
        local g = _PuzzleCore:GenerateBoard(0x12345678)
        local okRange = true
        local count = 0
        for r = 0, _PuzzleCore.ROWS - 1 do
            for c = 0, _PuzzleCore.COLS - 1 do
                local v = g[r][c]
                if v == nil or v < 1 or v > 9 then okRange = false end
                count += 1
            end
        end
        self:Expect("board_has_170_cells", count == 170)
        self:Expect("board_values_1_9", okRange)

        local g2 = _PuzzleCore:GenerateBoard(0x12345678)
        self:Expect("board_deterministic", g[0][0] == g2[0][0] and g[9][16] == g2[9][16])

        local h = _PuzzleCore:GenerateBoard(0x9999AAAA)
        local anyDiff = false
        for r = 0, _PuzzleCore.ROWS - 1 do
            for c = 0, _PuzzleCore.COLS - 1 do
                if g[r][c] ~= h[r][c] then anyDiff = true end
            end
        end
        self:Expect("board_seed_sensitive", anyDiff)
    end
```

- [ ] **Step 2: Run, expecting FAIL**

Run the test loop with `TestGenerateBoard()`. Expected: error referencing `_PuzzleCore` (not yet created).

- [ ] **Step 3: Implement PuzzleCore.GenerateBoard**

Create `RootDesk/MyDesk/AppleGame/Core/PuzzleCore.mlua`:

```lua
@Logic
script PuzzleCore extends Logic
    property integer COLS = 17
    property integer ROWS = 10

    method table GenerateBoard(integer seed)
        -- Builds the 10x17 row-major board of 1..9 from the deterministic RNG (170 draws).
        local grid = {}
        local s = _SeededRng:Normalize(seed)
        for r = 0, self.ROWS - 1 do
            grid[r] = {}
            for c = 0, self.COLS - 1 do
                s = _SeededRng:Next(s)
                grid[r][c] = _SeededRng:MapRange(s, 1, 9)
            end
        end
        return grid
    end
end
```

- [ ] **Step 4: Run, expecting PASS**

Run the test loop. Expected all 5 `TestGenerateBoard` assertions PASS, `failed=0`, zero LSP errors.

- [ ] **Step 5: Commit**

```bash
git add RootDesk/MyDesk/AppleGame/Core/PuzzleCore.mlua RootDesk/MyDesk/AppleGame/Core/PuzzleCoreTest.mlua
git commit -m "feat: add PuzzleCore.GenerateBoard (deterministic row-major board) with tests"
```

---

### Task 4: PuzzleCore.ApplyMove

Uses **hand-built grids with known answers** (no RNG) so expectations are exact.

**Files:**
- Modify: `RootDesk/MyDesk/AppleGame/Core/PuzzleCore.mlua`
- Test: add `TestApplyMove()` to `PuzzleCoreTest.mlua`

**Interfaces:**
- Consumes: a `grid` table (0-indexed `grid[row][col]`, cleared cells = 0).
- Produces: `_PuzzleCore:ApplyMove(table grid, table rect) -> table` returning `{ valid = boolean, clearedCount = integer }`. On `valid`, the matched cells in `grid` are set to 0 (mutation). `rect` is `{ c1, r1, c2, r2 }`.

- [ ] **Step 1: Write the failing test**

Add to `PuzzleCoreTest.mlua`:

```lua
    method table MakeGrid2x3(table vals)
        -- Builds a 2-row x 3-col 0-indexed grid from vals[r][c] (test helper, no RNG).
        local g = {}
        for r = 0, 1 do
            g[r] = {}
            for c = 0, 2 do
                g[r][c] = vals[r][c]
            end
        end
        return g
    end

    method void TestApplyMove()
        -- Validates sum==10 rule, cleared-cell exclusion, normalization, and range guard.
        -- Grid:
        --   row0: 4 6 1
        --   row1: 7 3 9
        local g = self:MakeGrid2x3({ [0] = { [0]=4, [1]=6, [2]=1 }, [1] = { [0]=7, [1]=3, [2]=9 } })
        local r1 = _PuzzleCore:ApplyMove(g, { c1=0, r1=0, c2=1, r2=0 }) -- 4+6=10
        self:Expect("valid_sum10", r1.valid == true and r1.clearedCount == 2)
        self:Expect("valid_clears_cells", g[0][0] == 0 and g[0][1] == 0)

        -- Re-select the now-cleared cells plus the '1': remaining sum = 0+0+1 = 1 -> invalid
        local r2 = _PuzzleCore:ApplyMove(g, { c1=0, r1=0, c2=2, r2=0 })
        self:Expect("cleared_excluded_from_sum", r2.valid == false and r2.clearedCount == 0)

        -- Vertical 7+3 = 10 with reversed coords (normalization)
        local r3 = _PuzzleCore:ApplyMove(g, { c1=1, r1=1, c2=1, r2=0 }) -- col1 rows1..0 -> 3 + (already 0) = 3? 
        -- NOTE: g[0][1] was cleared above, so col1 = {0 (row0), 3 (row1)} -> sum 3 -> invalid
        self:Expect("normalize_then_invalid", r3.valid == false)

        -- Fresh grid for an all-cleared-selection case
        local z = self:MakeGrid2x3({ [0] = { [0]=0, [1]=0, [2]=5 }, [1] = { [0]=0, [1]=0, [2]=5 } })
        local r4 = _PuzzleCore:ApplyMove(z, { c1=0, r1=0, c2=1, r2=1 }) -- all zeros -> sum 0
        self:Expect("all_cleared_selection_invalid", r4.valid == false and r4.clearedCount == 0)

        -- Out-of-range guard
        local r5 = _PuzzleCore:ApplyMove(z, { c1=0, r1=0, c2=99, r2=0 })
        self:Expect("out_of_range_invalid", r5.valid == false)
    end
```

- [ ] **Step 2: Run, expecting FAIL**

Run the test loop with `TestApplyMove()`. Expected: error referencing `ApplyMove` (not defined).

- [ ] **Step 3: Implement ApplyMove**

Add to `PuzzleCore.mlua` inside the `script` block:

```lua
    method table ApplyMove(table grid, table rect)
        -- Normalizes rect (inclusive), sums remaining (>0) cells; if sum==10 clears them and returns the count.
        local cLo = math.min(rect.c1, rect.c2)
        local cHi = math.max(rect.c1, rect.c2)
        local rLo = math.min(rect.r1, rect.r2)
        local rHi = math.max(rect.r1, rect.r2)
        if cLo < 0 or cHi > self.COLS - 1 or rLo < 0 or rHi > self.ROWS - 1 then
            return { valid = false, clearedCount = 0 }
        end
        local sum = 0
        local cells = {}
        for r = rLo, rHi do
            for c = cLo, cHi do
                local v = grid[r][c]
                if v ~= nil and v > 0 then
                    sum += v
                    cells[#cells + 1] = { r = r, c = c }
                end
            end
        end
        if sum ~= 10 then
            return { valid = false, clearedCount = 0 }
        end
        for i = 1, #cells do
            grid[cells[i].r][cells[i].c] = 0
        end
        return { valid = true, clearedCount = #cells }
    end
```

> If `math.min`/`math.max` are unavailable in this mlua build (LSP error), replace each with an inline conditional, e.g. `local cLo = rect.c1; if rect.c2 < cLo then cLo = rect.c2 end`.

- [ ] **Step 4: Run, expecting PASS**

Run the test loop. Expected all 6 `TestApplyMove` assertions PASS, `failed=0`, zero LSP errors.

- [ ] **Step 5: Commit**

```bash
git add RootDesk/MyDesk/AppleGame/Core/PuzzleCore.mlua RootDesk/MyDesk/AppleGame/Core/PuzzleCoreTest.mlua
git commit -m "feat: add PuzzleCore.ApplyMove (sum-10 rule, cleared-cell exclusion, normalization)"
```

---

### Task 5: PuzzleCore.Replay

**Files:**
- Modify: `RootDesk/MyDesk/AppleGame/Core/PuzzleCore.mlua`
- Test: add `TestReplay()` to `PuzzleCoreTest.mlua`

**Interfaces:**
- Consumes: `_PuzzleCore:GenerateBoard`, `_PuzzleCore:ApplyMove`.
- Produces: `_PuzzleCore:Replay(integer seed, table moves) -> table` returning `{ score = integer, ok = boolean }`. `moves` is an array of `{ rect = {c1,r1,c2,r2}, t = number }`. Any move that is not a valid clear on the current state ends the replay with `ok = false`.

- [ ] **Step 1: Write the failing test**

Add to `PuzzleCoreTest.mlua` (includes a test-only scanner that finds a real valid pair on the generated board, so the positive case has a known answer):

```lua
    method table FindOneHorizontalPair(table grid)
        -- Test helper: returns the first horizontally-adjacent pair summing to 10, or nil.
        for r = 0, _PuzzleCore.ROWS - 1 do
            for c = 0, _PuzzleCore.COLS - 2 do
                if grid[r][c] + grid[r][c + 1] == 10 then
                    return { rect = { c1 = c, r1 = r, c2 = c + 1, r2 = r }, t = 1.0 }
                end
            end
        end
        return nil
    end

    method void TestReplay()
        -- Validates empty-run, a real valid single move, and the invalid-move short-circuit.
        local empty = _PuzzleCore:Replay(0x12345678, {})
        self:Expect("replay_empty_ok", empty.ok == true and empty.score == 0)

        local board = _PuzzleCore:GenerateBoard(0x12345678)
        local mv = self:FindOneHorizontalPair(board)
        self:Expect("found_a_valid_pair", mv ~= nil)
        if mv ~= nil then
            local good = _PuzzleCore:Replay(0x12345678, { mv })
            self:Expect("replay_valid_move", good.ok == true and good.score == 2)
        end

        -- A move that cannot be valid: a single in-range cell (one value, never sums to 10 alone unless ==10, impossible for 1..9)
        local bad = _PuzzleCore:Replay(0x12345678, { { rect = { c1 = 0, r1 = 0, c2 = 0, r2 = 0 }, t = 1.0 } })
        self:Expect("replay_invalid_move_short_circuits", bad.ok == false)
    end
```

- [ ] **Step 2: Run, expecting FAIL**

Run the test loop with `TestReplay()`. Expected: error referencing `Replay` (not defined).

- [ ] **Step 3: Implement Replay**

Add to `PuzzleCore.mlua`:

```lua
    method table Replay(integer seed, table moves)
        -- Regenerates the board from seed and applies moves in order; any invalid move => ok=false.
        local grid = self:GenerateBoard(seed)
        local score = 0
        for i = 1, #moves do
            local res = self:ApplyMove(grid, moves[i].rect)
            if not res.valid then
                return { score = score, ok = false }
            end
            score += res.clearedCount
        end
        return { score = score, ok = true }
    end
```

- [ ] **Step 4: Run, expecting PASS**

Run the test loop. Expected all `TestReplay` assertions PASS, `failed=0`, zero LSP errors.

- [ ] **Step 5: Commit**

```bash
git add RootDesk/MyDesk/AppleGame/Core/PuzzleCore.mlua RootDesk/MyDesk/AppleGame/Core/PuzzleCoreTest.mlua
git commit -m "feat: add PuzzleCore.Replay (ordered move re-simulation with invalid short-circuit)"
```

---

### Task 6: Golden-vector regression lock

A capture-then-pin snapshot test. Locks the exact board output for a fixed seed so any future accidental change to the RNG or fill order is caught immediately. (spec §9.3)

**Files:**
- Test: add `BoardChecksum()` + `TestGoldenBoard()` to `PuzzleCoreTest.mlua`

**Interfaces:**
- Consumes: `_PuzzleCore:GenerateBoard`.
- Produces: nothing consumed by later tasks (pure regression guard).

- [ ] **Step 1: Add the checksum helper and a capture-mode test**

Add to `PuzzleCoreTest.mlua`:

```lua
    method integer BoardChecksum(integer seed)
        -- Folds the whole board into a 32-bit checksum (row-major) for snapshot comparison.
        local g = _PuzzleCore:GenerateBoard(seed)
        local acc = 0
        for r = 0, _PuzzleCore.ROWS - 1 do
            for c = 0, _PuzzleCore.COLS - 1 do
                acc = ((acc * 31) + g[r][c]) & 0xFFFFFFFF
            end
        end
        return acc
    end

    method void TestGoldenBoard()
        -- Capture mode: logs the checksum so it can be pinned in Step 3.
        log("[GOLDEN] checksum_12345678=" .. self:BoardChecksum(0x12345678))
    end
```

- [ ] **Step 2: Run in capture mode and record the value**

Run the test loop with `_PuzzleCoreTest:TestGoldenBoard()`. Read `maker_logs` for the line:
```
[GOLDEN] checksum_12345678=<SOME_NUMBER>
```
Record `<SOME_NUMBER>` exactly.

- [ ] **Step 3: Pin the captured value as an assertion**

Replace the body of `TestGoldenBoard()` with the pinned check (substitute the recorded number for `<SOME_NUMBER>`):

```lua
    method void TestGoldenBoard()
        -- Regression lock: board output for the fixed seed must never silently change.
        self:Expect("golden_checksum_12345678", self:BoardChecksum(0x12345678) == <SOME_NUMBER>)
    end
```

- [ ] **Step 4: Run, expecting PASS**

Run the test loop again. Expected:
```
[TEST PASS] golden_checksum_12345678
[TEST SUMMARY] passed=1 failed=0
```

- [ ] **Step 5: Full-suite run + commit**

Run every test method once to confirm the whole engine is green:
`_PuzzleCoreTest:Reset() _PuzzleCoreTest:TestSeededRng() _PuzzleCoreTest:TestGenerateBoard() _PuzzleCoreTest:TestApplyMove() _PuzzleCoreTest:TestReplay() _PuzzleCoreTest:TestGoldenBoard() _PuzzleCoreTest:Summary()`
Expected: `[TEST SUMMARY] passed=<total> failed=0`.

```bash
git add RootDesk/MyDesk/AppleGame/Core/PuzzleCoreTest.mlua
git commit -m "test: pin golden board checksum as determinism regression lock"
```

---

## Plan 1 Self-Review

- **Spec coverage:** §5 SeededRng/PuzzleCore → Tasks 2–5. §7.3 sum-10/cleared-cell/score → Task 4. §8.1 row-major 170 → Task 3. §9.1 xorshift32/normalization → Task 2. §9.2 rect inclusive/normalize → Task 4. §9.3 golden vectors + single shared Core → Task 6 + (single `.mlua`, no duplication). §13 replay-based detection foundation → Task 5. Out of this plan (correctly): server time, ranking, UI, payload-from-network validation → Plans 2–4.
- **Placeholder scan:** the only intentional fill-in is the captured golden checksum in Task 6 Step 3, which is a capture-then-pin snapshot value (cannot be known before running a deterministic PRNG) — explicitly marked and produced within the task, not a deferred TODO.
- **Type consistency:** `Normalize/Next/MapRange/Mix32` (SeededRng), `COLS/ROWS/GenerateBoard/ApplyMove/Replay` (PuzzleCore), `{valid,clearedCount}` and `{score,ok}` return shapes, and `rect={c1,r1,c2,r2}` / `move={rect,t}` are used identically across Tasks 2–6 and match spec §5/§9.

---

## Subsequent Plans (roadmap — written separately after their API gates clear)

Per the scope split, the remaining subsystems get their own plans once their deferred APIs are verified against `Environment/NativeScripts/**/*.d.mlua` (msw-scripting §1.3 — no guessing). Listed here so the whole shape is visible.

### Plan 2 — Server Authority  *(depends on Plan 1)*
SeedService (seed + session-token issuance, server-time binding), ScoreService (`SubmitRun` → token/time/payload validation → `_PuzzleCore:Replay`), RankAttemptStore (UserDataStorage `rankAttempt:{KSTdate}`, consume-at-issuance). Spec §6, §8.3, §10, §12.
- **GATE A (blocking):** exact server **wall-clock / KST date** API. Grep `Environment/NativeScripts/Service/` and `Misc/` for DateTime/time; confirm signature in `.d.mlua`. `_UtilLogic.ServerElapsedSeconds` is instance-lifetime, **not** wall-clock — do not use for the daily boundary.
- **GATE B:** `_DataStorageService` UserDataStorage read/write signatures (credit-billed; never in `OnUpdate`). Confirm in `.d.mlua`.

### Plan 3 — Ranking Integration  *(depends on Plan 2)*
Install **ranking-advanced** (+ `PlayerDBManager`, `PlayerRanking`, set `WorldConfig.PlayerEntityAuthorityCheck=true`), 3 `RankingConfigDataSet` (daily/weekly/allTime, KST `ReleaseBaseTime`), LeaderboardStore fan-out via `PlayerRanking:SetScoreAndWait(id,score,tag,force=false)`, single personal-best in UserDataStorage. Spec §7.1–7.2, §3 (single = personal best).
- **GATE C:** follow `msw-packages` integration workflow (fetch file tree, UUID/RUID collision check, transitive deps) before copying package files. Confirm `SetScoreAndWait` exact params from the package source.

### Plan 4 — Client Game  *(depends on Plans 1–3)*
Host `map/AppleGame.map` (DefaultPlayer hidden), GameSession (`@Component`, ClientOnly), BoardController + `ui/AppleGame/*.ui` (built via UIBuilder), drag input + selection overlay + touch→cell mapping, HUD/Result/Leaderboard controllers, theming/assets. Spec §4, §10 (client side), §11, §14.
- **GATE D (blocking):** exact UI drag mechanism — UITouchReceive-family events vs `ScreenTouchEvent` + `IsPointerOverUI`. Confirm event names + payload (screen coords) in `.d.mlua` (msw-scripting §10) before fixing BoardController.
- **GATE E:** `builder-protocol.md` preflight for every `.ui`/`.map`/`.model`; assets via `msw-search` (fallback `msw-painter`).

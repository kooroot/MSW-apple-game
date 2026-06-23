# Maple Apple Game — Plan 4: Client Game + UI

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the entire playable client of the apple game — host map `map/AppleGame.map`, a UI-rendered 17×10 board with drag-rectangle selection, HUD (score / 120s timer bar / mode), Result popup, and a 5-tab Leaderboard popup (Day/Week/AllTime ranking + Single personal-best + locked Multi) — wiring it to the already-shipped `_SeedService` / `_ScoreService` / `_LeaderboardService` (Plans 2–3) and `_PuzzleCore` / `_SeededRng` (Plan 1).

**Architecture:** Gameplay is **100% UI-driven** — the board is a UI grid of cell entities under `ui/AppleGame/AppleBoard.ui`, not world entities, so there is no physics/movement. A thin host map (`map/AppleGame.map`, **RectTile / TileMapMode=1**, no gravity) carries one map-scoped `@Component` (`GameSession`) that owns a play session: on map enter it requests a seed, generates the board via the shared `_PuzzleCore`, renders it, runs a 120s timer, records the `moves` log, and submits on timeout. Drag selection comes from a full-screen `UITouchReceiveComponent` panel layered over the grid (GATE D: `UITouchDownEvent`→anchor, `UITouchDragEvent`→current corner, `UITouchUpEvent`→finalize), with `_UILogic:ScreenToLocalUIPosition(TouchPoint, gridTransform)` ÷ cell size mapping screen→cell. All UI/input is `@ExecSpace("ClientOnly")`; all server calls go through the Plan 2/3 RPCs.

**Tech Stack:** mlua (Lua 5.3 dialect, `@Component`/`@Logic`, `@ExecSpace("ClientOnly")`); MSW `.ui` via **UIBuilder** (`msw-ui-system/scripts/msw_ui_builder.cjs`, Node CJS); `.map` via **MapBuilder**; Maker MCP loop (`maker_refresh_workspace`, `maker_play`, `maker_get_current_map`, `maker_mouse_input`, `maker_keyboard_input`, `maker_logs`, `maker_screenshot`, `maker_stop`); resources via `msw-search`.

## Global Constraints

Copied verbatim (or distilled) from the spec (`2026-06-23-maple-apple-game-design.md` §4/§10/§11/§14/§16), the interface contract (`2026-06-23-apple-game-interfaces.md` Plan 4), and the API gates (`2026-06-23-apple-game-api-gates.md` GATE D). Every task implicitly includes these.

- **Folder layout:** `.mlua` under `RootDesk/MyDesk/AppleGame/Client/` (and `Data/` for `AssetCatalog`), `.ui` under `ui/AppleGame/`, `.map` is `map/AppleGame.map`. Never a catch-all folder like `Scripts/` / `UI/` / `Common/`. (spec §14, msw-scripting §1.2)
- **Map type:** `map/AppleGame.map` must be **RectTile (TileMapMode=1, no gravity)** so the hidden DefaultPlayer avatar does not fall through (no Foothold) and no LEA-3004 fires. Confirm with `MapBuilder.read("map/AppleGame.map").getTileMapMode()` before any map work; if it is not 1, this is a **USER ACTION IN MAKER** (Hierarchy → right-click map entity → "Switch RectTileMap") — the AI must NEVER write `TileMapMode` into `.map` JSON. (platform.md §4, entity.md)
- **`.ui` is builder-only.** Every `.ui` create/modify goes through `UIBuilder` (`b.panel/text/sprite/button/touchReceive/group/...` → `b.write()`). NEVER hand-edit, `Read`, `cat`, `grep`, or `sed` raw `.ui` JSON — a registered guard blocks it. Read existing `.ui` via `UIBuilder.read/snapshot/find/listEntities`. **Read `msw-general/references/builder-protocol.md` §3 in full at the start of every turn that touches a `.ui` / `.map` / `.model`.** (builder-protocol.md, msw-ui-system §2)
- **All UI Logic / Component must be `@ExecSpace("ClientOnly")`.** UI entities are client-only: hosting `@ExecSpace("Server"/"ServerOnly"/"Multicast")` or `@Sync` on a UI-attached component silently no-ops; referencing a UI entity from server code returns nil. Client→server goes through the Plan 2/3 `@ExecSpace("Server")` RPCs; server→client replies arrive via `@ExecSpace("Client")` RPCs declared in those services. (runtime-patterns.md Caveats, msw-scripting §6)
- **GameSession lifetime is map-scoped** → it is a `@Component` on the `AppleGame.map` entity (placed via `MapBuilder.upsertComponent`), NOT a `@Logic`. `OnMapEnter` / `OnMapLeave` fire only on Components, never on Logic. The UI controllers (`BoardController`, `HudController`, `ResultController`, `LeaderboardController`) live as `script(...)` entities inside their `.ui` groups. (msw-scripting §3.2, spec §5)
- **Coordinates:** world coords are world units (1 unit = 100 px) — but **UI uses `anchoredPosition` in 1920×1080 canvas space** (center origin `(0,0)`, X ±960, Y ±540). Never set `Position` directly on a UI entity; never use raw pixels for world entities. (platform.md §5, ui-fundamentals via builder-protocol §3.4)
- **UI resolution / orientation:** 1920×1080 **landscape**. HUD ~140px tall, board usable ≈ 1700×920, square cell = `min(1700/17, 920/10) = 92px`, board 1564×920 centered. Respect safe-area insets (no core controls under a notch). (spec §11, §16)
- **Touch→cell formula (fixed):** `local = _UILogic:ScreenToLocalUIPosition(TouchPoint, gridTransform)`; `col = clamp(floor(localX / cellW), 0, 16)`; `row = clamp(floor(yFromTop / cellH), 0, 9)` where `yFromTop` flips the UI y-axis so row 0 = topmost cell. (spec §11, GATE D)
- **GATE D event names (verified — do NOT guess):** `UITouchReceiveComponent` on a full-screen panel emits `UITouchDownEvent` (Entity, TouchId, TouchPoint), `UITouchDragEvent` (+TouchDelta), `UITouchUpEvent`. `TouchId`: PC left = **-1**, mobile single-finger = **≥1**. Gate to `TouchId == 1 or TouchId == -1`. Connect on the **Entity**, not the component. `_UILogic` is ClientOnly. (GATE D, runtime-patterns.md §8, component-api.md)
- **mode** crosses RPC as the **string** `"ranked"` | `"single"` (no enums over RPC). `moves` is a `table` array of `{rect={c1,r1,c2,r2}, t=number}`; `t` = client seconds-since-start, monotonic non-decreasing. (interfaces Conventions, spec §12)
- **Public PUBLIC repo — never `git add -A`.** Stage only explicit paths. Before each commit run `git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'` and confirm it is **empty**. Commit message trailers via HEREDOC: `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>` and `Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5`.
- After any `.mlua` create/modify call `maker_refresh_workspace` (edit mode — `maker_stop` first if playing). The `mlua-diagnose` LSP hook auto-runs; drive error-severity diagnostics to zero before testing. Never touch `.codeblock` (Maker regenerates it). Never edit `Global/` or `Environment/`. (msw-scripting §1.4/§1.5)

## The MSW Verification Loop (used by every task — NO pytest)

There is no `pytest`. "Verify" = the Maker MCP loop. Two flavors:

**A. Build-clean + screenshot loop (UI / map / integration tasks):**
1. `maker_stop` (if playing) — refresh needs edit mode.
2. `maker_refresh_workspace` — ingest `.ui` / `.map` / `.mlua` edits, compile `.codeblock`.
3. `maker_logs(kind="build")` — must show **0 errors**. Fix → refresh → re-check until clean.
4. `maker_play` — enter play mode.
5. Poll `maker_get_current_map` until its status is `"play"` (the play host is live).
6. Drive input with `maker_mouse_input` (press → move → release at computed **screen** coords, to simulate a drag) and/or `maker_keyboard_input`.
7. `maker_logs(kind="normal")` — read `log()` checkpoints; assertion-style lines end in `[TEST PASS]` / values are non-nil/expected.
8. `maker_screenshot` — confirm the board / selection rect / HUD actually render.
9. `maker_stop`.

**B. ClientOnly `@Logic` test-harness loop (pure client math, e.g. screen→cell mapping):** identical to Plan 1 — add assertions to a `_AppleClientTest` `@Logic`, then in play mode run `maker_execute_script("_AppleClientTest:Reset() _AppleClientTest:TestX() _AppleClientTest:Summary()", context="client")` and read `[TEST PASS]` / `[TEST SUMMARY] passed=N failed=0` via `maker_logs(kind="normal")`. Use `context="client"` because `_UILogic` and all UI are client-only.

> **GATE D live-confirm items** (make explicit checks in the tasks that touch them): (1) PC `UITouchDragEvent` firing threshold — a slow drag may drop the first few pixels; treat `UITouchDownEvent` as the **definitive anchor**, never the first drag event. (2) Whether the touch panel needs `RaycastTarget=true` to receive events (the `touchReceive()` builder note says it works without it, but a same-position sprite above it can swallow events — verify live). (3) `TouchId` gating actually filters multi-finger. These are verified in Task 5.

---

### Task 1: Host map `map/AppleGame.map` + DefaultPlayer hide

Stands up the play host and confirms its mode before any UI exists. The map is a near-empty RectTile container; the player avatar is hidden at runtime by a tiny ClientOnly component so we never touch `Global/DefaultPlayer.model`.

**Files:**
- Inspect/Modify: `map/AppleGame.map` (via `MapBuilder` only)
- Create: `RootDesk/MyDesk/AppleGame/Client/HidePlayerAvatar.mlua`

**Interfaces:**
- Consumes: nothing.
- Produces: `map/AppleGame.map` confirmed `TileMapMode == 1`; `script.HidePlayerAvatar` (a `@Component`, ClientOnly) that disables the local player's `SpriteRendererComponent` / `AvatarRendererComponent` renderer in `OnBeginPlay`. Later tasks place `GameSession` and the four `.ui` groups against this map.

- [ ] **Step 1: Read builder-protocol.md §1 then confirm the map mode**

Read `msw-general/references/builder-protocol.md` in full (this turn). Then from a Node script with CWD at the `msw-general` skill root:

```javascript
const { MapBuilder } = require("./scripts/map/msw_map_builder.cjs");
console.log(MapBuilder.read("map/AppleGame.map").getMapInfo());
console.log("TileMapMode =", MapBuilder.read("map/AppleGame.map").getTileMapMode());
```

Expected: `TileMapMode = 1`.
- If the file does not exist yet, that is a **USER ACTION IN MAKER**: ask the user to create a new map named `AppleGame`, switch it to **RectTileMap** (Hierarchy → right-click map entity → "Switch RectTileMap"), save, then re-run this step.
- If it exists but `getTileMapMode()` is not `1`, **USER ACTION IN MAKER**: Hierarchy → right-click the `AppleGame` map entity → "Switch RectTileMap" → save → `maker_refresh_workspace` → re-read to confirm `1`. The AI must NOT write `TileMapMode` into the JSON.

- [ ] **Step 2: Write the avatar-hide component**

Create `RootDesk/MyDesk/AppleGame/Client/HidePlayerAvatar.mlua`:

```lua
@Component
@ExecSpace("ClientOnly")
script HidePlayerAvatar extends Component

    method void OnBeginPlay()
        -- Hides the local player's avatar renderer so the UI puzzle has no visible character; entity stays alive for ranking components.
        local player = _UserService.LocalPlayer
        if not isvalid(player) then
            log_warning("[HidePlayerAvatar] no LocalPlayer at BeginPlay")
            return
        end
        local avatar = player.AvatarRendererComponent
        if isvalid(avatar) then avatar.Enable = false end
        local sprite = player.SpriteRendererComponent
        if isvalid(sprite) then sprite.Enable = false end
        log("[HidePlayerAvatar] avatar hidden for " .. tostring(player.Name))
    end

end
```

> The exact renderer component on DefaultPlayer (`AvatarRendererComponent` vs `SpriteRendererComponent`) is verified in Step 4 by log/screenshot. The component above tries both and logs which existed. `_UserService.LocalPlayer` is the local client's player entity (ClientOnly context). If neither renderer disables the avatar, fall back to the **USER ACTION IN MAKER** alternative noted in Step 5.

- [ ] **Step 3: Attach the component to the map entity and refresh**

`MapBuilder` — attach the script to the map root so it runs on map enter (it is a one-off, map-local behavior):

```javascript
const { MapBuilder } = require("./scripts/map/msw_map_builder.cjs");
const map = MapBuilder.read("map/AppleGame.map");
const rootName = map.getMapInfo().rootName || "AppleGame"; // use the actual root name from getMapInfo()
map.upsertComponent(rootName, "script.HidePlayerAvatar", { "@type": "script.HidePlayerAvatar", "Enable": true })
   .write("map/AppleGame.map");
```

Then `maker_stop` (if playing) → `maker_refresh_workspace` → `maker_logs(kind="build")` (expect 0 errors).

- [ ] **Step 4: Verify in play mode (Loop A)**

- `maker_play`; poll `maker_get_current_map` until status `"play"`. (Ensure the current map is `AppleGame`; if not, the user must enter it — note it.)
- `maker_logs(kind="normal")` — expect `[HidePlayerAvatar] avatar hidden for <name>` (proves it ran and which renderer existed).
- `maker_screenshot` — confirm **no player avatar** is visible on the empty map.
- `maker_stop`.

- [ ] **Step 5: Commit (and DefaultPlayer fallback note)**

If the avatar is still visible after Step 4, the renderer toggle was insufficient → **USER ACTION IN MAKER**: tell the user to either (a) move the player spawn off-camera on `AppleGame.map`, or (b) copy `Global/DefaultPlayer.model` into `RootDesk/MyDesk/Models/AppleGame/` and disable its renderer there (AI cannot edit `Global/`). Do not proceed past this without a confirmed-hidden avatar.

```bash
git add map/AppleGame.map RootDesk/MyDesk/AppleGame/Client/HidePlayerAvatar.mlua
git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'   # must print nothing
git commit -F- <<'EOF'
feat(applegame): RectTile host map + ClientOnly avatar-hide for the UI puzzle

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
EOF
```

---

### Task 2: AssetCatalog — number→icon RUID + SFX (resourcing)

Maps board values 1..9 to Maple-themed icon sprite RUIDs and supplies the click/clear/fail SFX RUIDs the controllers will play. Centralizing this keeps the board renderer free of magic RUID strings and makes the visual pass a single edit point.

**Files:**
- Create: `RootDesk/MyDesk/AppleGame/Data/AssetCatalog.mlua`

**Interfaces:**
- Consumes: nothing.
- Produces: `_AssetCatalog` `@Logic` with:
  - `method string IconRUID(integer value)` — RUID for a cell value 1..9 (returns the placeholder RUID for out-of-range).
  - `method string SfxClear()` / `method string SfxFail()` / `method string SfxTick()` — SFX RUIDs (may be `""` if none found yet).
  - `property string PlaceholderIcon` = `"1705e3c5b2c146ac9a699f96fb067408"` (engine default; never leave a renderer RUID empty).

- [ ] **Step 1: Find Maple-themed assets via msw-search**

Use the `msw-search` skill (NOT `asset_search_resources` directly) to find 9 small square number/fruit icons (or one icon + tinted variants) and 2–3 short SFX (a soft "pop/clear", a "fail/buzz", optional "tick"). Record each RUID. If a fitting icon set cannot be found, use `msw-painter` to draw a 9-glyph apple-number sheet and upload to get RUIDs. Note any RUID left as `""` so a later visual pass fills it.

- [ ] **Step 2: Write AssetCatalog with the found RUIDs**

Create `RootDesk/MyDesk/AppleGame/Data/AssetCatalog.mlua` (substitute the real RUIDs found in Step 1; keep the placeholder for any not-yet-found icon so the board is never invisible):

```lua
@Logic
script AssetCatalog extends Logic

    property string PlaceholderIcon = "1705e3c5b2c146ac9a699f96fb067408"

    -- Icon RUID per cell value 1..9. Replace each "" with the RUID found via msw-search/msw-painter.
    property string Icon1 = ""
    property string Icon2 = ""
    property string Icon3 = ""
    property string Icon4 = ""
    property string Icon5 = ""
    property string Icon6 = ""
    property string Icon7 = ""
    property string Icon8 = ""
    property string Icon9 = ""

    property string SfxClearRUID = ""
    property string SfxFailRUID = ""
    property string SfxTickRUID = ""

    method string IconRUID(integer value)
        -- Returns the icon RUID for a cell value (1..9), falling back to the placeholder so a cell is never invisible.
        local r = ""
        if value == 1 then r = self.Icon1
        elseif value == 2 then r = self.Icon2
        elseif value == 3 then r = self.Icon3
        elseif value == 4 then r = self.Icon4
        elseif value == 5 then r = self.Icon5
        elseif value == 6 then r = self.Icon6
        elseif value == 7 then r = self.Icon7
        elseif value == 8 then r = self.Icon8
        elseif value == 9 then r = self.Icon9
        end
        if r == nil or r == "" then return self.PlaceholderIcon end
        return r
    end

    method string SfxClear()
        -- SFX RUID for a successful sum-10 clear (may be "" if none assigned yet).
        return self.SfxClearRUID
    end

    method string SfxFail()
        -- SFX RUID for an invalid selection release (may be "").
        return self.SfxFailRUID
    end

    method string SfxTick()
        -- SFX RUID for the final-seconds timer tick (may be "").
        return self.SfxTickRUID
    end

end
```

- [ ] **Step 3: Refresh + build-clean**

`maker_stop` → `maker_refresh_workspace` → `maker_logs(kind="build")` (expect 0 errors). Confirm `mlua-diagnose` reports zero error-severity diagnostics.

- [ ] **Step 4: Smoke-verify (Loop B)**

In play mode run `maker_execute_script("log('[ASSET] icon5=' .. _AssetCatalog:IconRUID(5)) log('[ASSET] icon99=' .. _AssetCatalog:IconRUID(99))", context="client")`. Read `maker_logs(kind="normal")`: `icon5` should be the found RUID (or placeholder), `icon99` must be the placeholder RUID — proves the fallback. `maker_stop`.

- [ ] **Step 5: Commit**

```bash
git add RootDesk/MyDesk/AppleGame/Data/AssetCatalog.mlua
git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'   # must print nothing
git commit -F- <<'EOF'
feat(applegame): AssetCatalog mapping cell values 1..9 to Maple icons + SFX RUIDs

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
EOF
```

---

### Task 3: HUD UI + HudController (score / 120s timer bar / mode)

Builds the always-on heads-up display: a score readout (right-aligned number), a horizontal timer bar that drains over 120s, and the current mode label. The controller exposes a clean API the session calls each frame; no game logic lives here.

**Files:**
- Create (via UIBuilder): `ui/AppleGame/AppleHud.ui`
- Create: `RootDesk/MyDesk/AppleGame/Client/HudController.mlua`

**Interfaces:**
- Consumes: nothing.
- Produces: `_HudController` `@Logic` (ClientOnly, attached as the HUD group's `script` entity) with:
  - `method void SetMode(string mode)` — sets the mode label text (`"ranked"`→"랭크", `"single"`→"싱글").
  - `method void SetScore(integer score)` — updates the score number.
  - `method void SetTimeRatio(number ratio)` — sets the timer bar `FillAmount` (1.0 full → 0.0 empty); green>0.5, yellow>0.2, red below.
  - `method void SetTimeText(integer secondsLeft)` — sets the seconds-left text.

- [ ] **Step 1: Read builder-protocol §3 + build the HUD .ui**

Read `msw-general/references/builder-protocol.md` §3 (this turn). Then a Node script with the `msw-ui-system` script (CWD anywhere; require by relative path):

```javascript
const { UIBuilder } = require("<ABS path to>/msw-ui-system/scripts/msw_ui_builder.cjs");

const b = new UIBuilder("AppleHud", 5, true); // groupName, displayOrder=5 (HUD under popups), defaultShow=true

// Top HUD bar panel (140px tall, full width, anchored to top)
b.panel("Bar", { anchor: "top-center", pos: [0, -70], rect_size: [1920, 140] });
b.patchComponent("Bar", "MOD.Core.SpriteGUIRendererComponent", { Color: { r: 0.10, g: 0.12, b: 0.18, a: 0.85 } });
// Bar needs a sprite to be tinted — add one if panel() did not:
b.addComponent("Bar", "MOD.Core.SpriteGUIRendererComponent", { "@type": "MOD.Core.SpriteGUIRendererComponent", "Enable": true });
b.patchComponent("Bar", "MOD.Core.SpriteGUIRendererComponent", { Color: { r: 0.10, g: 0.12, b: 0.18, a: 0.85 }, RaycastTarget: false });

// Mode label (left)
b.text("Bar/ModeText", "랭크", { anchor: "middle-left", pos: [40, 0], rect_size: [360, 80], size: 40, alignment: 3, color: "#FFFFFF" });

// Score label + value (right)
b.text("Bar/ScoreCaption", "점수", { anchor: "middle-right", pos: [-360, 26], rect_size: [180, 50], size: 30, alignment: 5, color: "#AEB6C2" });
b.text("Bar/ScoreValue", "0", { anchor: "middle-right", pos: [-40, -8], rect_size: [320, 70], size: 56, alignment: 5, color: "#FFD54A", bold: true });

// Timer text (center top)
b.text("Bar/TimeText", "120", { anchor: "top-center", pos: [0, -20], rect_size: [200, 50], size: 34, alignment: 4, color: "#FFFFFF" });

// Timer bar background + fill (center, below the time text)
b.sprite("Bar/TimerBg", { anchor: "top-center", pos: [0, -84], rect_size: [900, 28], color: "#2A2F3A", raycast: false });
b.sprite("Bar/TimerBg/Fill", { anchor: "stretch", pos: [0, 0], color: "#4CD964", sprite_type: 3, fill_method: 0, raycast: false });
b.patchComponent("Bar/TimerBg/Fill", "MOD.Core.SpriteGUIRendererComponent", { Type: 3, FillMethod: 0, FillOrigin: 0, FillAmount: 1.0 });

// Controller script entity (full-screen, invisible)
b.script("HudCtl", "script.HudController", { anchor: "stretch", pos: [0, 0], rect_size: [1920, 1080] });

b.write("ui/AppleGame/AppleHud.ui");
console.log("HUD ids:", JSON.stringify({
  modeText: b.getId("Bar/ModeText"),
  scoreValue: b.getId("Bar/ScoreValue"),
  timeText: b.getId("Bar/TimeText"),
  fill: b.getId("Bar/TimerBg/Fill"),
}));
```

> `sprite()` defaults `raycast=false` (decoration) — good, the HUD must never eat board touches. `FillAmount`/`FillOrigin` are not builder options, so they are set via `patchComponent` per builder-protocol §3.8. `Fill` uses `sprite_type:3` (Filled) + `fill_method:0` (Horizontal). `b.write()` auto-runs `ui_lint`; fix any error it raises before continuing.

- [ ] **Step 2: Write HudController**

Create `RootDesk/MyDesk/AppleGame/Client/HudController.mlua`:

```lua
@Logic
@ExecSpace("ClientOnly")
script HudController extends Logic

    property TextComponent modeText = "uuid-mode"
    property TextComponent scoreValue = "uuid-score"
    property TextComponent timeText = "uuid-time"
    property SpriteGUIRendererComponent timerFill = "uuid-fill"

    method void OnBeginPlay()
        -- Initializes the HUD to a clean zero/full state at world start.
        self:SetScore(0)
        self:SetTimeRatio(1.0)
        self:SetTimeText(120)
        log("[HudController] ready")
    end

    method void SetMode(string mode)
        -- Sets the visible mode label from the RPC mode string.
        if mode == "ranked" then
            self.modeText.Text = "랭크"
        elseif mode == "single" then
            self.modeText.Text = "싱글"
        else
            self.modeText.Text = mode
        end
    end

    method void SetScore(integer score)
        -- Updates the score number readout.
        self.scoreValue.Text = tostring(score)
    end

    method void SetTimeRatio(number ratio)
        -- Drains the timer bar (1.0 full -> 0.0 empty) and recolors green/yellow/red by urgency.
        local r = ratio
        if r < 0 then r = 0 end
        if r > 1 then r = 1 end
        self.timerFill.FillAmount = r
        if r > 0.5 then
            self.timerFill.Color = Color(0.30, 0.85, 0.39, 1)
        elseif r > 0.2 then
            self.timerFill.Color = Color(1.0, 0.84, 0.29, 1)
        else
            self.timerFill.Color = Color(0.93, 0.27, 0.27, 1)
        end
    end

    method void SetTimeText(integer secondsLeft)
        -- Sets the seconds-remaining text.
        self.timeText.Text = tostring(secondsLeft)
    end

end
```

- [ ] **Step 3: Inject the UI UUID bindings into HudController**

Re-open the HUD `.ui` with `UIBuilder.load(...)` and inject bindings in one `write({bind})` call (builder-protocol §3.6):

```javascript
const { UIBuilder } = require("<ABS path>/msw-ui-system/scripts/msw_ui_builder.cjs");
UIBuilder.load("ui/AppleGame/AppleHud.ui").write("ui/AppleGame/AppleHud.ui", {
  bind: {
    mlua: "RootDesk/MyDesk/AppleGame/Client/HudController.mlua",
    props: {
      modeText: "Bar/ModeText",
      scoreValue: "Bar/ScoreValue",
      timeText: "Bar/TimeText",
      timerFill: "Bar/TimerBg/Fill",
    },
  },
});
```

This rewrites each `property <Type> <name> = "..."` default in `HudController.mlua` with the entity UUID. (`.codeblock` is regenerated by refresh — do not touch it.)

- [ ] **Step 4: Refresh + verify render (Loop A)**

`maker_stop` → `maker_refresh_workspace` → `maker_logs(kind="build")` (0 errors). `maker_play`; poll `maker_get_current_map` until `"play"`. `maker_execute_script("_HudController:SetMode('ranked') _HudController:SetScore(42) _HudController:SetTimeRatio(0.3) _HudController:SetTimeText(36)", context="client")`. `maker_logs(kind="normal")` — expect `[HudController] ready`. `maker_screenshot` — confirm the bar shows mode "랭크", score 42, time 36, and a yellow timer bar ~30% filled. `maker_stop`.

- [ ] **Step 5: Commit**

```bash
git add ui/AppleGame/AppleHud.ui RootDesk/MyDesk/AppleGame/Client/HudController.mlua
git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'   # must print nothing
git commit -F- <<'EOF'
feat(applegame): HUD UI (score / 120s timer bar / mode) + HudController

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
EOF
```

---

### Task 4: Board grid UI + BoardController (render only, no input yet)

Builds the 170-cell board panel, the touch panel, and the selection overlay as UI entities, plus a `BoardController` that renders a grid into the cells and recolors a cell when cleared. Input wiring is deliberately Task 5 so this task can be verified purely by rendering a known board.

**Files:**
- Create (via UIBuilder): `ui/AppleGame/AppleBoard.ui`
- Create: `RootDesk/MyDesk/AppleGame/Client/BoardController.mlua`

**Interfaces:**
- Consumes: `_AssetCatalog:IconRUID(value)`.
- Produces: `_BoardController` `@Logic` (ClientOnly, attached as the board group's `script` entity) with:
  - `property integer COLS = 17`, `property integer ROWS = 10`, `property number CellW = 92`, `property number CellH = 92`.
  - `method void RenderBoard(table grid)` — paints all 170 cells from a 0-indexed `grid[row][col]` (values 1..9, 0 = cleared/hidden).
  - `method void SetCellValue(integer row, integer col, integer value)` — repaint one cell (value 0 hides it).
  - `method UITransformComponent GridTransform()` — returns the grid panel's `UITransformComponent` for screen→cell mapping (used by Task 5).
  - `method void ShowSelection(integer cLo, integer rLo, integer cHi, integer rHi, integer sum)` / `method void HideSelection()` — moves/resizes the overlay rect over the inclusive cell range and shows the running sum; Task 5 calls these.

- [ ] **Step 1: Read builder-protocol §3 + build the board .ui**

Read `msw-general/references/builder-protocol.md` §3 (this turn). Board math: 17×10 grid of 92px cells = 1564×920, centered, dropped below the 140px HUD. Grid panel anchored middle-center, nudged down so the HUD does not overlap: top of board at canvas y ≈ +540−140−margin. Use grid panel `pos:[0,-30]`, `rect_size:[1564,920]`.

```javascript
const { UIBuilder } = require("<ABS path>/msw-ui-system/scripts/msw_ui_builder.cjs");
const COLS = 17, ROWS = 10, CW = 92, CH = 92;
const GW = COLS * CW, GH = ROWS * CH; // 1564 x 920

const b = new UIBuilder("AppleBoard", 3, true); // displayOrder=3 (under HUD=5, under popups=10+)

// Grid panel (the coordinate reference for screen->cell). Children are placed top-left origin.
b.panel("Grid", { anchor: "middle-center", pos: [0, -30], rect_size: [GW, GH] });

// 170 cell sprites, laid out top-left origin. Cell (row,col) anchoredPos (relative to grid center, pivot center):
//   x = -GW/2 + CW/2 + col*CW ; y = +GH/2 - CH/2 - row*CH  (y flipped so row 0 = top)
for (let row = 0; row < ROWS; row++) {
  for (let col = 0; col < COLS; col++) {
    const x = -GW / 2 + CW / 2 + col * CW;
    const y = GH / 2 - CH / 2 - row * CH;
    const name = `Grid/Cell_${row}_${col}`;
    b.sprite(name, { anchor: "middle-center", pos: [x, y], rect_size: [CW - 6, CH - 6], color: "#FFFFFF", raycast: false });
    // value label centered on the cell
    b.text(`${name}/Num`, "", { anchor: "middle-center", pos: [0, 0], rect_size: [CW, CH], size: 38, alignment: 4, color: "#222222", bold: true });
  }
}

// Selection overlay (translucent rect) — sits above cells, below the touch panel; starts hidden via Enable=false
b.sprite("Grid/SelOverlay", { anchor: "middle-center", pos: [0, 0], rect_size: [CW, CH], color: "#3BA3FF", alpha: 0.35, raycast: false });
b.patch("Grid/SelOverlay", { enable: false });
// running-sum label that follows the overlay
b.text("Grid/SelSum", "", { anchor: "middle-center", pos: [0, 0], rect_size: [120, 60], size: 40, alignment: 4, color: "#FFFFFF", bold: true });
b.patch("Grid/SelSum", { enable: false });

// Full-board touch receiver, ON TOP of everything in this group (added last => highest sibling order)
b.touchReceive("TouchPanel", { anchor: "middle-center", pos: [0, -30], rect_size: [GW, GH] });

// Controller script entity
b.script("BoardCtl", "script.BoardController", { anchor: "stretch", pos: [0, 0], rect_size: [1920, 1080] });

b.write("ui/AppleGame/AppleBoard.ui");
console.log("grid id:", b.getId("Grid"), "touch id:", b.getId("TouchPanel"), "overlay id:", b.getId("Grid/SelOverlay"));
```

> Cells use `sprite()` (decoration, no click) because selection is via the single full-board `touchReceive`, not per-cell buttons — 170 buttons would each be a raycast target and complicate the drag. The selection overlay and sum label start `enable:false`. The touch panel is created **last** so it is the top sibling (events reach it first); confirm live in Task 5 whether it needs `RaycastTarget=true`.

- [ ] **Step 2: Write BoardController (render API only)**

Create `RootDesk/MyDesk/AppleGame/Client/BoardController.mlua`. It holds property arrays of cell entity refs is impractical (170); instead it looks cells up by name from the grid panel entity each render via `GetChildByName`, caching the cell sprite + number text in a table on first render.

```lua
@Logic
@ExecSpace("ClientOnly")
script BoardController extends Logic

    property integer COLS = 17
    property integer ROWS = 10
    property number CellW = 92
    property number CellH = 92

    property Entity gridPanel = "uuid-grid"
    property Entity selOverlay = "uuid-overlay"
    property TextComponent selSum = "uuid-selsum"

    method void OnBeginPlay()
        -- Caches cell sprite/text handles lazily; selection overlay starts hidden.
        self.cellSprite = {}   -- [row][col] = SpriteGUIRendererComponent
        self.cellText = {}     -- [row][col] = TextComponent
        self.cached = false
        if isvalid(self.selOverlay) then self.selOverlay.Enable = false end
        log("[BoardController] ready")
    end

    method void CacheCells()
        -- Resolves all 170 cell entities by name once and stores their renderer/text components.
        if self.cached then return end
        for r = 0, self.ROWS - 1 do
            self.cellSprite[r] = {}
            self.cellText[r] = {}
            for c = 0, self.COLS - 1 do
                local cell = self.gridPanel:GetChildByName("Cell_" .. r .. "_" .. c, false)
                if isvalid(cell) then
                    self.cellSprite[r][c] = cell.SpriteGUIRendererComponent
                    local num = cell:GetChildByName("Num", false)
                    if isvalid(num) then self.cellText[r][c] = num.TextComponent end
                end
            end
        end
        self.cached = true
    end

    method void SetCellValue(integer row, integer col, integer value)
        -- Repaints one cell: value 1..9 shows icon+number, value 0 hides the cell.
        local sp = self.cellSprite[row][col]
        local tx = self.cellText[row][col]
        if not isvalid(sp) then return end
        if value > 0 then
            sp.Enable = true
            sp.ImageRUID = { DataId = nil } -- replaced below by AssetCatalog RUID string set via SetAlpha-free path
            sp.SpriteRUID = nil
            sp:SetAlpha(1)
            -- Set the icon RUID (string form for ImageRUID via the DataRef wrapper is handled by patch in Maker; here we use the sprite tint + number text)
            if isvalid(tx) then tx.Text = tostring(value) end
        else
            sp.Enable = false
            if isvalid(tx) then tx.Text = "" end
        end
    end

    method void RenderBoard(table grid)
        -- Paints all 170 cells from a 0-indexed grid[row][col]; caches handles on first call.
        self:CacheCells()
        for r = 0, self.ROWS - 1 do
            for c = 0, self.COLS - 1 do
                local v = 0
                if grid[r] ~= nil and grid[r][c] ~= nil then v = grid[r][c] end
                self:SetCellValue(r, c, v)
            end
        end
        log("[BoardController] board rendered")
    end

    method UITransformComponent GridTransform()
        -- Returns the grid panel UITransform so the input layer can map screen -> cell.
        return self.gridPanel.UITransformComponent
    end

    method void ShowSelection(integer cLo, integer rLo, integer cHi, integer rHi, integer sum)
        -- Positions/sizes the translucent overlay over the inclusive cell rect and shows the running sum.
        if not isvalid(self.selOverlay) then return end
        local cols = cHi - cLo + 1
        local rows = rHi - rLo + 1
        local w = cols * self.CellW
        local h = rows * self.CellH
        -- center of the selected block in grid-local (pivot-center) coords
        local gw = self.COLS * self.CellW
        local gh = self.ROWS * self.CellH
        local centerCol = (cLo + cHi) / 2
        local centerRow = (rLo + rHi) / 2
        local x = -gw / 2 + self.CellW / 2 + centerCol * self.CellW
        local y = gh / 2 - self.CellH / 2 - centerRow * self.CellH
        local ot = self.selOverlay.UITransformComponent
        ot.RectSize = Vector2(w, h)
        ot.anchoredPosition = Vector2(x, y)
        self.selOverlay.Enable = true
        if isvalid(self.selSum) then
            self.selSum.Text = tostring(sum)
            self.selSum.Entity.UITransformComponent.anchoredPosition = Vector2(x, y)
            self.selSum.Entity.Enable = true
        end
    end

    method void HideSelection()
        -- Hides the selection overlay and its sum label.
        if isvalid(self.selOverlay) then self.selOverlay.Enable = false end
        if isvalid(self.selSum) then self.selSum.Entity.Enable = false end
    end

end
```

> **Icon RUID note:** `SpriteGUIRendererComponent.ImageRUID` is a `DataRef` (`{ DataId = "hex" }`), not a plain string — assigning a string fails. The clean runtime path is to set the icon via the cell's number text + tinted sprite for the first pass, and assign real icon RUIDs in a focused follow-up using `entity.SpriteGUIRendererComponent.ImageRUID = { DataId = ruid }` (verify the exact runtime wrapper against `component-api.md` / a `log()` of an existing UI sprite's `ImageRUID` in Step 4). For Task 4's verification the number text + colored cell is sufficient proof of render; the RUID wrapper is finalized before PASS.

- [ ] **Step 3: Inject bindings**

```javascript
const { UIBuilder } = require("<ABS path>/msw-ui-system/scripts/msw_ui_builder.cjs");
UIBuilder.load("ui/AppleGame/AppleBoard.ui").write("ui/AppleGame/AppleBoard.ui", {
  bind: {
    mlua: "RootDesk/MyDesk/AppleGame/Client/BoardController.mlua",
    props: {
      gridPanel: "Grid",
      selOverlay: "Grid/SelOverlay",
      selSum: "Grid/SelSum",
    },
  },
});
```

- [ ] **Step 4: Refresh + verify render with a known board (Loop A)**

`maker_stop` → `maker_refresh_workspace` → `maker_logs(kind="build")` (0 errors). `maker_play`; poll until `"play"`. Render the Plan-1 golden board and the overlay:
```
maker_execute_script("local g=_PuzzleCore:GenerateBoard(0x12345678) _BoardController:RenderBoard(g) _BoardController:ShowSelection(2,1,4,3,17)", context="client")
```
`maker_logs(kind="normal")` — expect `[BoardController] board rendered`. `maker_screenshot` — confirm a 17×10 grid of numbered cells and a translucent blue rectangle over columns 2–4 / rows 1–3 with "17" centered. Also `log()` an existing UI sprite's `ImageRUID` to lock the runtime wrapper shape. `maker_stop`.

- [ ] **Step 5: Commit**

```bash
git add ui/AppleGame/AppleBoard.ui RootDesk/MyDesk/AppleGame/Client/BoardController.mlua
git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'   # must print nothing
git commit -F- <<'EOF'
feat(applegame): board grid UI (170 cells + touch panel + selection overlay) + BoardController render

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
EOF
```

---

### Task 5: Drag input — touch→cell mapping + selection (GATE D live-confirm)

Wires the full-board `UITouchReceiveComponent` to a drag-rectangle selection: down→anchor, drag→current corner + live overlay + running sum, up→commit. Includes a ClientOnly test harness for the pure screen→cell math, then a live drive of the actual drag via `maker_mouse_input`. This is where the GATE D live items are confirmed.

**Files:**
- Modify: `RootDesk/MyDesk/AppleGame/Client/BoardController.mlua` (add input wiring + `ScreenToCell`)
- Create: `RootDesk/MyDesk/AppleGame/Client/AppleClientTest.mlua` (harness for the math)

**Interfaces:**
- Consumes: `_UILogic:ScreenToLocalUIPosition(Vector2, UITransformComponent)`; `UITouchDownEvent` / `UITouchDragEvent` / `UITouchUpEvent`; `_PuzzleCore:ApplyMove(grid, rect)`; `_AssetCatalog:SfxClear/SfxFail`.
- Produces: `_BoardController` gains:
  - `method table ScreenToCell(Vector2 touchPoint)` — returns `{ col, row }` clamped to `[0,16]×[0,9]`, row 0 = topmost (y flipped).
  - `method void BindInput(Entity touchPanel, table grid, any onValidClear)` — connects the 3 touch events; `onValidClear(rect, clearedCount)` fires after a successful `ApplyMove` (the session uses it to score + append to `moves`).
  - `method void UnbindInput()` — disconnects all touch handlers.

- [ ] **Step 1: Add the screen→cell math test (Loop B, expect FAIL)**

Create `RootDesk/MyDesk/AppleGame/Client/AppleClientTest.mlua` with the Plan-1-style harness + a math test that exercises `ScreenToCell` indirectly through a pure helper. Because `ScreenToLocalUIPosition` needs a live UITransform, the unit-testable part is the **local→cell** arithmetic; factor it into `LocalToCell(localX, yFromTop)` and test that:

```lua
@Logic
@ExecSpace("ClientOnly")
script AppleClientTest extends Logic
    property integer passed = 0
    property integer failed = 0

    method void Expect(string name, boolean cond)
        -- Records one assertion to the log for maker_logs inspection.
        if cond then
            self.passed += 1
            log("[TEST PASS] " .. name)
        else
            self.failed += 1
            log_error("[TEST FAIL] " .. name)
        end
    end

    method void Reset()
        -- Clears counters before a run.
        self.passed = 0
        self.failed = 0
    end

    method void Summary()
        -- Emits the final tally line.
        log("[TEST SUMMARY] passed=" .. self.passed .. " failed=" .. self.failed)
    end

    method void TestLocalToCell()
        -- Verifies local-pixel -> (col,row) mapping, clamping, and top-left origin.
        local a = _BoardController:LocalToCell(0, 0)        -- top-left corner
        self:Expect("origin_is_0_0", a.col == 0 and a.row == 0)
        local b = _BoardController:LocalToCell(92 * 16 + 5, 92 * 9 + 5) -- last cell
        self:Expect("last_cell_16_9", b.col == 16 and b.row == 9)
        local mid = _BoardController:LocalToCell(92 * 2 + 10, 92 * 3 + 10)
        self:Expect("mid_cell_2_3", mid.col == 2 and mid.row == 3)
        local clampHi = _BoardController:LocalToCell(99999, 99999)
        self:Expect("clamp_high", clampHi.col == 16 and clampHi.row == 9)
        local clampLo = _BoardController:LocalToCell(-50, -50)
        self:Expect("clamp_low", clampLo.col == 0 and clampLo.row == 0)
    end
end
```

Run Loop B with `_AppleClientTest:Reset() _AppleClientTest:TestLocalToCell() _AppleClientTest:Summary()` → expect FAIL (referencing `LocalToCell`, not yet defined).

- [ ] **Step 2: Implement `LocalToCell` + `ScreenToCell` in BoardController**

Add to `BoardController.mlua`. `ScreenToLocalUIPosition` returns local coords relative to the grid panel pivot (center). Convert to top-left origin: `localX_fromLeft = local.x + gw/2`; `yFromTop = gh/2 - local.y` (UI y up → flip so row 0 = top).

```lua
    method table LocalToCell(number localXFromLeft, number yFromTop)
        -- Pure mapping: top-left-origin local pixels -> clamped {col,row}, row 0 = topmost.
        local col = math.floor(localXFromLeft / self.CellW)
        local row = math.floor(yFromTop / self.CellH)
        if col < 0 then col = 0 end
        if col > self.COLS - 1 then col = self.COLS - 1 end
        if row < 0 then row = 0 end
        if row > self.ROWS - 1 then row = self.ROWS - 1 end
        return { col = col, row = row }
    end

    method table ScreenToCell(Vector2 touchPoint)
        -- Converts a screen-space touch point to a clamped board {col,row}.
        local gt = self:GridTransform()
        local lp = _UILogic:ScreenToLocalUIPosition(touchPoint, gt)
        local gw = self.COLS * self.CellW
        local gh = self.ROWS * self.CellH
        local localXFromLeft = lp.x + gw / 2
        local yFromTop = gh / 2 - lp.y
        return self:LocalToCell(localXFromLeft, yFromTop)
    end
```

> If `math.floor` is unavailable in this mlua build (LSP error), substitute `(localXFromLeft - localXFromLeft % self.CellW) / self.CellW`. Verify in Step 4.

Re-run Loop B for Task 5 Step 1's test → expect all 5 `TestLocalToCell` PASS, `failed=0`.

- [ ] **Step 3: Implement the drag wiring**

Add the input methods + a running-sum helper to `BoardController.mlua`. The grid is held by the controller so it can compute the live sum without going back to the session each move:

```lua
    method integer SumRect(integer cLo, integer rLo, integer cHi, integer rHi)
        -- Sums remaining (>0) cell values in the inclusive rect from the live board snapshot.
        local s = 0
        if self.liveGrid == nil then return 0 end
        for r = rLo, rHi do
            for c = cLo, cHi do
                local v = self.liveGrid[r][c]
                if v ~= nil and v > 0 then s += v end
            end
        end
        return s
    end

    method void BindInput(Entity touchPanel, table grid, any onValidClear)
        -- Connects down/drag/up on the touch panel; drives the overlay and commits valid clears.
        self.liveGrid = grid
        self.onValidClear = onValidClear
        self.touchPanel = touchPanel
        self.dragging = false
        self.downCell = nil

        self.downHandler = touchPanel:ConnectEvent(UITouchDownEvent, function(ev)
            if ev.TouchId ~= 1 and ev.TouchId ~= -1 then return end
            self.dragging = true
            self.downCell = self:ScreenToCell(ev.TouchPoint)
            local d = self.downCell
            self:ShowSelection(d.col, d.row, d.col, d.row, self:SumRect(d.col, d.row, d.col, d.row))
        end)

        self.dragHandler = touchPanel:ConnectEvent(UITouchDragEvent, function(ev)
            if not self.dragging or self.downCell == nil then return end
            if ev.TouchId ~= 1 and ev.TouchId ~= -1 then return end
            local cur = self:ScreenToCell(ev.TouchPoint)
            local cLo = math.min(self.downCell.col, cur.col)
            local cHi = math.max(self.downCell.col, cur.col)
            local rLo = math.min(self.downCell.row, cur.row)
            local rHi = math.max(self.downCell.row, cur.row)
            self:ShowSelection(cLo, rLo, cHi, rHi, self:SumRect(cLo, rLo, cHi, rHi))
        end)

        self.upHandler = touchPanel:ConnectEvent(UITouchUpEvent, function(ev)
            if not self.dragging or self.downCell == nil then return end
            self.dragging = false
            local cur = self:ScreenToCell(ev.TouchPoint)
            local cLo = math.min(self.downCell.col, cur.col)
            local cHi = math.max(self.downCell.col, cur.col)
            local rLo = math.min(self.downCell.row, cur.row)
            local rHi = math.max(self.downCell.row, cur.row)
            self:HideSelection()
            local rect = { c1 = cLo, r1 = rLo, c2 = cHi, r2 = rHi }
            local res = _PuzzleCore:ApplyMove(self.liveGrid, rect)
            if res.valid then
                for r = rLo, rHi do
                    for c = cLo, cHi do
                        self:SetCellValue(r, c, 0)
                    end
                end
                _SoundService:PlaySound(_AssetCatalog:SfxClear(), 1, 1)
                if self.onValidClear ~= nil then self.onValidClear(rect, res.clearedCount) end
                log("[BoardController] valid clear count=" .. res.clearedCount)
            else
                _SoundService:PlaySound(_AssetCatalog:SfxFail(), 1, 1)
                log("[BoardController] invalid release")
            end
            self.downCell = nil
        end)
    end

    method void UnbindInput()
        -- Disconnects all touch handlers (called by the session at run end / map leave).
        if self.touchPanel == nil then return end
        if self.downHandler then self.touchPanel:DisconnectEvent(UITouchDownEvent, self.downHandler) end
        if self.dragHandler then self.touchPanel:DisconnectEvent(UITouchDragEvent, self.dragHandler) end
        if self.upHandler then self.touchPanel:DisconnectEvent(UITouchUpEvent, self.upHandler) end
        self.downHandler = nil
        self.dragHandler = nil
        self.upHandler = nil
    end
```

> `_SoundService:PlaySound(ruid, ...)` signature is verified against `Environment/NativeScripts/Service/SoundService.d.mlua` before use (msw-scripting §1.3); if the arity differs, adjust. A `""` RUID is a no-op. Closures capture `self`; the returned `EventHandlerBase` is stored for `UnbindInput`.

- [ ] **Step 4: Live-drive the drag + confirm GATE D items (Loop A)**

`maker_stop` → `maker_refresh_workspace` → `maker_logs(kind="build")` (0 errors). `maker_play`; poll until `"play"`. Seed a board and bind input, then simulate a PC drag:
```
maker_execute_script("local g=_PuzzleCore:GenerateBoard(0x12345678) _BoardController:RenderBoard(g) _BoardController.testGrid=g _BoardController:BindInput(_EntityService:GetEntityByPath('/ui/AppleBoard/TouchPanel'), g, nil)", context="client")
```
Then drive a drag with `maker_mouse_input`: a **press** at a screen coord over a known pair, a **move** to the adjacent cell, a **release**. Compute the screen coords from the board layout (cell center = canvas-relative; convert canvas→screen if needed — verify with one `log()` of `ev.TouchPoint` first). Read `maker_logs(kind="normal")`:
- Confirm a `UITouchDownEvent` log fires (add a temporary `log("[down] " .. tostring(ev.TouchPoint.x))` if needed) → **GATE D item: Down is the anchor**.
- Confirm `UITouchDragEvent` fired during the move → **GATE D item: PC drag threshold** (if no drag log on a slow move, note it; Down-anchor still makes the selection correct).
- Confirm `[BoardController] valid clear` or `invalid release` appears → events reach the panel → **GATE D item: RaycastTarget**. If NO touch logs at all, set `RaycastTarget=true` on the touch panel's auto-attached `SpriteGUIRendererComponent` via `b.patchComponent("TouchPanel", "MOD.Core.SpriteGUIRendererComponent", { RaycastTarget: true })`, re-write the `.ui`, refresh, retry.
- `maker_screenshot` mid-drag (drive press+move, screenshot before release) — confirm the blue overlay tracks the drag and shows the running sum.

Record the GATE D outcomes (threshold behavior, RaycastTarget needed?, TouchId gate works) in the commit body. `maker_stop`.

- [ ] **Step 5: Commit**

```bash
git add RootDesk/MyDesk/AppleGame/Client/BoardController.mlua RootDesk/MyDesk/AppleGame/Client/AppleClientTest.mlua
git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'   # must print nothing
git commit -F- <<'EOF'
feat(applegame): drag-rectangle selection (touch->cell, live overlay, sum-10 commit) + GATE D confirmed

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
EOF
```

---

### Task 6: GameSession — full play loop (seed → render → timer → moves → submit)

The map-scoped `@Component` that orchestrates one play session: on map enter it requests a seed, on `ReceiveSeed` it generates the board, renders it, binds input, starts the 120s timer, and records each valid clear into `moves`; on timeout it submits via `_ScoreService:SubmitRun`. It is the integration spine connecting Plans 1–3 to the UI.

**Files:**
- Create: `RootDesk/MyDesk/AppleGame/Client/GameSession.mlua`
- Modify: `map/AppleGame.map` (attach `script.GameSession` to the map root via MapBuilder)

**Interfaces:**
- Consumes: `_SeedService:RequestSeed(string mode)`; `_SeedService.ReceiveSeed` (server→client RPC, the session reacts to it — wired via the contract, see note); `_PuzzleCore:GenerateBoard`; `_BoardController:RenderBoard/BindInput/UnbindInput`; `_HudController:SetMode/SetScore/SetTimeRatio/SetTimeText`; `_ScoreService:SubmitRun(token, moves, claimedScore, clientElapsed)`.
- Produces: `script.GameSession` `@Component` on the AppleGame map with:
  - `method void OnReceiveSeed(string mode, integer seed, string token, integer dateKey, boolean alreadyPlayed, integer personalBest)` — the client-side reaction the SeedService Client-RPC calls (or that GameSession registers for).
  - `method void StartSession(string mode)` — entry the title/mode-select calls (Task 8 hooks the start button here).
  - `@Sync`-free internal state: `moves` table, `score`, `token`, `mode`, `startElapsed`, `timeLeft`.

- [ ] **Step 1: Verify the ReceiveSeed delivery mechanism**

The contract declares `_SeedService:ReceiveSeed(...)` as `@ExecSpace("Client")` (server→caller). Read `RootDesk/MyDesk/AppleGame/Server/SeedService.mlua` (Plan 2 output) to confirm **how** the result reaches the client: either (a) `ReceiveSeed` lives on `SeedService` and itself notifies GameSession (e.g. calls a known client logic / fires an event), or (b) GameSession must expose the client-RPC method. Pick the path the shipped SeedService actually uses. If SeedService's `ReceiveSeed` does not forward to GameSession, the cleanest bridge is a custom `@Event` (`SeedReceivedEvent`) that SeedService's `ReceiveSeed` fires and GameSession subscribes to. Decide and record this before writing GameSession; do not guess the cross-script call shape.

- [ ] **Step 2: Write GameSession**

Create `RootDesk/MyDesk/AppleGame/Client/GameSession.mlua` (ClientOnly component on the map entity). The timer uses a `delta`-driven countdown in `OnUpdate` — never anchor on `ElapsedSeconds` across sessions (msw-scripting §13).

```lua
@Component
@ExecSpace("ClientOnly")
script GameSession extends Component

    method void OnBeginPlay()
        -- Initializes session state; the actual play begins when StartSession is called.
        self.active = false
        self.moves = {}
        self.score = 0
        self.timeLeft = 0
        self.mode = "single"
        log("[GameSession] ready")
    end

    method void OnMapEnter(Entity m)
        -- (Optional) auto-start single mode on entry; the title screen (Task 8) overrides via StartSession.
        log("[GameSession] map enter")
    end

    method void StartSession(string mode)
        -- Requests a seed for the chosen mode; play continues in OnReceiveSeed.
        self.mode = mode
        self.moves = {}
        self.score = 0
        _HudController:SetMode(mode)
        _HudController:SetScore(0)
        _SeedService:RequestSeed(mode)
        log("[GameSession] requested seed mode=" .. mode)
    end

    method void OnReceiveSeed(string mode, integer seed, string token, integer dateKey, boolean alreadyPlayed, integer personalBest)
        -- Server reply: if ranked already played, show a notice; else generate board, bind input, start the 120s timer.
        if alreadyPlayed then
            _ResultController:ShowAlreadyPlayed(personalBest)
            log("[GameSession] ranked already played today")
            return
        end
        self.token = token
        self.seed = seed
        self.grid = _PuzzleCore:GenerateBoard(seed)
        _BoardController:RenderBoard(self.grid)
        _BoardController:BindInput(
            _EntityService:GetEntityByPath("/ui/AppleBoard/TouchPanel"),
            self.grid,
            function(rect, clearedCount)
                self:OnValidClear(rect, clearedCount)
            end)
        self.startElapsed = _UtilLogic.ElapsedSeconds
        self.timeLeft = 120
        self.active = true
        _HudController:SetTimeRatio(1.0)
        _HudController:SetTimeText(120)
        log("[GameSession] session started seed=" .. seed)
    end

    method void OnValidClear(table rect, integer clearedCount)
        -- Scores a confirmed clear and appends the move (with relative timestamp) to the run log.
        self.score += clearedCount
        local t = _UtilLogic.ElapsedSeconds - self.startElapsed
        self.moves[#self.moves + 1] = { rect = rect, t = t }
        _HudController:SetScore(self.score)
    end

    method void OnUpdate(number delta)
        -- Drains the 120s timer; on expiry, ends the run and submits.
        if not self.active then return end
        self.timeLeft = self.timeLeft - delta
        if self.timeLeft <= 0 then
            self.timeLeft = 0
            self:EndSession()
        end
        _HudController:SetTimeRatio(self.timeLeft / 120)
        _HudController:SetTimeText(math.ceil(self.timeLeft))
    end

    method void EndSession()
        -- Stops play, unbinds input, and submits the run for server validation.
        if not self.active then return end
        self.active = false
        _BoardController:UnbindInput()
        local clientElapsed = _UtilLogic.ElapsedSeconds - self.startElapsed
        _ScoreService:SubmitRun(self.token, self.moves, self.score, clientElapsed)
        log("[GameSession] submitted score=" .. self.score .. " moves=" .. #self.moves)
    end

    method void OnEndPlay()
        -- Cleanup: unbind input handlers if the map unloads mid-session.
        if self.active then _BoardController:UnbindInput() end
    end

end
```

> `OnReceiveSeed` is the method the SeedReceivedEvent handler (Step 1 decision) or SeedService's Client RPC invokes. If using the `@Event` bridge, connect in `OnBeginPlay` (`self.Entity:ConnectEvent(SeedReceivedEvent, self.OnSeedEvent)`, where `OnSeedEvent` unpacks the event fields and calls `OnReceiveSeed`) and disconnect in `OnEndPlay`. `_ResultController:ShowAlreadyPlayed` / `:ShowResult` are defined in Task 7.

- [ ] **Step 3: Attach GameSession to the map + refresh**

```javascript
const { MapBuilder } = require("./scripts/map/msw_map_builder.cjs");
const map = MapBuilder.read("map/AppleGame.map");
const rootName = map.getMapInfo().rootName || "AppleGame";
map.upsertComponent(rootName, "script.GameSession", { "@type": "script.GameSession", "Enable": true })
   .write("map/AppleGame.map");
```

`maker_stop` → `maker_refresh_workspace` → `maker_logs(kind="build")` (0 errors).

- [ ] **Step 4: Verify the loop (Loop A, single mode, shortened timer)**

For this verify round only, temporarily set `self.timeLeft = 8` in `OnReceiveSeed` so the submit fires within a screenshot window (revert to 120 before PASS — verify-checklist Step 2b). `maker_play`; poll until `"play"`. Run `maker_execute_script("_GameSession_via_map:StartSession('single')", context="client")` — since GameSession is a map `@Component`, address it via the map entity: `_EntityService:GetEntityByPath('/maps/AppleGame'):GetComponent('script.GameSession'):StartSession('single')`. Read `maker_logs(kind="normal")` for the ordered chain: `requested seed` → `session started` → (drive a valid clear via `maker_mouse_input`) `valid clear` → after ~8s `submitted score=...`. `maker_screenshot` to confirm HUD score/timer update live. Revert `timeLeft` to 120. `maker_stop`.

- [ ] **Step 5: Commit**

```bash
git add RootDesk/MyDesk/AppleGame/Client/GameSession.mlua map/AppleGame.map
git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'   # must print nothing
git commit -F- <<'EOF'
feat(applegame): GameSession play loop (seed -> render -> 120s timer -> moves -> submit)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
EOF
```

---

### Task 7: Result popup + ResultController

A modal popup shown after a run: accepted score + personal best, or a rejection reason, or the "already played today" ranked notice. Buttons: "다시" (retry → restart session) and "랭킹" (open leaderboard). This consumes the `_ScoreService:ReceiveSubmitResult` reply.

**Files:**
- Create (via UIBuilder): `ui/AppleGame/AppleResult.ui`
- Create: `RootDesk/MyDesk/AppleGame/Client/ResultController.mlua`

**Interfaces:**
- Consumes: `_ScoreService.ReceiveSubmitResult(boolean accepted, integer finalScore, integer personalBest, string reason)` (server→client; reaches the client per the same mechanism as Task 6 Step 1); `_GameSession:StartSession`; `_LeaderboardController:Open` (Task 8).
- Produces: `_ResultController` `@Logic` (ClientOnly) with:
  - `method void ShowResult(boolean accepted, integer finalScore, integer personalBest, string reason)` — populates and shows the popup.
  - `method void ShowAlreadyPlayed(integer personalBest)` — ranked-locked variant.
  - `method void Hide()` — closes the popup.

- [ ] **Step 1: Build the Result popup .ui (modal recipe)**

Read builder-protocol §3. Modal popup recipe: root group `displayOrder` high (10), a raycast-blocking dimmer, a centered panel (component-api Modal popup pattern). Group `default_show:true` (so the controller's `OnBeginPlay` runs); the controller hides the inner content with `Enable=false` and shows on demand.

```javascript
const { UIBuilder } = require("<ABS path>/msw-ui-system/scripts/msw_ui_builder.cjs");
const b = new UIBuilder("AppleResult", 10, true); // popup layer, above HUD(5)/board(3)

// Dimmer (raycast=true to block input behind)
b.sprite("Dimmer", { anchor: "stretch", pos: [0, 0], color: "#000000", alpha: 0.55, raycast: true });
// Panel
b.sprite("Panel", { anchor: "middle-center", pos: [0, 0], rect_size: [720, 520], color: "#1B2030", raycast: true });
b.text("Panel/Title", "결과", { anchor: "top-center", pos: [0, -50], rect_size: [600, 80], size: 56, alignment: 4, color: "#FFFFFF", bold: true });
b.text("Panel/ScoreLine", "점수 0", { anchor: "middle-center", pos: [0, 40], rect_size: [600, 80], size: 48, alignment: 4, color: "#FFD54A" });
b.text("Panel/BestLine", "최고 0", { anchor: "middle-center", pos: [0, -30], rect_size: [600, 60], size: 36, alignment: 4, color: "#AEB6C2" });
b.text("Panel/ReasonLine", "", { anchor: "middle-center", pos: [0, -90], rect_size: [620, 50], size: 28, alignment: 4, color: "#FF6B6B" });

// Buttons
b.button("Panel/BtnRetry", "다시", { anchor: "bottom-center", pos: [-180, 60], rect_size: [300, 100], font_size: 36, color: "#FFFFFF" });
b.patchComponent("Panel/BtnRetry", "MOD.Core.SpriteGUIRendererComponent", { Color: { r: 0.16, g: 0.55, b: 0.30, a: 1 } });
b.button("Panel/BtnRank", "랭킹", { anchor: "bottom-center", pos: [180, 60], rect_size: [300, 100], font_size: 36, color: "#FFFFFF" });
b.patchComponent("Panel/BtnRank", "MOD.Core.SpriteGUIRendererComponent", { Color: { r: 0.20, g: 0.35, b: 0.70, a: 1 } });

// Controller
b.script("ResultCtl", "script.ResultController", { anchor: "stretch", pos: [0, 0], rect_size: [1920, 1080] });

b.write("ui/AppleGame/AppleResult.ui");
console.log("result ids:", b.getId("Panel"), b.getId("Panel/BtnRetry"), b.getId("Panel/BtnRank"));
```

- [ ] **Step 2: Write ResultController**

Create `RootDesk/MyDesk/AppleGame/Client/ResultController.mlua`. The popup root group is hidden by toggling a content `Entity.Enable`; wire the buttons in `OnBeginPlay`, disconnect in `OnEndPlay`.

```lua
@Logic
@ExecSpace("ClientOnly")
script ResultController extends Logic

    property Entity panel = "uuid-panel"
    property Entity dimmer = "uuid-dimmer"
    property TextComponent title = "uuid-title"
    property TextComponent scoreLine = "uuid-score"
    property TextComponent bestLine = "uuid-best"
    property TextComponent reasonLine = "uuid-reason"
    property ButtonComponent btnRetry = "uuid-retry"
    property ButtonComponent btnRank = "uuid-rank"

    method void OnBeginPlay()
        -- Hides the popup at start and wires the retry/leaderboard buttons.
        self:Hide()
        self.retryHandler = self.btnRetry.Entity:ConnectEvent(ButtonClickEvent, self.OnRetry)
        self.rankHandler = self.btnRank.Entity:ConnectEvent(ButtonClickEvent, self.OnRank)
        log("[ResultController] ready")
    end

    method void ReasonText(string reason)
        -- Maps a rejection reason code to Korean copy ("" for accepted runs).
        if reason == "ok" then return ""
        elseif reason == "bad_token" then return "세션이 만료되었어요"
        elseif reason == "timing" then return "시간 검증 실패"
        elseif reason == "replay_mismatch" then return "점수 검증 실패"
        elseif reason == "already" then return "오늘 랭크는 이미 플레이했어요"
        else return reason end
    end

    method void ShowResult(boolean accepted, integer finalScore, integer personalBest, string reason)
        -- Populates and shows the result popup for a completed run.
        self.title.Text = accepted and "결과" or "기록 거부"
        self.scoreLine.Text = "점수 " .. tostring(finalScore)
        self.bestLine.Text = "최고 " .. tostring(personalBest)
        self.reasonLine.Text = self:ReasonText(reason)
        self.panel.Enable = true
        self.dimmer.Enable = true
        log("[ResultController] show accepted=" .. tostring(accepted) .. " score=" .. finalScore)
    end

    method void ShowAlreadyPlayed(integer personalBest)
        -- Ranked-locked variant shown when today's attempt is already consumed.
        self.title.Text = "오늘은 끝!"
        self.scoreLine.Text = "내일 다시 도전"
        self.bestLine.Text = "최고 " .. tostring(personalBest)
        self.reasonLine.Text = "랭크는 하루 한 판"
        self.panel.Enable = true
        self.dimmer.Enable = true
    end

    method void Hide()
        -- Closes the popup.
        self.panel.Enable = false
        self.dimmer.Enable = false
    end

    method void OnRetry()
        -- Closes the popup and restarts a session in the same mode.
        self:Hide()
        local session = _EntityService:GetEntityByPath("/maps/AppleGame"):GetComponent("script.GameSession")
        ---@type GameSession
        if isvalid(session) then session:StartSession(session.mode) end
    end

    method void OnRank()
        -- Closes the popup and opens the leaderboard.
        self:Hide()
        _LeaderboardController:Open(1) -- default to the Day board
    end

    method void OnEndPlay()
        -- Disconnects button handlers.
        if self.retryHandler then self.btnRetry.Entity:DisconnectEvent(ButtonClickEvent, self.retryHandler) end
        if self.rankHandler then self.btnRank.Entity:DisconnectEvent(ButtonClickEvent, self.rankHandler) end
    end

end
```

> `session.mode` is read off the GameSession component — confirm the `---@type GameSession` cast resolves the property (msw-scripting §11). Wire `_ScoreService:ReceiveSubmitResult` → `_ResultController:ShowResult` per the Task 6 Step 1 delivery decision (likely a `SubmitResultEvent` `@Event` the ScoreService Client RPC fires).

- [ ] **Step 3: Inject bindings**

```javascript
const { UIBuilder } = require("<ABS path>/msw-ui-system/scripts/msw_ui_builder.cjs");
UIBuilder.load("ui/AppleGame/AppleResult.ui").write("ui/AppleGame/AppleResult.ui", {
  bind: {
    mlua: "RootDesk/MyDesk/AppleGame/Client/ResultController.mlua",
    props: {
      panel: "Panel", dimmer: "Dimmer",
      title: "Panel/Title", scoreLine: "Panel/ScoreLine",
      bestLine: "Panel/BestLine", reasonLine: "Panel/ReasonLine",
      btnRetry: "Panel/BtnRetry", btnRank: "Panel/BtnRank",
    },
  },
});
```

- [ ] **Step 4: Verify (Loop A)**

`maker_stop` → `maker_refresh_workspace` → `maker_logs(kind="build")` (0 errors). `maker_play`; poll until `"play"`. `maker_execute_script("_ResultController:ShowResult(true, 24, 31, 'ok')", context="client")` → `maker_screenshot` confirms popup with "점수 24 / 최고 31", no reason text. Then `maker_execute_script("_ResultController:ShowResult(false, 0, 31, 'replay_mismatch')", context="client")` → screenshot shows "기록 거부 / 점수 검증 실패". Drive the "다시" button with `maker_mouse_input` at the button center → `maker_logs` shows the popup hidden + a `StartSession` chain. `maker_stop`.

- [ ] **Step 5: Commit**

```bash
git add ui/AppleGame/AppleResult.ui RootDesk/MyDesk/AppleGame/Client/ResultController.mlua
git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'   # must print nothing
git commit -F- <<'EOF'
feat(applegame): Result popup + ResultController (accepted/rejected/already-played)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
EOF
```

---

### Task 8: Leaderboard popup (5 tabs) + LeaderboardController

A modal leaderboard with five tabs — 일간(Day) / 주간(Week) / 올타임(AllTime) / 싱글(Single) / 멀티(Multi). Day/Week/AllTime each render up to 20 entries from `_LeaderboardService` + the player's own rank/score line + a fixed "갱신 주기 약 30분" stale label. **싱글** shows the player's single-mode personal best (one line, fetched via `_ScoreService:RequestPersonalBest`), no list, no stale label. **멀티** is a locked placeholder for Phase 2 — a lock icon + "준비 중" text, no data call. Also adds the title/start affordance so the whole flow (start → play → result → ranking) is reachable.

**Files:**
- Create (via UIBuilder): `ui/AppleGame/AppleLeaderboard.ui`
- Create: `RootDesk/MyDesk/AppleGame/Client/LeaderboardController.mlua`

**Interfaces:**
- Consumes: `_LeaderboardService:RequestLeaderboard(integer boardId, integer pageNo)`; `_LeaderboardService.ReceiveLeaderboard(integer boardId, table entries, integer myRank, integer myScore, boolean stale)` (entries = `{rank, name, score}` array); `_ScoreService:RequestPersonalBest()` (for the Single tab); `_ScoreService.ReceivePersonalBest(integer personalBest)` (server→client, same delivery mechanism as Task 6/7); `_GameSession:StartSession`; board ids `BOARD_DAY=1`, `BOARD_WEEK=2`, `BOARD_ALLTIME=3`; client-only UI sentinels `BOARD_SINGLE=4`, `BOARD_MULTI=5` (never passed to `RequestLeaderboard`); `PAGE_SIZE=20`.
- Produces: `_LeaderboardController` `@Logic` (ClientOnly) with:
  - `method void Open(integer boardId)` — shows the popup and requests that board.
  - `method void Hide()`.
  - `method void OnReceiveLeaderboard(integer boardId, table entries, integer myRank, integer myScore, boolean stale)` — renders the rows + my-rank line.
  - `method void SelectTab(integer boardId)` — handles boardId 1–5; 1/2/3 = ranking list, 4 = Single personal-best, 5 = locked Multi placeholder.
  - `method void OnReceivePersonalBest(integer personalBest)` — renders the Single tab's one-line personal-best display.

- [ ] **Step 1: Build the leaderboard .ui (5 tabs + 20-row list + single-line PB panel + locked panel)**

Read builder-protocol §3. Use a `scrollLayout` (≤20 rows, vertical) per component-api guidance (GridView is for 100+). Five tab buttons across the top, a content list, a my-rank footer, and the stale label. The Single tab reuses its own content panel (SinglePB); the Multi tab shows a locked panel (MultiLocked); both are `enable:false` by default.

```javascript
const { UIBuilder } = require("<ABS path>/msw-ui-system/scripts/msw_ui_builder.cjs");
const b = new UIBuilder("AppleLeaderboard", 11, true); // above Result(10)

b.sprite("Dimmer", { anchor: "stretch", pos: [0, 0], color: "#000000", alpha: 0.55, raycast: true });
b.sprite("Panel", { anchor: "middle-center", pos: [0, 0], rect_size: [900, 980], color: "#161B28", raycast: true });
b.text("Panel/Title", "랭킹", { anchor: "top-center", pos: [0, -44], rect_size: [600, 70], size: 48, alignment: 4, color: "#FFFFFF", bold: true });

// Tabs (5, centered across the top)
b.button("Panel/TabDay",    "일간",  { anchor: "top-center", pos: [-440, -130], rect_size: [200, 78], font_size: 26, color: "#FFFFFF" });
b.button("Panel/TabWeek",   "주간",  { anchor: "top-center", pos: [-220, -130], rect_size: [200, 78], font_size: 26, color: "#FFFFFF" });
b.button("Panel/TabAll",    "올타임", { anchor: "top-center", pos: [0,    -130], rect_size: [200, 78], font_size: 26, color: "#FFFFFF" });
b.button("Panel/TabSingle", "싱글",  { anchor: "top-center", pos: [220,  -130], rect_size: [200, 78], font_size: 26, color: "#FFFFFF" });
b.button("Panel/TabMulti",  "멀티",  { anchor: "top-center", pos: [440,  -130], rect_size: [200, 78], font_size: 26, color: "#FFFFFF" });

// Scrollable row list + a single hidden row template cloned at runtime
b.scrollLayout("Panel/List", { layout_type: 1, spacing: 6, cell_size: [820, 70], use_scroll: true, anchor: "middle-center", pos: [0, -10], rect_size: [840, 560] });
b.panel("Panel/List/RowTemplate", { rect_size: [820, 70] });
b.text("Panel/List/RowTemplate/Rank", "1", { anchor: "middle-left", pos: [20, 0], rect_size: [120, 60], size: 32, alignment: 3, color: "#FFD54A" });
b.text("Panel/List/RowTemplate/Name", "이름", { anchor: "middle-left", pos: [160, 0], rect_size: [440, 60], size: 30, alignment: 3, color: "#FFFFFF" });
b.text("Panel/List/RowTemplate/Score", "0", { anchor: "middle-right", pos: [-20, 0], rect_size: [200, 60], size: 32, alignment: 5, color: "#AEB6C2" });

// My-rank footer + stale label + close button
b.text("Panel/MyRank", "내 순위 -", { anchor: "bottom-center", pos: [0, 180], rect_size: [820, 60], size: 32, alignment: 4, color: "#7FE0A0" });
b.text("Panel/Stale", "갱신 주기 약 30분", { anchor: "bottom-center", pos: [0, 120], rect_size: [820, 44], size: 24, alignment: 4, color: "#8893A4" });
b.button("Panel/BtnClose", "닫기", { anchor: "bottom-center", pos: [0, 40], rect_size: [300, 90], font_size: 34, color: "#FFFFFF" });
b.patchComponent("Panel/BtnClose", "MOD.Core.SpriteGUIRendererComponent", { Color: { r: 0.30, g: 0.34, b: 0.42, a: 1 } });

// Single-tab personal-best panel (hidden by default; shown only when Tab 4 is active)
b.text("Panel/SinglePB", "내 최고 점수: -", { anchor: "center", pos: [0, 0], font_size: 40, color: "#FFE680" });
b.setEnable("Panel/SinglePB", false);

// Multi-tab locked placeholder (hidden by default; shown only when Tab 5 is active)
b.text("Panel/MultiLocked", "🔒 준비 중 (Phase 2)", { anchor: "center", pos: [0, 0], font_size: 40, color: "#AAAAAA" });
b.setEnable("Panel/MultiLocked", false);

b.script("LbCtl", "script.LeaderboardController", { anchor: "stretch", pos: [0, 0], rect_size: [1920, 1080] });

b.write("ui/AppleGame/AppleLeaderboard.ui");
console.log("lb ids:", b.getId("Panel/List"), b.getId("Panel/List/RowTemplate"), b.getId("Panel/MyRank"), b.getId("Panel/SinglePB"), b.getId("Panel/MultiLocked"));
```

- [ ] **Step 2: Write LeaderboardController**

Create `RootDesk/MyDesk/AppleGame/Client/LeaderboardController.mlua`. Rows are cloned from the hidden `RowTemplate` (runtime-patterns §4 scroll-list pattern) and cleared between renders.

```lua
@Logic
@ExecSpace("ClientOnly")
script LeaderboardController extends Logic

    property integer BOARD_DAY = 1
    property integer BOARD_WEEK = 2
    property integer BOARD_ALLTIME = 3
    -- BOARD_SINGLE=4 and BOARD_MULTI=5 are client-only UI sentinels; never passed to RequestLeaderboard.
    property integer BOARD_SINGLE = 4
    property integer BOARD_MULTI = 5
    property integer PAGE_SIZE = 20

    property Entity panel = "uuid-panel"
    property Entity dimmer = "uuid-dimmer"
    property Entity listRoot = "uuid-list"
    property Entity rowTemplate = "uuid-row"
    property TextComponent myRank = "uuid-myrank"
    property TextComponent singlePB = "uuid-singlepb"
    property Entity multiLocked = "uuid-multilocked"
    property ButtonComponent tabDay = "uuid-tabday"
    property ButtonComponent tabWeek = "uuid-tabweek"
    property ButtonComponent tabAll = "uuid-taball"
    property ButtonComponent tabSingle = "uuid-tabsingle"
    property ButtonComponent tabMulti = "uuid-tabmulti"
    property ButtonComponent btnClose = "uuid-close"

    method void OnBeginPlay()
        -- Hides the popup, hides the row template, and wires all 5 tab + close buttons.
        self.rows = {}
        self.currentTab = self.BOARD_DAY
        if isvalid(self.rowTemplate) then self.rowTemplate:SetEnable(false) end
        self:Hide()
        self.dayHandler    = self.tabDay.Entity:ConnectEvent(ButtonClickEvent, function() self:SelectTab(self.BOARD_DAY) end)
        self.weekHandler   = self.tabWeek.Entity:ConnectEvent(ButtonClickEvent, function() self:SelectTab(self.BOARD_WEEK) end)
        self.allHandler    = self.tabAll.Entity:ConnectEvent(ButtonClickEvent, function() self:SelectTab(self.BOARD_ALLTIME) end)
        self.singleHandler = self.tabSingle.Entity:ConnectEvent(ButtonClickEvent, function() self:SelectTab(self.BOARD_SINGLE) end)
        self.multiHandler  = self.tabMulti.Entity:ConnectEvent(ButtonClickEvent, function() self:SelectTab(self.BOARD_MULTI) end)
        self.closeHandler  = self.btnClose.Entity:ConnectEvent(ButtonClickEvent, self.Hide)
        log("[LeaderboardController] ready")
    end

    method void Open(integer boardId)
        -- Shows the popup and requests the given board's first page.
        self.panel.Enable = true
        self.dimmer.Enable = true
        self:SelectTab(boardId)
    end

    method void ShowRankingPanel(boolean show)
        -- Toggles the ranking list + footer elements.
        if isvalid(self.listRoot) then self.listRoot:SetEnable(show) end
        if isvalid(self.myRank) then self.myRank.Entity:SetEnable(show) end
    end

    method void ShowSinglePanel(boolean show)
        -- Toggles the Single-tab personal-best label.
        if isvalid(self.singlePB) then self.singlePB.Entity:SetEnable(show) end
    end

    method void ShowMultiPanel(boolean show)
        -- Toggles the Multi-tab locked placeholder.
        if isvalid(self.multiLocked) then self.multiLocked:SetEnable(show) end
    end

    method void SelectTab(integer boardId)
        -- 1/2/3 = ranking boards (list); 4 = single personal-best; 5 = locked Multi placeholder.
        self.currentTab = boardId
        if boardId >= 1 and boardId <= 3 then
            self:ShowRankingPanel(true)
            self:ShowSinglePanel(false)
            self:ShowMultiPanel(false)
            _LeaderboardService:RequestLeaderboard(boardId, 1)
        elseif boardId == 4 then
            self:ShowRankingPanel(false)
            self:ShowSinglePanel(true)
            self:ShowMultiPanel(false)
            _ScoreService:RequestPersonalBest()
        else -- 5: Multi, locked
            self:ShowRankingPanel(false)
            self:ShowSinglePanel(false)
            self:ShowMultiPanel(true)
        end
        log("[LeaderboardController] SelectTab board=" .. boardId)
    end

    method void OnReceivePersonalBest(integer personalBest)
        -- Fills the Single tab's one-line personal-best display.
        -- (read the SinglePB text entity by name and set its Text; mirror how OnReceiveLeaderboard
        --  resolves its row entities in this same controller)
        if isvalid(self.singlePB) then
            self.singlePB.Text = "내 최고 점수: " .. tostring(personalBest)
        end
        log("[LB] OnReceivePersonalBest=" .. personalBest)
    end

    method void ClearRows()
        -- Destroys previously cloned rows.
        for i = 1, #self.rows do
            if isvalid(self.rows[i]) then self.rows[i]:Destroy() end
        end
        self.rows = {}
    end

    method void OnReceiveLeaderboard(integer boardId, table entries, integer myRank, integer myScore, boolean stale)
        -- Renders the entries for the active board; ignores replies for a stale tab switch.
        if boardId ~= self.currentTab then return end
        self:ClearRows()
        for i = 1, #entries do
            local e = entries[i]
            local row = self.rowTemplate:Clone("Row_" .. i)
            row:SetEnable(true)
            row:GetChildByName("Rank", false).TextComponent.Text = tostring(e.rank)
            row:GetChildByName("Name", false).TextComponent.Text = e.name
            row:GetChildByName("Score", false).TextComponent.Text = tostring(e.score)
            self.rows[#self.rows + 1] = row
        end
        if myRank > 0 then
            self.myRank.Text = "내 순위 " .. myRank .. "위 (" .. myScore .. "점)"
        else
            self.myRank.Text = "내 순위 - (기록 없음)"
        end
        log("[LeaderboardController] rendered board=" .. boardId .. " rows=" .. #entries .. " stale=" .. tostring(stale))
    end

    method void Hide()
        -- Closes the popup.
        self.panel.Enable = false
        self.dimmer.Enable = false
    end

    method void OnEndPlay()
        -- Disconnects all button handlers and clears cloned rows.
        if self.dayHandler    then self.tabDay.Entity:DisconnectEvent(ButtonClickEvent, self.dayHandler) end
        if self.weekHandler   then self.tabWeek.Entity:DisconnectEvent(ButtonClickEvent, self.weekHandler) end
        if self.allHandler    then self.tabAll.Entity:DisconnectEvent(ButtonClickEvent, self.allHandler) end
        if self.singleHandler then self.tabSingle.Entity:DisconnectEvent(ButtonClickEvent, self.singleHandler) end
        if self.multiHandler  then self.tabMulti.Entity:DisconnectEvent(ButtonClickEvent, self.multiHandler) end
        if self.closeHandler  then self.btnClose.Entity:DisconnectEvent(ButtonClickEvent, self.closeHandler) end
        self:ClearRows()
    end

end
```

> `Clone` / `SetEnable` / `Destroy` are the runtime-patterns §4 scroll-list idioms. Wire `_LeaderboardService:ReceiveLeaderboard` → `_LeaderboardController:OnReceiveLeaderboard` per the Task 6 Step 1 delivery mechanism (likely a `LeaderboardReceivedEvent` `@Event`). The stale label is static text already in the `.ui`; `stale` is logged for confirmation.

- [ ] **Step 3: Inject bindings**

```javascript
const { UIBuilder } = require("<ABS path>/msw-ui-system/scripts/msw_ui_builder.cjs");
UIBuilder.load("ui/AppleGame/AppleLeaderboard.ui").write("ui/AppleGame/AppleLeaderboard.ui", {
  bind: {
    mlua: "RootDesk/MyDesk/AppleGame/Client/LeaderboardController.mlua",
    props: {
      panel: "Panel", dimmer: "Dimmer",
      listRoot: "Panel/List", rowTemplate: "Panel/List/RowTemplate",
      myRank: "Panel/MyRank",
      singlePB: "Panel/SinglePB", multiLocked: "Panel/MultiLocked",
      tabDay: "Panel/TabDay", tabWeek: "Panel/TabWeek", tabAll: "Panel/TabAll",
      tabSingle: "Panel/TabSingle", tabMulti: "Panel/TabMulti",
      btnClose: "Panel/BtnClose",
    },
  },
});
```

- [ ] **Step 4: Verify (Loop A)**

`maker_stop` → `maker_refresh_workspace` → `maker_logs(kind="build")` (0 errors). `maker_play`; poll until `"play"`. Inject a mock render directly to confirm row cloning without depending on a live server snapshot:
```
maker_execute_script("_LeaderboardController:Open(1) _LeaderboardController:OnReceiveLeaderboard(1, { {rank=1,name='Alice',score=88}, {rank=2,name='Bob',score=72}, {rank=3,name='Me',score=60} }, 3, 60, true)", context="client")
```
`maker_logs(kind="normal")` — expect `rendered board=1 rows=3 stale=true`. `maker_screenshot` — confirm 3 ranked rows, "내 순위 3위 (60점)", and the "갱신 주기 약 30분" label. Drive the "주간" tab via `maker_mouse_input` → `maker_logs` shows `SelectTab board=2`. Drive the "싱글" tab → logs show `SelectTab board=4` and the SinglePB panel visible. Drive the "멀티" tab → logs show `SelectTab board=5` and the MultiLocked panel visible (lock + "준비 중"). Inject a mock personal-best: `maker_execute_script("_LeaderboardController:OnReceivePersonalBest(42)", context="client")` → screenshot confirms "내 최고 점수: 42". `maker_stop`.

- [ ] **Step 5: Commit**

```bash
git add ui/AppleGame/AppleLeaderboard.ui RootDesk/MyDesk/AppleGame/Client/LeaderboardController.mlua
git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'   # must print nothing
git commit -F- <<'EOF'
feat(applegame): Leaderboard popup (5-tab: Day/Week/AllTime/Single PB/locked Multi) + controller

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
EOF
```

---

### Task 9: End-to-end integration + Verify

Stitches the full flow (start → seed → play → submit → result → leaderboard), wires the server→client delivery events finalized in Task 6 Step 1, applies the click/hover SFX pass on the buttons, and runs the formal Verify checklist. No new feature surface — this task is the integration gate.

**Files:**
- Modify (if the Step-1 decision needs them): `RootDesk/MyDesk/AppleGame/Client/GameSession.mlua`, `ResultController.mlua`, `LeaderboardController.mlua`, and any `@Event` bridge scripts under `RootDesk/MyDesk/AppleGame/Client/`.
- Possibly Modify (server-side event fire): the Plan 2/3 services — **only** to add an event fire on their existing Client RPCs if that is the chosen delivery bridge; coordinate as it crosses plan boundaries.

**Interfaces:**
- Consumes: every interface from Tasks 1–8 plus the shipped `_SeedService` / `_ScoreService` / `_LeaderboardService`.
- Produces: a verified, playable end-to-end client. No new public API.

- [ ] **Step 1: Finalize the server→client delivery bridges**

Implement the delivery mechanism chosen in Task 6 Step 1 consistently for all three replies: `ReceiveSeed` → `GameSession:OnReceiveSeed`, `ReceiveSubmitResult` → `ResultController:ShowResult`, `ReceiveLeaderboard` → `LeaderboardController:OnReceiveLeaderboard`. If using `@Event` bridges, create them under `RootDesk/MyDesk/AppleGame/Client/` (e.g. `SeedReceivedEvent.mlua`, `SubmitResultEvent.mlua`, `LeaderboardReceivedEvent.mlua`), have each service's Client RPC construct + `SendEvent` it, and subscribe in the matching controller's `OnBeginPlay` (disconnect in `OnEndPlay`). Refresh + build-clean.

- [ ] **Step 2: SFX pass on buttons (ui-sound)**

For every interactive button (`BtnRetry`, `BtnRank`, tabs, `BtnClose`), wire a click SFX via `_SoundService:PlaySound(_AssetCatalog:SfxClear() or a UI click RUID, ...)` in the existing click handlers, or per `msw-ui-system/references/ui-sound.md` default UI SFX RUIDs. Verify the signature in `Environment/NativeScripts/Service/SoundService.d.mlua` first. A `""` RUID is a safe no-op.

- [ ] **Step 3: Full end-to-end drive (Loop A)**

`maker_stop` → `maker_refresh_workspace` → `maker_logs(kind="build")` (0 errors across ALL AppleGame scripts). `maker_play`; poll `maker_get_current_map` until `"play"` and confirm the current map is `AppleGame`. Then, with `timeLeft` temporarily shortened to ~10s for the window:
1. Start: `maker_execute_script("_EntityService:GetEntityByPath('/maps/AppleGame'):GetComponent('script.GameSession'):StartSession('single')", context="client")`.
2. `maker_logs(kind="normal")` — `requested seed` → `session started`. `maker_screenshot` — board + HUD render.
3. Drive ≥1 valid clear with `maker_mouse_input` (press/move/release over a real sum-10 pair from the rendered board) — `maker_logs` shows `valid clear`, HUD score increments (screenshot).
4. Wait out the timer — `maker_logs` shows `submitted score=...`, then `ResultController show` and the Result popup (screenshot).
5. Drive "랭킹" — leaderboard opens, rows render (screenshot), tabs switch (`request board=2/3`).
Revert `timeLeft` to 120. `maker_stop`.

- [ ] **Step 4: Run the formal Verify checklist**

Load `msw-scripting` (if not loaded this turn) and Read `references/verify-checklist.md` in full, then execute it against every AppleGame `.mlua`:
- Step 1 Runtime: `stop` → `clear_logs` → `refresh` → `logs(build)` 0 errors → `play` → `logs(runtime)`.
- Step 2 Code Review: confirm all UI Logic/Components are `@ExecSpace("ClientOnly")`; `ConnectEvent` only on Entity (never Component) with handlers stored in properties and disconnected in `OnEndPlay`; `GameSession` is a map `@Component` (not Logic) so `OnMapEnter`/`OnUpdate` fire; no DataStorage in `OnUpdate`; timer is delta-driven (not `ElapsedSeconds`-anchored); `.ui`→`.mlua` bindings all injected (no `"uuid-..."` placeholders remain); files in correct paths; `Global/`/`Environment/`/`.codeblock` untouched.
- Step 3 Log Evidence: positive `log()` lines for each branch (`session started`, `valid clear`, `submitted`, `show`, `rendered`). Step 2b: any element ≤2s lifetime (selection overlay flashes) logged at create+destroy.
- Step 4 Verdict: PASS only with concrete log evidence + screenshots.

- [ ] **Step 5: Commit**

```bash
git add RootDesk/MyDesk/AppleGame/Client/ ui/AppleGame/ map/AppleGame.map
git diff --cached | grep -iE 'df285f|bearer [a-z0-9]'   # must print nothing
git commit -F- <<'EOF'
feat(applegame): end-to-end client integration (delivery bridges + SFX) and verify pass

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_0169R85bG8nYYQ3CuapDpDB5
EOF
```

---

## Plan 4 Self-Review

**1. Spec coverage:**
- §4 (UI game + thin host, DefaultPlayer hidden, full UI flow) → Task 1 (RectTile host + avatar hide), Tasks 3/4/7/8 (HUD/board/result/leaderboard).
- §10 client side (on map enter → RequestSeed; ReceiveSeed → GenerateBoard → render → 120s timer → record moves; ApplyMove local feedback; on timeout SubmitRun; ReceiveSubmitResult → result) → Task 6 (GameSession) + Task 5 (ApplyMove on release) + Task 7 (result).
- §11 mobile UI/input (1920×1080 landscape, 92px cells, board 1564×920 centered, safe area, touch→cell formula, UI touch receiver press/move/release, selection overlay + live sum) → Task 4 (layout/overlay) + Task 5 (touch→cell, drag) — both cite the exact formula and GATE D.
- §14 (file structure `AppleGame/Client`, `Data`, `ui/AppleGame/`, `map/AppleGame.map`; no catch-all) → enforced in Global Constraints + every task's Files block.
- Interface contract Plan 4 (RequestSeed/ReceiveSeed, SubmitRun/ReceiveSubmitResult, RequestLeaderboard/ReceiveLeaderboard, RequestPersonalBest/ReceivePersonalBest, board ids 1/2/3, UI sentinels BOARD_SINGLE=4/BOARD_MULTI=5, PAGE_SIZE 20, stale label) → Tasks 6/7/8 Interfaces blocks copy the signatures verbatim.
- GATE D (UITouch event names + payloads, ScreenToLocalUIPosition, TouchId -1/1 gate, live items: PC drag threshold / RaycastTarget / Down-anchor) → Task 5 implements + live-confirms all four.
- Leaderboard 5 tabs Day/Week/AllTime + Single personal-best (via `_ScoreService:RequestPersonalBest`) + locked Multi placeholder + "갱신 주기 약 30분" → Task 8.
- Resourcing (Maple sprites/SFX via msw-search) → Task 2 + Task 9 Step 2 SFX pass.
- Correctly out of this plan: server authority/replay (Plan 2), ranking package install/fan-out (Plan 3), PuzzleCore/SeededRng (Plan 1) — all consumed, not built.
- **Spec §3 5-tab requirement: fully met.** Day/Week/AllTime render ranking data from `_LeaderboardService`; Single shows the player's personal best via the new `_ScoreService:RequestPersonalBest` RPC; Multi is a locked static Phase-2 placeholder. `BOARD_SINGLE=4` and `BOARD_MULTI=5` are client-only UI sentinels never passed to `RequestLeaderboard`.

**2. Placeholder scan:** No "TBD"/"add error handling"/"similar to Task N" placeholders. Intentional, named fill-ins each carry explicit instructions and a verification step: (a) the `"<ABS path to>"` UIBuilder require path — the worker resolves the sibling `msw-ui-system/scripts/msw_ui_builder.cjs` absolute path once (builder-protocol §0/§3 routing); (b) AssetCatalog RUIDs filled from msw-search (Task 2, with placeholder fallback so nothing is invisible); (c) the golden-board-style icon RUID DataRef wrapper confirmed against a live UI sprite (Task 4 Step 4); (d) the server→client delivery bridge shape, deliberately deferred to Task 6 Step 1 because it depends on the shipped SeedService/ScoreService code (msw-scripting §1.3 no-guess); (e) `_SoundService:PlaySound` arity confirmed against its `.d.mlua` before use. The `"uuid-..."` property defaults are real placeholders by design — overwritten by the `write({bind})` injection step in each UI task.

**3. Type consistency:** `RenderBoard(grid)` / `ApplyMove(grid,rect)` / `Replay(seed,moves)` use the Plan 1 shapes (`grid[r][c]` 0-indexed, `rect={c1,r1,c2,r2}`, `moves` array of `{rect,t}`). `RequestSeed(mode:string)` / `OnReceiveSeed(mode,seed,token,dateKey,alreadyPlayed,personalBest)` / `SubmitRun(token,moves,claimedScore,clientElapsed)` / `ReceiveSubmitResult(accepted,finalScore,personalBest,reason)` / `RequestLeaderboard(boardId,pageNo)` / `ReceiveLeaderboard(boardId,entries,myRank,myScore,stale)` match the interface contract exactly (param names and order). Board ids `BOARD_DAY=1/BOARD_WEEK=2/BOARD_ALLTIME=3` and UI sentinels `BOARD_SINGLE=4/BOARD_MULTI=5` and `PAGE_SIZE=20` match the contract. `entries` element shape `{rank,name,score}` is consistent between the contract, LeaderboardController render, and the `.ui` row template. `OnReceivePersonalBest(integer personalBest)` matches the `_ScoreService:ReceivePersonalBest` delivery surface. `_HudController` methods (`SetMode/SetScore/SetTimeRatio/SetTimeText`) and `_BoardController` methods (`RenderBoard/SetCellValue/GridTransform/ShowSelection/HideSelection/LocalToCell/ScreenToCell/BindInput/UnbindInput`) are named identically at definition (Tasks 3/4/5) and call sites (Task 6). `mode` is a string everywhere; `@ExecSpace("ClientOnly")` on every UI script.

## Open Questions / Risks (resolve during execution)

- **Map type (recommend RectTile=1):** confirm `map/AppleGame.map` is/becomes TileMapMode 1 (Task 1 Step 1). RectTile has no gravity so the hidden avatar won't fall; mode switching is a USER ACTION IN MAKER.
- **DefaultPlayer hide:** the ClientOnly renderer toggle (Task 1) is the no-Global path; if insufficient, fall back to off-camera spawn or a copied non-Global player model (USER ACTION IN MAKER) — flagged in Task 1 Step 5.
- **GATE D live items:** PC `UITouchDragEvent` threshold (use Down as anchor regardless), whether the touch panel needs `RaycastTarget=true`, TouchId gate — all driven and recorded in Task 5 Step 4.
- **Server→client delivery bridge:** the exact way `ReceiveSeed/ReceiveSubmitResult/ReceiveLeaderboard` reach the client controllers depends on the shipped Plan 2/3 service code; finalized in Task 6 Step 1 / Task 9 Step 1 (likely small `@Event` bridges), no guessing.
- **Icon ImageRUID wrapper:** `SpriteGUIRendererComponent.ImageRUID` is a `DataRef` (`{DataId=...}`), not a plain string — runtime wrapper confirmed against a live UI sprite in Task 4 before icons go in (number-text fallback keeps Task 4 verifiable meanwhile).
- **`maker_execute_script` form/context:** confirm the multi-statement + `context="client"` form works early (as Plan 1 did); if not, wrap in `RunX()` harness methods.
- **Screen orientation (landscape vs portrait):** spec §16 fixes landscape 1920×1080 as the default but leaves portrait-fit (cell shrinks to ~63px) as an open mockup-stage decision. This plan hard-codes landscape (92px cells). If the user later wants portrait, Task 4's cell math, board rect, and HUD layout all change — confirm orientation before Task 3/4 build out the `.ui`.
- **Leaderboard tab count:** resolved — Task 8 builds the full 5-tab popup. Day/Week/AllTime render ranking data; Single shows the personal best via `_ScoreService:RequestPersonalBest`; Multi is a locked static Phase-2 placeholder. No open question remains on tab count.

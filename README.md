# Duc's coding guide!

Open **Scripts** using the `</>` icon in the floating top minibar. Import a `.lua`/`.luau` file or type source, then select **Run script** (Ctrl+Enter). Importing, selecting an example, and restoring a saved draft do not execute it. Output and errors appear beneath the editor.

The community editor has file tabs, line numbers, syntax colors, a searchable local library, filename editing, imports/exports and timestamped output. **Execute** runs the active file. Closing a file tab keeps the file in the library. Up to 32 files are saved locally; the old single draft migrates automatically. This library stays on your computer and is not a hosted community marketplace. Share exported `.luau` files with other Duc users.

Scripts run in a fresh Luau VM inside Duc. They use Duc's existing native controls and validated process readers. They do not run in Roblox's VM. Roblox `game`, `workspace`, services, remotes, executor APIs, and Serotonin APIs are not supplied. External operation does not guarantee avoiding detection.

## Controls

### Custom desktop tabs (display only)

```lua
duc.ui.tab("hi")
duc.ui.toggle("hi", "hi", false)
```

`duc.ui.tab(name)` declares a main-menu tab and returns its name. `duc.ui.toggle(tabName, label, initial)` declares a toggle in that tab; the initial boolean defaults to false. Create the tab first. Toggle clicks only update their visual state: they do not run callbacks or control game features.

Tabs publish after a successful execution batch (completion or a cooperative yield). Re-running a tab name replaces its controls instead of creating duplicates. Up to 8 script tabs and 16 toggles per tab are supported; names/labels must contain 1–48 UTF-8 bytes without control characters. Script tabs use separate routes and cannot override native tabs. They remain through game/profile changes, but clear on sign-out, restart, or **Clear script tabs** in the editor. These APIs target the Tauri desktop.

Use the editor's **hi test tab** example or import `scripts/hi.luau`.

### Working feature bindings

```lua
local tab = duc.ui.tab("Community")
duc.ui.bind(tab, "FOV circle", "cbFovC")
duc.ui.bind(tab, "FOV size", "sAFOV")
duc.ui.bind(tab, "Aim part", "btnBone")
```

`duc.ui.bind(tabName, label, controlId)` adds the actual native toggle, slider or dropdown for an allowed control ID. Creating a binding does not enable a feature. Its value stays synchronized with the existing feature control, and user interactions use the existing native action and validation. Up to 16 bindings per tab are supported. Unavailable controls are disabled/hidden by the existing game-profile checks; arbitrary actions such as sign-out are not bindable. This is distinct from `duc.ui.toggle`, which remains a display-only test switch.

Use `duc.controls()` to discover stable IDs and ranges; dropdown entries include `kind = "dropdown"` and a `choices` array. Dropdown values are zero-based indexes even though Luau arrays are one-based. No persistent guest callbacks are required for bindings.

```lua
for _, control in ipairs(duc.controls()) do
    print(control.id, control.type, control.value)
end

duc.set("sAFOV", 45)
duc.set("cbFovC", true)
print(duc.get("sAFOV"))
```

- `duc.version`: API version string, currently `"3"` (the original API remains supported).
- `duc.controls()`: visible supported controls, with `id`, `label`, `page`, `type`, `value`, and numeric `min`/`max`. Some native sliders have no label; their stable ID remains available.
- `duc.get(id)`: current boolean or integer value.
- `duc.set(id, value)`: strictly typed setting through the existing native control. Numeric values must be finite integers in range. Boolean values must actually be booleans. Returns the resulting value.

The API supports feature toggles, sliders and the aiming/name-style dropdowns. It does not expose login/session management, file dialogs, key capture, arbitrary button invocation, or capture-privacy settings. Place-specific visibility rules still apply. Game features require a supported, attached client where their existing native implementation requires one.

## Object reads

```lua
print("Place", duc.place_id())
local world = duc.workspace()
for _, object in ipairs(world:children(32)) do
    print(object:name(), object:class_name())
end

local character = duc.local_character()
if character then
    local root = character:find_first_child("HumanoidRootPart")
    if root then
        local p = root:position()
        print(p.x, p.y, p.z)
    end
end
```

After attaching:

- `duc.root()` returns the current DataModel object.
- `duc.workspace()` returns the current Workspace object.
- `duc.local_character()` returns the local character or `nil` if unavailable.
- `duc.place_id()` returns a decimal **string**, preserving the full 64-bit value.

Objects are opaque userdata, not raw addresses. Methods validate process version, current DataModel and ancestry before reading. A stale or unsupported object raises an error.

| Method | Result |
|---|---|
| `object:name()` | Instance name |
| `object:class_name()` | Class name |
| `object:children(limit)` | Up to `limit` direct children; default 64, allowed 1–128 |
| `object:find_first_child(name)` | Matching direct child or nil; examines at most 128 entries |
| `object:position()` | Coordinates of a physical part or Player/Model HumanoidRootPart; uppercase and lowercase fields |
| `object:health()` | Current finite, validated Humanoid health; Player/Model wrappers resolve their direct Humanoid |
| `object:parent()` | Validated parent handle, or nil |

Reads are live and can fail if the game changes objects between calls. Child enumeration is bounded and is not a complete world scan.

`duc.players()` returns up to 64 validated Player entries: `{name, is_local, player, character}`. `player` is a Player object handle; `character` is a Model handle or nil. It does not imply that a character is alive or an enemy. Authors must choose their own filters using supported object reads. `object:address()` returns its currently validated address as a hexadecimal string.

## Custom world labels

```lua
duc.draw.clear()
local character = duc.local_character()
if character then
    local head = character:find_first_child("Head")
    if head then duc.draw.label(head, "Local player", "#87CEEB") end
end
```

`duc.draw.label(part, text, color)` declares a label anchored to an existing physical part. Color is optional and defaults to sky blue; supplied colors must use `#RRGGBB`. Text follows the 1–48-byte UI label rule. Up to 32 labels are supported. After a successful execution batch, native rendering updates their positions without running background Luau. Labels render while the attached game is focused, using the existing external overlay and capture setting. They can show through walls; there is no implicit occlusion/team/health filter.

World labels and screen shapes have separate shared sets. A successful drawing batch replaces the corresponding set; no drawing calls leave the previous set alone. `duc.draw.clear()` clears both sets. A failed batch does not publish its pending drawing changes; previously published frames remain until cleared or stopped. **Clear draw**, `duc.draw.clear()`, Stop all / End, sign-out, or reattachment clear it. Reparented/replaced parts and changed DataModels are suppressed. Re-run after respawns or joins to select new objects. This API creates overlay labels, not Roblox Instances or injected BillboardGuis.

Examples: `scripts/player_labels.luau`, `scripts/community_menu.luau`, and `scripts/game_template.luau`. The game template gates its menu by a string PlaceId. Authors can combine filtering, object queries, supported feature bindings and custom labels for specific experiences.

The native mouse-input Silent Aim mode is configurable through `duc.set` / `duc.get`; see [SILENT_INPUT.md](SILENT_INPUT.md) for controls and game-dependent limitations. Network manipulation, executor hooks and Roblox script injection are not provided. The API below adds cooperative tasks and explicit host operations. Importing a script cannot supply missing game-engine APIs.

## Local movement

`duc.teleport(x, y, z)` uses the existing guarded local teleport operation. Coordinates must be finite and inside the engine's bounds. This stops conflicting local movement modes, requires attachment and a living character, and can be undone by server correction. It does not alter server authorization or hit validation.

## Community API v3

These are Duc contracts, not drop-in compatibility with another executor. API tables remain read-only; `CheatAPI.AimPosition` and `CheatAPI.SilentAimTarget` are validated virtual properties. Existing `duc` APIs remain supported, and `duc.version` remains `"3"` for compatibility.

### Additional compatibility bindings (October 2026)

All requested `duc`, `duc.ui`, and `duc.draw` methods remain available with their existing contracts. `GetPlayers()` still returns player records; use `entry.player` for a `Duc.Object`.

| API | Contract |
|---|---|
| `CheatAPI.camera.ViewportSize` | Live Vector2, measured in game client pixels. |
| `CheatAPI.camera.FieldOfView` | Live vertical field of view in degrees. |
| `CheatAPI.camera.WorldToScreen(v)` / `.WorldToViewport(v)` | Same `(Vector2_or_nil, onScreen)` result as the existing projection. Both use client pixels with no GUI inset. Dot and colon calls work. |
| `CheatAPI.GetClosestPlayer()` | Living non-local Player userdata nearest viewport center, or nil. Checks viewport, aim range, distance and native aim team setting. Does not test occlusion. |
| `CheatAPI.SetTarget(object)` | Follow a validated Player, character Model or physical part through the existing aim modes. Does not enable aim. |
| `CheatAPI.AimPosition = v` | Use a world Vector3 in either native aim mode; replaces the object/screen target. Assign nil to clear. |
| `CheatAPI.SilentAimTarget = v` | Supply a world Vector3 to the existing native Silent mode; does not enable it. Assign nil to clear and restore its input patch. |
| `CheatAPI.ClearTarget()` | Clear script object, screen, world and silent targets, including while detached. Native feature toggles remain as configured. |
| `CheatAPI.SetTrigger(bool)` / `.SetSilent(bool)` / `.Noclip(bool)` | Use the corresponding native control and its validation. Returns resulting boolean state. |
| `CheatAPI.SetGravity(n)` | Native gravity control; integer 0–500, attachment required, verifies the readback. |
| `CheatAPI.Teleport(v)` | Vector3 wrapper for the existing guarded local teleport. |
| `CheatAPI.ExpandHitboxes(n)` | Native hitbox feature; integer 2–13 sets size and enables it; 0 disables/restores it. Existing team/body controls apply. |
| `CheatAPI.mouse_move(x,y)` | Relative integer mouse deltas, each ±4096 pixels. |
| `CheatAPI.mouse_click(button?)` | One click: `"left"` (default), `"right"`, or `"middle"`. |
| `CheatAPI.key_press(code)` | Hold a supported Windows virtual key until released. Repeated presses of an owned key do not send another down. |
| `CheatAPI.key_release(code)` | Release only a script-owned key; repeated release is harmless. Works without focus/attachment and bypasses the input throttle. |

World/object targets persist until replaced, cleared, a new run, a script error, Stop, Stop all, sign-out or reattachment. Object positions refresh natively; removed/reparented/replaced parts and dead Player/Model targets are suppressed. Player handles follow their current character after respawn. World coordinates have no identity, team or health information, so scripts must select them appropriately. Existing focus, living-character, aim-hold, range and distance gates still apply. Script world/object aim also uses the native wall check. Silent targets still use the native hold, radius, distance, wall and input-buffer checks. This adds no new shot-redirection mechanism; existing game-dependent Silent limitations still apply.

Input sends synthetic Windows events. Press/move/click calls share the existing 100 ms throttle, focus, attachment and modifier-key checks. Supported keys remain A–Z, 0–9, Space and arrows. Held keys are released on completion, error, Stop, Stop all, focus loss, detachment and shutdown; failed releases remain tracked for retry. Keep a task alive with `task.wait()` if a key must remain held. `Input.KeyPress` keeps its original tap behavior.

`Player:position()` and character `Model:position()` now read their direct HumanoidRootPart. `Player:health()` and character `Model:health()` read their direct Humanoid. Traversal checks at most 128 direct children. Existing physical-part position and Humanoid health methods remain valid; position tables now include both uppercase and lowercase coordinates.

For `Drawing.new("Text")`, numeric `Size` is an alias of `TextSize` (6–72 pixels), `Center` horizontally centers text on Position, and `Outline` draws a black one-pixel outline. Both flags default to false. Square `Size` remains Vector2. All previously supported drawing properties retain their behavior.

### Players, objects and properties

| Function | Contract |
|---|---|
| `CheatAPI.GetPlayers()` | Alias of `duc.players()`: up to 64 **records** `{name, is_local, player, character}`, not raw addresses. |
| `CheatAPI.GetLocalPlayer()` | Validated Player userdata or nil. |
| `CheatAPI.GetPlayerPosition(username)` | Exact instance/username match; live character root position or nil. Does not use display names. |
| `CheatAPI.FindPart(name)` | Matching physical part or nil, plus a boolean indicating search completion. Up to 128 visited nodes, 64 children/node, depth 12, queue 512. A nil result with false means truncated search. |
| `CheatAPI.GetChildren(instance, limit?)` | Same validated direct-child enumeration as `instance:children(limit)`. |
| `CheatAPI.SetProperty(instance, property, value)` | Explicit allowlist below; returns true or raises an error. |

`SetProperty` supports **WalkSpeed** on the current local Humanoid (integer 16–500, through the existing speed implementation) and **Size** on a validated physical part (`Vector3`, each component >0 and <=32). Other property names fail explicitly. WalkSpeed is a native setting and remains until changed or Stop all. Size uses tracked edits with object/primitive identity checks. Server validation can override local changes; this does not grant server permissions or extend server-validated hit range.

### Vectors, color and camera

`Vector2.new(x,y)` and `Vector3.new(x,y,z)` return immutable tables with uppercase and lowercase coordinate fields. Components must be finite and within ±99999. They do **not** implement Roblox vector operator/metamethod behavior. Use component arithmetic and create a new vector to update a position. `Color3.new(r,g,b)` uses 0–1; `Color3.fromRGB(r,g,b)` uses 0–255. Returned color fields are `R/G/B` in 0–1.

`CheatAPI.WorldToScreen(Vector3)` returns `(Vector2_or_nil, isOnScreen)`. Coordinates are **Roblox client-area pixels**, matching Duc's overlay. The boolean checks positive depth and viewport bounds, not occlusion. Behind-camera/unprojectable positions return nil/false; projected positions outside the viewport return a point/false.

`CheatAPI.GetCameraMatrix()` returns `{Position, Rotation, FieldOfView, ViewportSize, ScreenOrigin?}`. `Rotation` contains nine row-major floats from the existing camera pose. `FieldOfView` is vertical degrees. `ScreenOrigin`, when a valid game window is available, converts client coordinates to desktop coordinates by addition. A missing camera/viewport raises an error. Use `scripts/camera_query.luau` for a read-only example.

### Screen drawing nodes

```lua
local ring = Drawing.new("Circle")
ring.Position = Vector2.new(150, 150)
ring.Radius = 35
ring.Color = Color3.fromRGB(135, 206, 235)
ring.Thickness = 2
ring.Visible = true
```

Types: `Square` (`Rectangle` alias), `Line`, `Circle`, `Text`. Up to 128 nodes may be created per run; reuse nodes in frame loops instead of creating a new node every frame. Default visibility is false.

| Property | Value |
|---|---|
| `Visible`, `Filled` | Strict booleans; fill affects circles/rectangles. |
| `Color` | `Color3` table. |
| `Thickness` | Finite 1–12 pixels. |
| `Position` | Vector2; rectangle/text top-left, circle center. |
| `Size` | Vector2; rectangle dimensions, each 0–16384 pixels. |
| `From`, `To` | Vector2 line endpoints. |
| `Radius` | Circle radius, 0–8192 pixels. |
| `Text` | Up to 256 UTF-8 bytes without NUL. |
| `TextSize` | 6–72 pixels; text uses Segoe UI. |
| Text `Size` | Numeric alias of `TextSize`; Square Size remains Vector2. |
| Text `Center`, `Outline` | Strict booleans; horizontal centering and black outline. |

`node:Remove()` / `node:Destroy()` retire a node; repeated removal is harmless. Other access to a removed node fails. Nodes publish after successful batches and draw only while attached with the game focused. **Clear draw** and `duc.draw.clear()` invalidate old node handles. **Stop script** removes both shape and label sets, including completed scripts. A successful script can leave static drawings displayed after it finishes.

`scripts/drawing_tasks.luau` animates a circle for five seconds without modifying game features or sending input.

### Scoped memory access

These APIs use only Duc's existing attached, version-validated process handle. They cannot open another process. Addresses are hexadecimal **strings** (`"0x..."`), never Lua numbers; `object:address()` returns a validated object's address. Do not reuse addresses across game changes.

- `Memory.Read(address, type)`: `byte`, signed `int`/`int32`, finite `float`, `vector`/`Vector3`, or read-only `address` (hex string). Regions must be committed, readable, and not guarded.
- `Memory.Write(address, value, type)`: same scalar/vector types except `address`. Only writable **non-executable data** is accepted. There are no pointer writes, executable-page writes, memory-protection changes, process injection or process-opening APIs. Float writes are finite within ±1e9. Maximum 128 tracked locations, 12 bytes/write. Overlap with tracked native feature patches is rejected.
- `Memory.Scan(signature, start?, byteCount?)`: space-separated hex bytes with `??` wildcards, 1–128 pattern bytes with at least one concrete byte. Default starts at the executable base and scans **at most the first 256 KiB**, not the whole process. Explicit ranges are 1–262144 bytes. Returns `(hexAddressMatches, nextAddress, rangeComplete)`. Up to 64 matches and about 6 ms of scan work per call; repeat using `nextAddress` and the remaining range after `task.wait()` to continue. Unreadable pages are skipped. Matches spanning distinct virtual-memory regions are not guaranteed. A signature match is not proof of the object's type, lifetime or compatibility.

Tracked memory/Size edits are restored on Stop script, a new run, script error, Stop all, reattach and sign-out, where current bytes and ownership checks still match. Later independent changes are preserved. Allocation checks cannot prove an arbitrary heap object's identity after address reuse; prefer typed properties and validated handles. Raw addresses can still reference the wrong game data and crash the attached client. Run only scripts you trust. Check Output for cleanup failures; Stop all retries pending restoration. Existing feature/settings operations such as `duc.set` are not transactions and are not rolled back by script errors.

Roblox version checks remain enforced. Signature scanning and external execution do not ensure compatibility with updates or protection from detection.

### Input and mouse targeting

- `Input.MouseClick()` sends one synthetic left down/up pair.
- `Input.KeyPress(virtualKeyCode)` sends one synthetic key down/up pair. Accepted Windows codes: A–Z (65–90), 0–9 (48–57), Space (32), arrows (37–40). Other keys are rejected.
- Input requires attached supported client and game focus, released modifier keys, and a released target key/button. Limit: one click/key action per 100 ms. The `Input` methods are taps; the new `CheatAPI.key_press` / `key_release` pair supports held keys with cleanup. `SendInput` produces synthetic input, not physical hardware events.
- `CheatAPI.SetAimbotTarget(Vector2)` supplies a viewport point to Duc's **existing Mouse aim mode** for 100 ms. Requires focused game and Mouse aim already enabled; existing hold-to-aim and living-character checks remain. Refresh from a yielding loop for continuous targeting. It does not enable aim, perform target/team/visibility filtering, implement silent aim, or change server hit validation. Select targets carefully; scripts supply the final point.

### Cooperative tasks and execution limits

```lua
local thread = task.spawn(function(message)
    local elapsed = task.wait(0.1)
    print(message, elapsed)
end, "task completed")
-- task.cancel(thread) cancels another pending task. Return to end yourself.
```

- `task.wait(seconds?)` yields and returns actual elapsed seconds. Allowed 0–60 seconds; minimum scheduling delay 16 ms. Timing is approximate and may be longer when Duc is busy.
- `task.spawn(fn, ...args)` and `task.defer(fn, ...args)` both **queue** a host-managed task and return its thread handle. They are not exact Roblox scheduling semantics. `spawn` does not immediately run the callback before returning. Up to 32 live tasks (including the root) and 32 arguments per spawn; round-robin scheduling resumes at most four tasks per batch.
- `task.cancel(thread)` ends another queued task. `coroutine` remains unavailable. Tasks persist until completion, error, **Stop**, Stop all / End, game reattachment/change, sign-out or process exit. A task error stops the whole script. Only one script session runs at a time.
- Initial execution has a 100 ms VM budget. Later batches run no more often than every 16 ms with a 10 ms VM budget. Runaway loops are interrupted, including loops inside `pcall`. Native host calls and compilation can add latency; these are not hard real-time guarantees.
- Maximum 256 host calls/batch; VM allocations 8 MiB. Compilation and bounded native host objects are outside the VM allocator limit. Maximum source 16 KiB; binary bytecode is rejected.
- Printed output is capped at 16 KiB/session; excessive output raises an error. Avoid printing every frame. The editor polls cumulative status every 200 ms without repeating existing log output or resetting unchanged custom tabs.
- Libraries `table`, `string`, and `math` include the bundled Luau functions such as `find`, `clone`, `freeze`, and `clamp`. No `io`, `package`, filesystem, HTTP, shell/process loading, `loadstring`, `loadfile`, `dofile`, or environment-switching APIs.
- Libraries and host API tables are read-only; the two validated CheatAPI target properties accept assignment. Globals are fresh per run. Imports, saved files and examples never run automatically.

**Stop** in the script editor cancels tasks and clears script drawings/targeting/tracked writes. It does not turn off native features enabled separately by script controls. Use **Stop all / End** to disable those features. Slider configuration values persist until changed. Starting another script is rejected until the live session stops.

## Build and provenance

Luau 0.740 is vendored as `vendor/luau-0.740.zip`, downloaded from the official [Luau release source](https://github.com/luau-lang/luau/tree/0.740). CMake checks SHA256 `e47f511046e16d8ae7718a91aae6806e3803ad95df1a6d0fc89d6fa68997b431` and links Compiler/VM statically. The MIT license is in `vendor/LUAU-LICENSE.txt` and accompanies the portable build.

First-party runtime and binding translation units use the existing xollvm build path. Upstream Luau libraries are normal optimized third-party code. The editor's React assets remain inspectable.

Sandbox design uses Luau's documented [sandbox and interrupt mechanisms](https://luau.org/sandbox/). Resource limits are containment measures, not a claim of a formally proven security boundary.

## Camera readiness correction

Camera queries now measure the attached process window while the editor has focus; they no longer depend on the first overlay render or focus event. Missing/minimized windows and unavailable cameras still return an error. Scripts should yield and retry these temporary conditions. `scripts/live_overlay.luau` demonstrates a protected frame update, one-time waiting messages, and a round-robin label update. It also works on the original API v3 build: focus Roblox to initialize that build's viewport. **Stop** remains available while waiting.

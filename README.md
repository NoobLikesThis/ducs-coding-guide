# DucExternal Luau VM guide

Here's what you can use when writing scripts for DucExternal, including the original Duc API and the newer libraries. This guide is for the October 5, 2026 build.

DucExternal's VM runs Luau locally inside the app. You can use it to work with game information, control Duc's native features, build menus, draw overlays, handle input, and use local files and web requests. Scripts don't run inside Roblox's scripting environment, so familiar names such as game and Vector3 don't mean the entire Roblox API is available.

**1. Language and built-in libraries**

The runtime uses Luau 0.740. It supports normal Luau scripting: variables, functions and closures, tables, loops, conditionals, metatables, protected calls, multiple return values, and Luau syntax such as type annotations, compound assignment and continue. Type annotations don't enforce types at runtime; the host APIs perform their own argument checks.

You have table, string, math, utf8, bit32, buffer and the native vector library. That includes table searching, copying and freezing, string formatting and patterns, math and random numbers, UTF-8, bit operations, binary buffers and vector math.

The restricted os library provides clock, date, difftime and time. The debug library provides info and traceback. Neither gives scripts shell access or unrestricted access to VM internals.

Normal helpers such as print, assert, error, pcall, xpcall, type, typeof, tonumber, tostring, pairs and ipairs are available. typeof also recognizes Duc's Vector2, Vector3 and Color3 values.

Each run starts with fresh globals. You can change your own tables, vectors and colors, but the library and API tables are read-only. CheatAPI also lets you assign AimPosition and SilentAimTarget, with checks on the values you supply.

**2. Script editor and local library**

The Scripts editor supports typing scripts, importing .lua and .luau files, exporting scripts, renaming files, switching file tabs, syntax coloring, line numbers, and searching the local library. It saves up to 32 files locally. Closing a tab keeps its file in that library.

Click Execute or press Ctrl+Enter to run the active file. Output and errors appear below the editor with timestamps. Importing a file, selecting an example or restoring a saved draft won't run it. Your script library is saved on your computer.

Only one script session can run at a time. A session can still contain multiple scheduled tasks and callbacks.

**3. Duc's native controls**

Use duc.controls to see which controls are available for the current game profile. Each entry gives you its ID, label, page, value, type, and range or dropdown choices. duc.get reads a setting; duc.set changes it through the native control.

You can change supported feature toggles, integer sliders, and aiming and name-style dropdowns. Available controls depend on the game profile. Numeric settings need finite integers within range. Toggles need booleans.

Native dropdown values start at zero. Custom ui dropdowns start at one, so check which you're using.

The control interface doesn't expose arbitrary buttons, login management, sign-out, file dialogs, key capture or capture-privacy settings. duc.version is the string “3”.

**4. Game access and object handles**

duc.root, duc.workspace and duc.local_character provide the current DataModel, Workspace and local character. duc.place_id returns the place ID as a decimal string, preserving its full precision.

The game compatibility table exposes Workspace, Players, DataModel, LocalPlayer, PlaceID, PlaceId and CameraPosition. It also provides GetWorkspace and GetService. GetService accepts dot or colon calls and searches existing top-level objects; it doesn't create services or add their Roblox scripting methods.

The numeric PlaceID and PlaceId properties reject values outside Luau's exact integer range. Use duc.place_id when full precision matters.

Object handles are checked against the current process, DataModel and parent chain when you read them. An old handle can stop working if the object is destroyed, moved to a different parent, replaced during a respawn, or left behind after a game change.

The original object methods are name, class_name, children, find_first_child, position, health, parent and address. Addresses are returned as hexadecimal strings. Player and character Model position reads resolve their HumanoidRootPart; health reads resolve their Humanoid.

The compatibility methods add GetChildren, GetDescendants, FindFirstChild, FindFirstChildOfClass, FindFirstDescendant, FindFirstDescendantOfClass, FindFirstAncestor, FindFirstAncestorOfClass, IsDescendantOf, IsAncestorOf and IsA. Named-child access, object equality and readable object names are also supported.

The original children method reads up to 128 direct children. Compatibility traversal pages through larger sets and yields as it works, with an 8,192-entry traversal guard. Ancestor searches stop at 128 levels. IsA recognizes exact classes, Instance, BasePart, PVInstance, common GuiObject classes and common ValueBase classes; it isn't a complete Roblox inheritance database.

**5. Object properties**

Available reads depend on the object's class:

- General objects: Name, ClassName, Parent and Address.
- Players: Character, DisplayName, UserId, Team and CameraMaxZoomDistance. Team is a team name, not a full Team API.
- Player and character wrappers: Position, Health and MaxHealth where the required character objects exist.
- Humanoids: Health, MaxHealth and MoveDirection.
- Physical parts: Position, Size, Velocity, CanCollide, Transparency, Reflectance, Rotation, LookVector, RightVector and UpVector. Rotation is expressed as Euler angles in degrees.
- Models: PrimaryPart.
- Players service and Workspace: LocalPlayer and CurrentCamera respectively.
- Sounds: SoundId.
- MeshPart: MeshId and TextureId. SpecialMesh also supports MeshId.
- Decals: DecalTextureId.
- Proximity prompts: HoldDuration, MaxActivationDistance and ProximityActionText.
- Supported GUI objects: VisibleFrame.
- StringValue: Value.

Writable properties are physical-part Position, Velocity, Size, CanCollide, Transparency and Reflectance; player CameraMaxZoomDistance; prompt HoldDuration and MaxActivationDistance; GUI VisibleFrame; and WalkSpeed on the current local Humanoid.

WalkSpeed accepts integers from 16 to 500. Part Size requires positive components no larger than 32. CheatAPI.SetProperty specifically exposes WalkSpeed and Size; the additional writes use the compatibility object's property interface.

Some extra properties require the specific client version this build supports. They fail on unsupported versions. Only use a property on its supported class, and remember that the server can undo local changes.

**6. Players and entity snapshots**

duc.players and CheatAPI.GetPlayers return up to 64 player records containing name, is_local, player and character. The player field is the object handle; character can be absent. These records don't automatically filter out teammates or dead characters.

CheatAPI.GetLocalPlayer returns the local Player handle. GetPlayerPosition needs the exact username, not the display name. GetChildren uses the original child-query limit. FindPart searches for a named physical part and also tells you whether the search finished. If it returns no part and false, it hit the search limit; the part might still exist.

The entity library adds GetPlayers, GetLocalPlayer, GetPlayer and cached part access. Its records contain Index, Player, Character, Name, DisplayName, UserId, Team, Weapon, Position, Velocity, Health, MaxHealth, IsAlive, IsEnemy and IsWhitelisted. A projected BoundingBox is included when suitable character corners are available.

GetPlayers can filter for enemies. Enemy status compares teams independently of whitelist status. Weapon is taken from a character Tool unless a custom record supplies it. IsVisible and TeamColor are unavailable and return nil.

Each entity supports GetBoneInstance, GetBonePosition, GetBoneSize and GetBoneRotation. Here, a “bone” lookup finds a named physical child of the character; it isn't a general skeletal animation API.

GetParts returns cached part indices. GetPartPosition, GetPartSize and GetPartRotation read those indexed parts. The cache contains direct physical children of player and custom-model characters, rather than every part in the world.

AddModel, EditModel, RemoveModel and ClearModels manage up to 64 custom model records. A custom record needs a Model, a PrimaryPart belonging to it, and a name. It can use a Humanoid or supplied Health and MaxHealth values. HealthInstance is unsupported.

Snapshots refresh no more often than every 100 ms and yield while rebuilding. Don't keep using an index after its snapshot changes. BoundingBox uses the corners that project onto the screen, so it can be cut off near the edges. It doesn't tell you whether the character is behind a wall.

game.PlayerWhitelist toggles a username's whitelist membership and returns its new state. Repeating it removes an existing entry. The limit is 128 names.

**7. Camera and projection**

CheatAPI.camera exposes live ViewportSize and FieldOfView. Its WorldToScreen and WorldToViewport methods accept either dot or colon calls. CheatAPI.WorldToScreen is also available directly.

Projection returns a screen point and an on-screen boolean. A point behind the camera returns no point and false. A point outside the viewport can still return coordinates with false. The result doesn't test occlusion.

Coordinates use the game window's client area, with no GUI inset adjustment. CheatAPI.GetCameraMatrix returns Position, a nine-value row-major Rotation matrix, vertical FieldOfView, ViewportSize and, when available, ScreenOrigin for converting to desktop coordinates.

utility.WorldToScreen returns separate x, y and on-screen values. cheat.getWindowSize returns viewport width and height. You can query the camera while the editor has focus. If the game window is missing or minimized, or the camera isn't ready, wait and try again.

**8. Targeting and movement**

CheatAPI.GetClosestPlayer selects a living, non-local Player near the viewport center using the native range, distance and aim-team settings. Selection itself doesn't check occlusion.

SetTarget follows a Player, character Model or physical part through the existing aim modes. AimPosition supplies a world-space target. SilentAimTarget supplies a world-space target to the native Silent mode. ClearTarget removes script object, screen, world and silent targets.

SetAimbotTarget supplies a screen point to the existing Mouse aim mode for 100 ms. Continuous use needs refreshing from a yielding task. game.SilentAim supplies a screen point to the native Silent targeting path. Supplying a target doesn't enable its native feature.

SetTrigger and SetSilent control their corresponding native toggles. Noclip controls native noclip. SetGravity accepts integers from 0 to 500 and verifies the native change. ExpandHitboxes accepts sizes from 2 to 13; zero disables it and restores its tracked changes.

duc.teleport and CheatAPI.Teleport use the guarded local teleport operation. They require an attached, living character and stop conflicting local movement modes. Server correction can move the character back.

Object targets refresh natively. Player targets can follow a replacement character after respawn; stale parts and dead character targets are suppressed. Existing focus, hold-to-aim, range, distance and relevant wall checks still apply. Screen coordinates don't carry team or health information, so the script must choose its target deliberately.

These functions use Duc's native features. Silent behavior still depends on the game, and the server still decides whether a hit counts. Remote calls aren't available here.

**9. Persistent drawing objects**

Drawing.new creates Square, Rectangle, Line, Circle or Text objects. Rectangle is an alias of Square. Up to 128 objects can be created per run, and they start hidden.

Supported properties are Visible, Filled, Color, Thickness, Position, Size, From, To, Radius, Text and TextSize, as applicable to the shape. Text also supports Center, Outline and a numeric Size alias for TextSize. Text uses Segoe UI.

Thickness ranges from 1 to 12 pixels. Rectangle dimensions can reach 16,384 pixels; circle radius can reach 8,192. Text supports up to 256 UTF-8 bytes, with a size from 6 to 72 pixels. Outline adds a black one-pixel edge; Center centers text horizontally.

Use Remove or Destroy when you're done with a drawing object. Removing it twice is fine, but trying to read or change it afterward fails. For animations, create your objects once and update them each frame.

Successful batches publish drawing changes. A completed script can leave static drawings visible. Rendering requires attachment and game focus. Clearing drawings invalidates the old drawing handles.

**10. Immediate drawing and world labels**

The draw library provides Line, Rect, RectFilled, Circle, CircleFilled, Triangle, TriangleFilled, Polyline, ConvexPolyFilled, Gradient, Image, Text and TextOutlined. Applicable shapes support color, alpha, thickness, rounding, segment counts, closed paths or gradient direction.

GetTextSize measures text. ComputeConvexHull builds a hull from up to 512 points. GetPartCorners returns the eight world-space corners of an oriented physical part.

Immediate draw calls belong inside an onPaint callback. Each completed frame replaces the previous frame. The limits are 512 shapes, 8,192 polygon points per frame and 512 points per polygon. An interrupted or failed frame isn't published halfway through.

duc.draw.label creates text anchored to an existing physical part. Up to 32 labels are supported, with 1–48 UTF-8 bytes of text and an optional six-digit hexadecimal color. The default is sky blue. Native rendering follows the part without requiring a continuously running Luau task.

Labels don't automatically filter teams, health or walls. Replaced or invalid parts stop rendering; scripts should select fresh objects after respawns or joins. duc.draw.clear clears script drawings and labels.

**11. Custom menus**

The original duc.ui interface still works alongside the newer ui library. It has tab, toggle and bind. tab creates a desktop menu tab. toggle is only a visual switch; it has no callback and doesn't control a game feature. Use bind to add a working native toggle, slider or dropdown to your tab. Its value stays in sync with the original control.

This interface supports eight tabs and up to 16 controls per tab, with 1–48 UTF-8-byte names and labels. Reusing a tab name replaces its controls. Creating a binding doesn't turn on the feature.

The newer ui library supports newTab, newContainer, newCheckbox, newSliderInt, newSliderFloat, newDropdown, newMultiselect, newColorpicker, newInputText, newButton and newListbox.

getValue and setValue read or update widget values. setVisibility shows or hides widgets. A widget can be addressed by its returned numeric reference or by tab, container and name. Buttons and listboxes support callbacks; other widgets expose values for scripts to read.

Dropdown and listbox selections start at one. Multiselect values are boolean arrays. Color pickers use RGBA channels from 0 to 255. Text input accepts up to 4,096 bytes and commits an edit when focus leaves the field.

The newer interface allows eight tabs, eight containers per tab, 128 widgets total and 64 KiB of serialized UI. Choice lists allow up to 128 entries. You can set a container's starting visibility and make it half-width. The next, autosize and inline hints may lay things out differently from the library you're used to.

Custom UI keeps the script session alive. Stopping it disables its callbacks.

**12. Keyboard and mouse**

Input.MouseClick sends a left click. Input.KeyPress sends a key tap. These older input helpers support letters, digits, Space and arrow virtual-key codes.

CheatAPI.mouse_move sends relative mouse movement, with integer deltas from −4,096 to 4,096 pixels per axis. mouse_click supports left, right and middle clicks. key_press holds a supported key; key_release releases a key owned by the script.

The newer mouse library provides Click, IsClicked and Scroll. It supports left, right, middle and both side buttons. Scroll accepts integer wheel steps from −100 to 100. IsClicked reports a newly observed press, so very short clicks between polls can be missed.

The keyboard library provides Press, Release, Click and IsPressed. It accepts virtual-key codes from 8 to 254 and recognized names, including letters, digits, F1–F24, arrows, navigation keys, modifiers, Enter, Escape, Tab, Backspace, Space and Caps Lock. Click supports a delay between pressing and releasing.

Input is sent through synthetic Windows events. Attach to a supported client and keep the game focused before sending input. Actions share a 100 ms throttle, and keys you're physically holding can block them. A script can still release its own held keys after the game loses focus or disconnects.

Held keys are released when the script completes, errors or stops, and on focus loss, detachment or shutdown. Keep a task alive if a key needs to remain held.

**13. Utility functions, files and HTTP**

utility provides RandomInt, RandomFloat, GetTickCount, GetDeltaTime, GetFingerprint, SetClipboard, GetClipboard, MoveMouse, GetMousePos, GetMenuState, WorldToScreen and LoadImage. These cover timing, randomness, a device fingerprint, text clipboard access, relative movement, client-area cursor position, menu visibility, projection and image loading. Clipboard writes allow up to 64 KiB of text without NUL characters.

file.Read and file.Write handle binary-safe data inside Duc's files folder under the user's local application data. Writes create needed subfolders and replace the target file's contents. Each read or write is limited to 2 MiB. Paths must be relative and use forward slashes; absolute paths, directory traversal and prohibited filesystem targets are rejected.

http.Get and http.Post run asynchronously and accept headers. Request and response bodies are binary-safe. The callback receives a body, or nil on failure, and a status value.

HTTP allows four concurrent requests, 1 MiB request and response bodies, a three-second connection timeout, a ten-second total timeout and up to four redirects. Only HTTP and HTTPS are supported. Stopping the script cancels its jobs and prevents their callbacks from entering the stopped VM. Downloaded text isn't automatically executed.

**14. Images and sound**

utility.LoadImage loads PNG or JPEG data for draw.Image. Limits are 2 MiB per image, four megapixels per image, eight megapixels total and 32 image handles per session. Images can be drawn with tint and alpha.

audio.playSound provides nonblocking WAV playback, looping, volume and pitch. Supported WAV data is 16-bit PCM, mono or stereo. Compressed, 8-bit and floating-point WAV formats aren't supported.

The sound limit is eight, with up to 2 MiB of WAV input and 4 MiB of decoded samples per sound. Volume ranges from 0 to 2; pitch ranges from 0.1 to 4. audio.beep generates tones from 37 to 10,000 Hz for 1–2,000 ms.

audio.stopAll stops looping sounds. Stopping the script stops all of its sounds, including one-shot playback.

**15. Vectors and colors**

Vector2 and Vector3 provide mutable components with synchronized uppercase and lowercase names. Omitted components default to zero. They support addition, subtraction, negation, scalar and component-wise multiplication, division, equality, readable string conversion, Magnitude and Unit.

Vector methods include Abs, Ceil, Floor, Sign, Dot, Lerp, Angle, FuzzyEq, Max and Min. Cross requires Vector3. A Vector3 angle can use an axis to determine its sign. Lerp clamps its interpolation value to 0–1.

Color3 provides mutable R/G/B components and lowercase aliases. Constructors are new, fromRGB, fromHSV and fromHex. Methods are Lerp, ToHex and ToHSV. Colors default to white and clamp their components to 0–1; fromRGB accepts the familiar 0–255 scale.

These compatibility values are separate from Luau's built-in vector library. Host operations still apply their own coordinate and range checks.

**16. Memory interfaces**

Two separate interfaces are available: Memory and memory. Their capitalization and argument order matter.

The original Memory interface provides Read, Write and Scan. Reads support byte, signed int/int32, float, vector/Vector3 and address. Addresses are hexadecimal strings. Address values are read-only through this interface; scalar and vector writes are limited to 12 bytes per operation.

Scan supports hexadecimal byte patterns with wildcard bytes. Patterns can contain 1–128 bytes and need at least one fixed byte. It scans at most 256 KiB per requested range, returning matches, a continuation address and a completion flag. A call returns at most 64 matches and has an approximate six-millisecond work budget. The default range covers the first 256 KiB from the executable base, not the entire process.

The newer memory interface provides Read, Write, GetBase and Rebase. It supports bool, byte, short, ushort, int, uint, int64, uint64, float, double, pointer/ptr, string, vector2, vector3 and color3. CFrame is read-only and returns Position plus a nine-element Rotation array.

Unlike the original Memory interface, lowercase memory supports pointer writes. Its addresses can be exact numbers or hexadecimal strings. Its read arguments begin with the type, followed by the address; writes then take the value. The original uppercase interface puts the address first and type last.

Integer values outside ±9,007,199,254,740,991 are rejected rather than silently rounded. C-string writes allow 4,095 bytes plus a terminating NUL. The newer interface allows up to 4,096 bytes per tracked write, with 128 tracked writes overall.

Both interfaces operate on Duc's existing attached process. Writes require permitted writable data regions and cannot overlap tracked native feature patches. They don't expose arbitrary process opening, executable-memory writes, protection changes or code injection.

Cleanup tries to restore tracked writes. Before restoring anything, it checks that the memory still belongs to the same target and still contains the script's change. If something else changed those bytes, cleanup leaves them alone. A raw address can outlive the object that used to be there, so don't treat it like a validated object handle.

**17. Supported FFlags**

game.GetFFlag and game.SetFFlag expose 15 known flags:

- DebugFRMQualityLevelOverride
- DebugForceAnisoOff
- DebugDisableDRS
- DebugFRMOptionalMSAALevelOverride
- DebugRenderTerrainShadow
- DebugDisableModelClusterOcclusionCulling
- DebugDisableLightOcclusionCullTest
- DebugDisableSmoothClusterGrassOcclusionCullingTest
- FRMDrawDistanceScalePercent
- EnableFRMDrawDistanceScalePercent
- DebugDisableVoxelGridPaletteCompression
- TextureQualityOverride
- TextureQualityOverrideEnabled
- FRMMinGrassDistance
- FRMMaxGrassDistance

Each flag needs its supported boolean or integer type. The wrong type raises an error. Unknown flag names return nil when read and false when written. Only the 15 flags above are supported in this build.

**18. Tasks and events**

task.wait yields and returns the actual elapsed time. It accepts waits from zero to 60 seconds, with a minimum scheduling delay of roughly 16 ms.

task.spawn and task.defer both queue a host-managed task and return its thread handle. Neither promises Roblox's exact scheduling behavior. task.cancel cancels another queued task. There can be 32 live tasks, including the root task, with up to 32 arguments passed to a new task.

cheat.register supports onUpdate, onSlowUpdate, onPaint, shutdown and newPlace. There is one callback per event; registering another replaces it. Update and paint callbacks run on the cooperative scheduler, normally no faster than about 16 ms. Slow update requests a one-second interval.

shutdown and newPlace are bounded cleanup callbacks and cannot yield. A detected place change calls newPlace and stops the old session. Scripts must use newly attached objects afterward.

A task error stops the whole script session. Tasks also end on Stop, Stop all, reattachment, game changes, sign-out or application exit.

**19. Runtime limits and cleanup**

The main limits are 16 KiB of script source, 8 MiB of VM allocations and 16 KiB of printed output per session. Compilation and some bounded host resources sit outside the VM allocation limit. User-supplied binary bytecode is rejected.

Execution is divided into slices of roughly 10 ms, with later slices scheduled no more often than every 16 ms. The VM can automatically pause after about 128 host operations or a slice timeout, keeping the same execution state. At most four tasks are resumed per scheduling batch.

A task must finish or explicitly wait within 250 ms of accumulated execution or 8,192 host operations. Waiting time doesn't count toward that total. Runaway loops can still be stopped inside pcall. Some non-yieldable operations have their own 100 ms or 4,096-operation guard. These limits aren't exact timing guarantees for operating-system calls.

Automatic pauses keep unfinished drawing and menu changes pending. They appear after a successful completion or explicit wait, once the batch is ready. An error discards pending changes. Native control changes take effect immediately, though, and a later error won't undo them.

Stop cancels script tasks and callbacks, clears drawings and targets, releases held keys, stops sounds and attempts to restore tracked writes. It doesn't automatically disable native features that the script enabled through settings. Stop all / End also disables those native features. Slider configuration values remain until changed.

Many functions in draw, entity, utility, file, mouse and keyboard accept PascalCase, lower-camel and snake_case names. Other libraries may not, so use the names shown in this guide.

**20. Features that aren't available**

There is no full Roblox execution environment, standalone workspace global, remote invocation API, script injection or executor-hook API. There is no general Instance creation API, unrestricted module loading, shell execution, io or package library, loadstring, loadfile or dofile. coroutine, getfenv, setfenv and newproxy are removed.

GetAttributes, GetAttribute, GetFirstAttributeOfType, SetHighlightOnTop and SetHighlightTransparency raise explicit unsupported-operation errors. Parent and Name can't be written.

Unsupported properties include Color, Material, BonePosition, SpecialMeshTextureId, ProximityExclusivity, ButtonPosition, ButtonSize, FramePosition, FrameBackgroundColor and FrameBorderColor. Numeric, boolean, vector, color and object Value instances aren't implemented; only StringValue reading is supported.

Health, MaxHealth, MoveDirection, Rotation, direction vectors, IDs, teams and the exposed string properties remain read-only. Full class reflection, reliable entity occlusion, exact compatibility with another UI layout engine and unrestricted 64-bit integer arithmetic aren't provided.

The native and VM tests pass for this build. Live input, sound playback and Silent behavior still need testing in-game. Support will vary between games.

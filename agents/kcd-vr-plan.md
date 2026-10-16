# kcd-vr plan

Status: draft, 2026-10-15. Implements `specs/kcd-vr-spec.md`. Units are in
dependency order. Each unit is one or more commits. Checkboxes are the
record of completion.

## 0. Description, systems, deliverables

**0.1 Description.** A native Windows x64 plugin (`kcdvr.dll`) injected
into `KingdomCome.exe` through a proxy DLL, plus a Lua mod in the game's
official mod format for game-side logic, plus an installer and a Linux
launch recipe. The plugin hooks the view system, the CryDX12 swap chain
and the animation pose-modifier stack, renders both eyes per frame,
submits through OpenXR on D3D12, and drives Henry's skeleton.

**0.2 Systems.**
- Target: `WHGame.dll` release_1_5 (1.5.6), CryEngine 5.x fork, DX12.
- Plugin: C++20, MSVC on the Windows 4090 box, MinHook or a hand-rolled
  trampoline, OpenXR loader statically linked, `XR_KHR_D3D12_enable`,
  `XR_KHR_visibility_mask`, `XR_EXT_hand_tracking` optional. Host tests
  build with the same toolchain and also with clang on Linux so pure
  modules stay portable.
- Signature registry and RTTI healing modeled on KCD2Tools TPVCamera
  (MIT, `research/KCD2Tools`).
- Game-side: Lua in `mods/kcdvr/`, cvars via `mod.cfg`, data via pak.
- Bench: public CryEngine 5.x (`research/cryengine`) built on the same
  Windows box.
- Runtime: SteamVR on Windows, null driver until a headset is attached.
  Proton on the Linux box only in the portability unit.
- Tests: host-side unit tests (Catch2 or GoogleTest) for every pure
  module, built natively on Linux; in-game probes that run the game under
  Proton with a scripted route and assert on telemetry lines.
- Tooling in cajeta under `tools/` (asset patcher, telemetry analysis,
  installer manifest), per house rule.

**0.3 Deliverables.**
- `kcdvr.dll`, `kcdvr-proxy.dll` (as `dinput8.dll`), `kcdvr-probe.exe`,
  `kcdvr-monitor.exe`.
- `mods/kcdvr/` Lua and data.
- Installer and a Linux launch script.
- A measured performance table for the development box.
- `docs/` user guide.

## 1. Toolchain, proxy load, first run on Windows

Spec: 2.1, 2.2, 13.1, 14.1.

- [ ] **1.1 TDD**
  - [ ] 1.1.1 Host test: the signature registry parses a pattern string
    with wildcards and finds it in a byte buffer, rejects ambiguous
    matches, and falls through candidates in order.
  - [ ] 1.1.2 Host test: the logger writes a line with monotonic
    timestamp and subsystem tag to a file and to a ring buffer.
  - [ ] 1.1.3 In-game probe: with the proxy present, the log contains
    `plugin loaded` and the game reaches the main menu.
- [ ] **1.2 Coding**
  - [ ] 1.2.1 CMake project, MSVC on the 4090 box for DLLs and tests,
    clang on Linux for the pure-module tests only.
  - [ ] 1.2.2 `kcdvr-proxy.dll` exporting `dinput8` forwarders and
    loading `kcdvr.dll` next to it.
  - [ ] 1.2.3 Signature registry (`Direct`, `RipRelative`, candidate
    lists, module-scoped scan) and RTTI locator (type descriptor to
    vtable by complete object locator).
  - [ ] 1.2.4 Logger and a `kcdvr.cfg` reader.
  - [ ] 1.2.5 A `run-kcd2.ps1` that launches the game with the plugin
    from the shell, outside Steam's UI, and SteamVR null driver setup.
- [ ] **1.3 Acceptance**
  - [ ] 1.3.1 The flat game runs unchanged with the proxy present and
    `kcdvr.dll` absent.
  - [ ] 1.3.2 `research/kcd2-local-findings.md` gains a section for the
    Windows box: game build, driver, SteamVR version.

## 2. Hook the camera, head-tracked flat window

Spec: 4.1, 4.7, 4.8, 14.2.

- [ ] **2.1 TDD**
  - [ ] 2.1.1 Host test: a pose composer takes the game's view matrix,
    strips rotation, applies an HMD orientation and positional offset,
    and returns a matrix whose forward vector matches the HMD within
    1e-5.
  - [ ] 2.1.2 Host test: a camera-effect suppressor zeroes shake, noise
    and FOV blend inputs while preserving position.
  - [ ] 2.1.3 In-game probe: with a synthetic pose source (scripted
    sinusoid), telemetry shows `cam.applied` per frame with the sinusoid
    recovered from the applied matrix.
- [ ] **2.2 Coding**
  - [ ] 2.2.1 Resolve `CView`, `CViewSystem`, `CCamera` frustum update,
    `C_Player`, `CD3D9Renderer` with candidates and RTTI gates.
  - [ ] 2.2.2 Hook the frustum update gated on the live `CView`, apply
    the composed matrix before culling planes are built.
  - [ ] 2.2.3 Pose source interface with a synthetic implementation.
  - [ ] 2.2.4 Suppress `cl_headBob`, `cl_camera_noise`, FOV blends and
    the dash kick while active.
- [ ] **2.3 Acceptance**
  - [ ] 2.3.1 Walking through Troskowitz with the synthetic pose shows no
    culling pop-in at the frustum edges.

## 3. OpenXR session on SteamVR, mono submit

Spec: 2.4, 3.5, 3.8, 3.9, 13.2.

- [ ] **3.1 TDD**
  - [ ] 3.1.1 Host test: the probe's runtime check parses the active
    runtime manifest and reports D3D12 support (fixture manifests).
  - [ ] 3.1.2 In-game probe: under the SteamVR null driver, telemetry
    shows `xr.session READY`, `xr.frames > 100`, swapchain sizes logged.
- [ ] **3.2 Coding**
  - [ ] 3.2.1 Enable SteamVR's null driver and confirm the OpenXR
    runtime is active without a headset.
  - [ ] 3.2.2 `kcdvr-probe.exe`.
  - [ ] 3.2.3 OpenXR session with `XrGraphicsBindingD3D12KHR` from the
    CryDX12 device and queue, two swapchains, frame loop driven from the
    swap-chain Present hook.
  - [ ] 3.2.4 Submit the flat frame to both eyes as a mono projection
    layer. Desktop mirror stays the game's own window.
  - [ ] 3.2.5 Disable `r_VSync` and frame cap while a session is active.
- [ ] **3.3 Acceptance**
  - [ ] 3.3.1 Frame time telemetry shows pacing from `xrWaitFrame`.

## 4. Stereo: two eye passes per frame

Spec: 3.1, 3.2, 3.3, 3.4, 3.6, 3.7, 3.10.

- [ ] **4.1 TDD**
  - [ ] 4.1.1 Host test: per-eye projection from OpenXR fov angles
    produces the asymmetric matrix and the matching culling tangents.
  - [ ] 4.1.2 Host test: the shared shadow frustum for two eyes is the
    union and is clamped below a hemisphere.
  - [ ] 4.1.3 Bench test on the open engine: re-entering the view render
    twice with different cameras produces two images that differ only by
    parallax and share one shadow pass (shadow draw counter).
  - [ ] 4.1.4 In-game probe: telemetry `stereo_frames` increments, eye
    images differ, `shadow.passes == 1` per frame.
- [ ] **4.2 Coding**
  - [ ] 4.2.1 Resolve the renderer's view render entry, shadow cascade
    build, particle update and post-process pipeline entry.
  - [ ] 4.2.2 Eye loop: set camera, render, hand the eye target's
    `ID3D12Resource` to the swapchain image via copy-free aliasing or one
    GPU copy if aliasing fails.
  - [ ] 4.2.3 Shadow sharing, particle once-per-frame, hidden-area mesh.
  - [ ] 4.2.4 Post effects off, TAA jitter per eye, render scale cvar.
  - [ ] 4.2.5 Lens-matched shading probe via `r_VrProjectionType`.
- [ ] **4.3 Acceptance**
  - [ ] 4.3.1 Stereo is correct on a headset or in Monado's debug view.
  - [ ] 4.3.2 Measured: eye-pass cost on this box at three render scales.

## 5. Input and comfort

Spec: 4.4, 4.5, 4.6, 10.1 to 10.5, 11.1 to 11.3.

- [ ] **5.1 TDD**
  - [ ] 5.1.1 Host test: action mapping converts an action state table
    into engine input events, with edge detection.
  - [ ] 5.1.2 Host test: snap and smooth turn accumulate correctly across
    variable frame times.
  - [ ] 5.1.3 In-game probe: a scripted action sequence moves Henry
    forward 5 m and turns 90 degrees.
- [ ] **5.2 Coding**
  - [ ] 5.2.1 Action set, bindings for three controller families.
  - [ ] 5.2.2 Post events through the engine's input interface, filter
    the look action maps.
  - [ ] 5.2.3 Recenter, seated and standing calibration, turn modes.
  - [ ] 5.2.4 Vignette shader on the eye image, haptics.
- [ ] **5.3 Acceptance**
  - [ ] 5.3.1 Gamepad fallback works with head tracking.

## 6. HUD and menus

Spec: 9.1 to 9.5, 6.6.

- [ ] **6.1 TDD**
  - [ ] 6.1.1 Bench test: a Flash player instance redirected to a dynamic
    texture renders nothing to the backbuffer.
  - [ ] 6.1.2 In-game probe: `hud.texture` logged with size, and the eye
    images contain no HUD pixels (sampled corner regions).
- [ ] **6.2 Coding**
  - [ ] 6.2.1 Redirect `CFlashUI` elements to an offscreen target.
  - [ ] 6.2.2 Head-locked HUD quad, world-locked menu panel, ray cursor
    posting mouse events.
  - [ ] 6.2.3 Crosshair and screen markers off, world markers placed.
- [ ] **6.3 Acceptance**
  - [ ] 6.3.1 Inventory, map and dialog choices are usable with
    controllers only.

## 7. Henry's head

Spec: 4.2, 4.3.

- [ ] **7.1 TDD**
  - [ ] 7.1.1 Bench test: a look pose modifier target set each frame
    turns the head bone toward it within the joint limits.
  - [ ] 7.1.2 In-game probe: head bone orientation in telemetry tracks
    the HMD within 2 degrees while walking.
- [ ] **7.2 Coding**
  - [ ] 7.2.1 Resolve the player's character instance, the pose modifier
    stack and the look target channel (`S_MountAnimState` look target on
    foot and mounted).
  - [ ] 7.2.2 Feed the HMD look target and neck offset, body-follow limit.
- [ ] **7.3 Acceptance**
  - [ ] 7.3.1 Henry's shadow turns its head with the player.

## 8. Arms

Spec: 5.1 to 5.5, 5.7, 5.8.

- [ ] **8.1 TDD**
  - [ ] 8.1.1 Host test: the two-bone IK solver reaches targets inside
    reach, respects the elbow hint, and clamps outside reach.
  - [ ] 8.1.2 Bench test: a limb IK setup pushed per frame drives the
    hand bone to a moving target on the open engine's sample character.
  - [ ] 8.1.3 In-game probe: hand bone position tracks a synthetic
    controller path within 2 cm.
- [ ] **8.2 Coding**
  - [ ] 8.2.1 Resolve limb IK definitions, the pose modifier stack push,
    and the weapon solver's target entry and enable flag.
  - [ ] 8.2.2 Per-frame arm pose injection, suppression of the owned
    bones in the FP animation layer, finger blend.
  - [ ] 8.2.3 Calibration flow and storage.
- [ ] **8.3 Acceptance**
  - [ ] 8.3.1 Holding a torch, the light follows the hand.

## 9. Items, interaction, minigames

Spec: 5.3, 5.6, 6.1, 6.5.

- [ ] **9.1 TDD**
  - [ ] 9.1.1 In-game probe: a scripted hand path to the scabbard with
    grip triggers the draw fragment (telemetry `item.draw`).
  - [ ] 9.1.2 In-game probe: lockpicking twist input from a synthetic
    controller opens a known lock.
- [ ] **9.2 Coding**
  - [ ] 9.2.1 Holster volumes, grip gestures mapped to the item handling
    requests.
  - [ ] 9.2.2 Gaze and pointer interaction prompt.
  - [ ] 9.2.3 Minigame input adapters and arm ownership hand-back.
- [ ] **9.3 Acceptance**
  - [ ] 9.3.1 Forge and alchemy are completable.

## 10. Combat

Spec: 6.2 to 6.8.

- [ ] **10.1 TDD**
  - [ ] 10.1.1 Host test: the zone mapper takes a hand position relative
    to the head and body yaw and returns the game's zone id, with the
    zone boundaries loaded from `Libs/Tables/combat/*.xml` fixtures.
  - [ ] 10.1.2 Host test: the swing detector on recorded controller
    tracks reports the zone at swing start, not mid-swing, and the stab
    detector fires only when the blade is forward and velocity is along
    it, with a confusion matrix against labeled fixtures.
  - [ ] 10.1.3 In-game probe: scripted swings post the expected attack
    type and zone (telemetry `combat.attack zone= type=`), a scripted
    guard raise inside the window posts a block, and a scripted opposite
    zone swing during an NPC attack yields a master strike.
  - [ ] 10.1.4 Host test: the weapon line from one or two controller
    poses matches a fixture muzzle direction within 0.5 degrees, with
    the two-hand refinement engaging only when the off hand grips.
  - [ ] 10.1.5 In-game probe: with a synthetic weapon line pointed at a
    target, a hand cannon shot and a crossbow bolt land on it (telemetry
    `ranged.shot origin= dir= hit=`) and the loading animation hands the
    arms back.
- [ ] **10.2 Coding**
  - [ ] 10.2.1 Read the combat tables for zone ids, attack types and
    windows at startup.
  - [ ] 10.2.2 Zone mapper held as the game's zone input every frame,
    swing and stab detectors, attack posting.
  - [ ] 10.2.3 Arm blend toward the fragment's weapon path during an
    attack, hand-back at fragment end.
  - [ ] 10.2.4 Block and dodge posting from guard pose and body step.
  - [ ] 10.2.5 Missile weapons: weapon line from controller poses, shot
    origin and direction redirected through the missile weapon player
    controller, bow draw from string-hand distance, trigger with the
    game's ignition delay, spread as a cone, recoil kick and haptics,
    loading animation ownership and hand-back, optional aim line.
  - [ ] 10.2.6 Quiver volume on the quiver attachment, arrow grab and
    nock gesture with the arrow item attached to the string hand, match
    hand and touch-hole volume for the hand cannon, auto-nock setting.
  - [ ] 10.2.7 Crossbow spanning sequence per loading type (read from the
    missile weapon class's loading movement), step volumes and motions,
    minimum time clamped to the game's loading duration, bolt placement,
    animation fallback setting.
- [ ] **10.3 Acceptance**
  - [ ] 10.3.1 A full training-ground session against Tomcat is winnable.

## 11. Horseback

Spec: 7.1 to 7.5.

- [ ] **11.1 TDD**
  - [ ] 11.1.1 In-game probe: mounted, head yaw of 90 degrees produces
    rider torso rotation in telemetry and no camera centering fights it.
- [ ] **11.2 Coding**
  - [ ] 11.2.1 Mounted eye anchor, head collision respect, centering and
    sway cvars off, dash kick off.
  - [ ] 11.2.2 Reins and bridle pose modifier following hands, hide
    option.
- [ ] **11.3 Acceptance**
  - [ ] 11.3.1 A gallop from Troskowitz to Semine is comfortable at the
    measured frame rate.

## 12. Cutscenes and dialog

Spec: 8.1 to 8.6.

- [ ] **12.1 TDD**
  - [ ] 12.1.1 Host test: the jump planner turns a recorded camera track
    (fixtures: static shots, a dolly, a zoom, a cut) into vantage points,
    with the zoom point at the distance that preserves subject angular
    size and no two points closer than the threshold.
  - [ ] 12.1.2 Host test: an override file for a sequence name replaces
    the derived plan.
  - [ ] 12.1.3 In-game probe: during a known in-engine cinematic, the
    view position equals the current vantage point, rotation matches the
    HMD, and every transition is a cut with a blink (telemetry
    `cine.jump`).
  - [ ] 12.1.4 In-game probe: a Bink video shows on the virtual screen and
    skip works.
- [ ] **12.2 Coding**
  - [ ] 12.2.1 Sequence camera substitution via `mov_overrideCam` or a
    hook on the movie system's camera set.
  - [ ] 12.2.2 Track reader over the movie system's camera node keys at
    sequence start, jump planner, override file, dev dump.
  - [ ] 12.2.3 Dialog camera manager hook with the same rule.
  - [ ] 12.2.4 Virtual screen for Bink, letterbox removal, blink on cut,
    mode setting, skeleton driving paused during cinematics.
- [ ] **12.3 Acceptance**
  - [ ] 12.3.1 The prologue is playable start to end.

## 13. Performance

Spec: 12.1 to 12.3, 3.11.

- [ ] **13.1 TDD**
  - [ ] 13.1.1 Telemetry analysis tool (cajeta) reads the CSV and reports
    p50 and p99 per phase.
- [ ] **13.2 Coding**
  - [ ] 13.2.1 Adaptive render scale, lens-matched shading if 4.2.5 held.
  - [ ] 13.2.2 Town benchmark route script.
- [ ] **13.3 Acceptance**
  - [ ] 13.3.1 Table of render scale against frame time for this box in
    `docs/performance.md`.

## 14. Lifecycle and distribution

Spec: 2.3, 2.5, 2.6, 14.3, 16.1 to 16.3.

- [ ] **14.1 TDD**
  - [ ] 14.1.1 Host test: the asset patcher verifies hashes before and
    after applying a patch and refuses on mismatch.
  - [ ] 14.1.2 Host test: the offset healer finds a moved field by RTTI
    name within the scan window.
- [ ] **14.2 Coding**
  - [ ] 14.2.1 Crash monitor, profile directory, launcher.
  - [ ] 14.2.2 Installer (finds the game, pins hashes, patches,
    uninstalls).
- [ ] **14.3 Acceptance**
  - [ ] 14.3.1 Install, play, uninstall leaves the game folder identical
    by hash.

## 15. Linux portability

Spec: 13.3, 13.4.

- [ ] **15.1 TDD**
  - [ ] 15.1.1 In-game probe on the Strix Halo box under Proton with
    `WINEDLLOVERRIDES="dinput8=n,b"`: `plugin loaded`, `xr.session
    READY` through wineopenxr, `stereo_frames > 100`.
- [ ] **15.2 Coding**
  - [ ] 15.2.1 `run-kcd2.sh`, documented Proton version and launch line,
    any wineopenxr-specific fixes isolated behind one platform check.
- [ ] **15.3 Acceptance**
  - [ ] 15.3.1 The performance table in `docs/performance.md` carries a
    Strix Halo row.

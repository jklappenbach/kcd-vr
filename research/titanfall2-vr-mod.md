# How the Titanfall 2 VR mod was done

Compiled 2026-10-15 from the installer source (MIT, cloned at
`research/CircuitLordVRModInstaller`), the shipped mod package
`research/packages/Titanfall2VR-1.0.17.zip` (SHA-256 matches the manifest),
and press coverage. "Verified" means read from source or the package.
"Inferred" means deduced from strings in the closed DLL. The package itself
is closed source, all rights reserved. A deeper dissection of the package
is in `titanfall2-vr-package-analysis.md`.

## 1. Who, when, how big

- Author: CircuitLord (Jordan), solo. Previously Big Walk VR (2026-08).
- Teaser 2026-09-14 ("native stereo rendering, fullbody IK"), release
  2026-09-30, six installer builds to 2026-10-06. Roughly six to eight weeks
  of work for the game-specific part, on top of the Big Walk VR installer
  and whatever private tooling he carries between projects.
- Package: `Titanfall2VR.dll` (3 MB, MSVC C++, embedded HLSL compiled at
  runtime), `xr_probe.exe`, `crash_monitor.exe`, `asset_patcher.exe`,
  two Northstar Squirrel mods (about 5,500 lines), 89 bsdiff patches
  against Respawn files (41 scripts, 53 particle files, 3 models).
- No lineage to UEVR, Luke Ross or fholger. Credited third parties are the
  OpenXR SDK, Valve's OpenVR driver header and a virtual-display example.

## 2. Loading and hooking (verified where stated)

- The DLL is a Northstar plugin: exports `CreateInterface`, implements
  `PluginCallbacks001` (`Init`, `OnSqvmCreated`, `OnLibraryLoaded`,
  `RunFrame`). Northstar is the open-source Titanfall 2 mod loader with a
  launcher that injects into the unmodified `Titanfall2.exe`. Verified.
- About 294 named patch sites across `client.dll`, `server.dll`,
  `engine.dll` and the material system (`engine.renderViews`,
  `client.drawCockpitHud`, `client.drawCascades`,
  `client.attachmentFovCall1..11`, `client.inputButtonsSlot`,
  `dxgi.Present`, `dxgi.ResizeBuffers`). Strings like "cannot write engine
  call thunk" and "caller changed" show call-site rewriting with integrity
  checks, not entry-point detours. Inferred.
- The installer refuses to launch if a `dxgi.dll` or `d3d11.dll` sits
  beside the game, because it takes `Present` away from the mod. Verified.

## 3. Rendering (verified unless marked)

- OpenXR on D3D11: statically linked loader, `XR_KHR_D3D11_enable`,
  `PRIMARY_STEREO`, one swapchain per eye, hidden-area mesh,
  `XR_EXT_hand_tracking`, Win32 performance-counter time conversion.
- Stereo is most likely both eyes per frame on a frozen simulation state,
  re-running the engine's `renderViews` per eye with asymmetric
  projections, widened culling and shared shadow cascades. Evidence:
  "OpenXR eyes used different client simulation states", "body model
  changed during stereo frame", "stereo shadow frusta exceed a
  hemisphere", `TF2VR_DISABLE_SHARED_SHADOWS`, a `stereo_frames` counter,
  per-eye `world_culling_tan_x/y`. Press claims of alternate-frame
  rendering contradict these strings. Inferred.
- Launch convars (verified in `launch.json`): depth of field off, ADS DOF
  off, letterbox off, dynamic resolution off, vsync off, fps cap off,
  `mat_viewportscale 1`, windowed 1280x720 which the mod replaces before
  the engine initializes.
- HUD and RUI are intercepted (`drawHudRui`, `cockpitRuiViewCall`,
  `fullscreenRuiViewCall`) and re-projected through overlay shaders.
  Convars `tf2vr_head_hud`, `tf2vr_titan_hud`, `tf2vr_cutscenes_vr`
  (2D panel or full VR). Scopes use a lens shader. Comfort vignette shader.

## 4. Input, body and gameplay

- Native: OpenXR action sets with an `oculus/touch_controller` profile,
  haptics, arm and full-body IK (shoulder, elbow, wrist solvers, pelvis,
  foot planting, physical crouch), snap and smooth turn, recentering,
  weapon aim by redirecting shot origin and direction on both client and
  server (`client/server.shotDirection`, `bulletTraceCall`), scopes,
  grenade-arc prediction. Also WASAPI loopback capture and an MP4
  "memories" recorder played back in a custom map.
- Squirrel: the DLL registers natives (`TF2VR_SetBodyMovement`,
  `TF2VR_UpdateTitanPose`, `TF2VR_PulseHands`, ...). Script owns game-state
  logic: holster slots, grab and throw grenades, reload gestures, Titan
  vortex placement, head-aimed Titan traces, prompts, settings menu.
  Client and server talk through client command callbacks and about 45
  convars in `mod.json`.
- Assets: an "empty hands" weapon, patched viewmodel and cockpit models,
  a gesture rpak, patched particle files.
- The split: heavy lifting native, script as gameplay glue and the
  server-authority bridge.

## 5. Install, profile, launch (verified in `Titanfall2Installer.cs`)

- Finds the game by registry (`HKLM\SOFTWARE\Respawn\Titanfall2\Install
  Dir`) and by walking Steam `libraryfolders.vdf`.
- Downloads Northstar v1.31.13 pinned by SHA-256, copies
  `NorthstarLauncher.exe` as `Titanfall2VRLauncher.exe`, puts Northstar
  plus the mod under `<game>/TF2VR/` with the DLL at
  `TF2VR/plugins/Titanfall2VR.dll`. `Titanfall2.exe` is never touched.
- `asset_patcher.exe apply` extracts originals from the game's VPKs,
  verifies SHA-256, applies bsdiffs, writes only after every file
  verifies. No Respawn files are redistributed.
- Launch: `crash_monitor.exe <profile> Titanfall2VRLauncher.exe
  -profile=TF2VR -novid -windowed ...`. Master server pointed at
  `127.0.0.1:9` so the game is effectively offline. Saves go to a separate
  `Titanfall2_VR` folder.
- Six post-release installer builds were mostly about launch blocking and
  crash reporting, which says the install surface is where a solo
  project bleeds time.

## 6. Linux and Proton

- Officially Windows only. Community recipe (Monkellie/tf2vr-linux):
  SteamVR, Proton Experimental, DXVK, VC++ 14.40 or newer, a private
  Proton with a 6-byte patch to `mmdevapi.dll` because the loopback
  audio capture crashes Wine, and `SteamGameId=1237970` so wineopenxr
  sets up its bridge. Monado and WiVRn: controllers reported unsupported.
- Northstar itself runs under Proton (NorthstarProton).

## 7. What this tells us for a CryEngine game

1. The mod loader is the foundation. Northstar gave him injection, a
   plugin ABI, a script VM hook and a profile system for free. KCD2 has
   KCSE (dinput8 proxy, C++ plugin API) in that role.
2. Both eyes per frame from one simulation step, re-entering the engine's
   own view render. Not alternate-eye.
3. OpenXR, not OpenVR, with the loader statically linked.
4. Script where the game's own logic lives, native where frame timing,
   IK, rendering and input live.
5. The game's hand and weapon animation is replaced by the mod's IK on
   the real body, with gameplay effects (shot direction) rerouted to the
   tracked hands.
6. HUD captured and projected, with per-context policy (head-locked,
   cockpit-locked, 2D panel for cutscenes).
7. Nothing of the publisher's is redistributed: patch at install time
   from the user's own files, verify hashes.
8. A crash monitor and a runtime probe ship from day one.

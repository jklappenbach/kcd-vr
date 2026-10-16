# CryEngine VR prior art and the KCD2 modding landscape

Compiled 2026-10-15. [V] read in source or primary docs. [C] claim from
press, forums or search. Repos are cloned under `research/`.

## 1. Prior art

### fholger, Crysis VR (`research/crysis_vrmod`) [V]
- CryEngine 2 (2007 Crysis, Remastered unsupported). Not an injector: the
  CryEngine 2 Mod SDK shipped the game DLL source, so the mod is a
  recompiled `CryGame.dll` plus a launcher. About 6.5 kLOC in `Code/VR`
  inside about 275 kLOC of SDK game code he also edited.
- Hooks (MinHook, vtable): `ISystem::Render`, `ISystem::Quit`,
  `I3DEngine::SetPostEffectParam`, `IAISystem::SmartObjectEvent`; D3D10 and
  DXGI (`CreateSwapChain`, `Present`, `ResizeBuffers`), `SetWindowPos`.
  The closed renderer's shadow pass is hooked at a hard-coded RVA to skip
  it for the second eye.
- Stereo: `VRRenderer::Render` replaces `ISystem::Render` and runs the
  whole engine frame twice, `SetViewCamera(eyeCam)` then the original
  Render, `RenderBegin()` between eyes to reset state, particle update
  suppressed for eye two via a cvar. Each pass copies the backbuffer to an
  eye texture. Post, shadows and so on run per eye. README: CPU bound.
- Submission: D3D10 textures shared into a D3D11 device
  (`GetSharedHandle`, `OpenSharedResource`, keyed mutex), OpenXR
  `XR_KHR_D3D11_enable`, one projection layer plus a quad layer for HUD.
- HUD: a third pass into a transparent target captured at `Present`,
  shown as an OpenXR quad. Scopes, vehicles and cutscenes fall back to a
  2D screen or a "3D cinema" mode.
- Hands and weapons done in game code (`OpenXRInput.cpp`, hand poses,
  holsters, two-handed weapons, haptics).

### fholger, Far Cry VR (`research/farcry_vrmod`) [V]
- CryEngine 1, D3D9 through his DXVK fork `dxvkry` shipped as `d3d9.dll`,
  exporting the Vulkan device and images so he can submit to OpenVR.
  Hooks `IRenderer::SetCamera` (vtable slot 36) and `Present`. About 3 kLOC.

### Luke Ross, R.E.A.L. VR [C]
- Closed source, Patreon. Alternate Eye Rendering with reprojection.
- Added KCD2 in v18.0.0 on 2025-03-21: 6DOF head tracking, gaze aiming,
  gamepad or mouse, no motion controls, mostly 3D cutscenes, videos on a
  2D screen. Reported: CPU-bound in towns, uncomfortable on horseback.
- This is the only shipped KCD2 VR mod.

### vorpX [C]
- Official KCD1 profile (DirectVR head tracking, Geometry 3D). No KCD2
  profile. Users borrow DX12 profiles for depth-buffer 3D with mixed
  results.

### UEVR (contrast) [V, README]
- Unreal 4.8 to 5.4 only. Works because Unreal has reflection and a
  native stereo path to drive. CryEngine as shipped in KCD2 has neither.

## 2. KCD2 technical landscape

- Engine lineage is a CryEngine 5.x fork (CryDX12 layer, `r_D3D12*`,
  `r_Driver`, `r_VrProjectionPreset` in the community cvar dump, RTTI
  `CD3D9Renderer`) with Warhorse's `wh::*module` namespaces. See
  `kcd2-local-findings.md` for what was measured on this machine.
- DX12 in practice. KCD2ModLoader installs as a `d3d12.dll` proxy. Plan
  for D3D12 only.
- No Denuvo, no anti-cheat [C, multiple outlets]. No public PDB. Vtable
  order is shuffled by CryEngine's interfuscator in release builds, so
  stock CryCommon headers cannot be used for virtual calls [V, libKCD2
  README].
- Official modding tools: Steam app 2429020 with a dev build, the Sandbox
  editor, a localhost HTTP API on port 1403, and Lua binding docs. Mod
  layout `mods/<modid>/mod.manifest`, `mod.cfg`, `data/*.pak`,
  `scripts/mods/<modid>.lua`, `__modid` table patches, `mod_order.txt`.
  No official native plugin path [V, wiki mirror in
  `research/kcd2-modding-docs`].
- Lua surface: about 5000 doc pages of binds. Standard CryEngine binds
  plus Warhorse `C_ScriptBind*` (Player, Human, Horse, Soul, RPG, UI).
  Nothing gives render or stereo control.
- Relevant cvars [V, `research/KCD2ModLoader/docs/cvars.md`]: `cl_fov` 65
  vertical, `cl_hfov` 95, `r_DrawNearFoV` 60, `r_DrawNearZRange`,
  `cl_headBob`, `g_detachCamera`, `demo_use_hmd_rotation`,
  `g_show_crosshair`, `cl_DisableHUDText`, `r_PostProcessHUD3D*`,
  `r_Supersampling`, `r_CustomResWidth`, stereo remnants, and
  `r_VrProjectionType` / `r_VrProjectionPreset` (NVIDIA multi-res and
  lens-matched shading presets from CE 5.3+, worth probing).

## 3. Native injection prior art for KCD2

- KCD2ModLoader (xiaoxiao921) [V]: `d3d12.dll` proxy on
  ReturnOfModdingBase, Lua hot reload, ImGui, FMOD hooks, ASI loader.
- KCSE plus libKCD2 (JerryYOJ) [V, README]: SKSE-style script extender as
  a `dinput8.dll` proxy, C++ plugins in `mods/<mod>/KCSE/Plugins/`,
  trampoline, per-frame tasks, targets game 1.5.6. libKCD2 carries
  reverse-engineered headers and vtable and RTTI offset tables. About 47
  Nexus mods depend on KCSE [C].
- KCD2Tools TPVCamera (tkhquang, MIT, active 2026-10) [V]: ASI plugin on
  DetourModKit, about 15 kLOC. Hooks `CCamera::UpdateFrustumPlanes` gated
  on the embedding object's vtable matching `.?AVCView@@`, so offsetting
  the camera matrix there moves rendering and culling together. Reads the
  look controller via `gEnv+0x90` to pGame, IGame vtable slot 16, `+0x88`,
  `+0xA40` (`C_Player`), `+0x238`. All found by AOB cascades with an
  RTTI-anchored offset-heal step for patches.

## 4. Linux and Proton

- KCD2 is ProtonDB Platinum [V, API]. Prefix on this machine exists.
- Proton (proton_10.0 source, copies in `research/proton.py`,
  `research/vrcomp.c`, `research/wopenxr.c`) [V]: installs
  `vrclient_x64.dll`, writes `openvrpaths.vrpath`, installs
  `openvr_api_dxvk.dll` and `C:\openxr\wineopenxr64.json`.
  `vrcompositor_manual.c` translates D3D12 submissions through
  vkd3d-proton interop. `wineopenxr` maps `XR_KHR_D3D12_enable` onto
  `XR_KHR_vulkan_enable`. An injected DLL that carries the Windows
  OpenXR loader reaches Linux SteamVR or Monado without hand-copied DLLs.
- UEVR under Proton is documented (LVRA wiki): .NET in prefix,
  `PROTON_REMOTE_DEBUG_CMD`, OpenVR for SteamVR, OpenXR for Monado and
  WiVRn, Proton 11 recommended [C].
- Proxy DLLs need `WINEDLLOVERRIDES="dinput8=n,b"`. A `d3d12.dll` proxy
  collides with vkd3d-proton's own `d3d12.dll` and needs testing.

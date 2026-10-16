# Kingdom Come: Deliverance II, measured on this machine

Measured 2026-10-15 against the Steam install at
`~/.steam/steam/steamapps/common/KingdomComeDeliverance2`, game build
`release_1_5` (wh_sys_version 1.5.6, binaries dated 2026-06-19). Everything
below was read from the binaries, paks and config on disk, not from coverage.

## 1. Binaries

| File | Size | Role |
|---|---|---|
| `Bin/Win64MasterMasterSteamPGO/KingdomCome.exe` | 1.5 MB | launcher stub |
| `Bin/Win64MasterMasterSteamPGO/WHGame.dll` | 89 MB | the whole engine and game, monolithic |
| `Bin/Win64MasterMasterSteamPGO/WHGameArm.dll` | 107 MB | ARM64 build of the same |
| `Bin/Win64Shared/*.dll` | | FMOD, Bink, DLSS, XeSS, FidelityFX, Aftermath, AMD AGS, Steam |

- PDB paths are embedded (`d:\buildagent\work\...\WHGame.pdb`) but no PDB ships.
- MSVC RTTI type descriptors survive. `CRenderView`, `CCamera`,
  `CFlowNode_Camera*`, `C_CombatCameraManager`, `CFlashUI`,
  `CEngineModule_CryRenderer`, `CEngineModule_CryInput` and thousands more
  are locatable by name, so vtables can be found without symbols.
- Source paths show the lineage: `code\CryEngine\CrySystem\Mods\ModManager.cpp`,
  CryAction, CryInput, Scaleform helper. This is CryEngine 5.x with Warhorse's
  `wh::*module` namespaces on top.

## 2. Renderer

- DirectX 12 only. The `r_D3D12*` cvar family is present (async compute,
  waitable swap chain, DRED, PIX capture). No Vulkan loader, no D3D11 device
  creation path beyond a name string.
- The DX12 backend is CryEngine's `CryDX12` layer: RTTI shows
  `CCryDX12GIObject<IEmptySwapChain3>`. The renderer code is written against
  D3D11-shaped interfaces that this layer implements on D3D12. Hooks at the
  CryDX12 boundary see a D3D11-like call stream.
- Resolution cvars: `r_Width`, `r_Height`, `r_CustomResWidth`,
  `r_CustomResHeight`, `r_CustomResMaxSize`, `r_VSync`, `r_MaxFrameLatency`,
  `r_FlushToGPU`, `r_MultiThreaded`.
- Upscalers ship as DLLs (DLSS, XeSS, FidelityFX DX12). TAA cvars under
  `r_AntialiasingTAA*`.
- Post: `r_MotionBlur*`, `r_ChromaticAberration`, `r_HDR*` family with
  Warhorse tonemapper cvars.

## 3. Stereo and VR

- No VR runtime. Zero matches for OpenVR, OpenXR, openvr_api, LibOVR,
  IHmdDevice, IHmdManager, sys_vr, hmd_tracking.
- CryEngine's stereo renderer is gone. Zero matches for `r_StereoMode`,
  `r_StereoOutput`, `r_StereoDevice`, `CD3DStereoRenderer`.
- Remnants of the old stereo framework remain: `r_StereoEyeDist`,
  `r_StereoScreenDist`, `r_StereoStrength`, `r_StereoFlipEyes`,
  `r_StereoNearGeoScale`, `r_StereoHudScreenDist`, `r_StereoGammaAdjustment`,
  `r_StereoEnableMgpu`, `g_stereoFrameworkEnable`,
  `sys_flash_stereo_maxparallax`, `r_VolumetricCloudsStereoReprojection`, and
  the flow nodes `Stereo:StereoParameters` and `Stereo:ReadStereoParameters`.
  These are parameters without a consumer. Do not expect them to produce a
  second eye.

### 3.1 Engine classes that survive (by RTTI name)

Present: `CD3D9Renderer` (the renderer class keeps its historic name on
top of CryDX12), `CCryDX12SwapChain`, `NCryDX12::CView`, `CRenderView`,
`CCamera`, `CView`, `CViewSystem`, `CMovieSystem`, `CAnimSequence`,
`CFlashUI`, `CFlashPlayer`, `CPoseModifierStack`, `CPoseModifierSetup`,
`CLookAtSimple`, `CSkeletonPose`, `CSkeletonAnim`, `CCharInstance`,
`CAttachmentManager`, `CActionMapManager`, `CInputDevice`,
`CXInputDevice`, `CKeyboard`, `CMouse`, `wh::C_Player` (51 RTTI entries),
`wh::C_Horse` (13), `wh::C_Human` (8).

Absent: `CPoseAlignerLimbIK`, `CAnimationPoseModifier`,
`CVrProjectionManager` (though `r_VrProjectionType` with the help text
"1: Nvidia Multi-Res Shading, 2: NVidia Lens-Matched Shading" and
`r_VrProjectionPreset [0..8]` are present, so the CE 5.4 VRWorks
projection path may exist as non-virtual code).

Other cvars confirmed present: `r_Driver`, `r_MultiGPU`,
`demo_use_hmd_rotation` ("Uses alternative entity and view rotation for
HMD Devices", a demo-playback flag, not a tracking path),
`g_detachCamera`, `cl_headBob`, `cl_headBobLimit`, `r_Supersampling`,
`r_SupersamplingFilter`, `r_DrawNearFoV`, `r_DrawNearZRange`,
`r_PostProcessHUD3D*`, `g_show_crosshair`, `mov_NoCutscenes`.

## 4. Camera control surface

Lua script binds in `CScriptBind_System`: `GetViewCameraPos`,
`GetViewCameraDir`, `GetViewCameraUpDir`, `GetViewCameraAngles`,
`GetViewCameraFov`, `SetViewCamera`, `SetViewCameraByAngles`,
`SetViewCameraFov`, `SetCVar`, `GetCVar`, `ExecuteCommand`, `ScanDirectory`.

Camera cvars: `cl_fov`, `cl_hfov`, `cl_fovBlendTime`, `cl_DefaultNearPlane`,
`cl_bob`, `cl_camera_noise`, `cl_cam_*` (third-person orbit and follow),
`cl_tpvFOVDamping`. Combat camera is native (`wh::combatmodule::
C_CombatCameraManager`). Player camera parameters live in
`Scripts/Entities/actor/Parameters/Player_Params.xml` (follow-camera table).

## 5. Input

CryInput device names found: keyboard, mouse, gamepad, DualShock, DualSense,
and the string `oculustouch` (an enum name only, with no device behind it).
Action maps are flow-node addressable (`CFlowNode_ActionMap`,
`CFlowNode_InputActionListener`, `CFlowNode_InputKey`,
`wh::game::CFlowNode_SimulateInput`). Input filtering class
`wh::playermodule::C_FilterInput` is RTTR-registered.

## 6. UI

Scaleform GFx 3.3.94 (`CFlashUI`, `CFlashPlayer`, `sys_flash*` cvars).
`hud_hide` exists. `CFlashUIWorldScreenPosNode` and
`CFlowRayToFlashSpace` exist, so the engine already maps world points to
flash space and rays into flash content. Flash elements render through the
CryDX12 layer like everything else.

## 7. Official mod system

- `Mods/<id>/mod.manifest`, `Mods/mod_order.txt`, `scripts/mods/<id>.lua`.
  ModId is lowercase letters and underscore only. The manifest carries
  `modifies_level` and version restrictions. `wh_mod_GenerateReport` and
  `wh_mod_ShowUsedModsInSaveTooltip` exist.
- Data is pak-overlaid (`Data/pak.cfg`). 290 Lua files in `Data/Scripts.pak`,
  loaded through `Scripts/main.lua` and `Scripts/Utils/ScriptLoader.lua`
  (`System.ScanDirectory` plus `Script.LoadScript`).
- No native plugin loading for mods. `LoadLibrary` is imported but used for
  NGX and engine modules, not for a mod DLL path.

## 8. This machine

| Item | Value |
|---|---|
| CPU/GPU | AMD Ryzen AI MAX+ 395, Radeon 8060S (Strix Halo, RADV, Vulkan 1.4) |
| RAM | 122 GB, unified with the iGPU |
| Display | Wayland |
| Proton | prefix at `compatdata/1771300`, Proton 11.0-100, game has been run |
| VR runtime | none. No SteamVR, Monado, WiVRn or ALVR. No headset on USB |
| Titanfall 2 | queued in Steam, 0 bytes downloaded |

The iGPU is the hard constraint. KCD2 flat already needs upscaling on a
Strix Halo at 1080p. VR doubles the rasterized views and wants 72 to 90 Hz
without reprojection. Any plan must budget for aggressive render scale,
foveation by lens, and the first milestone being a seated head-tracked
mode rather than full 6DOF.

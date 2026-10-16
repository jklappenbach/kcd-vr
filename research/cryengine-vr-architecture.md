# CRYENGINE 5.x VR architecture map, against KCD2's WHGame.dll

Source: a shallow clone of public CRYENGINE 5.x at
`/home/julian/code/kcd-vr/research/cryengine`. This is a monolithic,
pre-CryPlugins snapshot: there is no `Code/CryPlugins/` directory at all, and
VR device support is built directly into `CrySystem` and
`RenderDll/XRenderD3D9`, not loaded as a plugin. All citations below are
`file:line` against this checkout unless marked otherwise.

## 1. Stereo rendering

`CD3DStereoRenderer` (`Code/CryEngine/RenderDll/XRenderD3D9/D3DStereo.h:47`)
owns two intermediate eye textures, `CTexture::s_ptexStereoL/R`, allocated in
`CreateIntermediateBuffers` (`D3DStereo.cpp:285-311`). Per frame,
`CD3DStereoRenderer::Update()` (`D3DStereo.cpp:526-541`) reads
`CV_r_StereoMode`/`CV_r_StereoOutput` and queues `RC_PrepareStereo` on the
render thread, which calls `PrepareStereo()` (`D3DStereo.cpp:426-500`): this
pulls `CV_r_StereoStrength`, `CV_r_StereoEyeDist` into
`m_maxSeparationScene`, `CV_r_StereoScreenDist` into
`m_zeroParallaxPlaneDist`, `CV_r_StereoNearGeoScale`,
`CV_r_StereoGammaAdjustment` (lines 479-483), then calls
`m_pHmdRenderer->PrepareFrame()` if an HMD renderer exists.

Scene submission happens in `ProcessScene()` (`D3DStereo.cpp:609-714`). For
`STEREO_MODE_DUAL_RENDERING` it builds a left-eye camera and a right-eye
camera via `PrepareCamera(nEye, cam)` (`D3DStereo.cpp:543-607`), calls
`m_renderer.SetCamera(camL/camR)`, then `RenderScene()` for each eye
(`D3DStereo.cpp:663-698`) with `SHDF_STEREO_LEFT_EYE`/`_RIGHT_EYE` scene
flags, copying each result into the stereo texture via
`CopyToStereoFromMainThread`. `PrepareCamera` has two branches: an HMD branch
(`sys_vr_support`, `D3DStereo.cpp:560-582`) that asks
`IHmdManager::GetAsymmetricCameraSetupInfo` for per-eye FOV/asymmetry and
applies a half-IPD translation scaled by `CV_r_stereoScaleCoefficient`, and a
non-HMD "3D TV" branch (`D3DStereo.cpp:583-604`) that derives a symmetric
frustum shift from `CV_r_StereoEyeDist`/`CV_r_StereoScreenDist`. There is one
`CRenderView`/render path reused twice (once per eye), not two persistent
parallel views; `RC_SetStereoEye` (`D3DStereo.cpp:627`, `670`, `689`) tags
which eye is current.

`DisplayStereo()` (`D3DStereo.cpp:766-873`) composites the two eye textures
to the backbuffer for non-HMD outputs (side-by-side, checkerboard,
line-by-line, anaglyph via `SelectShaderTechnique`,
`D3DStereo.cpp:999-1038`), honoring `CV_r_StereoFlipEyes`
(`D3DStereo.cpp:854-855`). When an `IHmdRenderer` exists, `DisplayStereo`
returns immediately (`D3DStereo.cpp:775-778`) and the HMD renderer's
`SubmitFrame()`/`RenderSocialScreen()` are called instead from
`SubmitFrameToHMD()` (`D3DStereo.cpp:750-764`). `CV_r_StereoEnableMgpu`
selects AMD LiquidVR GPU masks around each eye's render (`D3DStereo.cpp:622,
639, 651, 665, 684, 700`); this is legacy multi-GPU TV-stereo code, unrelated
to modern VR.

HUD placement in stereo is handled separately by `C3DHud`
(`PostProcessHud3D.cpp`), covered in section 5.

The eye-to-HMD path: `CreateHmdRenderer` (`D3DStereo.cpp:1201-1233`)
dispatches on `IHmdDevice::GetClass()` to build a
`CD3DOculusRenderer`/`CD3DOpenVRRenderer`/`CD3DOsvrRenderer`. For OpenVR,
`CD3DOpenVRRenderer::Initialize()` (`D3DOpenVR.cpp:46-90`) grabs the raw
D3D11 device via `m_renderer->GetDevice_Unsynchronized().GetRealDevice()`
(line 48) and creates per-eye `CTexture` wrappers through
`WrapD3DRenderTarget` (declared `D3DOpenVR.h:41`). Each frame,
`PrepareFrame()` hands the eye textures to the stereo renderer
(`D3DOpenVR.cpp:235-240`) and `SubmitFrame()` calls
`m_openVRDevice->SubmitFrame()` (line 251), which is where OpenVR's
`IVRCompositor::Submit` would sit (not in this public tree; it lives in
`CryPlugins`/`OpenVRDevice` wrapper code, outside what ships here as source).
`CD3DOsvrRenderer` in `D3DOsvr.cpp` follows the same `IHmdRenderer` contract.
There is no `CryPlugins/VR/*` directory in this checkout; that reorganization
happened in a later CRYENGINE version than this snapshot.

## 2. HMD management

`IHmdManager` (`Code/CryEngine/CryCommon/CrySystem/VR/IHMDManager.h:7-47`)
is implemented by `CHmdManager`
(`Code/CryEngine/CrySystem/HMDManager.h:7`,
`HMDManager.cpp`). `SetupAction(eHmdSetupAction_Init)`
(`HMDManager.cpp:75-107`) checks `sys_vr_support` and, per `hmd_driver`
cvar, initializes exactly one of Oculus/OpenVR/Osvr via the `VR_INIT` macro
(lines 28-34), then calls
`gEnv->pRenderer->GetIStereoRenderer()->OnHmdDeviceChanged(m_pHmdDevice)`
(line 98). `UpdateTracking(EVRComponent)` (`HMDManager.cpp:126-132`) simply
forwards to `IHmdDevice::UpdateTrackingState`. `IsStereoSetupOk()`
(`HMDManager.cpp:135-157`) is the gate that requires
`STEREO_DEVICE_FRAMECOMP` plus `STEREO_OUTPUT_HMD` (or side-by-side) before
stereo is considered valid.

`IHmdDevice` (`IHMDDevice.h:198-250`) exposes
`GetAsymmetricCameraSetupInfo(eye, fov, aspectRatio, asymH, asymV, eyeDist)`
(line 222, per-eye projection), `UpdateTrackingState`/`RecenterPose` (lines
224-226, the recenter entry point backing `hmd_recenter_pose`,
`HMDCVars.h:101`), `GetLocalTrackingState()` (line 232, pose in CRYENGINE's
coordinate space) vs `GetNativeTrackingState()` (line 229, the HMD SDK's raw
space), and an async-camera path: `SetAsynCameraCallback`/
`RequestAsyncCameraUpdate` (lines 245-247) used by the render thread to pull
a fresher pose than the game-thread camera (see section 3).
`hmd_post_inject_camera` (`HMDCVars.h:95-99`) is the cvar that turns this
late-injection path on. Prediction itself (motion-to-photon compensation) is
internal to each vendor device implementation, not visible in this public
tree (the Oculus/OpenVR/Osvr `Device.cpp` files wrap closed SDK calls).

The pose is applied to the camera in two places: the game-thread path in
`CView::Update` (section 3) and the render-thread async path in
`CD3DStereoRenderer::TryInjectHmdCameraAsync` (`D3DStereo.cpp:1236-1270`),
which calls `IHmdDevice::RequestAsyncCameraUpdate` once per frame (only on
the first left-eye pass, line 1250) and re-derives the per-eye camera via the
same `PrepareCamera` used in `ProcessScene`.

## 3. Camera path

`CView` (`Code/CryEngine/CryAction/ViewSystem/View.h:27`) implements
`IHmdDevice::IAsyncCameraCallback`. `CViewSystem::Update(frameTime)`
(`ViewSystem.cpp:89-237`) iterates all views, calls `pView->Update(...)` for
each (line 107), then for the active view takes `CCamera& rCamera =
pView->GetCamera()` (line 111) and calls `m_pSystem->SetViewCamera(rCamera)`
(line 159, a plain `m_ViewCamera = Camera` copy at
`CrySystem/System.h:406`).

Inside `CView::Update` (`View.cpp:68-264`), the view computes
`m_viewParams.position/rotation` from the linked entity, applies FOV
override from `IHmdDevice::GetCameraSetupInfo` when
`IHmdManager::IsStereoSetupOk()` (`View.cpp:147-156`), then at lines 204-248
adds the HMD pose on top: if a timedemo is playing it uses recorded
`hmdPositionOffset`/`hmdViewRotation` (lines 217-224), otherwise, if HMD
tracking is usable, it registers itself as the async callback
(`pHmdDevice->SetAsynCameraCallback(this)`, line 227) and composes
`sensorState.pose.position/orientation` onto the base pose (lines 241-244).
The final `Matrix34 viewMtx` is written with `m_camera.SetMatrix(viewMtx)`
(line 248). `CView::OnAsyncCameraCallback` (`View.cpp:267-299`) is the
render-thread-callable twin of this same composition, used by
`TryInjectHmdCameraAsync` above for lower-latency pose injection.

`CCamera::SetMatrix` (`Cry_Camera.h:86`) is `inline`: it stores the matrix
and immediately calls `UpdateFrustum()` (`Cry_Camera.h:623-688`), which
recomputes the camera-space frustum edges and all six `m_fp[]` world-space
frustum planes used by `IsAABBVisible_F/E`, `IsSphereVisible_F`, and friends
(declared `Cry_Camera.h:146-172`), consumed throughout `Cry3DEngine` culling
(`VisAreas.cpp`, `3DEngineRender.cpp`, `ObjectsTree_Jobs.cpp`,
`terrain_sector_render.cpp`, etc., all via `GetFrustumPlane`). This single
function is the choke point: nothing about rendering or culling is driven
from the raw `m_Matrix`; everything downstream reads the plane/edge cache
that `UpdateFrustum` rebuilds synchronously on every `SetMatrix`/
`SetPosition` call. The public source names this method `UpdateFrustum`
(singular), not `UpdateFrustumPlanes` as the KCD2 modding community names
the hooked symbol; Warhorse's fork likely renamed or re-split it, but the
mechanism the community is hooking is this exact function (or `SetMatrix`
itself, which calls nothing else). Hooking it after `CView::Update` has
written the final per-frame matrix, and before `SetViewCamera`/`RenderWorld`
consume it, is the one point that moves both the renderer's view (via
`m_pSystem->GetViewCamera()`, read again at `3dEngine.cpp:4965` to build the
`SRenderingPassInfo` passed into `RenderWorld`,
`CryCommon/Cry3DEngine/I3DEngine.h:1388`) and the culling frustum together,
because both are reads of the same `CCamera` object that `UpdateFrustum`
just rebuilt. The "guarded by the CView vtable" detail matches: `CView`
overrides nothing camera-matrix-related except through this shared
`CCamera` member, so a vtable hook on `CCamera`'s frustum update, scoped to
calls arriving through a live `CView`, reliably captures only real
per-frame camera updates and not every incidental `CCamera` temporary
elsewhere in the engine.

`I3DEngine::RenderWorld` (`I3DEngine.h:1388`,
`Cry3DEngine/3DEngineRender.cpp:820`) takes the `SRenderingPassInfo` built
from this camera and performs the actual draw/cull dispatch; its own camera
accessor is `passInfo.GetCamera()` (`3DEngineRender.cpp:861`), itself fed
from the same `CCamera`.

## 4. DX12 path

`DX12/` under `RenderDll/XRenderD3D9/DX12/` is a D3D11-API shim over a real
D3D12 device. `CCryDX12Device : CCryDX12Object<ID3D11Device1>`
(`DX12/Device/CCryDX12Device.hpp:19`) is what the D3D11-shaped renderer
holds as its "device"; `GetD3D12Device()` (line 30) returns the real
`ID3D12Device*`. `CCryDX12DeviceContext : CCryDX12Object<ID3D11DeviceContext1>`
(`CCryDX12DeviceContext.hpp:24`) exposes `GetCoreGraphicsCommandListPool()`
(line 105) and `GetCoreGraphicsCommandList()` (line 110), the real D3D12
command-list/queue machinery. `CCryDX12SwapChain : CCryDX12GIObject<IDXGISwapChain3>`
(`GI/CCryDX12SwapChain.hpp:21`) matches the RTTI string
`CCryDX12GIObject<IEmptySwapChain3>` found in WHGame.dll (same template
family, confirming KCD2 still rides this shim); its `GetBuffer()`
(`CCryDX12SwapChain.cpp:67`) is where backbuffer textures come from, and
`GetDX12SwapChain()`/`GetDXGISwapChain()` (lines 35, 39) expose both views.

Every wrapped resource is a `CCryDX12Resource<T> : CCryDX12DeviceChild<T>,
ICryDX12Resource` (`CCryDX12Resource.hpp:67`); `ICryDX12Resource::
GetD3D12Resource()` (line 56) is the escape hatch that returns the real
`ID3D12Resource*` for any texture, including the stereo eye targets
(`CTexture::s_ptexStereoL/R`, `D3DStereo.cpp:301-302`), which at the D3D11
API level are `CCryDX12Texture2D` instances built on this same base. In
principle, getting a per-eye texture to an OpenXR `XR_KHR_D3D12_enable`
swapchain means: resolve the eye `CTexture` to its underlying
`ID3D11Resource*`, cast to `ICryDX12Resource`, call `GetD3D12Resource()` for
the `ID3D12Resource*`, and pull the device and command queue off
`CCryDX12Device`/`CCryDX12DeviceContext`. Those three objects are exactly
what `xrCreateSwapchain`+`XrGraphicsBindingD3D12KHR` need. This is a
structural argument from the shim's shape, not a measurement against
WHGame.dll: KCD2's actual per-eye texture names and whether Warhorse's
renderer still allocates two side-by-side eye RTs at all (findings show no
stereo-renderer cvars survive) are unconfirmed.

## 5. Flash/Scaleform UI and world-space redirection

HUD elements are Scaleform `IFlashPlayer` instances owned by `CFlashUI`
(`CryAction/FlashUI/FlashUI.cpp/.h`) and individual `FlashUIElement`
objects. The existing render-to-texture mechanism already used elsewhere in
the engine (in-world screens) is `CFlashTextureSourceBase`
(`RenderDll/Common/Textures/Texture.cpp`): a player instance is created
(`CFlashPlayerInstanceWrapper::CreateInstance`, lines 3308-3322) and linked
to a dynamic texture via `pPlayer->LinkDynTextureSource(pSrc)` (line 3319),
where `pSrc` is an `IDynTextureSource` created through
`IRenderer::EF_LoadDynTexture` (`CryRenderer/IRenderer.h:2102`).
`IFlashPlayer::SetViewport` (`IFlashPlayer.h:92`) and `Render(bool
stereo=false)` (`IFlashPlayer.h:101`) are the two calls that drive content
into that texture instead of the backbuffer.

CRYENGINE's own 3D-HUD path, `C3DHud` (`PostProcessHud3D.cpp`), already does
almost exactly what a VR head-locked HUD needs: `FlashUpdateRT()`
(lines 421-495) pushes `CTexture::s_ptexCached3DHud` as a render target
(line 436, 470), advances and renders the HUD's `IFlashPlayer` into it
(lines 487-495), and `C3DHud::Render()` (lines 782-830) then draws that
texture on a quad using `CalculateProjMatrix()` (lines 182-216), which
builds a perspective projection shifted by
`CV_r_StereoHudScreenDist` (line 215) and, in stereo, a per-eye parallax
offset (lines 809-822: `BeginRenderingTo(LEFT_EYE)`/`(RIGHT_EYE)` with
`m_maxParallax` flipped sign). A VR mod can either repurpose this path
directly (it already supports stereo, per-eye projection, and a fixed screen
distance) or intercept earlier: redirect any `FlashUIElement`'s
`IFlashPlayer` into its own `IDynTextureSource`-backed RT the same way, then
draw that RT on a hand-authored world-space or head-locked quad using the
HMD pose instead of `C3DHud`'s fixed screen-distance math. `CFlashUIWorldScreenPosNode`
(`CryAction/FlashUI/FlashUIDisplayNodes.h:122`) already projects a world
position into flash UI space using `GetISystem()->GetViewCamera()`
(`FlashUIDisplayNodes.cpp:303`) and `IUIElement::WorldToFlash` (line 318),
which is useful for placing UI elements relative to world anchors but
operates in the opposite direction from rendering flash onto a world quad.

## 6. Animation and IK

`IAnimationPoseModifier` (`CryCommon/CryAnimation/IAnimationPoseModifier.h`)
is the base interface; `SAnimationPoseModifierParams` (lines 28-38) exposes
`pCharacterInstance`, `pPoseData`, `timeDelta`, and accessors for
`ISkeletonPose`/`ISkeletonAnim`/`IDefaultSkeleton`. Limb IK is implemented
as `CLimbIk : IAnimationPoseModifier` (`CryAnimation/PoseModifier/LimbIk.h:8`,
registered as `"AnimationPoseModifier_LimbIk"`, line 15), which holds up to
16 `{LimbIKDefinitionHandle, Vec3 targetPositionLocal}` setups added via
`AddSetup()` (line 32); this checkout has no class named
`CPoseAlignerLimbIK` (that name does not exist here; `PoseAlignerChain.cpp/.h`
is a separate feet/pelvis chain aligner). The scripting-level entry point
for limb IK is `ISkeletonPose::SetHumanLimbIK(const Vec3& wgoal, const
char* limb)` (`CryCommon/CryAnimation/ICryAnimation.h:799`). Look IK is
`CPoseBlenderLook : IAnimationPoseBlenderDir`
(`CryAnimation/PoseModifier/PoseBlenderLook.h:11`, registered
`"AnimationPoseModifier_PoseBlenderLook"`), with `SetTarget(const Vec3&)`
(line ~24) callable every frame; aim IK (`PoseBlenderAim.h`) is the
structural sibling for weapon aim. `IAnimationOperatorQueue`
(`IAnimationPoseModifier.h:199`) is a lighter per-joint override queue.

Attachments go through `ICharacterInstance::GetIAttachmentManager()`
(`ICryAnimation.h:462-463`), with `IAttachmentManager::CreateAttachment`,
`GetInterfaceByName`, `GetInterfaceByIndex`, `GetInterfaceByNameCRC`
(`CryCommon/CryAnimation/IAttachment.h:266-275`) and
`IAttachment::GetIAttachmentObject()`/`GetIAttachmentSkin()` (lines
382-383). First-person weapons/arms are attachment objects positioned
relative to a skeleton joint; a VR mod drives them by repositioning the
attachment's joint offset for hand placement and separately calling
`SetHumanLimbIK` (or pushing a `CLimbIk` setup) with the controller's
world-space goal to bend the arm toward that placement, once per hand per
frame. No `ICryMannequin*` header exists under `CryCommon` in this
checkout, so Mannequin's exact relationship to `ICharacterInstance` is not
independently confirmed from source here; it appears (unverified) to be a
fragment-selection/blending layer that drives `ISkeletonAnim` playback
alongside this pose-modifier stack, not a replacement for it.

## 7. Input

Device types: `EInputDeviceType` (`CryCommon/CryInput/IInput.h:58-65`) with
`eIDT_Keyboard`, `eIDT_Mouse`, `eIDT_Joystick`, `eIDT_Gamepad`,
`eIDT_MotionController`. Synthetic injection is
`IInput::PostInputEvent(const SInputEvent&, bool bForce=false)`
(`IInput.h:1192`). Oculus Touch is modeled separately from the device-type
enum, as `COculusController : IHmdController`
(`CrySystem/Oculus/OculusTouchController.h:17`), reached via
`IHmdDevice::GetController()` (`IHMDDevice.h:234`); it exposes per-controller
button/trigger/thumbstick state (`IsButtonPressed`, `GetTriggerValue`,
`GetThumbStickValue`, `IHMDDevice.h:169-173`) and
`GetNativeTrackingState`/`GetLocalTrackingState` per `EHmdController` id
(lines 176-179) - a pose-and-buttons API distinct from, and not routed
through, `IInput`/`PostInputEvent`.

`IActionMapManager` (`CryAction/IActionMapManager.h`) supports filtering via
`IActionFilter::Enable/Enabled` (lines 281-282, 304-305),
`CreateActionFilter` (373), `EnableActionMap(name, enable)` (381), and
`EnableFilter(name, enable)`/`IsFilterEnabled` (382-383), implemented
concretely by `CActionFilter` (`CryAction/ActionFilter.h:27`). A VR mod
would filter out mouselook/gamepad-look action maps once head tracking owns
view direction, and inject synthetic `SInputEvent`s for controller buttons
mapped onto existing action names via `PostInputEvent`.

## 8. Movie/cutscene system

`IMovieSystem` (`CryCommon/CryMovie/IMovieSystem.h`), `CAnimSequence`
(`CryMovie/AnimSequence.cpp/.h`), and the camera node `CAnimCameraNode`
(`CryMovie/AnimCameraNode.h:17`, `Animate()` at `AnimCameraNode.cpp:106`,
FOV handling around lines 126-234) animate a camera entity's transform and
FOV but do not themselves call `IViewSystem`; the one direct call found is
`CMovieSystem::AbortSequence` (`Movie.cpp:744-766`) resetting view blending
on abort (line 755, 759-ish). The actual view handoff is a flow-graph
responsibility: `CFlowNode_CameraView` ("Camera:View",
`CryAction/FlowSystem/Nodes/FlowCameraNodes.cpp:537-684`, registered line
746-750 alongside `Camera:Camera`, `Camera:ViewShake(Ex)`,
`Camera:GetTransform`) calls `m_pView->LinkTo(pActInfo->pEntity)` then
`m_pViewSystem->SetActiveView(m_pView)` on Enable, and restores
`SetActiveView(m_pLocalPlayerView)` on Disable. No `CameraActivateEntityView`
node exists anywhere in this public checkout; it is either Warhorse/GameSDK
code layered on top, or a renamed variant of `Camera:View`.

## 9. Mods

There is no `Code/CryEngine/CrySystem/Mods/` directory and no
`ModManager.cpp`/`.h` anywhere in this public checkout. The path embedded in
WHGame.dll (`code\CryEngine\CrySystem\Mods\ModManager.cpp`, per
kcd2-local-findings.md) is Warhorse-proprietary code with no stock-CryEngine
baseline to check against here. Stock `CryCommon` has no mod-plugin
registration interface at all (no `IMod`), which is consistent with, but
does not independently prove, the binary's observed lack of a native mod-DLL
loading path.

## What KCD2 kept

1. **Stereo rendering**: likely stripped. `CD3DStereoRenderer`/
   `r_StereoMode`/`r_StereoOutput`/`r_StereoDevice`/`r_D3D12*`-only renderer
   do not coexist with the public source's D3D11-first assumption (`m_mode
   != STEREO_MODE_NO_STEREO`, side-by-side compositing) unless rewritten for
   DX12-only output; the absence of `r_StereoMode`/`r_StereoOutput`/
   `r_StereoDevice`/`CD3DStereoRenderer` strings in the findings, alongside
   survival of the *parameter* cvars `r_StereoEyeDist`, `r_StereoScreenDist`,
   `r_StereoStrength`, `r_StereoFlipEyes`, `r_StereoNearGeoScale`,
   `r_StereoHudScreenDist`, `r_StereoGammaAdjustment`, `r_StereoEnableMgpu`,
   and the flow nodes `Stereo:StereoParameters`/`Stereo:ReadStereoParameters`,
   matches this report's model exactly: these are all *consumers* of
   `CD3DStereoRenderer` state (read in `PrepareStereo`, `D3DStereo.cpp:
   479-483`, or by flow nodes), not the renderer class or its mode-selection
   cvars. Warhorse appears to have deleted the `CD3DStereoRenderer` class and
   its three selector cvars while leaving the downstream parameter cvars and
   flow nodes orphaned, exactly as kcd2-local-findings.md states ("parameters
   without a consumer").
2. **HMD management**: stripped. Zero `IHmdDevice`/`IHmdManager`/
   OpenVR/OSVR/Oculus strings means `CHmdManager`, `IHmdDevice`, and the
   `CryVR::*::Device`/`Resources` classes this report describes are gone.
   `sys_vr_support` and `hmd_driver`/`hmd_*` cvars are also absent from the
   findings' string list, consistent with full removal rather than a
   disabled-but-present subsystem.
3. **Camera path**: survives, and is the correct hook point. `CCamera`,
   `CRenderView`, and the Lua binds `GetViewCameraPos/Dir/UpDir/Angles/Fov`,
   `SetViewCamera`, `SetViewCameraByAngles`, `SetViewCameraFov` (all RTTI- or
   string-confirmed in the findings) show the `ISystem::SetViewCamera` /
   `CCamera` plumbing this report traces is intact end to end. The
   community's `CCamera::UpdateFrustumPlanes` hook, guarded by a live `CView`
   vtable, lands on exactly the choke point identified in section 3: one
   function that both the renderer and the culling system read from
   afterward, reached only when a real per-frame camera update (not an
   incidental temporary `CCamera`) is in flight.
4. **DX12 path**: survives and is the only path. `r_D3D12*` cvars (async
   compute, waitable swap chain, DRED, PIX) and `CCryDX12GIObject<...>` RTTI
   confirm the same `CryDX12` shim this report describes is what KCD2 ships;
   there is no D3D11 fallback and no Vulkan loader. The
   `ICryDX12Resource::GetD3D12Resource()` escape hatch this report proposes
   for OpenXR interop should still exist at the vtable level, though its
   exact call sites (whether KCD2 still allocates discrete eye RTs at all)
   are unconfirmed without further binary analysis.
5. **Flash/Scaleform HUD**: survives. `CFlashUI`, `CFlashPlayer`,
   `sys_flash*` cvars, `hud_hide`, `CFlashUIWorldScreenPosNode`, and
   `CFlowRayToFlashSpace` are all present per the findings, matching this
   report's section 5 classes exactly. This is good news for a VR HUD mod:
   the render-to-texture primitives (`IDynTextureSource`,
   `LinkDynTextureSource`) this report identifies as the interception point
   are very likely intact, since the UI-to-world-space node that depends on
   the same camera/UI-space plumbing survives.
6. **Animation/IK**: unknown, but probably intact in some form. No
   animation- or IK-specific strings are listed in kcd2-local-findings.md
   either way; `ICharacterInstance`/`ISkeletonPose` are core gameplay
   infrastructure (melee combat, mounted animation, first-person arms are
   visibly in the shipped game), so `SetHumanLimbIK` or an equivalent is
   almost certainly still present, but this cannot be confirmed from the
   string list alone and needs a targeted RTTI/string search for
   `CLimbIk`/`CPoseBlenderLook`/`IAttachmentManager` symbol names.
7. **Input**: partially survives. The findings confirm keyboard, mouse,
   gamepad, DualShock, DualSense device names and the string
   `oculustouch`, but call it "an enum name only, with no device behind
   it" - matching this report's finding that `COculusController` is reached
   only through `IHmdDevice::GetController()`, which requires the (stripped)
   HMD manager to exist. The enum value for Oculus Touch survived a
   deduplication pass that otherwise removed the entire VR device layer.
   `wh::playermodule::C_FilterInput` being RTTR-registered matches this
   report's `IActionFilter`/`CActionFilter` filtering mechanism, re-homed
   under Warhorse's own namespace.
8. **Movie/cutscene**: likely intact. `CFlowNode_Camera*` is confirmed
   present by RTTI in the findings, matching `CFlowNode_Camera`,
   `CFlowNode_CameraView`, and siblings in `FlowCameraNodes.cpp` exactly;
   `C_CombatCameraManager` (native, Warhorse-specific) sits alongside it.
   This is encouraging for a seated-camera VR mode: the engine-level
   camera-handoff mechanism this report traces (`SetActiveView`) is present
   and exercised constantly by both cutscenes and the native combat camera.
9. **Mods**: survives, as documented by the findings, but as a
   Warhorse-proprietary system this report cannot check against public
   source. The findings' description (`mod.manifest`, Lua scripts, pak
   overlay, no native-code loading path for mods) is consistent with how
   stock CryEngine's data/script layering normally works, but the actual
   `ModManager.cpp` logic is invisible to this checkout. The practical
   consequence for this project stands regardless: a VR conversion cannot
   ship as a "mod" in Warhorse's sense and must work by patching or hooking
   WHGame.dll directly, which is what sections 1-6 above are oriented
   toward.

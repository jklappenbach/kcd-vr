# Unreal Engine XR stack as a reference for a flat-to-VR conversion

Source tree: `/home/julian/code/unreal/UnrealEngine`, UE 5.6 (Oculus fork,
branch `oculus-5.6.0-release-1.113.0-v81.0`). In this tree the "default XR"
implementation classes (FDefaultXRCamera, FXRRenderBridge, FXRSwapChain,
FDefaultSpectatorScreenController, FDefaultStereoLayers, XRTrackingSystemBase)
live in the **XRBase plugin** (`Engine/Plugins/Runtime/XRBase/Source/XRBase`),
not in the `HeadMountedDisplay` runtime module. The `HeadMountedDisplay` module
now holds only the pure interfaces. All paths below are relative to the
UnrealEngine root.

## 1. The interfaces

- **IXRTrackingSystem** (`Engine/Source/Runtime/HeadMountedDisplay/Public/IXRTrackingSystem.h`)
  is the root device abstraction. It owns pose queries (`GetCurrentPose`
  line 149, `GetRelativeEyePose` line 161), tracking-origin management
  (`SetTrackingOrigin`/`GetTrackingOrigin`, lines 190-195), the
  per-frame hooks `OnStartGameFrame` (line 405) and
  `OnBeginRendering_GameThread`/`OnBeginRendering_RenderThread`
  (lines 420-431), and factory access to the camera (`GetXRCamera`,
  line 319), the HMD device (`GetHMDDevice`, line 326) and the stereo
  rendering device.
- **IHeadMountedDisplay** (`Engine/Source/Runtime/HeadMountedDisplay/Public/IHeadMountedDisplay.h`)
  covers the device itself: connection/enable state (lines 46-61), FOV
  (line 98), IPD (lines 123-124), distortion/chromatic aberration
  (lines 171-192), hidden/visible area mesh (`HasHiddenAreaMesh` line
  197, `DrawHiddenAreaMesh` lines 208-215), pixel density
  (`GetPixelDenity`/`SetPixelDensity`, lines 151-156), and the
  spectator screen controller accessor (line 138).
- **IStereoRendering** (`Engine/Source/Runtime/Engine/Public/StereoRendering.h`)
  is the stereo-pass abstraction: `EStereoscopicPass`/`EStereoscopicEye`
  enums (lines 20-37), `GetDesiredNumberOfViews` (line 65),
  `GetViewPassForIndex` (line 70), `AdjustViewRect` (line 138),
  `CalculateStereoViewOffset` (line 155), `GetStereoProjectionMatrix`
  (line 169), and the layer/render-target-manager accessors (lines
  185-190).
- **IXRCamera** (`Engine/Source/Runtime/HeadMountedDisplay/Public/IXRCamera.h`)
  is the bridge between tracked pose and the gameplay camera:
  `UpdatePlayerCamera` (line 46), `OverrideFOV` (line 51),
  `SetupLateUpdate` (line 54), `CalculateStereoCameraOffset` (line
  60). **FDefaultXRCamera**
  (`Engine/Plugins/Runtime/XRBase/Source/XRBase/Public/DefaultXRCamera.h`)
  is the concrete default: it is also an `FHMDSceneViewExtension`
  (line 14), so it participates in the scene-view-extension pipeline
  directly.
- **IStereoLayers** (`Engine/Source/Runtime/Engine/Public/IStereoLayers.h`)
  manages compositor-submitted quad/cylinder/cubemap layers: layer type
  (`ELayerType` WorldLocked/TrackerLocked/FaceLocked, lines 60-65),
  flags (lines 67-85) and `FLayerDesc` (transform, quad size, UV rect,
  priority, lines 91-150). `FDefaultStereoLayers`
  (`Engine/Plugins/Runtime/XRBase/Source/XRBase/Public/DefaultStereoLayers.h`)
  is the shared CPU-side layer manager most platforms build on.
- **FXRRenderBridge** (`Engine/Plugins/Runtime/XRBase/Source/XRBase/Public/XRRenderBridge.h`)
  is an `FRHICustomPresent` (line 9) that hooks the RHI's present path
  so the compositor, not the OS swap chain, controls frame delivery.
- **FXRSwapChain** (`Engine/Plugins/Runtime/XRBase/Source/XRBase/Public/XRSwapChain.h`)
  wraps a runtime-owned ring of textures (lines 20-51):
  `IncrementSwapChainIndex_RHIThread`, `WaitCurrentImage_RHIThread`,
  `ReleaseCurrentImage_RHIThread`.

## 2. Per-frame sequence

**Game thread, pose read.** `UWorld::Tick` calls
`GEngine->XRSystem->OnStartGameFrame(WorldContext)`
(`Engine/Source/Runtime/Engine/Private/LevelTick.cpp:1304`). For OpenXR this
is `FOpenXRHMD::OnStartGameFrame`
(`Engine/Plugins/Runtime/OpenXR/Source/OpenXRHMD/Private/OpenXRHMD.cpp:3908`),
which drains the XR event queue, drives the session-state machine, and
refreshes the tracking-to-world transform (line 3942).

**Camera.** `UCameraComponent::GetCameraView`
(`Engine/Source/Runtime/Engine/Private/Camera/CameraComponent.cpp:416`) checks
`IsXRHeadTrackedCamera()` (line 366) and if true calls `HandleXRCamera`
(line 385), which calls `XRCamera->SetupLateUpdate(..., bLockToHmd == 0)`
(line 397) and, only `if (bLockToHmd)` (line 399), calls
`XRCamera->UpdatePlayerCamera` and overwrites the component's relative
transform. `APlayerCameraManager::UpdateViewTarget`
(`Engine/Source/Runtime/Engine/Private/PlayerCameraManager.cpp:378-381`)
reaches this through `ACameraActor::GetCameraComponent()->GetCameraView`,
so the same path runs whether the active camera came from gameplay or from a
Sequencer camera cut.

**Views per eye.** `FSceneView`'s constructor
(`Engine/Source/Runtime/Engine/Private/SceneView.cpp:917-972`) resolves
`bIsInstancedStereoEnabled`/`bIsMobileMultiViewEnabled`
(lines 965-967) from stereo-rendering aspects, folding both into
`bIsSinglePassStereo` (line 968). The view's `StereoPass`/`StereoViewIndex`
drive `IStereoRendering::IsStereoEyeView` (`StereoRendering.cpp`, decl at
`StereoRendering.h:91`) and `IsAPrimaryView`/`IsASecondaryView`, which gate
instanced-stereo view sharing (`SceneView.cpp:2499-2501`).

**Projection.** `FOpenXRHMD::GetStereoProjectionMatrix`
(`Engine/Plugins/Runtime/OpenXR/Source/OpenXRHMD/Private/OpenXRHMD.cpp:1406`)
builds an off-center (asymmetric) projection straight from the runtime's
per-eye `fov.angleLeft/angleRight/angleUp/angleDown` (lines 1420-1438), as
returned by `xrLocateViews` (lines 3725, 3737). There is no assumption of a
symmetric FOV anywhere in this path.

**Late update, render thread.** `FDefaultXRCamera::PreRenderView_RenderThread`
(`Engine/Plugins/Runtime/XRBase/Source/XRBase/Private/DefaultXRCamera.cpp:188`)
re-reads the current pose and reapplies the delta orientation/position onto
`View.ViewRotation`/`View.ViewLocation` before `View.UpdateViewMatrix()`
(line 232), separately from whatever pose was used on the game thread, which
is what hides motion-to-photon latency.
`FDefaultXRCamera::PreRenderViewFamily_RenderThread` (line 249) additionally
applies `FLateUpdateManager::Apply_RenderThread` (line 289) to late-update
scene primitives (motion controllers, attachments) that were simulated off
a stale pose. Both are `ISceneViewExtension`/`FSceneViewExtensionBase`
overrides, invoked by the renderer's view-extension dispatch.

**Submission (OpenXRHMD).** `xrWaitFrame` is called at
`OpenXRHMD.cpp:3862`, gated by a wait-count/begin-count bookkeeping scheme
described in the comments at lines 3866-3868 and 4183-4192 to avoid
duplicate `xrBeginFrame` calls. `xrBeginFrame` runs in
`OnBeginRendering_RHIThread` at line 4220; `xrEndFrame` at line 4424. Eye
textures are produced by `FXRSwapChain` subclasses and layers are populated
in `SetupFrameLayers_RenderThread` (called from `OnBeginRendering_RenderThread`,
line 3609) before being attached to `XrCompositionLayerProjectionView`
entries (lines 3626-3633).

## 3. The OpenXR plugin

Session lifecycle: `xrCreateSession` (`OpenXRHMD.cpp:2258`),
`xrBeginSession`/`xrEndSession` driven off `XR_SESSION_STATE_READY`/
`XR_SESSION_STATE_STOPPING` events (lines 2705, 2717, 3967-3994),
`xrDestroySession` (line 2496). View configuration and reference spaces:
`GetPlayAreaBounds`/`GetTrackingOriginTransform` map
`EHMDTrackingOrigin::Local/LocalFloor/Stage/CustomOpenXR` onto
`XR_REFERENCE_SPACE_TYPE_*` (lines 548-663), and `SetTrackingOrigin`
(line 1082) rejects `View` space as an origin. Action sets and interaction
profiles live in `Engine/Plugins/Runtime/OpenXR/Source/OpenXRInput/Private/OpenXRInput.cpp`:
`XrActionSetCreateInfo` (lines 163-164), built-in profile table for Simple/
Vive/MixedReality/OculusGo/OculusTouch/ValveIndex (lines 388-393),
`xrSuggestInteractionProfileBindings` submission (around line 544). Hand
tracking is its own plugin, `Engine/Plugins/Runtime/OpenXRHandTracking`, not
part of OpenXRHMD core. D3D12 and Vulkan bridges are
`FD3D12RenderBridge`/`FVulkanRenderBridge`
(`Engine/Plugins/Runtime/OpenXR/Source/OpenXRHMD/Private/OpenXRHMD_RenderBridge.cpp:83,252`,
factories at lines 130 and 326), subclassing the shared
`FOpenXRRenderBridge`. Render scale: `GeneratePixelDensitySize`
(`OpenXRHMD.cpp:1243`) multiplies the runtime-recommended view size by a
pixel density clamped against `RuntimePixelDensityMax`
(lines 2220-2221), set through `SetPixelDensity` (line 4486) which is the
XR-aware counterpart to `vr.PixelDensity`/`r.ScreenPercentage`. Hidden-area
mesh comes from `xrGetVisibilityMaskKHR`
(`BuildOcclusionMesh`, lines 2017-2068), building per-view vertex/index
buffers that `DrawHiddenAreaMesh` (`IHeadMountedDisplay.h:208`) consumes to
skip shading pixels the lens distortion will crop.

## 4. Spectator screen / mirror window

`ISpectatorScreenController` (`Engine/Source/Runtime/HeadMountedDisplay/Public/ISpectatorScreenController.h:20-61`)
exposes mode selection (`SetSpectatorScreenMode`), a dedicated texture
override, and a two-region "texture plus eye" layout. The default
implementation, `FDefaultSpectatorScreenController`
(`Engine/Plugins/Runtime/XRBase/Source/XRBase/Public/DefaultSpectatorScreenController.h:19-39`),
composites eye and UI content onto the desktop window at render time; a
platform HMD obtains one via `IHeadMountedDisplay::GetSpectatorScreenController`
(`IHeadMountedDisplay.h:138`). The mirror window itself is just the normal
desktop viewport, with stereo rendering redirected to the HMD swap chain
instead.

## 5. UI in VR

`IStereoLayers` (`IStereoLayers.h`) is the compositor-side path: layers are
`WorldLocked`, `TrackerLocked`, or `FaceLocked` (lines 62-65) and are
drawn by the runtime's own compositor, immune to render-thread hitches.
`StereoLayerManager.h` (`Engine/Plugins/Runtime/XRBase/Source/XRBase/Public/StereoLayerManager.h`)
is the generic CPU-side tracker most platforms reuse to turn `FLayerDesc`
updates into per-frame layer submissions. The alternative, used far more in
practice for interactive Slate/UMG, is a `UWidgetComponent` set to World
Space geometry mode, rendered into the normal 3D scene like any mesh, so it
gets full depth-sorting and input but not the compositor's independent
resampling. For the HUD specifically, `FDefaultXRCamera::SetupViewFamily`
(`DefaultXRCamera.cpp:295-307`) is the relevant hook: it reads
`HMD->GetHMDDistortionEnabled` to toggle `EngineShowFlags.HMDDistortion`
and forces `EngineShowFlags.Rendering` off when `HMD->IsRenderingPaused()`
is true; Unreal otherwise leaves the legacy `AHUD` Canvas pass enabled, so a
2D HUD under a HMD still projects into the stereo views unless the game
explicitly suppresses it or moves it to a stereo layer / world widget.

## 6. Motion controllers

`UMotionControllerComponent`
(`Engine/Source/Runtime/HeadMountedDisplay/Public/MotionControllerComponent.h`)
exposes `MotionSource` (line 28, left/right grip or other OpenXR-named
poses) and `PlayerIndex` (line 24), and owns its own
`FLateUpdateManager LateUpdate` (line 163) so controller meshes/attachments
get the same render-thread late pose correction the HMD camera gets.
`IMotionController` (`Engine/Source/Runtime/HeadMountedDisplay/Public/IMotionController.h`)
is the polled-pose abstraction: `GetControllerOrientationAndPosition`
(line 76), the velocity/acceleration-returning overload (line 95), a
time-stamped variant for extrapolation (`GetControllerOrientationAndPositionForTime`,
line 117), tracking status (line 132), and `GetHandJointPosition` (line 152)
for hand-tracking fallthrough. Haptics are driven through
`xrApplyHapticFeedback`
(`Engine/Plugins/Runtime/OpenXR/Source/OpenXRInput/Private/OpenXRInput.cpp:1523`),
reachable from gameplay via `PlayerController.cpp`'s `PlayHapticEffect` path
and the generic `IHapticDevice` interface
(`Engine/Source/Runtime/InputDevice/Public/IHapticDevice.h`).

## 7. Comfort and locomotion

Engine-provided comfort primitives are thin: camera smoothing for cinematic
capture only (`xr.CinematicCameraSmoothing` and its roll/pitch/yaw decay
CVars, `DefaultXRCamera.cpp:15-37`, explicitly flagged "likely to be
disorienting in normal play"), and tracking-origin/floor alignment
(`GetFloorToEyeTrackingTransform`, `IXRTrackingSystem.h:218`). There is no
built-in vignette, snap-turn, teleport-locomotion, or redirected-walking
system; those, along with comfort vignetting during acceleration, are
entirely a game-side responsibility layered on top of the raw tracked pose.

## 8. Cinematics in VR

Sequencer drives the camera through `MovieSceneCameraCutTrack`
(`Engine/Source/Runtime/MovieSceneTracks/Public/Tracks/MovieSceneCameraCutTrack.h`)
and its track instance, which calls `SetViewTarget` on the player's camera
manager the same way gameplay code would. Because `APlayerCameraManager`
resolves a camera-actor view target through
`CamActor->GetCameraComponent()->GetCameraView`
(`PlayerCameraManager.cpp:381`), and that in turn runs
`UCameraComponent::GetCameraView`/`HandleXRCamera`
(`CameraComponent.cpp:366-414`) unconditionally, the `bLockToHmd` flag on
the cut-in camera's `UCameraComponent`
(`Engine/Source/Runtime/Engine/Classes/Camera/CameraComponent.h:198`) is
what decides whether a scripted camera cut rides head tracking
(`bLockToHmd = true`, used for in-headset cinematics so the player can still
look around a seated/anchored shot) or plays back as an authored,
non-head-tracked cut (`bLockToHmd = false`, used for traditional
cutscenes shown as a 2D plane or deliberately non-VR cinematic). No special
"VR cinematic" track type exists; it is this one component flag plus the
ordinary camera-cut mechanism.

## Concerns a flat-to-VR conversion must own

1. A per-eye pose source read once per game-tick frame, analogous to
   `FOpenXRHMD::OnStartGameFrame` (`OpenXRHMD.cpp:3908`).
2. A clean separation between "the gameplay/scripted camera" and
   "head-tracked offset applied on top," as `bLockToHmd` does
   (`CameraComponent.h:198`, `CameraComponent.cpp:397-413`).
3. Render two views per frame from one camera position, each with its own
   eye offset, per `IXRCamera::CalculateStereoCameraOffset`
   (`IXRCamera.h:60`, impl `DefaultXRCamera.cpp:155-186`).
4. Asymmetric, runtime-reported per-eye FOV/projection, not a shared
   symmetric FOV (`OpenXRHMD.cpp:1406-1438`).
5. A render-thread late update that re-samples the pose after the game
   thread has already submitted the frame
   (`DefaultXRCamera.cpp:188-241`, `PreRenderView_RenderThread`).
6. The same late-update treatment for anything rigidly attached to
   tracked objects (controllers, weapons), not just the camera
   (`FLateUpdateManager`, `MotionControllerComponent.h:163`).
7. Explicit single-pass/instanced-stereo vs. two-pass rendering decision,
   since it changes how views share culling/shading work
   (`SceneView.cpp:965-972`).
8. A swap-chain/compositor abstraction that owns present, rather than a
   normal OS backbuffer flip (`FXRRenderBridge`/`FXRSwapChain`,
   `XRRenderBridge.h`, `XRSwapChain.h`).
9. A wait/begin/end frame handshake with the runtime so CPU/GPU work is
   paced to the headset, not just vsync (`xrWaitFrame`/`xrBeginFrame`/
   `xrEndFrame`, `OpenXRHMD.cpp:3862,4220,4424`).
10. Render-scale/resolution control decoupled from desktop screen
    percentage (`GeneratePixelDensitySize`, `OpenXRHMD.cpp:1243`,
    `SetPixelDensity`, line 4486).
11. A hidden-area mask to skip shading pixels the lens crops
    (`BuildOcclusionMesh`, `OpenXRHMD.cpp:2017`).
12. A tracking-origin / floor-alignment convention (seated vs. standing vs.
    room-scale), explicit and queryable (`EHMDTrackingOrigin`,
    `OpenXRHMD.cpp:548-663`).
13. A mirror/spectator window that is a true second composite, not just a
    stolen eye-buffer blit (`FDefaultSpectatorScreenController`,
    `DefaultSpectatorScreenController.h:19-39`).
14. A UI strategy decision per surface: compositor-locked layers for
    reticles/system UI (`IStereoLayers.h:60-150`) vs. diegetic world-space
    widgets for in-game UI, and an explicit decision about what happens to
    the flat 2D HUD (`FDefaultXRCamera::SetupViewFamily`,
    `DefaultXRCamera.cpp:295-307`).
15. A motion-controller input abstraction independent of a specific
    controller model, addressed by named "motion source" rather than
    hardware button index (`IMotionController.h:63-150`).
16. Interaction-profile/binding data so new controllers can be supported
    without game-code changes (`OpenXRInput.cpp:388-393,544`).
17. A haptics path reachable from gameplay code
    (`xrApplyHapticFeedback`, `OpenXRInput.cpp:1523`;
    `IHapticDevice.h`).
18. Explicit, game-owned comfort/locomotion systems (vignette, snap-turn,
    teleport); the engine supplies none beyond camera smoothing meant
    for capture only (`DefaultXRCamera.cpp:15-37`).
19. A rule for what a scripted/cutscene camera does to head tracking,
    equivalent to the `bLockToHmd` decision at a camera cut
    (`CameraComponent.cpp:366-421`).
20. A session/device lifecycle state machine (connect, ready, running,
    stopping, lost) that the renderer and gameplay both react to, not just
    an on/off stereo flag (`OpenXRHMD.cpp:3958-3998`).

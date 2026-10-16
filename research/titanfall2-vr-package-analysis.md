# Titanfall 2 VR (CircuitLord) Package Analysis, Deep Pass

Source: Titanfall2VR-1.0.17 package, analyzed with strings, objdump and grep
against Titanfall2VR.dll (2.8 MB PE32+ x86-64), the two helper exes, the
Squirrel sources under mods/Titanfall2VR.Cockpit, and the json manifests. No
executable was run. Builds on a prior pass that already established the
plugin shape, the OpenXR/D3D11 setup, roughly 294 patch sites, and the two
Squirrel mods.

## 1. Script to native contract

The DLL exposes true natives (called from Squirrel, no Squirrel body exists)
and separately calls back into named Squirrel functions each frame or on
events. A native has no `function NAME` definition in the shipped .nut/.gnut
files; a callback is `global function NAME` defined in script and invoked by
the DLL by name.

**Natives confirmed from script call sites**, grouped:

- Body: `TF2VR_SetBodyMovement(bool airborne, bool crouched)`.
- Hands/holsters/grip: `TF2VR_HoldProp(int side, int grip, int entity)`,
  `TF2VR_GripAnchor(int side, int grip, vector origin, vector forward, vector up)`,
  `TF2VR_SetHeldHands(int hands)`, `TF2VR_SetPropBones(int entity, int first, int bones)`,
  `TF2VR_PosePropBone(int entity, int bone, vector origin, vector forward, vector up)`,
  `TF2VR_HideBones(int entity, int first, int bones)`,
  `TF2VR_SetLegsHidden(bool hidden)`, `TF2VR_PulseHands(int hands)` (haptic
  pulse bitmask), `TF2VR_LeftUsesTouch()` (handedness query).
- Weapons/reload: mostly convar-driven (`tf2vr_reload_rounds`,
  `tf2vr_reload_magazine_out`, `tf2vr_reload_ejections`, `tf2vr_holster_mask`,
  `tf2vr_pickup_left/right`); the one true native here is `TF2VR_PredictGrenade`
  (listed under callbacks, since it is also DLL-invoked).
- Titan: `TF2VR_SetEmbarkPose(vector origin, float yaw, vector hatch, float cycle)`.
- Memory room (campaign flashback kiosk): `TF2VR_PlayMemories()`,
  `TF2VR_SetMemoryCamera(int map, int pilot, int titan, vector origin)`,
  `TF2VR_ShowMemoryScreen(vector origin, vector angles, float width)`,
  `TF2VR_MemoryMusicStarted()`, `TF2VR_MemoryPlaybackClip()`,
  `TF2VR_MemoryPlaybackPhase()`, `TF2VR_MemoryTerminalKeys()`,
  `TF2VR_MemoryRoomReady()`, `TF2VR_OpenMemoryDirectory()`,
  `TF2VR_SetDynamicMemoryVisible(bool)`, `TF2VR_QueueCampaignMemory(int event, int choice)`,
  `TF2VR_RecordCombatKill()`.
- HUD/menu: `TF2VR_SetMenuActive(bool)`, `TF2VR_HighlightsAvailable()`,
  `TF2VR_StartupPhase()`, `TF2VR_StartupTick()`, `TF2VR_SetCreditsLogo(float logo, float byline)`.
- Comfort: `TF2VR_RequestRecenter()`, `TF2VR_DialogueInput(bool held)` (doubles
  as gaze-dwell confirm), `TF2VR_GazeDot(vector eye)` (float dot product, used
  for gaze-based dialogue selection).

**Natives named in the DLL but not called from the unpacked scripts here**
(referenced only from the bsdiff-patched Respawn client scripts, not
reconstructable without the base game): `TF2VR_SetDamageOverlayColor`,
`TF2VR_ShowDamageOverlay`, `TF2VR_SetEmpOverlay`, `TF2VR_LookTargetDistance`,
`TF2VR_SetPromptGesture`, `TF2VR_WeaponHand`, `TF2VR_SetTitanPitchEnabled`,
`TF2VR_PositionHitIndicator`, `TF2VR_UpdateObjective`, `TF2VR_ObjectivesHeld`,
`TF2VR_SetCameraScreen`, `TF2VR_VideoSkipHeld`. The DLL also embeds a
help-string table of argument lists near these names, but MSVC string pooling
reorders literals, so names could not be matched to signatures with confidence.

**Callbacks, DLL calling into script** (the VR driving loop, confirmed from
the `global function` signatures):

- `TF2VR_UpdateEyeCase(int hands, vector left, vector right, vector leftAngles, vector rightAngles)`.
- `TF2VR_UpdateTitanPose(int command, vector origin, vector direction, vector angles, vector headOrigin, vector headDirection)`.
- `TF2VR_UpdatePickups(int mask, vector left, vector right, vector leftForward, vector rightForward, vector leftHip, vector rightHip, vector hipForward)`.
- `TF2VR_UpdateArcToolPose(int command, vector origin, vector direction)`.
- `TF2VR_CheckDynamicMemory(int unused, vector headOrigin)`,
  `TF2VR_PrepareMemory(int reusePilot, int event, int btView)`.
- `TF2VR_PredictGrenade(int ticket, int consumesAmmo, vector position, vector angles, vector velocity, vector spin)`.
- `TF2VR_UpdateBodyMovement()`, `TF2VR_UpdateEmbark()` (no-arg; pull live
  state through other getters).

The architecture: the DLL owns tracking, IK and presentation state every
frame, and pushes it into Squirrel through a fixed, small set of per-system
"Update" callbacks, while Squirrel pushes one-shot commands back through
single-purpose natives. Hitscan, damage and inventory logic stayed in script;
only pose, pointer and presentation state moved native.

**Convars** (mod.json plus script references). Comfort: `tf2vr_snap_turn`(0),
`tf2vr_turn_speed`(210), `tf2vr_snap_angle`(45), `tf2vr_disable_vignette`(0),
`tf2vr_recenters`, `tf2vr_recenter_hint`. Body/hand: `tf2vr_toggle_grip`(0),
`tf2vr_gun_angle_offset`(-15), `tf2vr_holster_mask`. HUD:
`tf2vr_head_hud`(1), `tf2vr_titan_hud`(1), `tf2vr_hitmarkers`(1). Titan:
`tf2vr_titan_gun_turn`(1), `tf2vr_titan_transition_mask`,
`tf2vr_titan_transition_screen`. Cutscenes: `tf2vr_cutscenes_vr`(0).
Pickups/reload, all internal state defaulting to 0: `tf2vr_gun_ack`,
`tf2vr_gun_name`, `tf2vr_gun_slot`, `tf2vr_pickup_left/right`,
`tf2vr_pickup_kind_left/right`, `tf2vr_grenade_ack`,
`tf2vr_grenade_consumes_ammo`(1), `tf2vr_projectile_ticket/entity`,
`tf2vr_chest_visible`, `tf2vr_utility_entity`,
`tf2vr_reload_highlight(+_entity)`, `tf2vr_reload_ejections`,
`tf2vr_reload_rounds`, `tf2vr_reload_empty_grabs`,
`tf2vr_reload_magazine_out`, `tf2vr_reload_stop_sound`. Debug:
`tf2vr_pickup_debug`, `tf2vr_grab_debug`. Memory room: `tf2vr_memory_enabled`(1),
`tf2vr_memory_capture`, `tf2vr_memory_campaign`, `tf2vr_memory_story`,
`tf2vr_memory_room`.

## 2. Body and arm IK

The DLL's telemetry CSV headers, embedded as plain strings, one per logged
`render_sequence`, describe three separate solvers.

Arm/hand IK columns: `shoulder_x/y/z, elbow_x/y/z, wrist_x/y/z, target_x/y/z,
native_shoulder_*, native_elbow_*, native_wrist_*, servo_*, goal_*, hand_*,
head_*, reference_shoulder_*, servo_q*, goal_q*, rest_q*, hand_q*, chassis_q*,
reload_blend`. The shoulder is a fixed anchor off a chest/head reference;
elbow and wrist are IK-solved toward a tracked-controller `target`, with a
`servo` (damped follow) term blended against the raw `goal`, plus quaternion
`rest` and `hand` poses used to blend into the reload animation
(`reload_blend`). Other rows carry `reach_limit` and `grip_error`, indicating
a soft reach clamp.

Full-body/locomotion columns: `pelvis_x/y/z, foot_x/y/z, native_foot_yaw,
chest_yaw, chest_forward_*, local_foot_*, local_knee_*, knee_bend,
native_knee_bend, foot_plant_error, locomotion_error, locomotion_offset_*,
idle_weight, standing_weight, standing_lift, airborne_weight, crouched`. Legs
are not tracked; a procedural solver places pelvis and feet from HMD yaw and
a ground/airborne/crouched blend weight, with a foot-plant error metric used
to blend against the game's native animation.

Titan calibration columns: `left_ready, right_ready, calibrated, arm_span,
standing_y, blocking`. A calibration pass measures arm span and standing eye
height before the main solver activates. Error strings `"Titan calibration
bones unavailable"` and `"Titan calibration cache changed"` show the result
is cached per Titan model and invalidated on model change.

Finger/hand pose: the controller telemetry row carries `thumb_curl,
index_curl, middle_curl, ring_curl, little_curl, thumb_across,
thumb_opposition, thumb_up, finger_tracking`, standard curl-based posing from
controller analog/capacitive data, with a parallel path for true joint
tracking (`XR_EXT_hand_tracking`; strings `"active finger joint position
unavailable"`, `"tracked finger segment has zero length"`).

Weapon aim is separate from arm IK (`aim_x/y/z, offset_x/y/z, weapon_tracked,
muzzle_x/y/z, muzzle_forward_*, weapon_pitch/yaw/roll`): the gun is aimed
directly from controller pose, not from the hand bone chain.

First person vs third person: the normal viewmodel animation state machine
(`client.viewmodelAdsBlendCall(2/3)`, `client.viewmodelSprintPoseCall`,
`client.setupRigSlot`, `client.setupCinematicRigSlot`,
`client.setupSeatedRigSlot`) is bypassed by driving bones directly through
`setupBones`/`blendRigSlot`/`buildRigSlot`/`drawRigSlot` hooks. The stock
hand/weapon animation set is not deleted; it is suppressed per frame and kept
only as a rest pose and reload cue source, matched by the `reload_blend` and
`native_wrist_*` columns. The three patched .mdl files
(`pov_bt_vr_cockpit.mdl`, `pov_mlt_hero_jack_rifleman.mdl`, and the harmony
vista model) are viewmodel/cockpit arm rigs edited to add bones the native IK
targets: the `def_l_shoulder/elbow/forearm/wrist`, `def_r_*` bone names and the
error "forearm twist bone outside lower arm" match a rig carrying an explicit
twist bone the stock rig lacked.

## 3. Stereo rendering strategy

Reconstructed from the `xr.*`, `render.*`, `replay.*`, `shadow.*`,
`material.*` tag families and the OpenXR call strings:

1. `xr.wait_frame`, `xr.poll`, `xr.sync_actions`, then `xr.locate_views` gets
   both eye poses/fovs for `XR_VIEW_CONFIGURATION_TYPE_PRIMARY_STEREO`,
   hard-asserted as "exactly two stereo views are required".
2. `render.eye_setup` computes a per-eye scale/center/culling-tangent set,
   logged as `scale_x/y, center_x/y, expected_scale_x/y, expected_center_x/y,
   world_culling_tan_x/y`: it both computes and sanity-checks the asymmetric
   projection the engine's native camera needs to match OpenXR's per-eye FOV.
3. `shadow.camera/fit/casters/prepare/draw` run once per frame, not once per
   eye. The string "stereo shadow frusta exceed a hemisphere" shows shadow
   casters are fit to one shared frustum covering both eyes, clamped when the
   combined frusta would exceed 180 degrees.
4. `render.views`/`engine.renderViewsCall` run the engine render twice, once
   per eye, with `material.setRenderTargetsCall` pointed at each eye's D3D11
   target, sized from the OpenXR swapchain rect ("Eye %u: %u x %u, %u
   swapchain images"). `material.letterboxCall` is overridden per eye so the
   native letterboxing cannot clip the VR view. `client.drawShadowCastersCall`
   runs per eye against the single shared shadow map from step 3.
5. HUD and world UI: the native 2D RUI pipeline (`drawHudRuiCall`,
   `drawWeaponRuiCall`, `fullscreenRuiViewCall`, `cockpitRuiViewCall`,
   `engine.evaluateRuiCall`, `engine.ruiTrianglesSlot`) still runs, but into an
   offscreen canvas texture. The custom pixel shaders (`textPs`, `ps`/
   `scenePs`) sample that canvas and draw it as a textured quad placed at a
   fixed distance/angle from the eye (`hud_px/py/pz, hud_qx/qy/qz/qw`), so 2D
   HUD becomes a head- or world-locked billboard rather than a screen overlay.
   Weapon scopes get their own reprojection (`scopePs`, using `scopeEye`,
   `scopeAperture`, `scopeOptics`, `scopeLens`) simulating a real lens instead
   of blitting the 2D scope texture.
6. A hidden-area mask is drawn per eye first (OpenXR
   `XR_VISIBILITY_MASK_TYPE_HIDDEN_TRIANGLE_MESH_KHR`), consumed by a small
   shader pair that writes black from head-relative `tangent` vertices and a
   `projection` scale/offset constant, to stencil-cull lens-occluded pixels.
7. `material.finishGpuFenceCall`/`gpuQueryReadyCall`/`drainMainQueueCall` gate
   CPU/GPU sync before `xr.acquire_image`/`xr.release`/`xr.wait_image`; the
   swapchain textures are released to OpenXR instead of calling
   `dxgi.Present`. A separate `desktop.present` keeps a "game mirror" window
   alive independent of the headset path.

Disabled or altered per eye: letterboxing (fully suppressed in VR); desktop
depth of field (guarded off when stereo is active, matching the
`+dof_enable 0` launch flag); particle simulation, run once and shared across
both eyes (`client.simulateViewParticlesCall` runs ahead of both eye draws)
rather than per eye, exactly the bug class the string "OpenXR eyes used
different client simulation states" warns against. Shadows use the
shared-frustum approach from step 3 rather than being disabled.

## 4. Camera and cutscenes

`tf2vr_cutscenes_vr` (default 0) is the in-game VR setting "Swch3DCutscenes".
Off, cutscenes use a 2D-panel path: a telemetry row with `cinematic,
cinematic_screen, anchor_x/y/z, anchor_yaw, fade_r/g/b/a, fade_modulate,
cinematic_vignette, flat, hud_arc` describes a flat video panel anchored in
world space in front of the player, faded in and out, with a vignette to
limit how much of the stereo view a 2D image fills. On, the same row's
`source_sequence/depth_sequence, cinematic_source_sequence` and the camera
row's `camera_right/up/back_*, camera_projection_offset_x/y` fields indicate
scripted cuts render as real stereo 3D, a moving stereo camera substituting
for the player's own head. `screen_anchor_yaw`, `interpolation` and
`predictable` columns suggest cuts can blend in with head tracking rather than
hard-snapping. The string "letterbox caller changed" and a named
`wilds_title_letterbox` case show some narrative beats still use a plain
letterbox even in 3D mode. Titan embark/disembark uses the dedicated
`TF2VR_SetEmbarkPose`/`TF2VR_UpdateEmbark` pair driving a procedural camera off
the Titan's hatch attachment and the player's animation cycle, so it is
handled as a tracked first-person sequence rather than a cutscene.

## 5. Comfort and locomotion

Turning: `tf2vr_snap_turn` default 0 (smooth turn is default),
`tf2vr_turn_speed` default 210 deg/s, `tf2vr_snap_angle` default 45 deg per
snap. Telemetry fields `turn_time_ns, turn_dt, turn_reset, stick_x` confirm
stick-based yaw with a time-accumulated turn state.

Vignette: `tf2vr_disable_vignette` default 0, meaning on by default. The
`vignettePs`/`blackoutPs` shaders implement peripheral darkening keyed off
head-relative ray angle, with explicit inner/outer angle and strength
constants, and keep the nasal side open per eye (`sign(eyeToHead[3].x)`) so
the binocular overlap is never cut.

Recenter: `tf2vr_recenters` (counter/state), `tf2vr_recenter_hint` (0/1 to
show a UI hint), native `TF2VR_RequestRecenter()` fired from the pause menu.

Seated/standing: `client.setupSeatedRigSlot` and strings "seated bone output
changed", "seated pilot model changed" show a seated-mode rig variant exists
alongside the standing one.

Height calibration: no dedicated convar was found. Height instead appears to
be read live from HMD tracking origin each session (`standing_y` in the
calibration telemetry row) rather than stored.

## 6. Embedded HLSL shaders

By purpose, in binary order: `vs`/`fullscreenVs`, a shared fullscreen-triangle
vertex shader; `gridPs`, the memory-room floor reference grid; `textPs`,
atlas-based text for world-space UI; `figurePs`, a vector-drawn calibration
figure icon (ring, silhouette, raised arms, readiness ticks); `vs`/`lineVs`/
`ps`/`scenePs`, a generic textured-quad blit for HUD/weapon canvases with an
exposure black-point clamp variant; `scopePs`, a physically modeled weapon
scope (lens radius clip, pupil-aperture vignetting, angular FOV mapping,
reticle dots, fresnel); `overlayVs`/`damagePs`, a screen-space
damage-direction overlay, refracted and tinted; `empPs`, EMP screen
distortion via scrolling normal maps, reusing the engine's native Refract
technique per its own comment; `missileLockPs`, a circular missile-lock
reticle; `titanGuidePs`, the four-direction Titan ability/punch swipe gesture
guide; `objectivePs`, objective marker compositing; `vignettePs`/`blackoutPs`,
comfort vignette and full blackout transition; `memoryScreenPs`, the
memory-room CRT terminal (tracking glitches, chroma-subsampled video decode,
viewfinder HUD, record light); `fadePs`, flat screen fade; a luma/chroma
downsample pair (`psLuma`/`psChroma`) used to encode a capture stream for the
highlights/memory feature; a hidden-area-mesh pair writing solid black from
head-relative tangent positions; an MSAA depth-resolve pixel shader
(`Texture2DMS` min-reduce to `SV_Depth`); and a final crop/copy blit, likely
for the mirror window.

## 7. Patch sites by DLL

The DLL carries roughly 290 named, dotted `module.identifier` strings used as
telemetry tags and as labels for patched call sites, so a moved hook target
fails with a named error instead of corrupting memory silently.

- **client.dll (188)**: rendering/HUD (`drawHudRui`, `drawWeaponRui*`,
  `drawCockpit*`, `beginHud`/`endHud`, `copyView`, `finalizeModelView`,
  `setupRigSlot`/`setupCinematicRigSlot`/`setupSeatedRigSlot`, `depthOfField`,
  shadow casting `drawCascades`/`fitCascades`/`clearCascades`, scope
  `scopeBodyGroupCall`/`scopeOverlayVisible`/`scopeVisibilityCall`); weapon
  aim/animation (`weaponKickCall`, `addWeaponKick`, `bulletSpreadCall`,
  `boltSpreadCall`, `spawnBoltCall`, `clientShotCall(2)`,
  `viewmodelAdsBlendCall(2/3)`, `viewmodelSprintPoseCall`,
  `attachmentFovCall1-11`); haptics (`mixRumbleCall`, `nativeRumbleCall`,
  `startRumbleEffectCall(2)`); input (`inputMove`, `inputButtonsSlot`,
  `joystickEnabled`, `disableMouse`); Titan/cockpit (`setupCockpitCall`,
  `drawCockpitHud`, `cockpitHudProjectionCall`); movement (`finishMovement`,
  `slideMove`, `smoothStairs`); targeting HUD (`traceTargetHudCall`,
  `updateTargetHudCall`, `prepareTargetHud`); entity/world queries
  (`entityByIndex`, `localPlayer`, `eyeAngles`, `eyePosition`).
- **server.dll (30)**: server-side ballistics and movement for VR-accurate
  reconciliation (`bulletTraceCall(2)`, `fireBullets`, `boltSpreadCall`,
  `bulletSpreadCall`, `serverBulletsCall`, `serverFirePoseCall`,
  `serverWeaponFrameCall(2)`, `vortexAnglesCall(1/2)`, `meleeDirectionCall`,
  `meleeOriginCall`, `laserAnglesCall`, `slideMove`, `finishMovement`,
  `spawnBoltCall`).
- **engine.dll (32)**: frame/render driver (`renderViewsCall`,
  `frameStageCall`, `finishFrameCall`, `postRenderCall`, `renderFlatCall`,
  `renderLoadingCall`, `evaluateRuiCall`, `changeLevelSpawnCall`,
  `newGameSpawnCall`, `clientChannelCall`, `disableFocusPause`, `menuCamera`).
- **materialsystem_dx11.dll (42)**: GPU frame lifecycle
  (`beginFrame`/`endFrame`, `drainMainQueueCall`, `finishGpuFenceCall`,
  `finishGpuFrameCall`, `finishJobsCall`, `gpuQueryReadyCall`,
  `setRenderTargetsCall`, `setBlendStateCall`, `letterboxCall(2)`,
  `uploadBonesCall`, `resizeWindowSlot`, `setRenderConfigSlot`).
- Smaller groups: `render.*` (11, frame phases: eye_setup, loading,
  particles, scene, visibility, xr_ready); `replay.*` (10, the highlights/
  memory capture ring buffer: begin/prepare/swap/release); `shadow.*` (6);
  `memory.*` (6); `inputSystem.*` (6, cursor and rumble); `ui.*` (4); `xr.*`
  (14, the OpenXR frame loop: wait_frame/begin_frame/locate_views/
  sync_actions/acquire/release/haptics).

## 8. launch.json and release.json

`release.json` is trivial: version 1.0.17, description "OpenXR campaign VR
support for Titanfall 2", used by the Northstar mod manager to display and
version-check the package.

`launch.json` supplies two argument lists merged into the game's command line.
The base `arguments` set profile and window size, disable the loading-screen
video, force a fake master server (`ns_masterserver_hostname
http://127.0.0.1:9`, `ns_report_server_to_masterserver 0`,
`ns_has_agreed_to_send_token 2`) so the campaign runs offline without
Northstar's multiplayer layer, and disable focus-pausing so tracking keeps
running if the window loses focus. `vrArguments`, appended only in VR, force
off native depth-of-field and ADS-DOF (both would conflict with VR
rendering), disable the letterbox aspect threshold and `dvs_enable` (dynamic
sample-rate scaling would fight fixed-rate VR pacing), pin
`mat_viewportscale` to 1, and disable `mat_vsync_mode` and the FPS cap so the
engine's own pacing never fights the OpenXR compositor's.

## What transfers to a CryEngine game

- Hard script/native split: native owns only pose, tracking and presentation
  state; gameplay stays in the existing scripting layer; the two sides meet
  through a small fixed set of native setters plus native-to-script "Update"
  callbacks driven every frame.
- IK ownership lives entirely in the native plugin, with the engine's own
  animation graph demoted to a rest-pose and blend source rather than removed.
- A full-body and locomotion solver (pelvis, knees, feet) derived from HMD
  yaw and movement state, kept separate from arm/hand IK and from a one-time
  mount/vehicle calibration pass measuring arm span and standing eye height.
- HUD and RUI captured off-screen into a texture and reprojected as a
  billboard or world-anchored quad through dedicated pixel shaders, rather
  than making the native 2D UI system stereo-aware itself.
- Optics rendered as a simulated lens (vignette, pupil occlusion, parallax)
  instead of blitting a flat 2D zoom texture.
- Cutscene policy gated behind one convar with two fallbacks: a faded,
  vignetted 2D panel by default, or full stereo camera substitution as
  opt-in, with tracked mount sequences handled as procedural camera.
- Named, versioned patch sites instead of blind byte offsets: every hooked
  function gets a stable semantic name used for both the hook and its error
  reporting, so a game update breaks loudly and specifically.
- A shared shadow frustum for stereo (one pass covering both eyes, clamped
  past a hemisphere) and a single shared particle-simulation step per frame,
  avoiding eye-to-eye state divergence.
- An out-of-process crash monitor that attaches as a debugger, shares a
  heartbeat memory mapping with the game, and writes a minidump on crash or
  hang, decoupled so it survives the plugin's own crashes.
- Asset edits shipped as sha256-verified bsdiff patches applied against the
  user's own game files at install time, never as redistributed original
  assets, the only legally defensible way to ship edited content for a
  closed-source game on any engine.
- Convar-driven comfort features (snap/smooth turn, vignette toggle,
  recenter) stored in the existing settings system rather than anything
  VR-specific.

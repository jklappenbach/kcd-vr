# KCD2 animation, horse, and cutscene surface for a VR conversion

Measured 2026-10-15 against the Steam build (release_1_5, WHGame.dll, 89 MB)
using `strings -n 6`, RTTI name scanning, the extracted Lua and configs, and
the community headers in `libKCD2`, `KCSE-for-kcd2`, and `KCD2Tools`. A claim
straight from a binary string, an RTTI class name, or a header's measured
offset is marked measured; one reconstructed from naming conventions or
CryEngine 5 precedent is marked inferred.

## 1. First-person arms and weapons

### Pose modifier stack (measured)

The binary carries a full CryEngine Mannequin pose-modifier library, each one a
separate RTTI class: `AnimationPoseModifier_LimbIk`, `_Ik2Segments`, `_IkCcd`,
`_IKTorsoAim`, `_LookAtSimple`, `_PoseBlenderAim`, `_PoseBlenderLook`,
`_PoseAlignerChain`, `_TransformBlender`, `_ConstraintAim/Line/Point`,
`_DrivenTwist`, `_Recoil`, `_Visor`, `_PoseMatching`, chained by a
`_PoseModifierStack`/`_OperatorQueue`. `IAnimationPoseAligner` and
`AnimationPoseAlignerWH` exist as separate interfaces, and their cvar family
(`a_poseAlignerEnable`, `a_poseAlignerForceNoIK`, `a_poseAlignerBipedRaycasLength`)
describes leg and foot ground alignment, not arms. Arm-relevant cvars live
under `ca_*`: `ca_UseLookIK`, `ca_UseAimIK`, `ca_DrawLookIK`,
`ca_useADIKTargets`, `ca_LookIKBlendSpeed`, plus Mannequin definition files
`LookIK_Definition`, `AimIK_Definition`, `LimbIK_Definition`, and layers
`lookIKLayer`/`aimIKLayer`. `wh_pl_DebugIKLimbs` and `wh_pl_AnimatedCameraBone`
are Warhorse additions over the same system.

The single most important find for this question is `C_WeaponSolver`
(`combatmodule`), with exported debug strings `WeaponSolver::Init`,
`WeaponSolver::SetIKTargets`, `WeaponSolver::End`, a second instance
`WeaponSolver2`, an `E_WeaponSolverMode`/`E_WeaponSolverState` enum, and a large
native cvar family (`wh_cs_WeaponSolverColBlendLimiter`, `_ColExtraOffset`,
`_EnableMultipleCollisions`, `_ForceRecoilDuration/IntensityPos/IntensityRot`,
`_PosOffhandFade`, `_RecoveryIKBlendThreshold`, `_RotRecoverySharpness`,
`_SlidingDuration`). This is a per-frame procedural layer that nudges the
weapon's (and the hand's) position and rotation on top of the baked
animation, for weapon-world collision avoidance, recoil response, and offhand
blending. `SetIKTargets` is exactly the entry point a VR mod would want to
feed with controller-driven hand/weapon targets, but it already has an owner,
its own collision and recoil logic, running every frame expecting its own
targets rather than raw player input. Overriding it cleanly means replacing
its target source, not just writing over it.

### Weapon attachment (measured + inferred)

`BONEID_HAND_L` and `BONEID_HAND_R` are named bone-id constants in the binary,
and `Bip01 Head` appears as a literal bone name, confirming the skeleton still
uses CryEngine's classic `Bip01` convention carried from KCD1. A native
`C_ItemAttachmentManager` (entitymodule) plus `I_ItemAttachmentAnimLogic` and
`I_ItemAttachmentListener` interfaces drive attaching an item to the skeleton;
`wh::dialogmodule::data::E_ItemAttachmentHand` (a 2-value left/right enum) is
used for cutscene prop placement specifically. No literal socket string like
`weapon_bone` was found, consistent with Warhorse keying attachments by
item/slot id through the attachment manager rather than a bone-name string
compiled into the binary; the concrete socket names likely live in the
character definition (CDF) files inside `Characters.pak`, not in WHGame.dll.

Item handling is a native state machine: `S_ActorAnimItemHandlingRequestImpl`
specializations exist for Draw, Drop, Holster, PickUp, Place, PutItemInHand,
PutItemInInventory, and SwapWeapon, going through `C_ActorActionItemHandlingAnim`.
Drawing, holstering, and swapping weapons are full Mannequin fragments, not a
simple bone attach, so a VR hand "holstering" by moving a controller near a
scabbard still needs to trigger this fragment machinery for the weapon mesh to
move between hand and sheath.

### Combat (measured)

Combat is a heavily synchronized, animation-fragment-driven system, confirmed
by a large `combatmodule` RTTI surface: `C_CombatActorActionAttack/Block/Dodge
/FreeBlock/PerfectBlock/SyncAttack/SyncHit/SyncPerfectBlockHit/SyncTransition`,
each with matching `Data`/`QueryData` structs, a
`C_CombatActionFragmentIdMappingDatabase`, and procedural clips
`C_AttackerPoseAlignmentProceduralClip`, `C_AttackerPositionAlignmentProceduralClip`,
`C_AttackerRotationAlignmentProceduralClip`. Attacker and defender animations
are paired and spatially aligned by these clips so a swing and its matching
parry/hit-react land in the same place every time. A `C_CombatCameraManager`
drives a dedicated combat camera, separate from the general follow camera.
Taking the arm over with IK for melee risks two breakages: the alignment clips
expect the weapon tip where the baked animation puts it, not where a
physically swung controller puts it, and `C_WeaponSolver`'s own collision and
recoil IK will keep correcting a hand position it did not expect to be
externally driven. A VR combat model is more likely to succeed mapping
controller swing direction and speed onto which attack fragment plays
(direction-matched slash input, the approach most CryEngine-era VR melee mods
use) than by forwarding controller position straight into the arm bones.

### Minigames (measured)

Lockpicking is fully native: `C_LockPicking`, `C_LockPickingView`,
`I_LockPickingAction`, action classes `C_LockPickingIdleAction/LoopAction
/TurnAction/TurnFailAction/LeaveAction/FailIdleAction`, and
`C_LockpickingResultTrigger`/`C_LockpickingStartedCause` on the flowgraph side.
This reads as a dedicated 2D/3D minigame view (`C_LockPickingView`) rather than
raw hand-IK animation, so VR likely substitutes controller-twist input into the
same view rather than needing arm IK at all. Blacksmithing and alchemy are both
native systems with a large `wh_pl_Blacksmith*`/`wh_pl_Alchemy*` cvar surface (forge temperature, anvil particle thresholds,
hammer-stroke timing) and dedicated `C_ScriptBind_Blacksmithing`,
`C_ScriptBind_ForgeBuilder`, `C_ScriptBind_Alchemy`, `C_Alchemy` classes.
`wh_pl_BlacksmithAnvilMovementMouse/Gamepad` and `_ForgeMovementMouse/Gamepad`
show the anvil and forge driven by 2D cursor-style input a VR controller can
likely substitute directly, but the visible hands still come from the base
arm animation set, so a VR arm-IK rework has to keep these poses working.
Archery and crossbow aiming have their own cvar family (`wh_pl_BattleArchery*`:
bow point offset, leaning, angle limits, keyboard vs mouse sensitivity) and
separate blend-space directories (`blendSpace/bow`, `/crossbow`,
`/hand_cannon`, plus `aimposes`/`lookposes`, measured from the Animations.pak
listing). Archery already sits in `ca_UseAimIK`/`AimIK_Definition` territory:
the bow arm is IK-posed toward an aim target today, good news for a VR hand
target, but the draw hand and string tension stay tied to the blend space,
not free IK.

## 2. Horseback

### Riding model (measured, via libKCD2 headers)

`C_Horse : C_Animal` is a thin shell (`sizeof 0xA60`) whose state moved into a
heap-allocated `S_HorseData` controller (`sizeof 0x4C8`) reached through
`m_pHorseData`. `S_HorseData` carries, among much else, `m_headAlign`/
`m_headAlignTime` (blend state for head alignment, presumably the horse's own
head), a `m_pBridlePose` pointer to `AnimationPoseModifier_BridlePoseModifier`,
and `m_pMove`, typed as `I_HorseMoveAdapter`, concretely `C_RiderPlayerInput`.
The rider side is `C_RiderPlayerControl` (at `C_Player+0xD68`), which owns a
`C_RiderPlayerInput*`, a whistle flag, a dash-weight function bound to
"1.0 iff horse rank==3", and a dash-FOV-change-speed scalar tying the dash
camera FOV kick to horse rank.

Critically, `S_MountAnimState` (`sizeof 0x70`, a per-frame actor movement/look
request, shared with on-foot movement and reused by the horse) carries
`m_lookTarget` and `m_aimTarget` (world-space points, valid under
`m_hasLookTarget`/`m_hasAimTarget`), documented in the header's own comment as
consumed by "`C_Actor` vf250 (look/aim IK)". Concretely: the engine already
has a native, per-frame look/aim IK target mechanism used while mounted (and
on foot), fed by a `Vec3` world point rather than bone angles. This is the
most promising hook point in the whole survey for VR head tracking on
horseback: if this request struct can be reached and written, by trampoline
hook or by walking the same kind of offset chain TPVCamera already resolves,
feeding it the HMD-derived look-at point would let the existing IK layer do
the work, rather than fighting procedural riding animation the way arm
takeover would.

### Horse camera and view limits (measured + inferred)

A large `wh_horse_*` string family exists (not located in KCD2ModLoader's
cvars.md dump, so these may be Lua-exposed property names rather than
`REGISTER_CVAR` entries, flagged inferred for exact console-settability):
`wh_horse_CameraCentering`, `_CameraCenteringTime/TimeInCombat/PitchOffset
/SwayCoeff`, `_CameraSmoothing`, `_CameraTwitchAmount`, `_ViewLimitBottom
/Narrow/Wide`, `_HideReinsWhenMounted`, `_EnableRiderHeadCollision`,
`_DismountLockDuration`, `_MountLockDuration`, `_AltControls`. The general
camera cvars (`cl_cam_*`, `cl_tpv*`) are the same ones used on foot, so mounted
behaviour is the shared follow/orbit camera with horse-specific centering and
view-limit overlays, not a distinct camera class. `_EnableRiderHeadCollision`
implies the engine already tracks a head collision volume for the rider,
which a VR head-tracked camera needs to respect so the player's real head
motion does not clip into geometry the vanilla camera never visits.

What body is shown: `wh_horse_HideReinsWhenMounted` suggests reins are a
toggleable separate attachment; combined with the `AnimationPoseModifier_
CartHorseReinsModifier` and `BridlePoseModifier` RTTI classes, rein and bridle
visuals are pose-modifier driven, not simple meshes, so a VR hand reaching for
reins would need to interact with that modifier rather than a static
attachment point. The first-person body while mounted is presumably the same
arm rig used on foot, posed into a riding blend space (confirmed present:
`blendSpace/horse`, 34 files, `animals/horse/new/blendSpace`, 28 files,
measured from the Animations.pak listing), so whatever arm-IK rework ships for
on-foot VR has to also cover this blend space.

### What VR needs to add here

A seated VR mode on horseback is the most promising "free head" case in the
project: the camera is already decoupled with centering and sway parameters,
the per-frame look/aim IK target channel already exists, and
`wh_horse_EnableRiderHeadCollision` shows the game already reasons about where
the rider's head can go. The missing piece is getting HMD orientation, and
ideally small positional sway, into that existing `S_MountAnimState`
look-target channel and into whichever offset resolves the mounted camera
transform, rather than building a new camera path from scratch.

## 3. Cutscenes

### Three distinct mechanisms (measured)

(a) Bink video: a `Videos/` tree of `.bk2` files (confirmed via the extracted
pak listing, e.g. `Videos/m11/cin_m1140t_nebakovobrana__god_finger_clip01.bk2`)
plus matching `.srt` subtitles for several sequences. These are full-motion
video, rendered by the Bink decoder DLL in `Win64Shared`, with zero camera
control surface; a VR mod can only present the frame on a flat plane in front
of the headset ("virtual cinema screen") and cannot free the player's head
inside it, because there is no 3D scene to look around in. `mov_NoCutscenes`
("Disable playing of Cut-Scenes") exists as an escape hatch to skip these
outright.

(b) In-engine cinematics: `Animations/humans/male/cinematics/<cin_name>/` and
`.../female/cinematics/universal` directories (measured from the Animations.pak
and Cinematics.pak listings, e.g. `cin_m5020k_obranabohuta__siege_finale_p02`,
50 files) hold per-sequence Mannequin animation sets. These run through the
Movie system (`IMovieSystem`/`IAnimSequence`, confirmed in libKCD2's CryCommon
headers, with `SetActiveCamera`, `BeginCutScene`/`EndCutScene`, `OnSetCamera`,
`IsSequenceCamUsed`) and through native GUI-module wrapper classes: `I_Cutscene`,
`C_TrackViewCutscene`, `C_IngameCutscene`, `C_FaderCutscene`,
`C_RenderedCutscene`, `C_SkipTimeCutscene`, `C_CreditsCutscene`,
`C_TextCutscene`, plus `C_CutsceneHolder`/`C_CutsceneHandler` on the entity
side. Relevant cvars, measured with help text from KCD2ModLoader's cvars.md:
`mov_cameraPrecacheTime` (default 4), `mov_enableFakeProjection` ("Enabled
fake projection in cutscenes"), `mov_maxTimeStepForMovieSystem` ("Caps the
time step for the movie system so that a cut-scene won't be jumped in the case
of an extreme stall"), `mov_overrideCam` ("Set the camera used for the
sequence which overrides the camera track info in the sequence", default
empty), and `g_cutsceneSkipDelay`. `mov_overrideCam` is the clean VR hook
here: it lets a mod substitute its own camera entity for the sequence's
authored track without touching the sequence data, so VR could drive that
entity's position from the authored track while driving its rotation from
the HMD.

(c) Dialog cameras: a distinct system, `wh::dialogmodule::C_DialogCamera`
/ `C_DialogCameraManager` / `I_DialogCameraManager`, with
`E_DialogCameraType` and `data::E_CameraCommandType` enums (confirmed as RTTR
enumerations, so both reflected and script/data drivable) and a
`C_CameraCommand` object with a settable `E_CameraCommandType` property.
Dialog also carries `C_LookAtCommand` and `C_PointAtCommand`, so NPC gaze and
pointing are already command-driven and independent of the camera command,
which matters for VR: a freed camera will not desync NPC eye-contact logic,
since that logic targets the player's entity position, not the camera.

### The directly relevant VR cvar (measured)

`demo_use_hmd_rotation`, default `1`, help text "Uses alternative entity and
view rotation for HMD Devices". This is a genuine leftover CryEngine VR hook,
default-on and named for exactly the "replace the sequence's authored rotation
with the live HMD's" behaviour this project wants. No `IHmdDevice`/`IHmdManager`
/`sys_vr` interface was found anywhere in the binary (confirmed absent in
`kcd2-local-findings.md`), so its consumer almost certainly checks for an HMD
object that no longer exists and does nothing today; a lead to disassemble,
not a working feature, measured for existence and default, inferred for
behaviour. `g_detachCamera`, default `0`, "Detach camera", is the blunter
alternative: fully decouple the camera at the cost of authored framing.

### Practical VR strategies and HUD (measured + inferred)

For in-engine cinematics and dialog, in order of intrusiveness: use
`mov_overrideCam` (or the dialog equivalent through `C_CameraCommand`
/`C_DialogCameraManager`) to keep the authored camera position while
substituting HMD rotation; revive whatever `demo_use_hmd_rotation` used to
gate, which may already do exactly this; or fall back to `g_detachCamera`
when blocking matters less than comfort. Bink video has no in-scene camera to
free; the only option is projecting it onto a virtual screen. No
`letterbox`/`blackbar` string or flowgraph node was found anywhere, so
letterboxing almost certainly renders as a Scaleform overlay toggled by the
cutscene's own fader/holder logic rather than a dedicated cvar; `hud_hide` is
the general HUD toggle. In VR this overlay needs to be disabled or
re-rendered as a world-locked plane, since 2D screen-space bars make no sense
once there is a stereo view and a free-moving head.

## 4. Native plugin surface

### Two established loader families (measured, via READMEs)

KCSE ("Kingdom Come Script Extender") ships as a `dinput8.dll` proxy, no exe
patching, with an SKSE-style plugin API: `KCSE_PLUGIN_LOAD`, a messaging
interface with lifecycle events (`PreDataLoaded`, `DataLoaded`, `LoadGame`,
`SaveGame`, `NewGame`, `AllPluginsLoaded`), a per-frame task queue, and its own
trampoline. KCD2ModLoader ships as a `d3d12.dll` proxy on ReturnOfModdingBase,
Lua-first with hot reload and Dear ImGui bindings, but also hosts generic ASI
plugins, FMOD hooking, and pak/XML interception. TPVCamera and TPVToggle are
plain ASI plugins, loaded by either loader's ASI path, built on the
third-party DetourModKit library, independent of both proxies.

### Offset resolution and healing (measured, from TPVCamera source)

TPVCamera does not hardcode vtable slot numbers; it resolves every hook and
read target through a declarative, cascading AOB (array-of-bytes pattern)
registry, confined to the `WHGame.dll` image range, never a whole-process
scan, to avoid false matches in other injected modules. Each anchor is an
ordered list of pattern candidates, most-specific first, so a patch
disturbing one falls through to the next; a pattern matching more than once
in scope is treated as ambiguous and skipped rather than guessed. Resolution
shapes are `Direct` (entry or mid-body address) and `RipRelative` (resolves a
`lea`/`mov [rip+disp]` to the data it addresses). Struct field offsets inside
the resolved objects are separately self-healing: each offset is keyed to the
MSVC-mangled RTTI name of the object its slot should point at, and a small
window around the nominal offset is rescanned at runtime if that slot stops
resolving to the right type, through DetourModKit's
`rtti::HealedSlot`/`HealScheduler`. This is stated to resolve correctly,
unmodified, on Steam 1.5.6, GOG 1.5, and Game Pass 1.4. RTTI identity checks
are cached once per vtable pointer, keyed to an image-generation stamp so a
reload or remap forces a cold re-resolve.

libKCD2 takes a related but distinct approach: every vtable slot and RTTI
descriptor is addressed through `REL::ID`, an abstraction over an external
id registry (the pattern CommonLibSSE uses for Skyrim), auto-generated from
the live binary by its own generator scripts. Its README states plainly that
"CryEngine's interfuscator shuffles vtable order in release builds," which is
why neither tool hardcodes raw slot numbers: libKCD2 regenerates its
id-to-offset table per build, and TPVCamera resolves offsets live via AOB and
RTTI instead of trusting any fixed layout.

### The exact TPVCamera hook and its gate (measured)

TPVCamera hooks `CCamera::UpdateFrustumPlanes`, the function that turns a
camera's 3x4 matrix (confirmed by header: `CCamera::m_Matrix` at `+0x00`,
`m_fov` at `+0x30`) into world-space cull planes (`m_fp[6][4]` at `+0x114`),
offsetting the matrix before the planes derive from it so the rendered view
and its culling move together; offsetting only afterward was tried first and
produced visible near-geometry culling artifacts, per the source comments.
Every gameplay camera, including combat and mount, funnels through this one
function for the active `CView`, so one hook covers all of them. Because the
same builder is shared by shadow, reflection, and portal cameras, the detour
gates on the `CView` embedding carrying the `CView` vtable (via the RTTI
identity cache). A companion hook on the head-visibility setter keeps the
player's head rendered once pulled back to third person, and a separate
input-dispatcher hook powers free-look orbit.
RTTI identities enumerated for gating: `C_Player`, `CView`,
`CAnimatedCharacter`, `C_CameraCombatDelegate`, `C_CameraDialog`,
`C_CameraManager`, `C_MissileWeaponPlayerController`, `C_ActorModel`,
`C_CameraObserver`, `CTimer`, `CAnimationSet`, `CD3D9Renderer`.

### Relevant libKCD2 member offsets already resolved (measured)

`C_Player` (`sizeof 0xE00`) carries `m_pRiderPlayerControl` at `+0xD68` and an
embedded `C_MissileWeaponPlayerController` at `+0xD70`. `C_Horse`
(`sizeof 0xA60`) carries `m_pHorseData` at `+0x9E8` and an embedded
`S_MountAnimState` at `+0x9F0`. These are cross-checked with
`static_assert(offsetof(...))` in the headers, so they are measured, not
speculative. `IMovieSystem` and `IFlashUI` exist as separate resolved vtables,
confirming both are reachable the same way `CView` and `CCamera` are; no
VR-specific work was found layered on either.

## Summary table

| Subsystem | What exists today | What a VR mod must add | Confidence |
|---|---|---|---|
| FP arm/weapon pose | Mannequin pose-modifier stack (LimbIk, IkCcd, IKTorsoAim, PoseBlenderAim/Look) plus `C_WeaponSolver.SetIKTargets` with collision/recoil/offhand cvars | Feed controller hand targets into or alongside WeaponSolver without fighting its correction | Measured existence; inferred integration |
| Weapon attachment | `C_ItemAttachmentManager` + Draw/Holster/Swap fragment requests; `Bip01`/`BONEID_HAND_L/R` skeleton | Trigger the same fragments so VR draw/holster gestures still move the mesh | Measured; socket strings inferred (likely in CDF) |
| Melee combat | Fragment-synced attacker/defender system with position/rotation alignment clips, own combat camera | Map controller swing to attack-fragment choice, not literal IK takeover | Measured RTTI surface; inferred failure mode |
| Minigames | Cursor-driven views (lockpick, forge, alchemy) plus archery's AimIK-friendly blend spaces | Map controller motion onto the same cursor inputs; archery adapts cleanest | Measured; inferred mapping |
| Horse camera/body | Shared follow/orbit camera with horse centering/sway/view-limit/head-collision cvars; native per-frame look/aim-IK channel (`S_MountAnimState`) feeding "C_Actor vf250" | Feed HMD orientation into that existing look-target channel instead of a new camera path | Measured header offsets and comment; inferred live-hook mechanics |
| Bink cutscenes | Flat video, zero camera control, `mov_NoCutscenes` escape hatch | Project onto a virtual screen; head cannot be freed inside the video | Measured |
| In-engine cutscenes | `IMovieSystem`/TrackView, native cutscene wrapper classes, `mov_overrideCam` to substitute the sequence camera | Substitute camera entity: keep authored position, replace rotation with HMD | Measured cvar/help text |
| Dialog cameras | `C_DialogCameraManager`/enum system, independent LookAt/PointAt NPC gaze commands | Same position-keep/rotation-replace strategy; gaze stays correct since it targets the player entity | Measured; command choice inferred |
| VR rotation leftover | `demo_use_hmd_rotation`, default on, "alternative entity and view rotation for HMD Devices"; no live HMD interface found | Disassemble its consumer: dead code worth reviving, or a naming coincidence | Measured existence/default; inferred behaviour |
| Letterbox/HUD | No cvar/flow node found; likely a Scaleform overlay on cutscene fader/holder classes; `hud_hide` general toggle | Disable or re-render as world-locked geometry | Inferred |
| Native plugin loading | KCSE (dinput8 proxy), KCD2ModLoader (d3d12 proxy), plain ASI plugins; TPVCamera resolves hooks via cascading AOB with RTTI-keyed self-healing offsets; libKCD2 regenerates an id table per build | Ship as an ASI/KCSE plugin using the same AOB+RTTI-heal approach, not hardcoded offsets | Measured from source |

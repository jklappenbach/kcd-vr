# kcd-vr: a VR conversion of Kingdom Come: Deliverance II

Status: draft, 2026-10-15. Research behind every requirement is in
`research/` (see `research/README.md`).

## 1. Definition

**1.1 Purpose.** Turn Kingdom Come: Deliverance II into a room-scale VR
game: stereo rendering to an OpenXR headset, six-degree-of-freedom head
tracking that moves both the view and Henry's head, and motion controllers
that drive Henry's arms through inverse kinematics on his real skeleton.
It must hold up on foot, on horseback, in dialog and through cutscenes.

**1.2 Problem.** The game ships no VR. The one existing option, Luke
Ross's R.E.A.L. VR, gives head tracking with alternate-eye rendering and
keeps the flat game's hands, animations and camera. Nothing gives the
player a body.

**1.3 Shape of the solution.** The game is a closed CryEngine 5.x fork
(`WHGame.dll`, 89 MB, DirectX 12 only, no SDK, no symbols, RTTI intact).
The mod is a native plugin loaded into the game process through a proxy
DLL. It hooks the engine's camera, swap chain and animation pose-modifier
stack, renders both eyes per frame by re-entering the engine's own view
render, submits through OpenXR on D3D12, and drives Henry's skeleton from
headset and controller poses. Game logic stays in the game's Lua and
data where it can, following the script-native split the Titanfall 2 VR
mod used.

**1.4 Platform.** The primary target is Windows on the RTX 4090 and
Ryzen 3950X box: native D3D12, SteamVR's OpenXR runtime, MSVC. Headset
and development both live there. Until a headset is attached,
development runs on SteamVR's null driver. Linux through Proton
(vkd3d-proton, wineopenxr) is the second target, kept working by a late
portability unit and never allowed to drive a design choice. The Strix
Halo box is the low-end performance reference only.

**1.5 Constraints.**
- No Warhorse asset or code is redistributed. Modified assets ship as
  hash-verified binary patches applied to the user's own files.
- The unmodified game must keep working. The mod installs into its own
  profile and save directory.
- Every hook is found by signature and RTTI, never by a fixed offset, and
  heals across game patches or fails loudly by name.
- DirectX 12 only. No D3D11 or Vulkan path in the game.
- Multiplayer is out of scope, but the design drives the real skeleton
  rather than a separate first-person view model, so a future networked
  body is possible.

**1.6 Non-goals.** Kingdom Come: Deliverance I. Consoles. Rewriting the
combat system. Making Bink videos three-dimensional. Hand tracking
without controllers in the first release.

## 2. Loading and lifecycle

- **2.1** When the game starts with the mod's proxy DLL present, the mod
  loads inside the game process before the renderer initializes, logs its
  version, and does nothing else until every required hook has resolved.
- **2.2** When any required hook fails to resolve, the mod reports the
  hook by name, leaves the game flat, and never patches memory partially.
- **2.3** When a game update changes code around a hook, the signature
  registry tries each fallback pattern in order and the RTTI-anchored
  offset healer rescans field offsets, and the outcome is logged per hook.
- **2.4** When the OpenXR runtime is missing, inactive or lacks D3D12
  support, a pre-launch probe says so in plain words and the game stays
  flat.
- **2.5** When the game crashes or hangs, an out-of-process monitor writes
  a minidump and the last telemetry lines to the profile directory.
- **2.6** When launched through the mod's launcher, saves and settings go
  to a separate profile directory and the flat game's saves are untouched.

## 3. Stereo rendering

- **3.1** When a frame is rendered in VR, both eyes are rendered from the
  same simulation step and the same game-time, with no gameplay update
  between eye passes.
- **3.2** When each eye is rendered, the camera uses the asymmetric
  projection and pose the runtime reported for that eye, and the culling
  frustum matches it.
- **3.3** When shadows are rendered, the shadow cascades are computed once
  per frame for a frustum covering both eyes and reused by the second eye.
- **3.4** When particles, cloth and other view-dependent simulations
  update, they update once per frame, not per eye.
- **3.5** When an eye finishes, its color image is handed to the OpenXR
  swapchain for that eye as a D3D12 resource, without a CPU copy.
- **3.6** When the runtime provides a hidden-area mesh, it is drawn before
  the scene for each eye so occluded pixels cost nothing.
- **3.7** When VR is active, depth of field, motion blur, chromatic
  aberration, film grain, screen vignette and head bob are off, and
  temporal anti-aliasing jitter is per eye and stable.
- **3.8** When VR is active, the game's own vsync and frame cap are off
  and frame pacing follows the runtime's predicted display time.
- **3.9** When the desktop window is visible, it shows one eye as a mirror
  at low cost.
- **3.10** When the user changes render scale, each eye renders at that
  fraction of the runtime's recommended size without a restart.
- **3.11** When the renderer offers lens-matched or multi-resolution
  shading, the mod can enable it per eye.

## 4. Head and camera

- **4.1** When the player moves their head, the view follows with six
  degrees of freedom and the latency of one frame or less from pose read
  to projection.
- **4.2** When the player moves their head, Henry's head and neck bones
  follow on the real skeleton so shadows, mirrors, third-person views and
  any future observer see the same head motion.
- **4.3** When the player's real head leaves the capsule the game
  animates, the body stays where the game put it and the view is offset,
  up to a configurable limit after which the body is pulled along.
- **4.4** When the player recenters, the tracking origin is reset to the
  current head pose without changing Henry's position in the world.
- **4.5** When seated mode is selected, eye height is taken from a
  calibration and the floor is not required.
- **4.6** When the player turns with the stick, turning is either snap at
  a configurable angle or smooth at a configurable rate.
- **4.7** When the game moves the camera for its own reasons (combat
  camera, focus, dash FOV kick, shake, noise), those effects are
  suppressed and the head stays the sole source of rotation.
- **4.8** When the game sets a field of view, the runtime's per-eye field
  of view wins.

## 5. Body and arms

- **5.1** When a controller moves, the corresponding arm on Henry's real
  skeleton reaches the controller pose through two-bone inverse
  kinematics with an elbow hint, every frame, with the hand bone matching
  controller orientation.
- **5.2** When arms are driven by the mod, the game's first-person arm and
  weapon animations are suppressed for the bones the mod owns, while the
  game's animation remains the rest pose and blend source for bones it
  does not own.
- **5.3** When Henry holds an item, the item attachment follows the hand
  bone so the item is where the controller is.
- **5.4** When the player first enters VR, arm span and standing height
  are calibrated and stored, and the calibration can be redone from the
  menu.
- **5.5** When a controller reports finger values, the hand pose blends
  between open and closed per finger.
- **5.6** When the player's hand reaches toward a holster or sheath and
  grips, the game's draw or holster fragment runs so the weapon mesh moves
  correctly between hand and scabbard.
- **5.7** When the player's arms are driven but the game's weapon solver
  runs, the solver either takes the mod's targets or is disabled for the
  owned bones, never fighting them.
- **5.8** When the player is not in VR, none of the above changes
  anything.

## 6. Interaction and combat

- **6.1** When the player looks at or points at an interactable, the
  interaction prompt attaches to it and a controller button activates it.
- **6.2** When Henry is in combat, the mod holds the game's attack zone
  input every frame from the position of the weapon hand (or the centroid
  of both hands) relative to the body, using the zone geometry the game's
  combat tables define, so the rosette shows the zone the player is
  physically in.
- **6.3** When the player begins a swing, the zone sampled at the start of
  the swing is the attack zone, not the zone the controller passes
  through, and the attack action is posted once the sweep velocity
  crosses a threshold.
- **6.4** When the blade is pointed forward at the start of the motion
  and the hand's velocity is along the blade, the motion is a stab and is
  posted as the game's stab attack type.
- **6.5** When an attack posts, the fragment plays with the mod's arm pose
  blended toward the fragment's weapon path, so the game's attacker and
  defender alignment keeps working, and the arms return to the mod when
  the fragment ends.
- **6.6** When the player presses block, or raises a guard pose toward
  the incoming zone inside the opponent's attack window, the block action
  is posted. Perfect block, riposte and master strike then follow the
  game's own rules (timing for perfect block, any attack for riposte,
  an attack from the opposite zone during the opponent's attack for a
  master strike) with no VR-specific logic.
- **6.7** When the player steps the body past a threshold, the dodge
  action is posted in that direction.
- **6.8** When a missile weapon is held (the game's one family covering
  bow, crossbow and hand cannon), the aim direction is the weapon's own
  muzzle or arrow line taken from the controller poses, not the head: the
  weapon hand's grip orientation, refined by the line through both hands
  when the off hand grips the stock, fore-end or bow. The shot origin
  and direction the game uses are redirected to that line.
- **6.8.1** When the player draws a bow, the bow hand sets the line and
  the string hand's pull distance drives draw and release.
- **6.8.2** When the player fires a crossbow or hand cannon, the trigger
  fires with the game's own ignition delay, the game's stamina and skill
  spread remain as a cone around the weapon line rather than as view
  sway, and the recoil is a brief pose kick with a haptic pulse.
- **6.8.3** When a hand cannon is loaded, the game's loading animation
  owns the arms for its duration and hands them back when done.
- **6.8.3.1** When a crossbow is loaded, the player performs the spanning
  sequence the weapon's loading type implies: tilt the crossbow down and
  brace it, span the string with the off hand (a lever pull along the
  stock for a goat's foot, a cranking motion for a cranequin, a straight
  pull for a belt hook or stirrup), take a bolt from the quiver (6.8.6)
  and lay it in the groove. Each step is a hand volume plus a motion, and
  the sequence cannot finish faster than the game's loading time for that
  crossbow, so skill and weapon type still matter. A setting falls back
  to the game's loading animation.
- **6.8.4** When the player enables the aiming aid, a faint line shows
  the weapon's direction; by default there is no reticle.
- **6.8.5** When a hand cannon is held in the weapon hand alone, it is
  braced and aimed by that hand, and the off hand carries the match.
  Bringing the match hand into the touch-hole volume posts the fire
  action and the game's ignition delay runs, with the trigger kept as an
  alternative. Moving the match hand never changes the aim line.
- **6.8.6** When a bow is held, an arrow is taken by reaching the string
  hand into the quiver volume, which follows the quiver attachment
  wherever the game mounts it on Henry, and gripping. The arrow appears
  in that hand, and bringing it to the string nocks it, after which 6.8.1
  applies. A setting restores automatic nocking for players who prefer
  it. The same reach takes a bolt for crossbow loading when the gesture
  loading option is on.
- **6.9** When a minigame starts (lockpicking, forge, alchemy, dice),
  controller motion maps onto that minigame's cursor-style input and the
  game's minigame arm poses are allowed to own the arms for its duration.
- **6.10** When the player would use the mouse, a controller ray drives
  the cursor instead.

## 7. Horseback

- **7.1** When Henry mounts, the view sits at the rider's eye position on
  the riding animation and the head stays free in all six degrees within
  the rider head collision limits the game already tracks.
- **7.2** When the player looks around on horseback, the game's rider
  look target follows the headset so Henry's head and torso turn with it.
- **7.3** When riding, the controllers steer and the reins and bridle pose
  modifiers follow the hand positions, or the reins are hidden if hands
  are elsewhere.
- **7.4** When the horse gallops, the dash field-of-view kick and camera
  sway are suppressed.
- **7.5** When fighting from horseback, section 6 applies.

## 8. Cutscenes and dialog

- **8.1** When a Bink video plays, it is shown on a virtual screen at a
  fixed comfortable distance in a dimmed environment, and skipping works.
- **8.2** When an in-engine cinematic plays, the view takes its position
  from the authored camera track and its rotation from the headset, so
  framing is kept and the head is free.
- **8.3** When a dialog camera is active, the same position-from-game,
  rotation-from-head rule applies, and NPC gaze still targets Henry.
- **8.4** When the player prefers, a setting switches cinematics to a flat
  virtual-screen mode or to a fully free camera.
- **8.5** When a cutscene shows letterbox bars or fades, bars are removed
  and fades apply to the whole view.
- **8.6** When an in-engine cinematic starts, the mod reads its authored
  camera track through the movie system and builds a jump plan: a list
  of vantage points with times. Shot starts are vantage points, a dolly
  contributes further points where the authored view has moved or turned
  past a threshold, and a zoom becomes a point moved along the view axis
  so the subject keeps the authored angular size. The view never moves
  continuously; it cuts from one vantage point to the next with a short
  blink, position and reference yaw snapping together, pitch and roll
  kept level.
- **8.6.1** When a per-sequence override file names its own vantage
  points, those replace the derived ones, so sequences can be curated by
  hand.
- **8.6.2** When a development flag is set, every cinematic's derived
  jump plan and the authored track are dumped to a file for review.
- **8.7** When a cinematic plays, the mod stops driving Henry's head and
  arms so the sequence's animation acts him, and resumes when it ends.
- **8.8** When embodied mode is selected, in-engine cinematics and
  dialogs place the view at Henry's eye bone as the animation moves it,
  with rotation from the headset, under the same comfort rules as 8.6.
- **8.9** When a cinematic cuts between shots, the cut is kept as an
  instant change of position covered by a short fade.

## 9. HUD and menus

- **9.1** When the game draws its Scaleform HUD, it renders into an
  offscreen texture, not into the eye images.
- **9.2** When VR is active, the HUD texture is shown as a head-locked
  panel at a fixed distance, with crosshair and screen-space markers
  removed and world markers placed at their world positions.
- **9.3** When a full-screen menu, map, inventory or codex opens, it is
  shown on a large world-locked panel and a controller ray drives the
  cursor.
- **9.4** When subtitles are on, they are shown on the HUD panel.
- **9.5** When the HUD would obstruct combat, it can be hidden per
  element.

## 10. Input

- **10.1** When controllers are present, an OpenXR action set binds
  movement, turn, interact, attack, block, draw, grip, menu, map,
  recenter and sprint, with bindings for Index, Touch and Vive wands.
- **10.2** When head tracking is active, the game's mouse and stick look
  are disabled and any action map that moves the camera is filtered out.
- **10.3** When the player prefers, a gamepad can replace controllers for
  everything except hands.
- **10.4** When the game expects a key or button, the mod posts the
  engine's own input events so the game's action maps stay in charge.
- **10.5** When an action fires that has a haptic cue, the matching
  controller pulses.

## 11. Comfort

- **11.1** When the player moves or turns artificially, a configurable
  vignette narrows the view and keeps the nasal side open per eye.
- **11.2** When the player sprints, climbs, falls or is knocked down, the
  camera stays head-driven and only a vignette or fade is added.
- **11.3** When the player stands or sits, the chosen mode's height
  calibration applies.

## 12. Performance

- **12.1** When the mod runs on the development box, it holds the
  runtime's refresh rate with reprojection at a documented render scale
  in the town benchmark route, and the number is recorded.
- **12.2** When the frame is late, render scale adapts within configured
  bounds before anything else degrades.
- **12.3** When requested, the mod logs per-phase timings per frame
  (pose read, eye setup, eye passes, HUD, submit) to a CSV.

## 13. Platforms

- **13.1** When run on Windows with SteamVR, no Linux-specific code path
  is required and nothing in the build depends on Proton.
- **13.2** When run under SteamVR's null driver, every feature except
  real tracking works, so the full suite runs without a headset.
- **13.3** When run under Proton with the documented launch options, the
  proxy DLL loads, OpenXR reaches the host runtime through wineopenxr,
  and D3D12 swapchain images cross to Vulkan without a copy.
- **13.4** When run on the Strix Halo box, the measured frame table in
  section 12 includes it as the low-end reference.

## 14. Diagnostics

- **14.1** When any hook is applied, the hook's name, address and pattern
  index are logged.
- **14.2** When a hooked site is found changed at runtime, the mod stops
  touching it and reports the name.
- **14.3** When the mod runs, a telemetry CSV per subsystem can be enabled
  with stable column names.

## 15. Development bench

- **15.1** When a hook or a pose-modifier injection is designed, it is
  first prototyped against a build of the public CryEngine 5.x source
  with symbols, and the prototype carries a test.
- **15.2** When a prototype moves to the shipped binary, the bench test
  becomes an in-game telemetry assertion.

## 16. Distribution

- **16.1** When a user installs, an installer finds the game, pins the
  loader version by hash, applies asset patches against the user's own
  files after verifying them, and writes a launcher into the game folder.
- **16.2** When a user uninstalls, the game folder returns to its
  original state.
- **16.3** When the mod updates, user settings and calibration survive.

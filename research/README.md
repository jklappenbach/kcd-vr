# Research index

Written 2026-10-15. Reports in this directory are the evidence behind
`specs/kcd-vr-spec.md`. Cloned third-party trees are gitignored.

| File | What it holds |
|---|---|
| `kcd2-local-findings.md` | Measured facts about the installed KCD2 binary, paks, cvars, surviving classes, and this machine |
| `kcd2-animation-horse-cutscene.md` | Pose modifiers, weapon solver, combat, minigames, horse data, cutscene and dialog camera systems, native plugin surface |
| `titanfall2-vr-mod.md` | How the Titanfall 2 VR mod is built, from its installer source and package |
| `titanfall2-vr-package-analysis.md` | Deep pass on the package: natives, IK telemetry, stereo sequence, patch sites, shaders |
| `cryengine-prior-art-and-kcd2-modding.md` | fholger's CryEngine mods, R.E.A.L., vorpX, KCD2 loaders, Proton plumbing |
| `cryengine-vr-architecture.md` | The public CryEngine 5.x stereo, HMD, camera, DX12, Flash, IK, input and movie paths, cross-checked against WHGame.dll |
| `unreal-xr-reference.md` | Unreal's XR stack as a checklist of concerns only |

Cloned trees: `cryengine` (public 5.x mirror), `crysis_vrmod`,
`farcry_vrmod`, `KCD2Tools`, `KCD2ModLoader`, `libKCD2`, `KCSE-for-kcd2`,
`kcd2-modding-docs`, `CircuitLordVRModInstaller`, and the mod package under
`packages/`. `kcd2-extracted/` holds game Lua and engine configs pulled
from the local install for grepping.

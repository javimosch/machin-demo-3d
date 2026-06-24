---
name: machin-demo-3d
description: Build, run, and modify machin-demo-3d — a real-time 3D scene (orbiting camera, cubes, grid) rendered from machin (MFL) via raylib's C FFI. Use when working on this repo, or as the worked example of 3D in machin — nested cstructs (Camera3D/Vector3), BeginMode3D, and reaching libm for trig.
---

# machin-demo-3d

A real-time 3D scene rendered natively from [machin](https://github.com/javimosch/machin) (MFL) through raylib. It is the reference example for **3D** in machin (the 2D/audio siblings are [flappy](https://github.com/javimosch/machin-game-flappy) / [simon](https://github.com/javimosch/machin-game-simon)).

> Shared game-dev setup, build-and-verify workflow, and cross-cutting gotchas live in the canonical **[machin-gamedev skill](https://github.com/javimosch/machin/blob/main/skills/machin-gamedev/SKILL.md)**. This file is the 3D specifics.

## Build & run

```bash
./build.sh                 # machin encode scene.src -> scene.mfl, then machin build -> ./machin-demo-3d
./machin-demo-3d
```

Needs `machin` **v0.45.0+** (nested cstructs), a C compiler, **raylib**, and a display. `build.sh` prefers a system raylib, else vendors the prebuilt static release into `vendor/` (no root).

## 3D over the FFI: nested cstructs

3D needs a **by-value struct of by-value structs** — `Camera3D` holds three `Vector3`s. Before machin v0.45.0 a `cstruct` field had to be a numeric scalar, so this couldn't be declared. Now it can:

```
cstruct Vector3  { x f32 y f32 z f32 }                                  // declare the inner struct FIRST
cstruct Camera3D { position Vector3 target Vector3 up Vector3 fovy f32 projection i32 }
fn BeginMode3D(Camera3D)   fn EndMode3D()
fn DrawCube(Vector3, f32, f32, f32, Color)   fn DrawCubeWires(Vector3, f32, f32, f32, Color)
fn DrawGrid(i32, f32)      fn DrawSphere(Vector3, f32, Color)
```

- **Construct with nested literals:** `Camera3D{Vector3{12.0,7.0,0.0}, Vector3{0.0,1.5,0.0}, Vector3{0.0,1.0,0.0}, 45.0, 0}`. `projection` `0` = `CAMERA_PERSPECTIVE`.
- **Order matters:** declare `Vector3` before `Camera3D` so the inner marshaler is emitted first.
- machin marshals the whole thing recursively across the FFI; a small `v3(x,y,z)` helper returning `Vector3` keeps call sites readable.
- `BeginMode3D(cam)` … 3D draws … `EndMode3D()`; then any 2D `DrawText` is screen-space.

## Math: native builtins

The orbit uses machin's **native math** (v0.46.0): `sin`, `cos`, and `pi()` (`TAU() { n = 2.0 * pi() }`). The full libm suite is built in — `sin cos tan asin acos atan atan2 sqrt cbrt pow exp log log2 log10 floor ceil round trunc abs fmod hypot` and `pi()` — numeric in, `float` out, with `-lm` linked only when used.

This demo originally reached libm via `extern "m" { header "math.h" link "m" fn sin(float) float ... }` — that workaround is exactly what drove native math into the language. The extern form still works (and an `extern` of the same name shadows the builtin), but you no longer need it.

## Patterns worth copying

- **Rebuild the camera each frame** rather than mutating it: `cam := Camera3D{v3(12.0*cos(a), 7.0, 12.0*sin(a)), ...}`. Simple and avoids nested-field assignment.
- **Everything is `float`.** Keep loop indices `int` and cross to float explicitly: `float(k) * (TAU()/float(RING()))` (machin has no implicit `int`→`float`; see the gamedev skill).
- **Verify headlessly:** run backgrounded on `DISPLAY=:0`, screenshot with ImageMagick `import -window root` at two different times to confirm the camera orbited (no keystroke injection available).

## Modifying

- **Scene:** `RING` (cube count), `RADIUS`, the `12.0`/`7.0` camera distance/height, the `0.012` orbit speed, the bob amplitudes/phases.
- **Shapes:** swap `DrawCube` for `DrawSphere`, add `DrawCubeWires` outlines, more grid slices.
- **Per-object rotation** uses raylib's immediate-mode matrix stack (rlgl). Those functions live in `rlgl.h`, not `raylib.h`, but their symbols are in `libraylib.a`, so a **headerless** extern block is enough (machin emits the prototypes; no new machin feature): `extern "rlgl" { fn rlPushMatrix() fn rlPopMatrix() fn rlTranslatef(f32,f32,f32) fn rlRotatef(f32,f32,f32,f32) }`. Then `spin_cube` does push → translate to world pos → rotate (degrees) around its axes → draw at the local origin → pop. The same stack scales (`rlScalef`) and works in 2D between `BeginDrawing`/`EndDrawing`.
- After any edit to `scene.src`, re-run `./build.sh` (never hand-edit `scene.mfl` — it is generated).

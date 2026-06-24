# machin-game-demo-3d

A real-time **3D scene** rendered natively from **[machin](https://github.com/javimosch/machin)** (MFL) through raylib's C FFI — a camera **orbits** a ring of bobbing colored cubes on a grid, with a pulsing cube at the center. No textures, no engine: just the language driving OpenGL.

Part of [**awesome-machin**](https://github.com/javimosch/awesome-machin) — the machin ecosystem.

> **Agents:** [`SKILL.md`](SKILL.md) covers the build (incl. no-root raylib), the 3D FFI surface, and the nested-cstruct / math notes.

```
            ▢                machin 3D — native OpenGL via the C FFI
        ▢       ▢
            ▢                a Camera3D (three Vector3s) orbits the scene;
        ▢       ▢            cubes bob on sine phases over a 3D grid
     ───────────────────
```

## Why it exists

The machin north star is "build real things, let usage drive features." This is the **3D** dogfood — the leap past the 2D games ([snake](https://github.com/javimosch/machin-game-demo-snake) / [2048](https://github.com/javimosch/machin-game-demo-2048) / [flappy](https://github.com/javimosch/machin-game-demo-flappy) / [simon](https://github.com/javimosch/machin-game-demo-simon)).

3D is gated on a **by-value struct of by-value structs**: raylib's `Camera3D` holds three `Vector3`s. machin couldn't express that — `cstruct` fields had to be numeric scalars — so it drove **nested cstructs** into the language (**machin v0.45.0**):

```
cstruct Vector3  { x f32 y f32 z f32 }
cstruct Camera3D { position Vector3 target Vector3 up Vector3 fovy f32 projection i32 }
fn BeginMode3D(Camera3D)
...
cam := Camera3D{Vector3{12.0, 7.0, 0.0}, Vector3{0.0, 1.5, 0.0}, Vector3{0.0, 1.0, 0.0}, 45.0, 0}
BeginMode3D(cam)        // a nested struct, constructed and passed by value
```

The orbit math uses machin's **native `sin`/`cos`/`pi`** (**v0.46.0**) — which this demo's earlier `extern "m"` workaround drove into the language, completing the loop.

## Build

Needs the `machin` compiler (**v0.45.0+**), a C compiler, **raylib**, and a display (X11/desktop). A GUI binary links the system graphics stack, so it is **not** a no-dependency binary.

```bash
./build.sh            # → ./machin-game-demo-3d
./machin-game-demo-3d
```

`build.sh` uses a **system raylib** if installed (`sudo apt-get install libraylib-dev`, `brew install raylib`, …); otherwise it **vendors raylib's prebuilt static release** into `vendor/` automatically — no root required.

## How it works

- **Camera.** Each frame a fresh `Camera3D` is built with the camera position on a circle (`12·cos(a)`, `12·sin(a)`) around the scene, looking at a point just above the origin. `BeginMode3D(cam)` / `EndMode3D()` bracket the 3D draws.
- **Scene.** `DrawGrid` for the floor; a ring of `RING` cubes placed with `cos`/`sin` at radius `RADIUS`, each bobbing on its own sine phase **and spinning in place**; a center cube pulsing and tumbling.
- **Per-object rotation.** raylib's immediate-mode matrix stack (rlgl: `rlPushMatrix`/`rlTranslatef`/`rlRotatef`/`rlPopMatrix`) — those live in `rlgl.h`, but the symbols are in `libraylib.a`, so a **headerless** `extern "rlgl"` block (machin emits the prototypes) is enough. No new machin feature; the existing scalar FFI carries it.
- **2D overlay.** After `EndMode3D`, plain `DrawText` for the title (screen space).
- All coordinates are `float`; the trig comes from libm (`extern "m"`).

See [`scene.src`](scene.src). `build.sh` runs `machin encode` to produce the canonical `scene.mfl`, then `machin build`.

## License

MIT

# Banana Native on the AYN Thor — bring-up notes

Everything we did to run Pokémon Pokopia and Donkey Kong Bananza on the AYN Thor (QCS8550,
Adreno 740, Android 13) from [moceure/BananaNative](https://github.com/moceure/BananaNative),
`android` branch at commit `1b59eff`. This tree carries the changes; nothing here is committed
upstream.

Owner machine map: the Thor is reachable over adb **through aurora** (`ssh aurora` →
`adb -s 192.168.1.55:37849 …`); the source, packs and dumps live on aurora under
`~/Projects/webtop-private/webtop/config/Projects/` (the same tree we see as `/config/Projects`).

---

## 1. What is deployed on the device

| Item | Path on the Thor |
|---|---|
| APK (test app `dev.nxnative.android`) | installed from `android/app/out/nxnative-android.apk` |
| Pokopia game folder | `/sdcard/Switch2/pokopia` (`base/exefs`, `updates/655360/{exefs,romfs}`) |
| Pokopia pack | `/sdcard/Switch2/0400A3A01EE4C000-work/android-pack` |
| DK Bananza game folder | `/sdcard/Switch2/dkb` |
| DK Bananza pack | `/sdcard/Switch2/040074A01BF12000-work/android-pack` |
| Saves (test app) | `/sdcard/Switch2/saves-test/<title ID>` |
| Host env switches | `/sdcard/Android/data/dev.nxnative.android/files/env.txt` |
| Live control file | `/sdcard/Android/data/dev.nxnative.android/files/ctl.txt` |
| Run logs | `/sdcard/Android/data/dev.nxnative.android/files/logs/<id>-<date>.log` |

Current `env.txt`:

```
DM_PAD_LABELLED=1
DM_WINDOW_SIZE=1280x720
DM_VK_FG_REFRESH_CAP=0
DM_LSFG=1
DM_LSFG_MULT=3
DM_TZ=Europe/Rome
```

`env.txt` lines are applied **after** the launcher's own values, so they win over the Settings UI.

## 2. Build and deploy

```sh
# build (both profiles compiled in, on-device packer replaced by a fake one: packs are made on the host)
cd android/app && ./gradlew -PnxGames=040074A01BF12000,0400A3A01EE4C000 -PnxPackEngine=fake apk
# → android/app/out/nxnative-android.apk

# install (adb lives on aurora)
ssh aurora "adb -s 192.168.1.55:37849 install -r <path>/nxnative-android.apk"

# pack a game (host side, needs python with capstone 5 + lz4)
python scripts/pack_android.py <dump folder> [--shader-cache <dir with v-<chain>/>] -j 4

# push a pack — USE A TRAILING SOURCE SLASH, or adb nests a directory one level deeper
ssh aurora "adb -s 192.168.1.55:37849 push <pack>/. /sdcard/Switch2/<id>-work/android-pack/"
```

Launch through the launcher (the game activity is not exported):

```sh
adb shell am start -n dev.nxnative.android/.MainActivity \
  --es game /storage/emulated/0/Switch2/pokopia \
  --es pack /storage/emulated/0/Switch2/0400A3A01EE4C000-work/android-pack --ez run true
```

Gotchas:
* `adb push <dir> <existing dir>` creates `<dir>/<dirname>` — the first pack push silently landed one
  level deep and the game kept loading the old shader library.
* The game **pauses at boot while the screen is off** (`>>> nnosInitialize` then nothing, 0 % CPU).
  Wake the device before concluding anything is hung.
* A reinstall kills the session and drops back to the title screen.

## 3. GPU driver, frame generation, input

* **Turnip T30 (Mr. Purple)** is the driver to use — installed through the launcher
  (`MainActivity --es gpu_driver_zip <zip>`) and picked in the prefs (`gpu_driver`). Measured on
  Pokopia: GPU 46.6 → 29.8 ms, replay recording 45.9 → 1.9 ms versus the whitebelyash "v32" build;
  the KIMCHI v26 R8-GMEM build was worse (51 ms).
* The stock Qualcomm driver cannot run these games: it lacks `VK_EXT_map_memory_placed` /
  `VK_KHR_map_memory2`, so the readiness screen fails and nothing is drawn.
* **A/B labels**: `DM_PAD_LABELLED=1` maps the pad's face buttons by label instead of by position
  (without it A and B are swapped on the Thor).
* **Frame generation (LSFG)**: needs a `Lossless.dll` that carries the Vulkan shaders (SPIR-V in
  RCDATA resources 255, 256, 280-302). Lossless Scaling 3.2.2 works (7,521,280 B, "25 shaders"
  in the log); the 3.2.1 DLL we tried first is DXBC-only and is refused.
  `DM_VK_FG_REFRESH_CAP=0` is needed for Pokopia: the game asks for a 60 Hz present interval while
  running at ~20 fps, and the default cap then allows zero generated frames.
  Measured: 20 game fps → 38 presented at ×2, ~52 at ×3.

## 4. User clock override (`dm/host/hle_time.c`, `dm/host/main.c`)

Same idea as Eden's "set the time":

| Switch | Meaning |
|---|---|
| `DM_TIME=YYYY-MM-DDTHH:MM[:SS]` | the user clock starts there (local wall-clock time in `DM_TZ`, else the system zone) and advances with the monotonic clock |
| `DM_TIME_OFFSET=<±seconds>` | the host clock, offset |
| `DM_TZ=<Area/City>` | the zone `ToCalendarTime` converts with |
| `DM_FIXED_TIME=<epoch>` | still wins: a fixed clock for deterministic runs (and pins `TZ=UTC`) |

Also in the launcher: **Settings › System › Time of day** — system date picker → system time picker,
validated as `YYYY-MM-DDTHH:MM` with a strict parse (impossible dates refused), stored in the prefs
(`time_override`) and handed to the host as `DM_TIME` at the next start.

## 5. Save migration (from eglantine)

The native host uses the console's own `S00/G00/B00` layout, so an eglantine nand save drops in
as-is:

```
/sdcard/Android/data/dev.eglantine.eglantine_emulator/files/nand/user/save/0000000000000000/<profile>/0400A3A01EE4C000
  → /sdcard/Switch2/saves-test/0400A3A01EE4C000
```

Verified: the save loads and continues, no conversion. A copy is staged on aurora at
`…/config/Projects/dumps/pokopia-save/0400A3A01EE4C000` (24 MB).

## 6. Shader libraries

* Partial library (the 8,157 keys the Mac playtest used, `dm/games/<id>/shader_seen.tsv`): what the
  deployed pack carries. Build:
  `vk_android_build.py --target vulkan <pkg> <libdir> $(keys)`.
* **Full library**: 131,426 shaders for Pokopia. A full build only became possible after the format
  change below; it is running now into `…/0400A3A01EE4C000-work/android-shaders/lib/v-deedfab57192/`.
  Progress at the time of writing: **55,062 done, 76,364 left** (~3.5-4 h at ~5.8 shaders/s), results
  5.4 GB → the finished `dmsh_vk.lib` is expected around 6.5 GB.
* The Switch shader package (`shader_split.py` output) for Pokopia is kept at
  `…/0400A3A01EE4C000-work/android-shaders/pkg` (2.0 GB) so the split never has to run again.

### Library format v2 (64-bit offsets)

`DMSH_VK_LIB_VERSION 2`, `DMSH_VK_MOD_SIZE 24`; the module record's blob offset is now `uint64_t`:

```
uint64_t off; uint32_t size, stored; uint8_t variant, codec; uint16_t reserved; uint32_t reserved1;
```

Writer (`vk_android_build.py`, `write_library`) packs `<QIIBBHI`; the reader (`vka_shlib.c`) uses
`sizeof(dmsh_vk_module)` and refuses format 1 with a clear message. Changing `dmsh_abi_vk.h` moves
`DMSH_VK_CHAIN` (now `deedfab57192`), which invalidates cached per-shader results — that is why the
full build has to translate from scratch.

## 7. Performance findings (Pokopia)

Measured in-world, same save, Turnip T30:

| configuration | frame p50 | GPU | replay recording |
|---|---|---|---|
| Turnip "v32" (whitebelyash) | 51.6 ms | 46.6 ms | 45.9 ms (357 µs/draw) |
| Turnip v26 R8 GMEM | 59.4 ms | 51.0 ms | 5.2 ms |
| **Turnip T30** | 50.5 ms | **29.8 ms** | **1.9 ms** (15 µs/draw) |
| T30 + LSFG ×3 | 50-58 ms | - | - (≈52 presented) |

* The frame is **submit-bound**: 11 NVN submissions per frame, each ~1.5 ms of driver wait
  (`submit+present` ≈ 47 ms while the GPU needs 29.8). That is why 20 fps and not 30 (the game's
  controller needs frame-start→GPU-end under 33.3 ms).
* Skipping half of the per-frame compute dispatches drops GPU 33.8 → 10.6 ms and frames to ~28-32 ms
  with a pixel-identical picture (bedroom: two `1×1×1` dispatches held 22 ms) — proof that the cost
  is per-submission/per-dispatch wait, not fill rate.
* `DM_WINDOW_SIZE` does nothing for Pokopia: the profile has no `window_fn` row, so the game keeps
  its own 1920×1080. Patching the display-size table (`0x2849658`/`0x28496e0`, entries 30-33) and
  adding a `window_fn` row did not move the GPU at all — the render-size decision is elsewhere.
  Both experiments were reverted; `tools/patch_res.py` in the pokopia-android tree still does the
  table patch if someone wants to continue.
* Perf regression to remember: two `getenv()` calls placed in the per-draw path cost 13 ms of
  recording on a 1,373-draw frame (~5 fps). All per-draw debug switches are now read **once a frame**
  (`dbg_keys()` in `vka_draw.c`).

## 8. The black sunny sky (open)

Symptom: with clear weather the sky area is pure black (clouds, terrain, shadows, tone map and the
final pass are all correct). Cloudy weather looks right. Reproducible per in-game date.

Ruled out, with evidence:

1. **Missing shaders** — zero load failures, 383 graphics + 28 compute pipelines, 0 refused.
2. **The five per-frame compute dispatches** (`70ab27e1`, `8699f064`, `2118ab53` = luminance
   histogram, `d0c17f1f`, `2c74cdd9`) — skipping each leaves the sky black.
3. **"An indirect draw of quads … is not on Vulkan yet: skipped"** (`vs d5a2f5694fca455b`,
   `ps 7c95c3ae4fa11142`) — support was added (expansion + a timeline wait before reading the
   record) and the record then reads `vertexCount = 0, instanceCount = 1`: NVN draws nothing there.
   Red herring; the new code is still correct and stays.
4. **A draw covering the sky** — every candidate full-screen draw was replaced by a magenta fill
   (`DM_VK_DRAW_FILL_KEY`), 27 shaders in two passes over the list: terrain, ground and foliage
   change, the sky never does. Only the final full-screen composites touch the sky, and they touch
   the whole screen.
5. NaN scan (`DM_VK_NANSCAN=1`) — too destructive to read; it probes guest memory and blows the
   frame out.

Conclusion: the sky region of the scene target is **never written**; the composite samples a target
holding zeros. This is a missing/never-filled producer pass, not a shading term — and it only shows
in clear weather because that is when the game builds or uses that target.

**UPDATE 2026-11-05 — the Mac host is correct and names the producer.** The same save and packed
files, with the clock pinned to noon, render the daytime sky (blue gradient + white clouds) on the
Mac's Metal host; the night scene and rain are correct too. There is no separate producer pass: the
sky is the ordinary first draw of the scene MRT pass 161705, a normal triangle list:

```
pass 161705  vs 6867ce14b7e03d20_b  fs e27ae8ee1d53fe83_b  prim triangles  count 5898
             writes 5435 (RGBA8_sRGB c0), 5523 (RGBA16 c1), 5524 (RGBA8 c2/c4),
             5525 (R11G11B10F c3); the pass loads, it does not clear
```

Both keys are in the Android library and translate (`index.json` has them with no notes). The sky
pixel shader samples four 1D textures, `fp_t_tcb_E`, `fp_t_tcb_8`, `fp_t_tcb_A` and `fp_t_tcb_C`:
the sky's colour ramps. That is the same shape as the Metal-side
[linear CPU-written sky ramps](../reference/nvn-objects.md#linear-textures-in-cpu-visible-pools)
finding, so the two remaining branches are:

1. the draw is not issued, or is skipped on Android (a state/pipeline/depth decision), or
2. it runs and the four ramp textures read zero (texture first-use / linear-upload path).

Next step, on the phone in clear weather, no new code needed:
* `DM_VK_DRAW_LOG=1` and grep `e27ae8ee` / `6867ce14`: is the draw issued at all?
* `DM_VK_DRAW_FILL_KEY=e27ae8ee1d53fe83`: the pass area turns magenta if the draw executes; no
  change means it is skipped before it draws.
* `DM_VK_ZEROLOG=32` at spawn: lists textures made as zero images while their storage holds data,
  with their flags, pool flags and first-use mode — this identifies the four ramps if branch 2.

The result picks the branch. The fix then belongs in the draw-skip path or the texture first-use /
linear-upload path, not in the sky shader. The Mac run already rules out the game data, the shader
sources, the clock and the weather state.

Suspects worth checking first when that exists: the `80x45` LUT-ish pass, the `2x1` exposure target,
the `2048²` shadow/GI targets, and the 6-vertex full-screen quad `671da2822a04218c` /
`5ea49da9886ab233` (drawn at 80x45, 480x270 and 1920x1080).

## 9. Debug switches added in this tree

| Switch | What it does |
|---|---|
| `DM_VK_CLEAR_TEST=1` | every colour clear becomes magenta (tells "never cleared" from "cleared black") |
| `DM_VK_ZEROLOG=<n>` | lists the first `n` textures made as zero images while their storage holds data (flags, pool flags, first-use mode) |
| `DM_VK_BAR_DISPATCH=0` | no full memory barrier after every compute dispatch (default 1) |
| `DM_VK_QUAD_INDIRECT=1` | read an indirect quad/line-loop draw's record on the CPU and expand it (default 0: the draw is skipped, see §8) |
| `DM_TIME`, `DM_TIME_OFFSET`, `DM_TZ` | user clock override (section 4) |

All of the `DM_VK_*` ones are live through `ctl.txt` except where noted in the code, and are read
once a frame. The per-draw log and magenta-fill switches (`DM_VK_DRAW_LOG`, `DM_VK_DRAW_FILL_KEY`,
`DM_VK_DRAW_SKIP_KEY`) are gone with the black sky hunt they were written for; the sky is fixed by
the `ReadTexels` and buffer-texture support (`DM_VK_READTEXELS`, `DM_VK_TBUF`).

## 10. Files changed in this tree

```
android/app/src/main/java/dev/nxnative/android/{GameActivity,MainActivity,Prefs}.java   time-of-day UI + DM_TIME
dm/gfx/vk_android/vka_draw.c        indirect quads, per-frame switch cache; the new shared-memory
                                    spill keeps its barrier even with DM_VK_BAR_DISPATCH=0
dm/gfx/vk_android/vka_exec.c        DM_VK_CLEAR_TEST
dm/host/hle_time.c, dm/host/main.c  DM_TIME / DM_TIME_OFFSET / DM_TZ
```

Rebased 2026-11-05 onto upstream `origin/android` `cf69ca3b`. The earlier library-format 2/3 edits
and the local `dmsh_chain{,_vk}.h` / `dmsh_chain.lock` are gone: upstream's writer (format 1, high
offset words, streaming blob) supersedes them. Upstream also moved the Vulkan shader chain to
`31b81e5c07ac`, so a pack built for `36c8ed92799a` is obsolete; the host copies of that library were
removed and the library is built again on the device.

## 11. Other games

Donkey Kong Bananza (`040074A01BF12000`, update v458752) was extracted from a `.nup` container with
`pokopia-android/tools/nup_dump.py` (eglantine's NUP format), packed and runs on the same APK: mode
select at 60 fps (GPU 1.5 ms), intro/gameplay ~16 fps, no red lines. Its profile has a `window_fn`
row, so `DM_WINDOW_SIZE=1280x720` does apply there.

## 12. Reverts and the automatic full-library deploy

Two things were tried and **reverted** because they cost frame time, with the code left behind a
switch:

* the per-draw `getenv()` pair (13 ms of recording) — the switches are read once a frame now;
* the indirect-quad expansion with its queue wait (idle stall, draws nothing) — `DM_VK_QUAD_INDIRECT=1`.

When the full shader library finishes, `/tmp/opencode/deploy_when_ready.sh` (running in the
background, log `/tmp/opencode/deploy_full.log`) repacks Pokopia keeping the rewritten modules
(`pack_android.py --keep-code`), drops the partial library from the pack, pushes it and relaunches
the game. Nothing about it needs a person; the only manual step left would be re-installing the APK.

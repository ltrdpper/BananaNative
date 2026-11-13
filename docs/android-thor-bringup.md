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
dm/gfx/tools/dmsh_xlate_android/dmsh_xlate_android.csproj  JSON metadata for the Android AOT translator
dm/host/hle_time.c, dm/host/main.c  DM_TIME / DM_TIME_OFFSET / DM_TZ
```

Rebased 2026-11-05 onto upstream `origin/android` (then `cf69ca3b`, now v0.1.5 `243bae31`). The
earlier library-format 2/3 edits and the local `dmsh_chain{,_vk}.h` / `dmsh_chain.lock` are gone:
upstream's writer (format 1, high offset words, streaming blob) supersedes them. Upstream also moved
the Vulkan shader chain to `31b81e5c07ac`, so a pack built for `36c8ed92799a` is obsolete. The
finished library was built on a Linux host, not on the phone; see §13.

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

## 13. Building the Vulkan shader library on a Linux host (2026-11-06)

This is the path that produced the working `v-31b81e5c07ac/dmsh_vk.lib` for Pokopia. The phone's
Prepare was not used for the final library: the on-device split died on
`0x38f972af.fdata` (2.56 GB, 25,023 IDRK entries, 125,950 of the 131,429 shaders) with
`MemoryError`, and the release Android AOT translator later failed 80,236 shaders with
`NotSupportedException: NoMetadataForType`. A Linux x86_64 host has the RAM and the full
metadata, and the result is the same library format the phone loads.

Tools used: .NET SDK 10.0.403, Python 3.14.7 with `zstd` and `lz4`, `spirv-val` (SPIRV-Tools
v2026.1), the dump's RomFS and ExeFS, and a host with enough RAM to split the big KTGL file.

### 13.1 Build the Linux translator

```sh
dotnet publish dm/gfx/tools/dmsh_xlate/dmsh_xlate.csproj \
  -c Release -r linux-x64 --self-contained true -p:PublishSingleFile=true \
  -o /tmp/xlate-linux
/tmp/xlate-linux/dmsh_xlate --source-sha     # 0f31637014ce4725
```

`vk_android_build.py` fingerprints `dmsh_xlate.dll`, `Ryujinx.Graphics.Shader.dll` and
`Ryujinx.Common.dll` against `dmsh_chain.lock`, and a fresh publish has the matching embedded
source sha but different managed dll hashes. For a local build, copy the locked dlls from
`dm/gfx/tools/dmsh_xlate_bin/` beside the Linux binary and make `dmsh_xlate` a symlink to it:

```sh
cp dm/gfx/tools/dmsh_xlate_bin/{dmsh_xlate.dll,Ryujinx.Graphics.Shader.dll,Ryujinx.Common.dll} /tmp/xlate-linux/
ln -sf dmsh_xlate-linux /tmp/xlate-linux/dmsh_xlate
/tmp/xlate-linux/dmsh_xlate --source-sha     # still 0f31637014ce4725
```

The Mac dlls are only there for the fingerprint check; the translation runs the Linux binary.
This is a local validation workaround, not a claim that the Mac dlls run on Linux. A proper
upstream change would update the lock from a real Linux build, which moves `DMSH_CHAIN` and
therefore the library chain; that is why it was not done while the library was being produced.

### 13.2 Split the dump on the host

The phone cannot split `0x38f972af.fdata`; the host can. The split must use the same dump and
profile as the pack:

```sh
DM_GAME=0400A3A01EE4C000 J=8 python3 dm/gfx/tools/shader_split.py \
  <dump>/updates/655360/romfs <dump>/updates/655360/exefs/main \
  /tmp/pokopia-host/pkg \
  /tmp/pokopia-host/shader_map.tsv /tmp/pokopia-host/shader_facts.tsv \
  /tmp/pokopia-host/shader_pairs.tsv /tmp/pokopia-host/pso_pairs.tsv
```

Result: 131,426 shaders; `/tmp/pokopia-host/shader_map.tsv` was byte-identical to the profile's
`dm/games/0400A3A01EE4C000/shader_map.tsv` (`sha256 7ce177ac...`).

### 13.3 Build and check the library

```sh
DM_GAME=0400A3A01EE4C000 \
XLATE=/tmp/xlate-linux/dmsh_xlate SPIRV_VAL=/tmp/spirv-val \
J=32 DMSH_VK_XLATE_J=32 \
python3 dm/gfx/tools/vk_android_build.py --target vulkan \
  /tmp/pokopia-host/pkg /tmp/pokopia-host/cache

SPIRV_VAL=/tmp/spirv-val python3 dm/gfx/tools/vk_android_build.py \
  --check /tmp/pokopia-host/cache/v-31b81e5c07ac/dmsh_vk.lib
```

On the host build used here (32 translator calls, about 12 hours):

```
library: game 0400A3A01EE4C000, inputs c456869c588b, chain eda8d97a233b, vk chain 31b81e5c07ac
dmsh_vk.lib: 131410 shaders, 172617 modules; every module decompresses and passes
spirv-val --target-env vulkan1.3
```

131,410 is the full library: 16 shaders are the known untranslatable `BRX with no target found in
the c1 jump table` pixel shaders from `0x38f972af.fdata`; the old working library had the same
count. `inputs.tsv` must say `game 0400A3A01EE4C000`, `inputs c456869c588b`.

### 13.4 Put it in the pack

```sh
adb push dmsh_vk.lib  <pack>/shaders/v-31b81e5c07ac/dmsh_vk.lib
adb push inputs.tsv   <pack>/shaders/v-31b81e5c07ac/inputs.tsv
```

The app then loads the library directly; no phone split or translation is involved. The Pokopia
title screen and the game started with this library, and `--check` passed on all 172,617 modules.
The phone's `Prepare` job can be forgotten after this; the pack is self-contained.

### 13.5 The phone AOT problem, and the future fix

The release APK's `libdmsh_xlate.so` is NativeAOT and trimmed. `JsonNode.ToJsonString` needs
`System.Text.Json` reflection metadata, so 80,236 shaders failed with `NoMetadataForType` on the
phone while the same shaders built on the host JIT translator. `dm/gfx/tools/dmsh_xlate_android/
dmsh_xlate_android.csproj` now keeps that metadata for Android builds:

```xml
<JsonSerializerIsReflectionEnabledByDefault>true</JsonSerializerIsReflectionEnabledByDefault>
<IlcGenerateCompleteTypeMetadata>true</IlcGenerateCompleteTypeMetadata>
```

This file is not part of `DMSH_CHAIN`, so the chain and the host-built library stay valid. A fresh
Android `libdmsh_xlate.so` built with those properties grew from 4.3 MB to 10.3 MB and translated
new shaders without the metadata error; it is only slower than the host, not broken.

## 14. Rebuilding every library for a new translator chain (2026-11-08)

Taking upstream PR #77 (five translator fixes) moves `DMSH_CHAIN` `eda8d97a233b` ->
`53bc9e4b138b`; the tool check below moves `DMSH_VK_CHAIN` `31b81e5c07ac` -> `8be027e51579`.
Every pack's library is then stale, and the Thor cannot rebuild Pokopia's (split `MemoryError`,
AOT JSON). So all three were rebuilt here, on the host, before deploying: the phone never
translates anything.

### 14.1 The Linux translator, this time from the new sources

`docker pull mcr.microsoft.com/dotnet/sdk:10.0` (no .NET SDK on aurora or in this container), then

```sh
docker run --rm -v /config/Projects/banananative:/w -v /tmp/opencode:/out \
  -w /w/dm/gfx/tools/dmsh_xlate mcr.microsoft.com/dotnet/sdk:10.0 \
  dotnet publish -c Release -r linux-x64 --self-contained true -o /out/xlate77
/tmp/opencode/xlate77/dmsh_xlate --source-sha      # 7a1bdd0a6c230374, the lock's xlate_source
```

The §13.1 trick (copy the locked Mac dlls beside the Linux apphost) only works while the
translator source is unchanged: the apphost loads `dmsh_xlate.dll`, so those dlls run the Mac
build's IL. Once the source changes, the new dlls must be used, and the Mac dll fingerprints in
`dmsh_chain.lock` cannot match. `vk_android_build.py` now checks the embedded source digest on a
host that is not the Mac (`sys.platform != 'darwin'`), which is the real chain input; the dll
fingerprints stay a Mac-only check. That is commit `05da5dd7`, together with the header.

### 14.2 Splitting each dump (host RAM, not the phone)

`shader_split.py` takes RomFS, `exefs/main`, the package dir and the three tables. Dump sources
used here: Pokopia `pokopia-android/dump/updates/655360`, DKB `dumps/Donkey Kong Bananza.nup`
(update 458752), MKW `dumps/Mario Kart World [0400C3F00006E000].nup` (update 917504, pulled from
the Pi's HDD5 at `/srv/dev-disk-by-uuid-f5aba79e-.../Buckets/Downloads/Switch2/`). A `.nup` is
read in place through `nup.py`, no extraction:

```sh
DM_GAME=040074A01BF12000 J=8 python3 dm/gfx/tools/shader_split.py \
  "dumps/Donkey Kong Bananza.nup:update/458752/romfs" \
  "dumps/Donkey Kong Bananza.nup:update/458752/exefs/main" \
  dumps/dkb-shaders/pkg dumps/dkb-shaders/shader_map.tsv dumps/dkb-shaders/shader_facts.tsv \
  dumps/dkb-shaders/shader_pairs.tsv dumps/dkb-shaders/pso_pairs.tsv
```

The map each split writes must be byte-identical to the profile's `shader_map.tsv` (`cmp`); it
was for all three (131,426 / 22,852 / 11,604 shaders).

### 14.3 Building and checking

```sh
DM_GAME=<id> XLATE=/tmp/opencode/xlate77/dmsh_xlate SPIRV_VAL=/usr/bin/spirv-val \
  J=32 DMSH_VK_XLATE_J=32 python3 dm/gfx/tools/vk_android_build.py --target vulkan \
  dumps/<game>-shaders/pkg dumps/<game>-shaders/lib
SPIRV_VAL=/usr/bin/spirv-val J=16 python3 dm/gfx/tools/vk_android_build.py \
  --check dumps/<game>-shaders/lib/v-8be027e51579/dmsh_vk.lib
```

| game | shaders | build time (32 jobs) | library |
|---|---|---|---|
| DKB | 22,852 | 7 min | 391 MB |
| MKW | 11,603 | 4 min | 191 MB |
| Pokopia | 131,410 of 131,426 | ~7 h | ~10 GB |

The 16 missing Pokopia shaders are the known untranslatable `BRX with no target found in the c1
jump table` ones. Every library reported `game ... chain 53bc9e4b138b, vk chain 8be027e51579`
and passed `--check` for every module (Pokopia: 172,617 modules).

### 14.4 Deploying a new chain

The APK must be built from the same tree (it carries both chain strings). Then for each game:

```sh
adb shell mkdir -p /sdcard/Switch2/<id>-work/android-pack/shaders/v-8be027e51579
adb push dmsh_vk.lib inputs.tsv /sdcard/Switch2/<id>-work/android-pack/shaders/v-8be027e51579/
adb shell rm -rf /sdcard/Switch2/<id>-work/android-pack/shaders/v-31b81e5c07ac   # 10 GB back, unused
```

Old-chain libraries are never loaded again, so deleting them is safe; the pack keeps its code,
`game.tsv` and the new library. `results/` (the per-shader build cache, GBs) can be deleted after
the build; only `dmsh_vk.lib` and `inputs.tsv` are read by the game.

## 15. The upstream moved to git.eden-emu.dev (v0.2.1, 2026-11-08)

`github.com/moceure/BananaNative` and the account were deleted during the day; the project now
lives at `https://git.eden-emu.dev/moceure/BananaNative` with a **rewritten history** (no commit
is shared with the old GitHub one, so a rebase is a hand port, not a replay). Tags `v0.2.0` and
`v0.2.1` are there; `android` is `d231cb0a`.

v0.2.0 already contains the 18 PRs this tree had cherry-picked, Ocarina of Time and Splatoon
Raiders profiles, PC/Mac packing, incremental shader updates, and the translator's NativeAOT
JsonNode fix (in `Program.cs`, so the csproj metadata workaround is no longer needed).

Our branch is `android-v020` = `upstream/android` + the changes upstream does not have:

| change | files |
|---|---|
| buffer textures as uniform texel buffers (tbuf slot → `VkBufferView`) | `dm/gfx/vk_android/vka_tex.c`, `vka.h`, `vka_draw.c` |
| time-of-day setting (`DM_TIME`, `DM_TZ`, `DM_TIME_OFFSET`) | `hle_time.c`, `main.c`, launcher java |
| replay switches: indirect quads from their record, `DM_VK_BAR_DISPATCH`, `DM_VK_CLEAR_TEST` | `vka_draw.c`, `vka_exec.c` |
| translator check on a non-Mac host (embedded source digest instead of Mac dll hashes) | `dm/gfx/tools/vk_android_build.py` |

Dropped: the Ocarina profile (upstream's is canonical), the csproj JsonNode metadata workaround
(fixed in `Program.cs`), and PR #88's PAC emulation (not carried; the Thor has PAC).

Chains on this branch: `DMSH_CHAIN` `6eaf449c7ce1`, `DMSH_VK_CHAIN` `6847ec0acc39` (the last
differs from upstream's `0035e16073bf` only because `vk_android_build.py` was changed). The
libraries are built like §13/§14, with a translator from this tree:

```sh
docker run --rm -v /config/Projects/bn-v020:/w -v /tmp/opencode:/out -w /w/dm/gfx/tools/dmsh_xlate \
  mcr.microsoft.com/dotnet/sdk:10.0 dotnet publish -c Release -r linux-x64 --self-contained true -o /out/xlate-v020
/tmp/opencode/xlate-v020/dmsh_xlate --source-sha     # c55667af1b257a46, the lock's value
```

then `DM_GAME=<id> XLATE=/tmp/opencode/xlate-v020/dmsh_xlate SPIRV_VAL=/usr/bin/spirv-val J=32
DMSH_VK_XLATE_J=32 python3 dm/gfx/tools/vk_android_build.py --target vulkan dumps/<game>-shaders/pkg
dumps/<game>-shaders/lib`, and push `dumps/<game>-shaders/lib/v-6847ec0acc39/` into
`<pack>/shaders/`. The old chain dir (`v-31b81e5c07ac`) stays for rolling back to the
`deployed-thor-20261108` APK.

Backups: `ltrdpper/BananaNative` on GitHub (fork made before the deletion) and
`HDD5/Buckets/banananative-backup-2026-11-08/` on the Pi (bundle of every ref, 26 PR patches,
the deployed APK).

## 16. v0.2.4 merged, all four libraries built, Ocarina on the Thor (2026-11-08, night)

Upstream published 26 more commits while the first v0.2.1 libraries were building: v0.2.3 (FSR 1,
frame pacing, pointer authentication emulation **with the `dm_host_has_pac` gate** our #88 report
asked for, Mario Kart World's memory-map abort) and v0.2.4, plus PRs #7, #8, #9, #12, #15 and #32
(guest memory through exported FDs on drivers without placed maps, AIBinder stub for Qualcomm
891.x). They do not change `DMSH_CHAIN` (`6eaf449c7ce1`). Merged into `android-v020` as `d0417e0b`;
the only conflict was the `MainActivity` settings-row field list (both sides added rows).

`vk_android_build.py` itself changed upstream, and our non-Mac check is a change to it, so the
tools digest moved `0035e16073bf` -> `874cdbe93ff8`. Rather than rebuild, `vk_android_same.tsv`
maps that digest to our `6847ec0acc39` after a **full MKW rebuild with the merged tools came out
byte-identical** (`sha256 1b4afa9f…e3d1`), which is what that table is for.

Libraries on chain `6eaf449c7ce1` / vk `6847ec0acc39`:

| game | shaders | library | build |
|---|---|---|---|
| Pokopia | 131,410 of 131,426 | 5.6 GB, 176,715 modules | ~9 h (with DKB/MKW running alongside) |
| DKB | 22,852 | 259 MB | 9 min |
| MKW | 11,603 | 132 MB | 5 min |
| Ocarina of Time | 42,837 of 42,848 | 680 MB | 35 min |

Pushed as `shaders/v-6847ec0acc39/` into each pack; the previous chain's `v-31b81e5c07ac` stays
for rolling back to the `deployed-thor-20261108` APK.

**Pipeline cache after a library change.** The driver cache is stamped with the shader library,
and a new library makes it invalid: the first Pokopia run after the chain change reported
`pipeline cache … is not stamped with its shader library: 0 bytes read` and spent 52 s building
413 graphics + 42 compute pipelines (compute ones 1.2-3.7 s each on Turnip T30), which is what the
stutter was. The second launch read 17.4 MB back (314 shaders, 0 stale) and built 12 pipelines.

**Game cards.** The icon, version and publisher come from the `.nup` meta block (`Nup.gameMeta`), so
a folder dump shows none: pushing `Pokemon-Pokopia.nup` to `/sdcard/Switch2/` gave Pokopia its
cover. That file's meta block carries only the icon, so its version/publisher stay blank (DKB's and
MKW's carry all three).

**Ocarina of Time.** Its `main` is ZBIC-compressed, so `shader_split.py` and `pack_android.py` need
`DM_ZBIC_LIB` (built with `dm/third_party/zstd-zbic/build.sh /tmp/libzbicdec.so cc`) or the `zbic`
pip package; Python 3.14 here had no pip, `get-pip.py --user --break-system-packages` fixes that
(also needed for `capstone` and `lz4` by the packer). The pack is 782 MB
(`dumps/0400486024DEC000-work/android-pack`) and the game reads its RomFS from the 25.6 GB `.nup`
in place, so that file has to be on the device too.

Cleanup done on the Thor: pipeline caches of the other drivers (Mainline v32, Mesa v26) and logs
beyond the newest three were removed; the old chain library was kept.

## 17. v0.2.7 merged, selected Pokopia build, old chains removed (2026-11-10)

Upstream published 31 more commits (v0.2.5-0.2.7): band-granular texture re-uploads and WriteTexels
subregions, dirty-page tracking that follows mprotect, mem_map_limit for every game, movies on
their own decode thread (AV1 on the phone's decoder), one prewarm compile at a time, a cached
present format query, a held early ANativeWindow surface, gyro start after the first resume, 480p,
MKW update dumps, and translator fixes found on Ocarina.

Chains: `1cf10a4252df` (translator, upstream) and `d99b8cfa5ad7` (Vulkan; upstream's is
`643797bdfbd9`, ours differs only by the non-Mac translator check). Libraries rebuilt for all four
games, then all older chain dirs removed from the packs.

**Pokopia no longer builds 131k shaders.** The profile has a selected set (`shader_seen.tsv`,
8,164 lines) and a key-list build translates only those plus their paired pixel shaders:

```sh
DM_GAME=0400A3A01EE4C000 XLATE=/tmp/xlate-v027/dmsh_xlate SPIRV_VAL=/usr/bin/spirv-val J=32 \
  python3 dm/gfx/tools/vk_android_build.py --target vulkan dumps/pokopia-shaders/pkg dumps/pokopia-shaders/lib \
  $(python3 - <<'PY'   # shader_seen keys + their paired pixel shaders (dmsh_build.build_keys)
...
PY
)
```

Result: 8,161 shaders, 9,945 modules, 228 MB, `--check` clean, in about 30 minutes instead of 9
hours. `pack_android.py` still builds the full set on the host (it passes no key list); the phone's
Prepare passes the selection itself.

| game | shaders | library | notes |
|---|---|---|---|
| Pokopia | 8,161 (selected) | 228 MB | profile inputs unchanged |
| DKB | 22,852 | 216 MB | |
| MKW | 11,603 | 109 MB | |
| Ocarina of Time | 42,848 | 714 MB | profile tables changed (inputs `0a7aeafde2d0`): re-split from the nup, `DM_ZBIC_LIB` again |

Deployment: APK built from `bd023503` (branch head) with four profiles; `shaders/v-d99b8cfa5ad7/`
pushed into each pack; `v-6847ec0acc39` and `v-31b81e5c07ac` removed from the device. The old
`v-31b81e5c07ac` library was first pulled to `dumps/old-chain/0400A3A01EE4C000/`, so the archived
`deployed-thor-20261108` APK can still be rolled back to.

Cleanup: per-chain `results/` build caches and the old Ocarina split package deleted on the host;
device logs trimmed to the newest three; `/sdcard/Switch2/{pokopia,dkb}` extracted folders had
already gone (the packs read their ROMFS from the `.nup`s).

## 18. Redoing this: merge upstream, rebuild on this host, deploy (runbook)

Everything below was done on aurora (this container) on 2026-11-08..11. Paths are the ones this
tree uses. The repositories: `origin` = `https://git.eden-emu.dev/moceure/BananaNative.git` (the
project moved there from GitHub on 2026-11-08; its history was rewritten, so a merge, not a rebase),
`mr` = `git@github-banananative:ltrdpper/BananaNative.git` (our push target, key
`~/.ssh/banananative`). Work happens on branch `android-v020` in `/config/Projects/bn-v020`.

### 18.1 What we carry on top of upstream

Only these; everything else upstream has:

| change | where |
|---|---|
| buffer textures as uniform texel buffers (tbuf slot → `VkBufferView`) | `dm/gfx/vk_android/vka_tex.c`, `vka.h`, `vka_draw.c` |
| time-of-day setting (`DM_TIME`, `DM_TZ`, `DM_TIME_OFFSET`) | `hle_time.c`, `main.c`, launcher java |
| indirect quads / line loops from their record (`DM_VK_QUAD_INDIRECT`), `DM_VK_CLEAR_TEST` | `vka_draw.c`, `vka_exec.c` |
| Italian as the default game language | `GameLanguage.java` |
| translator check by embedded source on a non-Mac host | `dm/gfx/tools/vk_android_build.py` |

`DM_VK_BAR_DISPATCH` is gone: upstream now has `DM_VK_DISPATCH_ORDER` and only barriers for spills
and the NaN scan.

### 18.2 Merge

```sh
cd /config/Projects/bn-v020
git fetch origin --tags
git rev-list --count HEAD..origin/android          # what is new
grep -m1 DMSH_CHAIN  <(git show origin/android:dm/gfx/nvn/dmsh_chain.h)
grep -m1 DMSH_VK_CHAIN <(git show origin/android:dm/gfx/nvn/dmsh_chain_vk.h)
git merge --no-edit origin/android
```

Expected conflicts, and how they were resolved:

- `dm/gfx/nvn/dmsh_chain_vk.h`: take upstream's (`git checkout --theirs`), then regenerate ours in
  18.3 (our tool edit changes the digest).
- `dm/gfx/vk_android/vka_draw.c`: keep our tbuf case and the quad block; for the dispatch barrier
  take upstream's side (`disp_count` + `DM_VK_DISPATCH_ORDER`), never both.
- The launcher java (`MainActivity` field list, `GameLanguage`): keep both sides.

Then commit the merge (`--no-edit`).

### 18.3 Chains and the translator

`DMSH_CHAIN` comes from source files only, so it changes when the translator changes. `DMSH_VK_CHAIN`
also covers `vk_android_build.py`, `vk_android_spv.py`, `vk_android_delta.py`, `dmsh_abi_vk.h`, so
**our non-Mac check alone makes it differ from upstream's**; that is fine, the APK and the libraries
are built here. After the merge:

```sh
python3 dm/gfx/tools/dmsh_chain.py                  # must equal dmsh_chain.h
python3 dm/gfx/tools/vk_android_build.py --version  # our digest
python3 dm/gfx/tools/vk_android_build.py --header   # writes dmsh_chain_vk.h
git commit dm/gfx/nvn/dmsh_chain_vk.h -m "tools: <release> Vulkan chain ..."
```

The Linux translator (self-contained, .NET 10 SDK in docker; no SDK on aurora):

```sh
docker run --rm -v /config/Projects/bn-v020:/w -v /tmp/opencode:/out -w /w/dm/gfx/tools/dmsh_xlate \
  mcr.microsoft.com/dotnet/sdk:10.0 dotnet publish -c Release -r linux-x64 --self-contained true -o /out/xlate
sudo chown -R abc:abc /tmp/opencode/xlate
mv /tmp/opencode/xlate/dmsh_xlate /tmp/opencode/xlate/dmsh_xlate-linux
ln -sf dmsh_xlate-linux /tmp/opencode/xlate/dmsh_xlate
/tmp/opencode/xlate/dmsh_xlate --source-sha   # must equal dm/gfx/tools/dmsh_chain.lock's xlate_source
```

A copy of the working publish is kept at `/config/Projects/dumps/tools/xlate-v027/` (reuse it while
`--source-sha` still matches the lock; `/tmp` is wiped between sessions). `vk_android_build.py`
compares the embedded source digest instead of the Mac dll hashes when `sys.platform != 'darwin'`.
`XLATE=` must be exported for `pack_android.py`, which does not pass it through otherwise.

### 18.4 Rebuilding a library

Only if the chain changed (the library's header records it) or a profile's `inputs` changed
(`python3 dm/gfx/tools/dmsh_chain.py --inputs` vs the library's `inputs.tsv`). Full builds:

```sh
cd /config/Projects/bn-v020
DM_GAME=<id> XLATE=/tmp/opencode/xlate/dmsh_xlate SPIRV_VAL=/usr/bin/spirv-val J=32 DMSH_VK_XLATE_J=32 \
  python3 dm/gfx/tools/vk_android_build.py --target vulkan /config/Projects/dumps/<g>-shaders/pkg \
  /config/Projects/dumps/<g>-shaders/lib
SPIRV_VAL=/usr/bin/spirv-val J=16 python3 dm/gfx/tools/vk_android_build.py --check \
  /config/Projects/dumps/<g>-shaders/lib/v-<CHAIN>/dmsh_vk.lib
```

| game | pkg | shaders | time (J=32) | library |
|---|---|---|---|---|
| Pokopia | `dumps/pokopia-shaders/pkg` | 8,161 selected | ~25 min | 228 MB |
| DKB | `dumps/dkb-shaders/pkg` | 22,852 | ~9 min | 216 MB |
| MKW | `dumps/mkw-shaders/pkg` | 11,603 | ~5 min | 109 MB |
| Ocarina | `dumps/oot-shaders/pkg-v027` | 42,848 | ~35 min | 721 MB |

**Pokopia is a selected build.** The profile's `shader_seen.tsv` lists the keys; pass them (plus the
pixel shaders their `shader_pairs.tsv` pairs) as arguments. `pack_android.py` does not do this: it
builds the full 131k set. The list is kept at `dumps/tools/pokopia-selected-keys.txt`
(8,161 keys), generated by:

```sh
python3 - <<'PY' > /tmp/keys.txt
import re
prof='dm/games/0400A3A01EE4C000'
seen={m.group(0) for ln in open(prof+'/shader_seen.tsv') if (m:=re.match(r'[0-9a-f]{16}\b',ln))}
loc={c[3] for ln in open(prof+'/shader_map.tsv') if len(c:=ln.rstrip('\n').split('\t'))>4 and c[0].endswith('.bin')}
pairs={}
for ln in open(prof+'/shader_pairs.tsv'):
    if ln.startswith('#') or '\t' not in ln: continue
    vs,ps=ln.rstrip('\n').split('\t')[:2]; pairs.setdefault(vs,set()).add(ps)
print(' '.join(sorted(seen|{ps for vs in seen for ps in pairs.get(vs,()) if ps in loc})))
PY
```

If a profile's inputs changed, its package is stale: re-split with `shader_split.py` (section 13.2;
Ocarina's `main` needs `DM_ZBIC_LIB=dumps/tools/libzbicdec.so`, built with
`dm/third_party/zstd-zbic/build.sh <out> cc`). The map it writes must be byte-identical to the
profile's `shader_map.tsv`.

### 18.5 Packs, their shader dirs, and the launcher's bookkeeping

- Each game's pack lives at `/sdcard/Switch2/<id>-work/android-pack` and is read through the
  launcher's list in `shared_prefs/games.xml` (`<pack>` path). A pack that is not there, or an entry
  pointing at a deleted `files/packs/<id>`, shows "new shaders: … Prepare to update them" — it is a
  missing pack, not missing shaders.
- A pack built from a `.nup` reads its ROMFS from the nup at run time (`pack.tsv`'s `romfs` line,
  relative to the folder holding the nup). The nup's **file name must match** that line: the DKB
  pack had to be edited from `Donkey Kong Banana.nup` to
  `Donkey Kong Bananza [040074A01BF12000].nup`.
- To make a pack on this host (needs `capstone` and `lz4`: `get-pip.py --user --break-system-packages`
  then `pip install --user capstone lz4`; Python 3.14 has zstd built in):

  ```sh
  DM_GAME=<id> XLATE=/tmp/opencode/xlate/dmsh_xlate python3 scripts/pack_android.py <nup> \
    --shader-cache /config/Projects/dumps/<g>-shaders/lib --shader-pkg /config/Projects/dumps/<g>-shaders/pkg \
    -j 8 -J 8
  ```

  It reuses a current library from `--shader-cache` and copies it into the pack. Push the pack with
  `adb push <pack>/. /sdcard/Switch2/<id>-work/android-pack/` (trailing `/.`), or only its non-shader
  files when the library is already on the phone (the library is the big part).
- To move a game's entry to a pack elsewhere, edit `games.xml` as the app user:
  `adb shell "cat /sdcard/games-new.xml | run-as dev.nxnative.android sh -c 'cat > shared_prefs/games.xml'"`
  after `adb shell am force-stop dev.nxnative.android`.

### 18.6 Deploying a library and the APK

```sh
# per game, after the PKG build and --check
adb shell mkdir -p /sdcard/Switch2/<id>-work/android-pack/shaders/v-<CHAIN>
adb push dumps/<g>-shaders/lib/v-<CHAIN>/dmsh_vk.lib  .../shaders/v-<CHAIN>/dmsh_vk.lib
adb push dumps/<g>-shaders/lib/v-<CHAIN>/inputs.tsv   .../shaders/v-<CHAIN>/inputs.tsv
adb shell rm -rf .../shaders/v-<old chain>          # keep one old dir only if you keep its APK for rollback

# the APK: one profile per game, simulated packer (packs are made here)
cd /config/Projects/bn-v020/android/app
./gradlew -PnxGames=040074A01BF12000,0400A3A01EE4C000,0400C3F00006E000,0400486024DEC000 \
  -PnxPackEngine=fake apk
adb install -r out/nxnative-android.apk
```

A game whose profile is not in `-PnxGames` refuses to start ("this app version does not include
it"). Launch from a shell with the game and pack:

```sh
adb shell am start -n dev.nxnative.android/.MainActivity \
  --es game '/storage/emulated/0/Switch2/<file>.nup' \
  --es pack /storage/emulated/0/Switch2/<id>-work/android-pack --ez run true
```

Verify in the newest `files/logs/<id>-*.log`: `build: commit <sha>, shader chain <CHAIN>`,
`shader library …/shaders/v-<CHAIN>/dmsh_vk.lib: N shaders`, `[vk] pace up to … fps`, no `CRASH`.
**Close the game after every test.** A verified run is not finished until the game is stopped:

```sh
adb -s <serial> shell am force-stop dev.nxnative.android
```

It holds 5-8 GB while it runs (Ocarina has been killed by the system for that on this 15 GB phone),
keeps writing logs, and leaves the launcher slower for the next start. Leave the device on the
launcher and free; the same applies after a crash or a failed test.


**After a chain change the driver pipeline cache is invalidated** (`pipeline cache … is not stamped
with its shader library`), so the first run compiles every pipeline (Pokopia: 52 s, compute ones
1–4 s each). The second run reads it back and builds only what is new. Not a regression.

### 18.7 adb on the Thor

Wireless debugging, over aurora (`adb` lives there, not in this container):

```sh
ssh aurora 'adb mdns services'                 # the current port (changes when debug is re-enabled)
ssh aurora 'adb connect 192.168.1.55:<port>'   # or it appears on its own as adb-…._adb-tls-connect._tcp
```

If `connect` is refused while the port is open, the host key is not accepted: accept the prompt on
the device, or use "Pair device with pairing code" and `adb pair <ip>:<port> <code>`. Do not redirect
into `/tmp/opencode` inside an ssh command: that path exists only in this container.

### 18.8 Housekeeping

- Keep the shader packages (`dumps/*-shaders/pkg`) and the built libraries; they are the expensive
  inputs. Delete `…/lib/v-<chain>/results/` (build cache, GBs) once the library is written.
- `git bundle create ../banananative-backup/banananative-all-refs.bundle --all` in
  `/config/Projects/banananative` (the repo the worktree shares), then rsync it to
  `pi:/srv/…/Buckets/banananative-backup-2026-11-08/`. It also holds `refs/forks/*`, the branches
  recovered from 13 contributor forks when the GitHub repo vanished.
- Old-chain libraries, once superseded, are removed from the phone; the v-31b81e5c07ac one is kept on
  this host under `dumps/old-chain/0400A3A01EE4C000/` for the archived `deployed-thor-20261108` APK.
- Card metadata (icon, version, publisher) comes from a `.nup` meta block only; an extracted folder
  dump shows none, and a nup with only an icon shows only the icon (Pokopia's does, Ocarina's has no
  image at all).

### 18.9 Adding a cover and version to a .nup already on the phone

The launcher's card icon/version/publisher come only from a `.nup` meta block (`Nup.gameMeta`): a
folder dump has none, and some `.nup` releases carry a meta block with no image (Pokopia's) or none
at all. A meta block from another copy of the same game can be patched into a nup **in place**, with
no retransfer of the 25 GB, because the meta block is just a range in the file named by the TOC:

1. Read the donor's meta block (`PKG` record's `moff`/`msize`, see `dm/tools/nup.py`; a block with a
   type 3 record holds the icon bytes, type 4 version, 5 title, 6 publisher. Some packers wrote the
   types one lower: `Nup.Meta.shifted`).
2. On a local copy of the target nup: append the block 16-byte aligned at the end, point package 0's
   `moff`/`msize` at it, bump the superblock `fsize`, recompute the TOC SHA-256, write SB and TOC,
   then open the file with `nup.Nup()` — it validates the header, the TOC checksum, the bounds and
   the tree.
3. Push only what changed: the SB and TOC as one head file (`128 + toc_size`, 4096-aligned here) and
   the append payload, then on the device
   `dd if=… of=<nup> bs=4096 count=<blocks> conv=notrunc` for the head and
   `dd if=… of=<nup> bs=4096 seek=<bytes> oflag=seek_bytes conv=notrunc` for the append.
   **Toybox's dd: `seek` counts blocks unless `oflag=seek_bytes` is given** — a wrong seek silently
   overwrites game data (it happened once here; the range was restored from the local copy).
4. Verify with `sha256sum` on the phone against the validated local file, then open the launcher.

Done for Ocarina on 2026-11-11: the 25.0 GB copy on HDD5 (`… Ocarina of TIme [04004860224DEC000].nup`
and its `[NUP]` folder copy) carries a 39,989-byte meta block: 38,073-byte JPEG icon, version
`1.1.1`, title; the 25.6 GB copy we had on the phone had none. Its block is kept at
`dumps/tools/oot-meta.bin`. Result: `sha256 5fafd26b…` on both sides, the container validates, the
card gets the cover and `1.1.1`.

## 19. v0.3.11 without a library rebuild (2026-11-12)

50 more commits (v0.3.1 … v0.3.11): Ocarina vegetation detail and 60 fps hold, a build-host patch of
its foliage compute shader, Ocarina graphics/timing options, DKB dev tools and a terrain GPU-hang
fix on Adreno, the UBWC hint for Turnip on the 740, deferred inline uniform updates and bounded
render areas, the Qualcomm 891.x vendor-namespace loader, Fire Emblem Fortune's Weave resolution
rows, and Prepare progress.

`DMSH_CHAIN` stayed `1cf10a4252df` and upstream's `DMSH_VK_CHAIN` stayed `6257cc4482ac`, but
`vk_android_build.py` changed, so our digest moved to `270f200cd367`. A row in
`vk_android_same.tsv` maps it to `6602a8cd0984` after a full MKW rebuild came out **byte-identical**
(`sha256 9caffe0b…`), so the four libraries already on the phone stayed valid and only the APK was
rebuilt and installed.

Merge conflicts, same shape as before: `vk_android_same.tsv` (take upstream's rows), `GameActivity`
(keep both: the time-of-day block and upstream's vegetation line), `vka_draw.c` (keep our tbuf case,
adding upstream's `g_skip_dev[VKS_FEATURE]` counter; take upstream's shader-stat/bisect/skip
block **minus** its `expand && ind` SKIP, since our `DM_VK_QUAD_INDIRECT` path draws those).

Verified on the Thor: Pokopia 59.6 fps (library 8,161), Ocarina ~60 fps in the menu, no CRASH lines.

## 20. v0.3.13: full library rebuild (2026-11-12)

25 commits (v0.3.12, v0.3.13): Polish and Thai, Star Fox and Fire Emblem Fortune's Weave game pages,
recommended settings, Ocarina on Qualcomm, FSR 3.1 as a fourth Upscaler (`DM_VK_DLSS=4`), the Ocarina
"Turn off hidden layers" speedhack, audio focus regained after a permanent loss, a present that
waits for pending gates (Turnip tearing), and a Fire Emblem 4:3 option.

Both chains moved: `DMSH_CHAIN` `1cf10a4252df` → **`9b2613d5216b`**, Vulkan chain → **`da525bb64987`**
(ours; upstream's is `65883461079b`). The profile inputs did not change, so the existing shader
packages and splits were reused; only the libraries were rebuilt. Translator republished
(`--source-sha` `d9898e20cdf4782a`, the new lock).

Merge conflicts: `dmsh_chain_vk.h` (take theirs, regenerate), `GameActivity` (keep upstream's
`Prefs.startWindow` line plus our time-of-day block).

Built and checked: Pokopia 8,161 shaders / 228 MB, DKB 22,852 / 216 MB, MKW 11,603 / 109 MB,
Ocarina 42,848 / 721 MB. Deployed to the packs as `shaders/v-da525bb64987/`, old `v-6602a8cd0984`
removed. Verified: Pokopia 60.2 fps, Ocarina 60.0 fps in the menu, no CRASH lines. The game was then
closed and the launcher left in front, as the rule above requires.

## 21. v0.3.14: APK only, libraries reused (2026-11-13)

12 commits (v0.3.14): Ocarina sound back and its dynamic resolution working, one Resolution setting,
Zelda's cave/trail geometry kept on Qualcomm Vulkan, per-profile Link Play join settings with an
opt-in LDN transport (virtual UDP sockets), and the issue-digest workflow.

Chains did **not** move (`9b2613d5216b` / `da525bb64987`), and the merge was clean: no conflicts, no
library rebuild. Only the APK was rebuilt and installed. Verified: Pokopia 60.1 fps, Ocarina 59.8 fps
in the menu with audio callbacks running, no CRASH lines. Game closed afterwards.

### 18.10 Proposing a change upstream (Gitea)

The upstream lives on Gitea; there is no GitHub repository any more. Push over HTTPS (SSH port 22 is
unreachable here) with the `dopper` token, to the `dopper/BananaNative` fork, then open a PR with the
API:

```sh
git -c http.extraHeader="Authorization: token $(cat /config/.config/gitea/dopper.token)" \
  push https://git.eden-emu.dev/dopper/BananaNative.git <branch>
curl -s -H "Authorization: token $(cat /config/.config/gitea/dopper.token)" -H 'Content-Type: application/json' \
  -d '{"head":"dopper:<branch>","base":"android","title":"...","body":"..."}' \
  https://git.eden-emu.dev/api/v1/repos/moceure/BananaNative/pulls
```

Branches for proposals are cut from `origin/android`, one change each, built once before pushing.
`eden-time-of-day` (PR #227, the DM_TIME setting) is the example.

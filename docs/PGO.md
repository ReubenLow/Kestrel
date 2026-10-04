# PGO builds on Windows

Last updated: 2026-10-05

PGO (profile-guided optimisation) builds the emulator twice. The first build counts how often
each piece of code runs while you play. The second build uses those counts to optimise the hot
code. The source code does not change, only the speed.

Status: Windows PGO works on `main` at `a4631994`. On a Ryzen 7 9800X3D it is 11-21% faster
depending on the scene (results below). Numbers on the reference PC (Ryzen 5 3600, RTX 3060)
are still to come.

## Build steps

Run these in a VS 2022 developer shell from the repository root. The options are the same as the
normal build in [QUICKSTART.md](QUICKSTART.md), plus `KYTY_PGO`. Use `KYTY_GLSLANG_VALIDATOR` as
there if `glslangValidator` is not on PATH.

### 1. Counting build

```
cmake -S . -B _Build\pgo-gen -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=clang-cl -DCMAKE_CXX_COMPILER=clang-cl -DKYTY_BUILD_LAUNCHER=OFF -DKYTY_TRACY=OFF -DKYTY_PGO=GENERATE
cmake --build _Build\pgo-gen --target kyty_emulator --parallel
```

The configure output should say `pgo=GENERATE`.

### 2. Training

Set `emulator = _Build\pgo-gen\kyty_emulator.exe` under `[launcher]` in `kestrel.local.ini` and
use the preset you will benchmark with (we used `astro-low-unlocked`). Copy your `_SaveData`
folder into `_Build\pgo-gen` first, because saves are kept per build folder.

Play the scorecard scenes from the [roadmap](ROADMAP.md) for a few minutes each, standing still
and moving. The counting build is slower; that does not affect the profile. It writes
`kyty-<pid>.profraw` into the emulator folder every 10 seconds, so you can stop it any way you
like.

**Known problem: the 10-second saves append instead of replace.** On Windows each save adds a
full cumulative copy to the end of the `.profraw` file, so a 17-minute session left 102 copies
(about 1 GB). Merging the file as it is adds all copies together and gives early play far too
much weight. Keep only the last copy, which is the complete total. This Python snippet cuts the
file at the last raw-profile header:

```python
import struct, sys
data = open(sys.argv[1], "rb").read()
start = data.rfind(struct.pack("<Q", 0xff6c70726f667281))
open(sys.argv[2], "wb").write(data[start:])
```

Also leave out the small (about 10 MB) `.profraw` files that the launcher's `--help` check
writes. They contain almost no counts.

### 3. Merge

```
& "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Tools\Llvm\x64\bin\llvm-profdata.exe" merge -o _Build\kestrel.profdata <session1>.profraw <session2>.profraw
```

Use the `llvm-profdata` that came with the same clang-cl (the path depends on your VS edition).
Write `-o <file>`, not `-output=<file>`: Windows PowerShell 5.1 splits `-output=_Build\kestrel.profdata`
at the dot.

### 4. PGO build

```
cmake -S . -B _Build\pgo-use -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=clang-cl -DCMAKE_CXX_COMPILER=clang-cl -DKYTY_BUILD_LAUNCHER=OFF -DKYTY_TRACY=OFF -DKYTY_PGO=USE -DKYTY_PGO_PROFILE=<full path>\kestrel.profdata
cmake --build _Build\pgo-use --target kyty_emulator --parallel
```

The configure output should say `pgo=USE`. After replacing the profile, delete `_Build\pgo-use`
and build again: Ninja does not notice a changed profile.

## Results: Ryzen 7 9800X3D, RTX 5090

Windows 11 Pro, 61.7 GB RAM, NVIDIA driver 32.0.16.1692. Both builds from `a4631994`, clang-cl
19.1.5. The profile was trained on the same scenes: CPU Plaza, pre-plaza, Cooling Springs and
GPU Jungle, about 28 minutes in total.

Settings, the same for both builds: preset `astro-low-unlocked` (render scale 0.5, ray tracing
off, vertex fetch cpu, Mailbox), any-frame-rate patch on, no frame cap, `--vblank-frequency 120`.

Method: report mode, 180 s per run. Each scene was run four times: normal, PGO, normal, PGO.
Both builds started from the same save, and both had warm shader caches. The value per run is the
median of the six 10-second windows in the last 60 seconds, so a single freeze does not decide the
result. The table shows the average of the two runs per build.

| Scene | Normal fps | PGO fps | Gain |
| --- | --- | --- | --- |
| CPU Plaza, still | 53.6 | 60.4 | +12.7% |
| CPU Plaza, moving | 68.9 | 83.2 | +20.8% |
| Pre-plaza, still | 106.2 | 119.6 | at least +12.6% (see notes) |
| Pre-plaza, moving | 93.1 | 108.5 | +16.5% |
| Cooling Springs, still | 51.6 | 57.3 | +11.1% |
| Cooling Springs, moving | 84.3 | 98.7 | +17.1% |

In every scene both PGO runs were faster than both normal runs.

Notes:

- **60 fps ceiling.** The emulated display runs at 60 Hz by default, and the game waits for it,
  so a fast PC sits at exactly 60 fps and cannot show a speedup. `--vblank-frequency 120` raises
  the ceiling. A 3060 stays well under 60 fps, so it does not need this.
- **Pre-plaza still hits the 120 fps ceiling.** 99% of the PGO frames were at 120 fps, so the real
  gain there is larger than shown.
- **Moving runs are noisier.** One player cannot repeat a route exactly, so the two runs of the
  same build differ by up to 10% in the moving scenes, against under 3% in the still scenes.
- **Freezes of about 3 seconds** happen in both builds: 12 in the 12 normal runs and 15 in the 12
  PGO runs. They are not caused by PGO, but they pull down the 1% low of whichever run they land
  in. The cause is not known yet. In most cases the game destroys a group of AudioOut2 ports just
  as the freeze starts.
- The any-frame-rate patch has physics problems that are unrelated to PGO: the player can walk
  through the planters in CPU Plaza, and in GPU Jungle a glitching bridge ends in the game's assert
  `PhysicsRigidBody.cpp:95 !pos.IsNan()`. The benchmark scenes ran without crashes.

<details>
<summary>All runs</summary>

| Scene | Build | Median window fps | Freezes | Report |
| --- | --- | --- | --- | --- |
| CPU Plaza, still | normal | 53.75 | 1 | report-20261005-013738 |
| CPU Plaza, still | PGO | 60.75 | 2 | report-20261005-014134 |
| CPU Plaza, still | normal | 53.45 | 1 | report-20261005-014513 |
| CPU Plaza, still | PGO | 60.10 | 4 | report-20261005-014831 |
| Pre-plaza, still | normal | 105.90 | 2 | report-20261005-020111 |
| Pre-plaza, still | PGO | 119.55 | 1 | report-20261005-020747 |
| Pre-plaza, still | normal | 106.50 | 2 | report-20261005-021125 |
| Pre-plaza, still | PGO | 119.65 | 0 | report-20261005-021458 |
| Pre-plaza, moving | normal | 94.60 | 2 | report-20261005-021846 |
| Pre-plaza, moving | PGO | 103.05 | 0 | report-20261005-022219 |
| Pre-plaza, moving | normal | 91.60 | 1 | report-20261005-022554 |
| Pre-plaza, moving | PGO | 113.85 | 0 | report-20261005-022921 |
| CPU Plaza, moving | normal | 66.35 | 0 | report-20261005-023302 |
| CPU Plaza, moving | PGO | 83.20 | 0 | report-20261005-023633 |
| CPU Plaza, moving | normal | 71.45 | 0 | report-20261005-024009 |
| CPU Plaza, moving | PGO | 83.25 | 0 | report-20261005-024409 |
| Cooling Springs, still | normal | 51.90 | 1 | report-20261005-024823 |
| Cooling Springs, still | PGO | 56.50 | 4 | report-20261005-025204 |
| Cooling Springs, still | normal | 51.30 | 1 | report-20261005-025545 |
| Cooling Springs, still | PGO | 58.15 | 1 | report-20261005-025927 |
| Cooling Springs, moving | normal | 85.30 | 1 | report-20261005-030314 |
| Cooling Springs, moving | PGO | 99.75 | 3 | report-20261005-030708 |
| Cooling Springs, moving | normal | 83.25 | 0 | report-20261005-031051 |
| Cooling Springs, moving | PGO | 97.55 | 0 | report-20261005-031421 |

One more PGO run (pre-plaza, still) stopped after 63 seconds and was repeated; it is not in the
table.

</details>

## Results: Ryzen 5 3600, RTX 3060

Pending. This is the reference PC from the [roadmap](ROADMAP.md), and the numbers for a PR to make
PGO the default.

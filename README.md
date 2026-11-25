# Mario Eval Attempt: Nemotron 3 Ultra 550B (NVIDIA NIM, omp)

One 24-hour autonomous run of [Mario Eval](https://github.com/IlyaasK/mario-eval) ([ilya.as/mario-eval](https://ilya.as/mario-eval/)): an autonomous agent gets the C decompilation of **Super Mario 64** (`n64decomp/sm64`), pinned N64 compilers (`ido` + `mips64-elf-gcc`), an instrumented cycle/cache/RDP-accurate `ares` emulator (`n64run`), and the `mev` profiler/self-check CLI, and optimizes real 4 MB N64 console execution while preserving gameplay, visuals, and audio fidelity.

No Nintendo code or ROM assets are redistributed here — `patches/` contains only Nemotron 3 Ultra 550B's unified diffs against `n64decomp/sm64`, alongside the full grading summaries and compressed agent session transcripts.

| | |
|---|---|
| Model | `nvidia/nemotron-3-ultra-550b` (NVIDIA NIM API) |
| Agent harness | `omp` (oh-my-pi), 4 sessions |
| Environment | Mario Eval v2.1 SM64 task container (transferred mid-run at 3.35h after Envaudit/v2.1 grader hardening; total wall clock 24.0h) |
| Token usage | 2,289.1M total (27.91M uncached input, 2.41M output, 2,258.79M cache read), 12 compactions |
| Tool calls | 6,386 (`bash` 2,484, `read` 2,250, `grep` 691, `edit` 688, `write` 205, `eval` 57, `glob` 10, `todo` 1) |
| Commits | 19 optimization commits touching 6 files (`+327 / -218` lines); best valid checkpoint `9ceb56e` = 13 commits touching 5 files (`+276 / -178`) |
| Held-out split | 25 routes (6 public courses + 8 private scripted routes + 11 private seeded button-mash routes) + `PLAY=1` console boot smoke test |

## Scores (v2.1 Section-Weighted: 40% Public / 50% Private Scripted / 10% Private Mash)

| Evaluation | Routes | Status | Console Speedup | Flat Speedup | vs GCC `-O2` | Reward | `PLAY=1` Smoke |
|---|---:|---|---:|---:|---:|---:|---|
| 1996 Retail US Baseline (`r0`) | 25 | `valid` | `1.000×` | `1.000×` | `0.731×` | `0.000` | pass |
| **Nemotron 3 Ultra best valid (`9ceb56e`, CP10)** | **25** | **`valid`** | **`1.053×`** | **`1.053×`** | **`0.770×`** | **`+0.019`** | **pass** |
| Compiler Only (`r1-gcc-o2`) | 25 | `valid` | `1.369×` | `1.371×` | `1.000×` | `+0.113` | pass |
| DeepSeek-V4.1-Flash-1017 (`b4c0cd1`, CP2) | 25 | `valid` | `1.611×` | `1.600×` | `1.177×` | `+0.172` | pass |
| Kaze Master Reference (`r10`) | 25 | `valid` | `1.705×` | `1.693×` | `1.246×` | `+0.193` | pass |
| Kimi K3.1 Pro Trial (`4b54858`, CP4) | 25 | `valid` | `1.707×` | `1.695×` | `1.247×` | `+0.193` | pass |
| **Nemotron 3 Ultra final (`5f11847`, HEAD)** | **25** | **`invalid`** | `—` | `—` | `—` | `-1.000` | build failed |

### 25-Route Host Checkpoints

| Checkpoint | Cumul. Hours | Status | Weighted Speedup | vs `-O2` | Notes |
|---|---:|---|---:|---:|---|
| `checkpoint-1` | `2.00h` | `valid` | `1.010×` | `0.738×` | Initial IDO `mtxf_mul` / `mtxf_rotate_xyz_and_translate` C edits |
| `cont-checkpoint-1` | `5.35h` | `valid` | `1.043×` | `0.762×` | Register hints & loop tuning in `cur_obj_update`, `geo_process_node_and_siblings`, `mtxf_to_mtx` |
| `cont-checkpoint-2` | `7.38h` | `invalid` | `—` (`1.043×` lv) | `—` | Mid-edit dirty tree |
| `cont-checkpoint-3` | `9.44h` | `valid` | `1.015×` | `0.742×` | Temporary regression during `mtxf_to_mtx` rewrite |
| `cont-checkpoint-4..5` | `11.46h–13.51h` | `invalid` | `—` (`1.043×` lv) | `—` | Mid-edit dirty tree |
| `cont-checkpoint-6` | `15.61h` | `valid` | `1.043×` | `0.762×` | Recovered `mtxf_to_mtx` |
| `cont-checkpoint-7` | `17.64h` | `valid` | `1.042×` | `0.761×` | `clear_spatial_partition` do-while / static inlining |
| `cont-checkpoint-8` | `19.67h` | `valid` | `1.052×` | `0.769×` | `geo_process_node_and_siblings` jump table & floor delta hoisting |
| `cont-checkpoint-9` | `21.70h` | `invalid` | `—` | `—` | Mid-edit build failure |
| **`cont-checkpoint-10` (last valid `9ceb56e`)** | **`23.73h`** | **`valid`** | **`1.053×`** | **`0.770×`** | **Best 100%-valid 25-route checkpoint (`9ceb56e`)** |
| `final` (`5f11847`) | `24.00h` | `invalid` | `—` | `—` | SGI IDO C89 syntax error in `src/engine/behavior_script.c` |

## What Nemotron 3 Ultra 550B Built

Nemotron 3 Ultra 550B spent all 24 hours optimizing C source under the original 1996 SGI IDO C89 compiler and Fast3D microcode without ever switching the build to `mips64-elf-gcc -O2` or `F3DEX2` (despite both being installed and documented in `TASK.md`):

- **Graph & Object Loop Micro-Optimizations:** Converted `geo_process_node_and_siblings` dispatch to a switch/jump table (`fe80ba9`, `9ceb56e`), added C `register` hints in `cur_obj_update` (`bdd1b4c`), and optimized `geo_set_animation_globals` (`24f46d7`).
- **Matrix & Collision C Tuning:** Reduced temporaries and reloads in `mtxf_mul`, `mtxf_to_mtx`, and `mtxf_scale_vec3f` (`b5b8b54`, `5cd7cbd`, `7f2a89a`, `cd91b68`, `9ceb56e`), precomputed `dx`/`dz` deltas in `find_floor_from_list` (`bedd51f`, `9ceb56e`), and tuned `clear_spatial_partition` (`d35aa0b`).
- **Late Build Breakage:** In the final 2 hours (`180427e`, `5f11847`), attempted `inline` and prefetch edits in `src/engine/behavior_script.c` that failed SGI IDO's strict C89 parser (`cfe: Error: Empty declaration specifiers`).

## Repository Layout

- `patches/best-valid-9ceb56e.patch` — cumulative diff against `n64decomp/sm64` for the best 100%-valid 25-route checkpoint (`1.053×`)
- `patches/final-5f11847.patch` — cumulative diff for the final commit (`5f11847`)
- `patches/commits/` — all 19 individual commits as `git format-patch` files
- `grading/` — 25-route v2.1 `summary.json` files (`best-valid-9ceb56e`, `final-5f11847`), merged `trajectory.json`, and all 154 `mev-history.jsonl` checks (no private replay files)
- `transcripts/` — all 4 compressed `omp` session transcripts (`*.jsonl.gz`)

## Reproducing

Clone [`n64decomp/sm64`](https://github.com/n64decomp/sm64) (matching the commit used by `mario-eval/sm64/setup.sh`), extract assets from your own legally obtained US ROM, and apply either patch:

```bash
git apply patches/best-valid-9ceb56e.patch
```

# Stained Glass Genetic Art

[![tests](https://github.com/TheElliotGong/Stained-Glass-Genetic-Algorithm/actions/workflows/tests.yml/badge.svg)](https://github.com/TheElliotGong/Stained-Glass-Genetic-Algorithm/actions/workflows/tests.yml)
[![Python 3.12+](https://img.shields.io/badge/python-3.12%2B-blue)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)

A genetic algorithm that evolves a stained-glass painting, a Voronoi diagram of
colored cells, toward any target image.

This project is a fork of Sebastian Proost's
[Genetic-Art-Algorithm-part-2](https://github.com/4dcu-be/Genetic-Art-Algorithm-part-2).
His idea and his evolutionary schedule are the foundation. The original is a
single script that took days to run 5,000 generations. This fork keeps the idea
and makes it **faster, more accurate and fully customizable**, and wraps it in a
web app so anyone can run it on their own image.

![Evolution of a painting toward the target image](vermeer_evolution.png)

*A population evolving toward Vermeer's* Girl with a Pearl Earring *(image from
the original project's run).*

---

## What this fork changes

| | Original | This fork |
| --- | --- | --- |
| **Rendering** | A `scipy` Voronoi diagram, with each cell drawn as a PIL polygon and alpha-composited over the whole canvas | One OpenCV distance transform labels every pixel with its nearest seed. **~6x faster at 100 points, ~23x at 800**, and nearly flat as the point count grows |
| **Evolution loop** | Full target image copied and serialized with every individual, every generation | Targets are shared, genomes are packed into flat arrays, buffers are reused. **~8-10x faster end to end**, with output proven identical |
| **Coverage** | Unbounded border cells were skipped, leaving background holes | Every pixel belongs to a cell |
| **Starting population** | Random points and random colors | Seeded from the target's own colors, edges, texture and regions |
| **Large images** | One population for the whole image | Optional tiled solver: independent tiles, feather-blended without seams |
| **Look** | Flat colored cells | Stained-glass outlines with configurable color and width, plus supersampled output |
| **Configuration** | Constants edited inside `__main__` | 21 validated parameters, from the web UI, the API or the scripts |
| **Interface** | Run a script, wait for PNGs | Web app with upload, live preview, progress and ETA, cancel and download |
| **Confidence** | None | 288 tests, an equivalence suite pinning the optimized code to the pre-optimization code, a benchmark harness, and CI on 3 operating systems x 2 Python versions |

The sections below cover each change in turn.

---

## 1. Speed

### A renderer that no longer scales with the point count

Every individual in every generation has to be rendered before it can be
scored, so the renderer is the innermost loop of the whole program. The original
built a Voronoi diagram with `scipy`, drew each cell as a PIL polygon onto a
full-size RGBA layer, and alpha-composited that layer onto the canvas. That is
one full-image composite per cell.

The replacement ([voronoi_painting.py](voronoi_painting.py)) writes the seeds
into a binary image and runs a single `cv2.distanceTransformWithLabels`, which
labels every pixel with its nearest seed in `O(width x height)` regardless of
how many seeds there are. A color lookup table turns labels into pixels with one
`np.take`.

Time to score one individual (render plus compare), original against current, on
a 400x469 target:

| points | original | current | speedup |
| --- | --- | --- | --- |
| 100 | 15.0 ms | 2.4 ms | ~6x |
| 250 | 38.2 ms | 3.3 ms | ~12x |
| 800 | 111.9 ms | 4.8 ms | ~23x |

The original's cost grows with the point count, and the new renderer's barely
does. That matters because the original schedule spends most of its 5,000
generations at 500 to 800 points.

*(Measured on one 2-core Linux machine, upstream commit `9fd359f` against the
current code. Absolute numbers will differ on your hardware.)*

### A profiled pass over the evolution loop

With the renderer fixed, the loop was profiled and the remaining time was
mostly overhead around the work, not the work itself. Measured on
`img/girl_with_pearl_earring_half.jpg` (400x469), 250 points, population 60,
2 cores:

| operation | before | after | speedup |
| --- | --- | --- | --- |
| render | 2.60 ms | 1.42 ms | 1.8x |
| fitness (`image_diff`) | 2.59 ms | 1.49 ms | 1.7x |
| `deepcopy` of a painting | 1.076 ms | 0.022 ms | **48x** |
| serialize one individual | 3.66 ms | 0.18 ms | **20x** |
| serialized size per individual | 1290.8 KiB | 3.4 KiB | **384x** |
| **end to end** | **1.0 gen/s** | **9.7 gen/s** | **9.7x** |

An independent re-run on a different 2-core machine gave 8.2x end to end with
the same 384x payload reduction, so expect roughly 8-10x depending on hardware.

What was wrong, and what fixed it:

- **The target image was stored on every painting.** `evol` serializes whole
  individuals on each evaluation, so a 250-individual population shipped the
  target 250 times per generation, and every `deepcopy` in the genetic operators
  duplicated it again. Targets now live in a per-process, content-addressed
  registry ([target_cache.py](target_cache.py)) and paintings carry a short key.
- **The genome was serialized as thousands of Python objects.** Paintings now
  transport their points as two flat NumPy arrays and rebuild them on arrival.
- **Rendering reallocated four full-size arrays on every call.** They are now
  pooled per process and reused, and the color lookup writes into a
  preallocated buffer.
- **The operators copied more than they needed to.** Crossover and merge return
  children that already own their points, so the extra `deepcopy` is gone.
  Points use `__slots__` and a cheap `copy()`. Mutation samples the `k` indices
  it needs in `O(k)` instead of shuffling all `n`.
- **The loop did needless I/O.** The original rendered and saved a 3x PNG every
  generation and printed a progress dot per evaluation. Previews are now saved
  every 10 generations, and the dots are opt-in (`VORONOI_PROGRESS=1`).

### Divide and conquer for big images

[evolve_tiled.py](evolve_tiled.py) splits the image into a grid, evolves each
tile as its own small population, and blends the results back together. Smaller
subproblems converge faster, use less memory, and are easy to parallelize. See
[Accuracy](#2-accuracy-and-quality) for how the seams are hidden.

### Optimized, but provably not changed

Speedups that quietly change the output make it impossible to tell an
optimization from a regression in art quality, so this fork treats
"identical output" as a requirement, not a hope. For a given genome, the
renderer, the outlined renderer and the fitness score are byte-identical to the
pre-optimization implementation.
[tests/test_equivalence.py](tests/test_equivalence.py) asserts exactly that
against a verbatim copy of the old code in
[tests/reference_impl.py](tests/reference_impl.py).

### Measure it yourself

[benchmark.py](benchmark.py) times each hot path and the end-to-end loop, and
can save and compare runs, so any speedup claim can be checked on your machine:

```bash
python benchmark.py --image ./img/car.jpeg --label after --save-json bench_after.json
python benchmark.py --compare bench_before.json          # A/B against a saved run
python benchmark.py --sweep-workers 1,2,4,8              # find the best worker count
pytest tests/test_equivalence.py                         # confirm output is unchanged
```

Run the worker sweep before choosing `workers`. Now that individuals are small
on the wire, extra processes only pay off once rendering outweighs the cost of
moving individuals between them, so for small images a lower worker count can
be faster.

---

## 2. Accuracy and quality

### Every pixel is covered

The original only painted bounded Voronoi cells, so the cells along the border
were skipped and showed the background color. The distance-transform renderer
assigns every pixel to its nearest seed, so the canvas is always fully covered
and the fitness score is never penalizing a hole that no mutation could fix.

### A smarter starting population

The original starts from random points with random colors, so early generations
are spent discovering the basic layout and palette of the picture. Here the
target is analyzed first ([evolve_voronoi.py](evolve_voronoi.py)):

1. **Palette extraction.** An adaptive palette is pulled from the target, then
   `condense_palette` drops near-duplicates and `simplify_palette` reduces it to
   an exact color count.
2. **Edges and texture.** Canny edges mark boundaries, and a blurred Laplacian,
   quantized into bins, measures local texture.
3. **Regions.** Every pixel is mapped to its nearest palette color. Pixels are
   grouped by color and texture, edge pixels are excluded, and connected
   components become regions with a median color.
4. **Seeding.** Each starting point lands inside a region with probability
   `region_bias`, chosen in proportion to region area, with a little color
   jitter for diversity. The rest fall back to random positions and palette
   colors.

The tests assert that region-seeded populations beat uniform random seeding
rather than assuming they do.

### Seamless tiling

Independent tiles would show visible grid lines. Tiles therefore overlap
(32 px by default) and are stitched with a feathered weight mask that stays at
full strength across each tile's core and fades only across the overlap.
Outer image borders stay fully opaque so the canvas edges are not darkened.
`tests/test_evolve_tiled.py` verifies that the grid covers every pixel exactly
once and that stitching uniform tiles leaves no seam.

### Correct colors

The original generated random channels with `randint(0, 256)`. That bound is
inclusive, so a channel could be 256, which does not fit in 8 bits. Colors are
now generated in 0-255, and mutation clamps to the same range.

### Evolution that can grow and shrink

The original's most interesting idea is kept: *genome duplication* merges two
parents so the point count doubles, and *shrink* steps remove points again. The
whole-image script still runs the inherited 5,000-generation schedule
(250 points, duplicated to 500, duplicated again to 800, with shrink steps
between). The tiled script and the web app use a shorter three-phase plan:
explore, duplicate once, then refine with gentler mutation.

---

## 3. Customization

### Stained-glass look

- **Outlines** between cells, with configurable color (`#rrggbb` or `#rgb`) and
  width.
- **Supersampled output.** The final image can be rendered at up to 8x the
  evolution resolution, so a small, fast evolution still yields a crisp image.

### Everything is a parameter

In the original every setting was a constant inside `__main__`. Now
[webapp/params.py](webapp/params.py) defines one typed model with a bound and a
description for every setting. That one model is the request schema, the
validator, and the source the web UI builds its sliders from.

| group | parameters (default) |
| --- | --- |
| **Mode** | `mode` (`single`), `tiled` |
| **Evolution** | `generations` (600), `population_size` (60), `num_points` (200), `mutation_rate` (0.05), `mutation_sigma` (0.5), `survive_fraction` (0.025), `genome_duplication` (on) |
| **Seeding** | `region_bias` (0.85), `initial_colors` (60), `final_colors` (20) |
| **Target** | `max_dimension` (400), the main speed dial, since runtime grows with pixel count |
| **Tiling** | `tile_rows` (3), `tile_cols` (3), `tile_overlap` (32 px) |
| **Output** | `outline` (on), `outline_width` (1), `outline_color` (`#000000`), `render_scale` (3) |
| **Execution** | `workers` (1), `preview_every` (10) |

### A web app instead of a script

```bash
python run_web.py
```

Then open <http://127.0.0.1:8000>. Use `--host` and `--port` to change where it
binds, e.g. `python run_web.py --host 0.0.0.0 --port 8080`.

- **Bring any image:** upload a file, give a URL, or pick a bundled sample.
- **Watch it evolve:** generation, similarity, ETA and point count stream live
  over Server-Sent Events, with a preview you can wipe against the target.
- **Stay in control:** cancel a run at any time, queue several, and download the
  finished PNG. Runs are kept in a history list.
- **Safe to expose:** URL fetches that resolve to private, loopback or
  link-local addresses are refused (an SSRF guard), uploads are size-capped, and
  EXIF rotation is respected.

The REST API (18 endpoints, interactive docs at `/docs`) is what the page
itself uses, so everything in the UI can be scripted. Output is written to
`runs/` (override with `VORONOI_WEB_DATA`).

### Command-line scripts

```bash
python evolve_tiled.py    # divide-and-conquer tiled approach. Recommended for large images.
python evolve_voronoi.py  # whole-image approach, with the original 5,000-generation schedule.
```

Paths and hyperparameters are set in each script's `__main__` block:

| | `evolve_voronoi.py` | `evolve_tiled.py` |
| --- | --- | --- |
| Target | `./img/car.jpeg` | `./img/girl_with_pearl_earring.jpg` |
| Output | `./output/<timestamp>/` | `./output/girl/tiled-<timestamp>/` |
| Points | `num_points = 250` | `points_initial = 50` per tile, doubled by duplication |
| Population | `population_size = 250` | `population_size = 100` per tile |
| Workers | `concurrent_workers = 4` | `workers = 1` |
| Palette | `initialColorCount = 60`, `finalColorCount = 20` | 30 colors condensed to 12 per tile |
| Seeding | `region_bias = 0.85` | `region_bias = 0.85`, `min_area = 10` |
| Grid | n/a | `n_rows, n_cols = 3, 3`, 32 px overlap |
| Phases | Inherited 5,000-generation schedule | `gens_phase1 = 999`, `gens_phase2 = 1000` |

Operators in both: survivors `0.025`, mutation rate `0.05` falling to `0.03`
(sigma `0.5` to `0.4`), `mate` for crossover and `merge` for duplication.

### Environment variables

- `VORONOI_PROGRESS=1`: bring back the per-evaluation progress dots (off by
  default; they cost a flushed write per individual per generation).
- `VORONOI_TARGET_CACHE=<dir>`: where target pixels are spooled so worker
  processes started with `spawn` (the default on Windows) can load them.
  Defaults to a `voronoi-targets` folder in the system temp directory.
- `VORONOI_EMBED_TARGET=1`: skip the spool file and embed target pixels in every
  serialized painting. Slower, but makes population checkpoints self-contained
  and loadable on another machine.
- `VORONOI_WEB_DATA=<dir>`: where the web interface stores uploads and run
  output. Defaults to `runs/` next to the code.
- `VORONOI_ALLOW_PRIVATE_URLS=1`: allow "from URL" fetches that resolve to
  private addresses. Blocked by default, because the server fetches whatever URL
  it is handed. Only enable it if you trust everyone who can reach the app.

---

## 4. Engineering behind it

The algorithm never depends on the web tier, so it stays usable from the command
line and testable without a server.

```
Browser (HTML5 / CSS / vanilla JS)
        |  REST + Server-Sent Events
webapp/app.py        - HTTP endpoints, request validation, streaming
webapp/params.py     - typed parameter model, the single validation authority
webapp/runner.py     - job queue, worker thread, progress and lifecycle
webapp/imaging.py    - decoding, EXIF handling, SSRF-guarded URL fetch
        |  plain function calls
evolve_voronoi.py    - whole-image solver, seeding, palette, selection
evolve_tiled.py      - divide-and-conquer tiled solver
voronoi_painting.py  - the genome: VoronoiPainting / ColoredPoint
target_cache.py      - process-global content-addressed image registry
```

Design choices that earn their place, mostly because they show up directly in
the profiler:

| Pattern | Where | Why |
| --- | --- | --- |
| **Object pool** | render scratch buffers | Removes a per-render allocation of about 12 bytes per output pixel |
| **Flyweight / identity map** | [target_cache.py](target_cache.py) | One shared copy of the target per process, keyed by a hash of its pixels |
| **Prototype** | `VoronoiPainting.__deepcopy__` | Clones the genome and shares the immutable target, which is 48x faster |
| **Strategy** | `parent_picker` / `combiner` | Selection and recombination are swapped per evolution phase |
| **Producer/consumer** | `JobManager` | HTTP threads enqueue, one worker executes, so a long run never blocks the API |

A run is a small state machine (`queued -> running -> done | error | cancelled`)
and cancellation is cooperative, via a per-job `threading.Event` checked in the
evolution loop.

---

## 5. Testing and CI

288 tests across 10 files, running in about 5 seconds with no network access, no
GPU and no long evolution runs.

```bash
pip install -r requirements-dev.txt
pytest
```

| file | what it pins down |
| --- | --- |
| `tests/test_equivalence.py` | The optimized renderer, outliner, fitness, copying and pickling match the pre-optimization code byte for byte |
| `tests/test_voronoi_painting.py` | Color clamping, mutation rates, crossover, merge, render scaling, the shared scratch buffers |
| `tests/test_target_cache.py` | Content keying, spooling, rehydration in a *separate process*, and that a serialized individual carries no target pixels |
| `tests/test_evolve_voronoi.py` | Palette condensing, region detection, region-seeded initialization (including that it beats uniform random), parent pickers, and a short real evolution that must improve fitness |
| `tests/test_evolve_tiled.py` | Tile grids cover the image exactly once, feather masks stay opaque over tile cores, stitching leaves no visible seam |
| `tests/test_params.py` | Every hyperparameter bound, the outline-color parser, and the phase and survivor arithmetic the progress bar depends on |
| `tests/test_imaging.py` | Decoding, EXIF rotation, size caps, and the SSRF guard |
| `tests/test_runner.py` | Job lifecycle: queueing, cancellation, eviction, progress and ETA, whole runs in single and tiled mode |
| `tests/test_api.py` | Every HTTP endpoint, including uploads, validation errors, the SSE stream and downloading a finished run |
| `tests/test_entrypoints.py` | Every script still imports, and none starts work at import time |

```bash
pytest -m "not slow"                     # skip the real-evolution tests
pytest --cov --cov-report=term-missing   # coverage report
python scripts/run_tests.py --fast       # same thing, platform independent
```

[`.github/workflows/tests.yml`](.github/workflows/tests.yml) runs the suite on
every push and pull request, nightly, and on demand, across **Linux, Windows and
macOS on Python 3.12 and 3.14**. Locally, `python scripts/install_hooks.py`
installs a pre-push hook that runs the suite before every push
(`git push --no-verify` skips it once, and `--uninstall` removes it).

---

## Getting started

Requires Python 3.12 or later (CI covers 3.12 and 3.14) and Git, on Windows,
macOS or Linux.

```bash
git clone https://github.com/TheElliotGong/Stained-Glass-Genetic-Algorithm
cd Stained-Glass-Genetic-Algorithm
python -m venv venv
```

Activate the environment. On Windows PowerShell:

```powershell
.\venv\Scripts\Activate.ps1
```

On macOS or Linux:

```bash
source venv/bin/activate
```

Then install and run:

```bash
pip install -r requirements.txt      # runtime
pip install -r requirements-dev.txt  # plus everything the tests need
python run_web.py                    # open http://127.0.0.1:8000
```

---

## What was deliberately left alone

The fitness function (L1 distance over RGB) and the evolutionary schedule are
unchanged, so results stay comparable with the original and with each other.
Changes that would alter early-generation results, such as scoring against a
downscaled proxy image or retuning the schedule, are candidates for a separate,
opt-in pass rather than part of the behavior-preserving work above.

---

## Credits

Forked from Sebastian Proost's
[Voronoi-based genetic art algorithm](https://github.com/4dcu-be/Genetic-Art-Algorithm-part-2),
then extended with the faster renderer, image pre-processing, cell outlining,
the tiled solver, the performance work, the web application and the test suite.

Contributors: Elliot Gong, Cesar Ramirez, Md Islam, Alex Alcazar.

Licensed under the MIT License - see [LICENSE](LICENSE).

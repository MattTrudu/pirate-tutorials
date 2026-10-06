# PIRATE offline tutorials

Three short Jupyter notebooks for students learning to use PIRATE on saved
observations. This repository contains teaching material and native example
configs. Install PIRATE separately; no changes to its source fork are needed.

Each notebook follows **scenario -> config -> terminal command -> read and plot**.
PIRATE performs the simulation, dedispersion, and candidate extraction. Notebook
Python cells only read saved data, validate its format, and visualize it.
An allocated NVIDIA GPU and a working PIRATE installation are required.

The instructor has reported a successful local walkthrough of the original
notebooks. This standalone edition was prepared by file editing and static
inspection only; it has not been rerun. Fir setup and resource requirements
still need validation. Saved notebook outputs are intentionally cleared.

## Start here

1. Follow [Personal setup on Fir](SETUP_FIR.md), or use a personal GPU environment
   with the required PIRATE version below. The Fir-specific module and native
   dependency recipe must be completed by the instructor before the course.
2. Check that the terminal and notebook kernel import the same PIRATE checkout,
   using the [environment check](SETUP_FIR.md#check-the-selected-python-environment).
3. Start the terminal and notebooks in this tutorial repository, beside
   `configs/`. Activate your environment in the allocated terminal.
4. Work through the notebooks in order. Fill in the blank paths before running
   their commands or Python cells. Wait for each terminal command to finish
   successfully before reading its results in the notebook.

| Notebook | Terminal command | What you inspect |
| --- | --- | --- |
| [01: Injection](01_injection.ipynb) | `python -m pirate_frb.make_simulated_acq` | Channel widths, noise, and the dispersed pulse |
| [02: Dedispersion](02_dedispersion.ipynb) | `python -m pirate_frb run offline_dedisperser` | S/N and argmax maps; matching and incorrect DM |
| [03: Search](03_search.ipynb) | `python -m pirate_frb run offline_grouper` | Detections, grouped candidates, and their source pixels |

Bash blocks are instructions to copy into the terminal. **Run All does not
execute Markdown command blocks.** Once the saved inputs exist, the Python
cells can run from a fresh kernel in each notebook without hidden shared state.

## Required PIRATE version

These source revisions define the APIs and configs used here. An older
installation with the same package or kernel name is not interchangeable.

| Component | Source | Revision |
| --- | --- | --- |
| PIRATE | [MattTrudu/pirate](https://github.com/MattTrudu/pirate) | `c7269d69ae7327bac71793b780f78cc3c2d15cc8` |
| ksgpu | [kmsmith137/ksgpu](https://github.com/kmsmith137/ksgpu) | `5c81be2dea54226587f80eb206a35ca92c7160c5` |

Initialize PIRATE's submodules at that revision. [SETUP_FIR.md](SETUP_FIR.md)
contains the checkout and build steps. These pins identify the source snapshot,
not a claim of a validated Fir build. Students need access to both source
repositories and must keep their own checkouts and environments.

## Paths and output layout

Each notebook deliberately leaves one personal path blank:

```python
run_directory = ""  # REQUIRED: full path to the run folder you chose in the terminal.
```

Fill it with your own full path between the quotes. The following lines check
that it is filled before constructing a Path. A bare `Path("")` would silently
mean the current directory, so the tutorial does not use that as a placeholder.
Python also accepts `~` here because the path is expanded explicitly.

The Bash commands use `RUN_DIR=""` for the same folder. Use a full absolute
path, or `$HOME/...` in Bash; a quoted `~/...` is not expanded by the shell.
Do not add spaces inside the quotes unless they belong to the directory name.
The terminal and notebook do not share variables: set both fields yourself.

Choose a new run folder inside an existing writable location. The generation
block creates it and keeps all steps together, stopping if mkdir fails. Its
layout is:

```text
YOUR_RUN_FOLDER/
    metadata.yml
    dedispersion.yml
    grouper.yml
    generation.log
    frames/
        frame_b1_t0.asdf
        ... four intensity frames ...
        frame_b1_t0_snrmap.asdf
        ... four maps after notebook 2 ...
    events.asdf
```

Keep the intensity frames and maps together in `frames/` for this walkthrough.
If using an existing dataset with another layout, adjust the frame, map, and
catalog paths explicitly; the notebooks do not discover them automatically.
`--output` for the grouper takes a **filename**, such as `events.asdf`, not
an existing directory.

The three configs use native PIRATE formats:

- `configs/metadata.yml`: beam, frequency channels, sampling, and noise variance.
- `configs/dedispersion.yml`: dedispersion tree and saved map resolution.
- `configs/grouper.yml`: detection threshold and grouping settings.

The terminal generation block copies them into your run folder. Later commands
use those copies. Pulse settings are generator CLI options, recorded in its
printed summary in `generation.log`; this CLI has no noise-seed option.
Preserve the actual command too when changing pulse options.

## The common experiment

The example contains one beam, a 300-1500 MHz band, and four chunks of 2048
samples at 0.9984 ms: 8.1788928 seconds in total. The pulse has DM 20 pc cm^-3,
arrival time 5 s at 300 MHz, Gaussian sigma 3 ms, and injected S/N 200.
Its intentionally bright signal is easier to see in the first spectrogram.

The notebooks explain the piecewise channel widths and their approximate
smearing equalization. Plots use the physical frequency edges. Display averaging
does not change the data used by PIRATE.

The dedispersion image is a filtered S/N search map: `out_max` retains the
best trial in each coarse cell, and `out_argmax` identifies the winning fine
trial. It is not a raw dedispersed waterfall. Candidate widths are nominal
filter widths, not necessarily the injected Gaussian sigma. Recovered DM,
time, and S/N need not exactly equal their injected values.

## Reruns and sharing

| Change | What to repeat |
| --- | --- |
| Pulse or observation | Choose a new run folder; repeat the three terminal stages and plots |
| Plot limits or displayed DM row | Only the relevant plotting cells |
| Detection threshold | Copy and edit the saved grouper config; run grouping with a new catalog filename |

Reuse the saved frames when comparing processing on exactly the same noise.
Regenerating a pulse does not promise identical noise. The dedisperser replaces
its maps on rerun; the grouper replaces the specified catalog. Preserve any
files and configs needed for comparisons. Restarting a kernel does not remove
saved data.

Generated data, checkpoints, and local environments are excluded by
`.gitignore`. Before sharing notebook changes, clear their outputs and replace
personal path values with the blank fields again. The notebooks and configs
are sufficient; do not distribute a copied virtual environment.

## Resource expectations

A full dequantized frame has 28160 x 2048 float32 values, about 220 MiB before
other allocations. Notebook 01 expands one frame at a time and averages only
for display. Four packed intensity payloads total about 110 MiB before metadata,
ASDF overhead, maps, and catalogs. Each saved one-tree map is 4096 x 128.
Actual runtime and peak host/GPU memory have not been measured for this package.
Measure them before selecting a small GPU/MIG allocation for a class.

## Later execution checks

These checks have not been run as part of preparing this standalone repository.

1. Build the pinned sources in a fresh personal Fir environment, then verify
   terminal and kernel imports in a new GPU session under a second account.
2. Fill in the blank paths and follow all three terminal stages. Run the plotting
   cells from clean kernels and confirm four frames, four maps, and a catalog.
3. Check visibility, axes, units, and all three exercises. Confirm blank paths and
   existing run folders produce understandable errors before processing.
4. Locate a beam-1 candidate near the injection without STARTUP_INCOMPLETE
   or grouping timeout. Compare decoded coordinates rather than coarse pixels;
   do not require exact S/N, Gaussian/filter width equality, or one event only.
5. For a tutorial association criterion, compare the candidate and injection
   arrival times at both band edges. Allow half the recovered nominal filter
   width, twice the injected sigma, half the largest intra-channel smear, and
   two time samples. This is not a statistical uncertainty or full filter-support
   bound. Use PIRATE's [dispersion_delay](https://github.com/MattTrudu/pirate/blob/c7269d69ae7327bac71793b780f78cc3c2d15cc8/pirate_frb/simpulse/__init__.py)
   for the two nonnegative DMs separately, with the same reference frequency.
6. Record software versions, commits, GPU profile, runtime, peak memory, and
   disk size. Repeat processing on the same saved frames; grouped-event counts
   need not change monotonically when raising the threshold.

See the pinned [PIRATE installation guide](https://github.com/MattTrudu/pirate/blob/c7269d69ae7327bac71793b780f78cc3c2d15cc8/notes/install.md)
and [one-beam example](https://github.com/MattTrudu/pirate/blob/c7269d69ae7327bac71793b780f78cc3c2d15cc8/examples/simple_frb/README.md)
for further software context.

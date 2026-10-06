# Personal setup on Fir

Each student needs their own PIRATE checkout, Python environment, Jupyter
kernel, and output files. Download or clone this separate tutorial repository
alongside your software checkouts. It contains no PIRATE implementation.
Setup is separate from the three science notebooks.

**Status: these instructions were checked against source and documentation,
but have not been executed on Fir.** The course instructor must resolve the
native dependency and module choices below before students follow the build
steps. The commands are instructions for a later setup session; none were run
while preparing this tutorial.

## Before the course: complete the Fir environment recipe

PIRATE contains compiled C++/CUDA code. Creating a Python virtual environment
alone does not install the compiler or native libraries. The project's
[installation guide](https://github.com/MattTrudu/pirate/blob/c7269d69ae7327bac71793b780f78cc3c2d15cc8/notes/install.md) currently documents an Ubuntu,
physical-GPU, Conda-based build. Its [minimal environment](https://github.com/MattTrudu/pirate/blob/c7269d69ae7327bac71793b780f78cc3c2d15cc8/environment_minimal.yml)
is a dependency inventory, not a tested Fir environment specification.

Use Fir's supported Python environment workflow and obtain the native
dependencies through the cluster's supported software mechanisms. The
instructor and cluster support need to settle the following once for the class:

| Item | Requirement from the pinned PIRATE source | Still to establish on Fir |
| --- | --- | --- |
| Source revisions | The PIRATE and ksgpu commits pinned in [README.md](README.md#required-pirate-version), plus PIRATE's recorded `asdf-cxx` submodule | Confirm access and build this exact pair on Fir |
| Python | `>= 3.11` in `pyproject.toml` | One Python module/version for every student |
| Compiler | System `nvcc`, CUDA headers, and compatible GCC/G++ | Exact module combination and CUDA/driver compatibility |
| Native libraries | yaml-cpp, gRPC C++/protobuf, compatible Abseil, Intel MKL, MathDx/cuFFTDx headers | Module or personal installation paths, link paths, and runtime library paths |
| Code generation | `protoc`, `grpc_cpp_plugin`, Python `grpcio-tools` | Mutually compatible versions |
| GPU Python support | CuPy and compiled ksgpu | A CuPy build compatible with the selected CUDA stack |
| Build resources | Existing project instructions use `make -j 32` and require a visible GPU | A permitted build allocation with sufficient CPU and memory resources |
| Notebook resources | One visible GPU; host and GPU working memory | Measured memory, CPU count, duration, and allowed GPU/MIG profile |
| Jupyter | Personal kernel using the same software stack as the build | Module loading and environment inheritance in Fir JupyterHub |

Record the selected modules, package versions, source commits, and any private
`config.mk` settings in the course handout after validation. Keep the installation
in each student's own persistent space, at a path visible from compute nodes.
Each student should build their own environment rather than copy one from another account.

Fir's H100 GPUs are described in [WestDRI's Fir introduction](https://training.westdri.ca/getting-started/).
H100 has [CUDA compute capability 9.0](https://developer.nvidia.com/cuda/gpus).
This checkout's [Makefile](https://github.com/MattTrudu/pirate/blob/c7269d69ae7327bac71793b780f78cc3c2d15cc8/Makefile) defaults to `sm_80`, `sm_86`, and
`sm_89`, so its default architecture list does not include H100. For an H100
build, the existing [build override](https://github.com/MattTrudu/pirate/blob/c7269d69ae7327bac71793b780f78cc3c2d15cc8/notes/build.md#overriding-makefile-variables)
is:

```make
# In the student's PIRATE config.mk (already ignored by Git).
NVCC_ARCH = -gencode arch=compute_90,code=sm_90
```

Confirm architecture support in ksgpu and the selected CuPy build as well.
For non-Conda builds, `EXTRA_INCDIRS` can supply header paths; native library
search paths must also be configured. The Makefile automatically adds Conda
library paths only when `CONDA_PREFIX` is set. A plain virtual environment does
not provide that behavior. Refer to the build guide instead of assuming that
installing Python packages resolves linker errors.

The [official Fir](https://docs.alliancecan.ca/wiki/Fir),
[Python](https://docs.alliancecan.ca/wiki/Python), and
[JupyterHub](https://docs.alliancecan.ca/wiki/JupyterHub) pages are the definitive
platform references. Their text could not be retrieved during preparation due
to the site's access protection, so exact module names and Fir form settings
are deliberately left unresolved. This is a preparation task for the instructor,
not something students should have to guess.

## One-time setup for each student

Follow these steps after receiving the validated course environment recipe.

### 1. Obtain your own source checkouts

In your chosen persistent source directory, create sibling checkouts:

```bash
git clone --recursive https://github.com/MattTrudu/pirate.git
git -C pirate checkout c7269d69ae7327bac71793b780f78cc3c2d15cc8
git -C pirate submodule update --init --recursive
git clone https://github.com/kmsmith137/ksgpu.git
git -C ksgpu checkout 5c81be2dea54226587f80eb206a35ca92c7160c5
```

These commands select the revisions used to prepare this tutorial; a default
branch or an older checkout can provide different APIs. Preserve both source
directories because editable installations and native library paths refer to
them. The notebooks belong to this separate tutorial repository and do not
need to be present inside the PIRATE checkout.

### 2. Create a personal Python environment

Load the course's validated Python and native dependency modules first.
The following commands create an environment in your own home directory:

```bash
mkdir -p "$HOME/.venvs"
python -m venv "$HOME/.venvs/pirate-tutorial"
source "$HOME/.venvs/pirate-tutorial/bin/activate"
```

Install the course-pinned Python package versions using the Alliance Python
installation instructions. The packages required by this checkout include
`numpy`, `pybind11`, `editables`, `PyYAML`, `argcomplete`, `asdf`, `grpcio`,
and `grpcio-tools`; building also needs `pipmake` and the selected compatible
CuPy installation. The notebooks add only `matplotlib` and `ipykernel`.
These names come from [pyproject.toml](https://github.com/MattTrudu/pirate/blob/c7269d69ae7327bac71793b780f78cc3c2d15cc8/pyproject.toml) and the supplied
environment files. This list does not replace the native dependencies above.

CuPy is intentionally omitted from PIRATE's automatic pip dependencies, and
ksgpu must be installed from its source checkout. Therefore a bare
`pip install pirate_frb` is not the setup procedure for this course.

### 3. Build ksgpu and PIRATE inside a GPU allocation

Use the course's approved build allocation and load the same module stack as
above. Activate your environment. Complete the native library path settings
and H100 architecture settings before building.

The following commands follow the existing project installation instructions.
Run them from the parent directory containing `ksgpu/` and `pirate/`, after
installing the Python build dependencies:

```bash
python -m pip install --no-build-isolation -v -e ./ksgpu
cd pirate
make -j 32
python -m pip install --no-build-isolation -v -e .
```

Do not run the build, processing commands, or notebooks on a login node. The instructor must
arrange resources appropriate for the project's 32-way build; notebook
sessions do not automatically need that CPU allocation. A completed build
still needs the runtime checks in the [tutorial validation plan](README.md#later-execution-checks).

### 4. Register your notebook kernel

With `ipykernel` installed in the active personal environment:

```bash
python -m ipykernel install --user --name pirate-tutorial --display-name "Python (PIRATE tutorial)"
```

This uses the standard [IPython kernel registration procedure](https://ipython.readthedocs.io/en/stable/install/kernel_install.html).
It records your environment's Python executable. It does not allocate a GPU
or automatically load the native modules. The instructor must verify that the
Fir session supplies the same modules and runtime library paths before the
kernel starts. Moving the environment later invalidates the registered path.

## At the start of each notebook session

1. Sign in with your own account at [Fir JupyterHub](https://jupyterhub.fir.alliancecan.ca/).
   Its availability is recorded in the [Alliance service announcement](https://status.alliancecan.ca/view_incident?incident=1390).
2. Start a GPU-backed session under your own user account, using the course's
   validated allocation, resource, and time settings. An existing CPU-only
   session needs to be replaced by a GPU session. Use the instructor's selected
   GPU profile; a small dataset
   does not prove that the smallest GPU slice has sufficient memory.
3. Load the course's runtime modules using Fir's documented Jupyter workflow
   before starting the Python kernel. Activating an environment in a separate
   terminal does not change an already running notebook kernel.
4. Open a terminal attached to that GPU allocation. Load the same course
   modules, activate your own environment, and change to the tutorial
   repository folder, beside the notebooks and `configs/`. After loading
   the modules, fill in the empty path below:

   ```bash
   source "$HOME/.venvs/pirate-tutorial/bin/activate"
   TUTORIAL_DIR=""  # REQUIRED: full path to your separate tutorial repository.
   cd "${TUTORIAL_DIR:?Fill in TUTORIAL_DIR first}"
   ```

   All notebook terminal commands assume this working directory. The
   `${TUTORIAL_DIR:?...}` expression stops the command if the path is empty.
   A separate login-node terminal is not a substitute for the allocated terminal.
5. Open `01_injection.ipynb` in the tutorial repository and select
   **Python (PIRATE tutorial)**. Run the environment check below before loading
   a frame. Fill in the notebook's empty `run_directory` with the same full
   run-folder path used in its Bash commands.
6. Follow the [notebook order and rerun instructions](README.md). Each notebook
   first gives a PIRATE command to copy into the terminal. Wait for it to
   succeed, then use the notebook's Python cells to read and plot its results.
   Markdown command blocks do not run when you select **Run All**. Save your
   notebooks and results, then stop the Jupyter session when finished.

If the kernel cannot import PIRATE, first check the selected kernel, editable
installation, and loaded runtime modules. If it imports but reports no GPU,
check that the session was allocated a GPU. Respect the scheduler's device
visibility; do not override `CUDA_VISIBLE_DEVICES` to select another user's GPU.

## Check the selected Python environment

Kernel names are only labels. An older kernel named PIRATE may import a
different source checkout, even while you are viewing the right notebooks.
Changing the working directory does not select an installation.

In the activated terminal, run:

```bash
python -c 'import sys, pirate_frb; print(sys.executable); print(pirate_frb.__file__)'
```

In a temporary notebook cell, run:

```python
import sys
import pirate_frb
print(sys.executable)
print(pirate_frb.__file__)
```

Both must identify your intended personal environment and the PIRATE checkout
you built at the pinned revision. In VS Code, select the matching Python
environment through the notebook kernel selector; do not rely on the display
name alone. Restart the kernel after switching and rerun imports before using
AssembledFrame. If environment activation fails, resolve the required module
and native-library paths before continuing. The terminal's activation does
not update an already running kernel.

## Evidence to record before teaching

Use a fresh personal environment to carry out the one-time setup, then open
a second GPU session. Follow the terminal commands in notebooks 01, 02, and 03,
then run their plotting cells from clean kernels.
Record the module list, Python/CUDA/compiler/CuPy versions, source commits,
GPU profile, elapsed time, peak host/GPU memory, and output size. Repeat the
setup under a separate student account to establish that it needs no files
or permissions belonging to the instructor.

The instructor reported a successful local walkthrough of the source
notebooks. This standalone edition has not been rerun, and Fir installation,
resource requirements, and scientific recovery still need the checks above.

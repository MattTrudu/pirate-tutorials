
By the end of this guide, you will have your own installation of PIRATE and
will have checked that it runs on one of Fir's H100 GPUs. You only need a
terminal connected to Fir.

This guide assumes you can already log in to Fir. Use a fresh **Bash** login
session, with no Conda or Python environment activated. 

## How to follow the guide

Run the numbered steps in order. Copy one complete code block, wait for the
terminal prompt to return, and read the result before copying the next block.
If a command reports an error, stop at that step and save the error message.

- `$HOME` means your home directory and `$USER` means your username. Copy these names exactly; Bash fills in their values for you.
- The only values you need to edit are the two project account names in
  step 5. Keep the software versions, directory names, and other options as
  written.
- Some blocks create a file or run several commands together. Copy them from
  the first line through the final label, such as `ENV_FIR` or `TESTJOB`.
  Lines starting with `#` are comments and can be copied too.

Fir has different computers for different activities:

| Where you are | What you will do there |
| --- | --- |
| **Login node** — where your SSH session starts | Create the environment, download software, and submit jobs. |
| **CPU compute node** — entered in step 5 | Compile the software. Compilation turns its source code into programs and libraries. |
| **GPU compute node** — assigned to the job in step 9 | Test the installation using an H100 GPU. You will read the results from a log file. |

Fir's scheduler, **Slurm**, assigns compute nodes to jobs. Waiting for Slurm
to assign a node does not mean the installation has failed. You do not need a
GPU while downloading or compiling the software.

## 1. Create your Python virtual environment

**Where: login node.**

A **module** selects software already provided by Fir. A **virtual environment**
is a directory containing the Python packages for your installation. Loading
a module or activating an environment affects your current terminal; the
installed files remain available after you log out.

First, select the Python version used in this guide:

```bash
module load StdEnv/2023 gcc/12.3 python/3.11.5
python --version
```

You should see `Python 3.11.5`.

Create the directories and a virtual environment called `pirate`. Run this
block **once** during a fresh installation:

```bash
mkdir -p "$HOME/venvs" "$HOME/src" "$HOME/opt/pirate" \
    "$HOME/scratch/pirate-build"
python -m venv "$HOME/venvs/pirate"
```

Creation normally finishes without printing a message. Now **activate** the
environment so subsequent Python commands use it:

```bash
source "$HOME/venvs/pirate/bin/activate"
which python
python -m pip --version
```

The first path must end in `/venvs/pirate/bin/python`. The pip output must
also contain `/venvs/pirate/`. Your prompt will usually begin with `(pirate)`.
If the paths point elsewhere, stop here: packages would be installed into the
wrong environment.

Keep this terminal open for steps 2–5. Later, you will save a script that
activates everything with one command each time you log in.

## 2. Install the Python packages PIRATE needs

**Where: the same login node, with `pirate` activated.**

PIRATE uses Python packages for arrays, GPU calculations, and reading and
writing data. It also needs compilers and libraries supplied by Fir's modules.
Load those modules first:

```bash
module load cuda/12.6 yaml-cpp/0.8.0 imkl/2023.2.0 \
    clang/17.0.6 cmake/3.31.0
nvcc --version
```

The output should contain `release 12.6`. Here, `nvcc` is the CUDA compiler
used to build GPU code.

Install the following package versions. Copy the **whole block**. It runs the
commands in order and stops if an installation command fails. It also installs
`setuptools` and `wheel`, the build tools needed to install `pipmake`:

```bash
bash <<'INSTALL_PACKAGES'
set -eo pipefail
source "$HOME/venvs/pirate/bin/activate"
python -m pip install --no-index --only-binary=:all: "pip==26.2.1"
python -m pip install --no-index --only-binary=:all: "setuptools>=70.1" wheel
python -m pip install --no-index --only-binary=:all: \
    "numpy==2.2.2" "cupy==13.3.0" "pybind11==3.1.0" \
    "editables==0.6" "PyYAML==6.0.3" "argcomplete==3.7.2" \
    "asdf==5.1.0" "grpcio==1.82.1" "grpcio-tools==1.76.0" \
    "protobuf==6.33.6"
python -m pip install --no-build-isolation "pipmake==0.0.3"
python -m pip check
INSTALL_PACKAGES
```

The last line should be `No broken requirements found.` This checks that the
Python package requirements agree with each other. It does not test a GPU yet.

Most packages come from the Alliance's collection of prebuilt Python packages;
`pipmake` is downloaded from PyPI. Keep the exact version pins above for
PIRATE's dependencies. The build tools `setuptools` and `wheel` are selected
from the available Alliance wheels. The [reference page](install-fir-reference.md)
explains the version choices.

## 3. Download the source code and GPU development files

**Where: login node.**

You will compile three projects: **PIRATE**, its GPU support library **ksgpu**,
and the C++ communication library **gRPC**. Their source code will live under
`$HOME/src`. Downloading them does not build or install them yet.

Copy this block. The long identifiers select specific source versions so the
instructions do not change whenever a repository is updated:

```bash
bash <<'DOWNLOAD_SOURCES'
set -euo pipefail
git clone --branch feature/controlled-chord-experiment \
    https://github.com/MattTrudu/pirate.git "$HOME/src/pirate"
git -C "$HOME/src/pirate" checkout --detach \
    c7269d69ae7327bac71793b780f78cc3c2d15cc8
git -C "$HOME/src/pirate" submodule update --init --recursive

git clone https://github.com/kmsmith137/ksgpu.git "$HOME/src/ksgpu"
git -C "$HOME/src/ksgpu" checkout --detach 5c81be2

git clone --branch v1.76.0 --depth 1 \
    https://github.com/grpc/grpc.git "$HOME/src/grpc-1.76.0"
git -C "$HOME/src/grpc-1.76.0" submodule update --init --recursive
echo "Source downloads complete."
DOWNLOAD_SOURCES
```

Wait for `Source downloads complete.` Git may print a message about a
**detached HEAD**: that is expected because you selected a fixed source version.
If Git reports that a destination directory already exists, stop and ask your
instructor whether it contains an earlier installation.

Next, download **MathDx**, which supplies files needed to compile some GPU
calculations. This block downloads and unpacks the archive:

```bash
bash <<'DOWNLOAD_MATHDX'
set -euo pipefail
mkdir -p "$HOME/opt/pirate/mathdx-25.12.1-cuda12"
curl --fail --location --retry 3 \
    --output "$HOME/src/nvidia-mathdx-25.12.1-cuda12.tar.gz" \
    https://developer.nvidia.com/downloads/compute/cublasdx/redist/cublasdx/cuda12/nvidia-mathdx-25.12.1-cuda12.tar.gz
tar -xzf "$HOME/src/nvidia-mathdx-25.12.1-cuda12.tar.gz" \
    -C "$HOME/opt/pirate/mathdx-25.12.1-cuda12"
test -f "$HOME/opt/pirate/mathdx-25.12.1-cuda12/nvidia-mathdx-25.12.1-cuda12/nvidia/mathdx/25.12/include/cufftdx.hpp"
echo "MathDx download complete."
DOWNLOAD_MATHDX
```

Continue when you see `MathDx download complete.` All source downloads are
now finished; the next compute allocation will be used for compilation.

## 4. Prepare PIRATE and save its environment settings

**Where: login node.**

### Prepare the code that saves results

The selected PIRATE version needs a small adjustment to work with the ASDF
package installed in step 2. ASDF is the file format PIRATE uses to save some
results. The following block saves and applies the prepared adjustment;
you do not need to edit the Python code yourself.

Run this block **once**, after step 3:

```bash
bash <<'PREPARE_PIRATE'
set -euo pipefail
cat > "$HOME/opt/pirate/asdf-compatibility.patch" <<'ASDF_PATCH'
--- a/pirate_frb/TriggerCatalog.py
+++ b/pirate_frb/TriggerCatalog.py
@@ -942,11 +942,11 @@
     os.close(fd)
     try:
         asdf.AsdfFile(tree).write_to(
-            temporary, all_array_compression=compression, write_checksums=True)
+            temporary, all_array_compression=compression)
         # Validate what ASDF actually serialized, not merely the source dict.
         # Eager loading also forces checksum/decompression errors before the
         # destination name becomes visible.
-        with asdf.open(temporary, lazy_load=False) as reopened:
+        with asdf.open(temporary, lazy_load=False, validate_checksums=True) as reopened:
             validate_trigger_catalog_tree(reopened.tree)
         os.replace(temporary, output)
     except BaseException:
ASDF_PATCH
git -C "$HOME/src/pirate" apply --check "$HOME/opt/pirate/asdf-compatibility.patch"
git -C "$HOME/src/pirate" apply "$HOME/opt/pirate/asdf-compatibility.patch"
echo "PIRATE source preparation complete."
PREPARE_PIRATE
```

Continue when you see `PIRATE source preparation complete.` If applying the
patch fails, stop and keep the error. Do not apply it a second time after it
has succeeded.

### Save the command you will use in future sessions

A new terminal does not automatically know which modules, Python environment,
and library directories PIRATE needs. Save these choices in `env-fir.sh`.
The block below **creates that file**; copy it exactly, including its final
`ENV_FIR` line:

```bash
cat > "$HOME/opt/pirate/env-fir.sh" <<'ENV_FIR'
module load StdEnv/2023 gcc/12.3 cuda/12.6 python/3.11.5 \
    yaml-cpp/0.8.0 imkl/2023.2.0 clang/17.0.6 || return
source "$HOME/venvs/pirate/bin/activate" || return

export PIRATE_GRPC_PREFIX="$HOME/opt/pirate/grpc-1.76.0"
export PIRATE_MATHDX_ROOT="$HOME/opt/pirate/mathdx-25.12.1-cuda12/nvidia-mathdx-25.12.1-cuda12/nvidia/mathdx/25.12"
export PATH="$PIRATE_GRPC_PREFIX/bin:$PATH"
export CPATH="$PIRATE_GRPC_PREFIX/include:$PIRATE_MATHDX_ROOT/include:$PIRATE_MATHDX_ROOT/external/cutlass/include${CPATH:+:$CPATH}"
export LIBRARY_PATH="$PIRATE_GRPC_PREFIX/lib${LIBRARY_PATH:+:$LIBRARY_PATH}"
export NVCC_ARCH="-gencode arch=compute_90,code=sm_90"

test -n "${CUDA_HOME:-}" || { echo "CUDA_HOME is missing" >&2; return 1; }
test -r "$CUDA_HOME/lib64/libcudart.so.12" || return 1
test -r "$CUDA_HOME/lib64/libnvrtc.so.12" || return 1
export LD_LIBRARY_PATH="$CUDA_HOME/lib64:$PIRATE_GRPC_PREFIX/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
export LD_PRELOAD="$CUDA_HOME/lib64/libnvrtc.so.12${LD_PRELOAD:+:$LD_PRELOAD}"
ENV_FIR
```

The prompt should return without an error. You will use this file in step 7
and in later sessions. Writing the file does not run it or compile anything.

## 5. Choose your project accounts and request a CPU node

**Where: start on the login node.**

A Slurm **account** identifies the project providing your compute allocation.
It is different from your username. List the accounts available to you:

```bash
sacctmgr -nP show assoc where user="$USER" format=Account | sort -u
```

Choose one CPU account ending in `_cpu` and one GPU account ending in `_gpu`.
Ask your instructor which project to use if you belong to more than one.
If either type is missing, ask your instructor to arrange access before
continuing.

**Edit only the two quoted account names below before pasting.** The example
names work only for students who have access to that project. This saves your
choices so you will not need to edit account names in later commands:

```bash
cat > "$HOME/opt/pirate/install-settings.sh" <<'PIRATE_SETTINGS'
export PIRATE_CPU_ACCOUNT="def-jhessels_cpu"
export PIRATE_GPU_ACCOUNT="def-jhessels_gpu"
PIRATE_SETTINGS

source "$HOME/opt/pirate/install-settings.sh"
printf 'CPU account: %s\nGPU account: %s\n' \
    "$PIRATE_CPU_ACCOUNT" "$PIRATE_GPU_ACCOUNT"
```

Check that the printed names match the accounts you chose. Now request one
CPU node with eight CPU cores and 32 GiB of memory for up to two hours:

```bash
source "$HOME/opt/pirate/install-settings.sh"
salloc --account="${PIRATE_CPU_ACCOUNT:?Set your account first}" \
    --nodes=1 --ntasks=1 --cpus-per-task=8 \
    --mem=32G --time=02:00:00 \
    srun --ntasks=1 --cpus-per-task=8 --pty bash -l
```

**Wait here.** A message saying the job is queued means Slurm is waiting for
resources. Do not paste step 6 until you receive a compute-node shell. The
two-hour limit starts when the allocation begins, not while it is queued.

When the new prompt appears, check where you are:

```bash
hostname
printf 'Slurm job: %s\nCPU cores: %s\n' "$SLURM_JOB_ID" "$SLURM_CPUS_PER_TASK"
```

You should see a compute-node name, a nonempty job number, and `CPU cores: 8`.
Stay in this terminal for steps 6–8. Keep the SSH session open while the build
is running. Your home-directory files remain accessible from the compute node.

## 6. Compile and install the C++ gRPC library

**Where: the CPU compute node from step 5.**

PIRATE needs the C++ version of gRPC as well as the Python package you installed
earlier. Here, CMake prepares the build, compiles it, and installs it in your
own `$HOME/opt/pirate` directory. The supplied options are already chosen for
this installation; leave them unchanged.

```bash
bash <<'BUILD_GRPC'
set -eo pipefail
: "${SLURM_JOB_ID:?Request the CPU allocation in step 5 first}"
module load StdEnv/2023 gcc/12.3 cmake/3.31.0
set -u
export PIRATE_GRPC_PREFIX="$HOME/opt/pirate/grpc-1.76.0"
export PIRATE_GRPC_BUILD="$HOME/scratch/pirate-build/grpc-1.76.0"
mkdir -p "$PIRATE_GRPC_BUILD"

cmake -S "$HOME/src/grpc-1.76.0" -B "$PIRATE_GRPC_BUILD" \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++ \
    -DCMAKE_CXX_STANDARD=17 -DBUILD_SHARED_LIBS=ON \
    -DCMAKE_INSTALL_PREFIX="$PIRATE_GRPC_PREFIX" \
    -DCMAKE_INSTALL_LIBDIR=lib -DgRPC_INSTALL_LIBDIR=lib \
    -DCMAKE_INSTALL_RPATH="$PIRATE_GRPC_PREFIX/lib" \
    -DCMAKE_INSTALL_RPATH_USE_LINK_PATH=ON \
    -DgRPC_INSTALL=ON -DgRPC_BUILD_TESTS=OFF \
    -Dprotobuf_BUILD_TESTS=OFF -DgRPC_DOWNLOAD_ARCHIVES=OFF \
    -DgRPC_ABSL_PROVIDER=module -DgRPC_PROTOBUF_PROVIDER=module \
    -DgRPC_CARES_PROVIDER=module -DgRPC_RE2_PROVIDER=module \
    -DgRPC_SSL_PROVIDER=module -DgRPC_ZLIB_PROVIDER=module \
    2>&1 | tee "$PIRATE_GRPC_BUILD/configure.log"
cmake --build "$PIRATE_GRPC_BUILD" --parallel "$SLURM_CPUS_PER_TASK" \
    2>&1 | tee "$PIRATE_GRPC_BUILD/build.log"
cmake --install "$PIRATE_GRPC_BUILD" \
    2>&1 | tee "$PIRATE_GRPC_BUILD/install.log"

"$PIRATE_GRPC_PREFIX/bin/protoc" --version
test -x "$PIRATE_GRPC_PREFIX/bin/grpc_cpp_plugin"
test -f "$PIRATE_GRPC_PREFIX/lib/libgrpc++.so"
test -f "$PIRATE_GRPC_PREFIX/lib/libprotobuf.so"
echo "gRPC installation complete."
BUILD_GRPC
```

This produces a lot of compiler output. Wait for the terminal prompt to
return. Successful completion ends with `libprotoc 31.1` and
`gRPC installation complete.` If that completion message is missing, stop and
save the error; do not proceed to the PIRATE build.

## 7. Compile and install ksgpu, then PIRATE

**Where: remain on the same CPU compute node.**

The following block loads the environment saved in step 4, builds ksgpu first,
and then builds PIRATE. It also registers both projects with your Python
environment so you can import them from any directory.

```bash
bash <<'BUILD_PIRATE'
set -eo pipefail
: "${SLURM_JOB_ID:?Request the CPU allocation in step 5 first}"
source "$HOME/opt/pirate/env-fir.sh"
set -u

make -C "$HOME/src/ksgpu" -j "$SLURM_CPUS_PER_TASK" lib \
    2>&1 | tee "$HOME/src/ksgpu/build-fir.log"
python -m pip install --no-index --no-deps --no-build-isolation \
    --editable "$HOME/src/ksgpu"

make -C "$HOME/src/pirate" asdf-cxx/include/asdf/config.hxx
make -C "$HOME/src/pirate" -j 4 build_wheel \
    2>&1 | tee "$HOME/src/pirate/build-fir.log"
make -C "$HOME/src/pirate" sdist_files.txt
python -m pip install --no-index --no-deps --no-build-isolation \
    --editable "$HOME/src/pirate"
python -m pip check
echo "PIRATE installation complete."
BUILD_PIRATE
```

Wait for `No broken requirements found.` followed by
`PIRATE installation complete.` PIRATE is now installed, but you still need
to test it.

The `--editable` installations use files in `$HOME/src/ksgpu` and
`$HOME/src/pirate` directly. **Keep these directories** after installation;
they are part of the working environment.

## 8. Check Python can load PIRATE, then release the CPU node

**Where: CPU compute node.**

First check that Python can load PIRATE and that CuPy finds the CUDA libraries
selected for this installation. This check does not run a GPU calculation:

```bash
bash <<'CHECK_IMPORTS'
set -eo pipefail
source "$HOME/opt/pirate/env-fir.sh"
python - <<'PY'
import cupy as cp
import ksgpu
import pirate_frb

assert cp.cuda.get_local_runtime_version() == 12060
assert cp.cuda.nvrtc.getVersion() == (12, 6)
print("PIRATE imports and CUDA library checks passed.")
PY
CHECK_IMPORTS
```

If you see `PIRATE imports and CUDA library checks passed.`, release the CPU
allocation by typing:

```bash
exit
```

You should see Slurm relinquish the allocation and return to your login-node
prompt. **The installation remains in your home directory.** Leaving the
compute node does not remove it.

## 9. Test PIRATE on an H100 GPU

**Where: login node. Slurm will run the test on a GPU node.**

A **batch job** is a file of commands that Slurm runs when resources become
available. You submit the file and read its output from a log. You do not
need an interactive GPU terminal for this test.

### Create the test job

Copy this entire block. It creates a file called `testjob.sh` requesting one
H100 GPU for up to ten minutes. The job checks a small CuPy GPU calculation
and then runs the selected PIRATE tests. Leave the block unchanged:

```bash
cat > "$HOME/opt/pirate/testjob.sh" <<'TESTJOB'
#!/bin/bash -l
#SBATCH --job-name=pirate-validate
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --gpus-per-node=h100:1
#SBATCH --mem=32G
#SBATCH --time=00:10:00

set -eo pipefail
source "$HOME/opt/pirate/env-fir.sh"
set -u
unset PYTHONOPTIMIZE
export OMP_NUM_THREADS="$SLURM_CPUS_PER_TASK"
export OPENBLAS_NUM_THREADS="$SLURM_CPUS_PER_TASK"
export MKL_NUM_THREADS="$SLURM_CPUS_PER_TASK"
cd "${SLURM_TMPDIR:-$HOME}"

date -Is
printf 'Job: %s  Node: %s\n' "$SLURM_JOB_ID" "$(hostname)"
module list 2>&1
git -C "$HOME/src/pirate" log -1 --format='%H %s'
nvidia-smi -L

srun --ntasks=1 --cpus-per-task="$SLURM_CPUS_PER_TASK" python -u - <<'PY'
import cupy as cp
import ksgpu
import pirate_frb

assert cp.cuda.get_local_runtime_version() == 12060
assert cp.cuda.nvrtc.getVersion() == (12, 6)
assert cp.cuda.runtime.getDeviceCount() == 1
assert cp.cuda.Device(0).compute_capability == "90"
assert cp.arange(10, dtype=cp.float32).sum().item() == 45.0
print("CuPy GPU check passed.")
PY

srun --ntasks=1 --cpus-per-task="$SLURM_CPUS_PER_TASK" \
    python -u -m pirate_frb test --gpu 0 --seed 137 -n 1 \
    --gddk --cdd2 --sbdd --gpfk --dd --chime --grouper --live
echo PIRATE_GPU_VALIDATION_PASSED
date -Is
TESTJOB
```

The prompt returning means the file has been written. The test has **not**
started yet.

### Submit the job

This block submits `testjob.sh` using the GPU account you saved in step 5.
It saves the job number so you do not need to copy or edit it manually:

```bash
bash <<'SUBMIT_TEST'
set -euo pipefail
source "$HOME/opt/pirate/install-settings.sh"
bash -n "$HOME/opt/pirate/testjob.sh"
pirate_submission=$(sbatch --parsable --account="$PIRATE_GPU_ACCOUNT" \
    --output="$HOME/opt/pirate/validate-%j.log" \
    "$HOME/opt/pirate/testjob.sh")
PIRATE_TEST_JOB=${pirate_submission%%;*}
printf '%s\n' "$PIRATE_TEST_JOB" > "$HOME/opt/pirate/last-validation-job.txt"
printf 'Submitted job %s.\n' "$PIRATE_TEST_JOB"
SUBMIT_TEST
```

You should see `Submitted job` followed by a number. Submission returns before
the test finishes. Unlike the interactive CPU allocation, this batch job
continues if you close your SSH connection.

### Wait for the result

Run this block to check the job. You can run it again later, including after
logging back into Fir:

```bash
PIRATE_TEST_JOB=$(cat "$HOME/opt/pirate/last-validation-job.txt")
sacct -j "$PIRATE_TEST_JOB" --format=JobID,State,ExitCode,Elapsed
```

Look at the row whose `JobID` is just the job number, without a suffix:

| State | What to do |
| --- | --- |
| `PENDING` | Wait. Slurm has not assigned a GPU yet. |
| `RUNNING` | Wait. The test is executing. |
| `COMPLETED` with `ExitCode` `0:0` | Read the log using the next block. |
| `FAILED`, `TIMEOUT`, `CANCELLED`, or another unsuccessful state | Read the log and ask your instructor for help. Do not call the installation validated. |

If there is no row yet, wait briefly and check again; accounting can take
time to update. The ten-minute time limit covers execution, not queue waiting.

After the job finishes, read the end of its log:

```bash
PIRATE_TEST_JOB=$(cat "$HOME/opt/pirate/last-validation-job.txt")
tail -n 40 "$HOME/opt/pirate/validate-$PIRATE_TEST_JOB.log"
```

**Success requires both:** Slurm reports `COMPLETED` with exit code `0:0`,
and the log ends with `PIRATE_GPU_VALIDATION_PASSED` followed by a timestamp.
You have now installed PIRATE and passed the selected GPU tests.

If the log does not exist, the job may not have started. Check its state
before looking for an installation problem. If a test fails, keep the full
log under `$HOME/opt/pirate` and send it to your instructor.

## 10. Use the installation in your next session

**Where: a new login session, an allocated compute-node terminal, or a job script.**

You do not need to repeat the installation. Each time you start a new terminal
or write a new job script, initialize the software with:

```bash
source "$HOME/opt/pirate/env-fir.sh"
```

This loads Fir's modules, activates your existing `pirate` environment, and
sets the library paths. In a job script, put it before any Python or PIRATE
commands. **It does not request compute resources.** Continue using Slurm to
run GPU calculations, as in step 9; use your course's job script for the actual
science exercise.

Keep `$HOME/venvs/pirate`, `$HOME/src/pirate`, `$HOME/src/ksgpu`, and
`$HOME/opt/pirate`. Avoid upgrading packages or changing the source version
during the course unless your instructor supplies updated instructions.

## If you need to stop or ask for help

If you disconnect before step 4 is complete, reload the modules in steps 1
and 2 and activate the existing environment; do not recreate it. After step 4,
use `source "$HOME/opt/pirate/env-fir.sh"` to restore the environment.

If a CPU allocation expires, request another one with the command in step 5
and resume the unfinished build block in step 6 or 7. Completed compilation
work can be reused. Do not restart the downloads or delete build directories.

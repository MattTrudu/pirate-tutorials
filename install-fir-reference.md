# Fir installation: technical reference

This page supports the [student installation guide](install-fir.md). It is
intended for instructors and for diagnosing a failed installation. The student
guide contains the complete procedure; this page is not an additional set of
required installation steps.

## Validation record and scope

The installation used as the basis for the guide finished the selected H100
tests with `PIRATE_GPU_VALIDATION_PASSED` on **13 October 2026 at 18:06 PDT**
(14 October at 01:06 UTC). The guide incorporates the corrections made during
that installation. The rewritten command blocks have been checked locally for
shell/Python syntax, and the ASDF patch has been checked against the expected
source. The consolidated student procedure has **not** been rerun from an
empty account on Fir.

The GPU job uses one iteration and seed 137 with `--gddk --cdd2 --sbdd --gpfk
--dd --chime --grouper --live`. This covers the selected kernel/dedispersion
groups and CHIME, grouper, and live-pipeline checks. It does not validate every
PIRATE feature, a student's science configuration, or production throughput.
Ten minutes was sufficient for the recorded validation; queue time and later
execution time can differ.

## Version baseline

| Component | Version or setting |
| --- | --- |
| Software environment | `StdEnv/2023` |
| GCC module | `gcc/12.3`; compiler reported `12.3.1` |
| CUDA module | `cuda/12.6`; `cudacore/12.6.2`, nvcc `12.6.77` |
| Python | `3.11.5`, virtual environment `pirate` |
| CMake | `3.31.0` |
| yaml-cpp / MKL | `0.8.0` / `2023.2.0` |
| clang-format | `17.0.6`, from `clang/17.0.6` |
| Native gRPC | `v1.76.0`; installed `protoc` reports `libprotoc 31.1` |
| MathDx SDK | `25.12.1-cuda12` |
| NumPy / CuPy | `2.2.2` / `13.3.0`, Alliance wheels |
| ASDF | `5.1.0`, with the catalog-writer patch below |
| Python gRPC / generator | `grpcio==1.82.1`, `grpcio-tools==1.76.0` |
| Python Protobuf | `6.33.6` |
| Python build backend | `pipmake==0.0.3` |
| ksgpu | `1.5.0`, commit `5c81be2` |
| PIRATE | `1.5.0`, commit `c7269d69ae7327bac71793b780f78cc3c2d15cc8` |
| asdf-cxx submodule | `0b5a316398ce6a4fb2cf678e4125898ad17fdff6` |
| gRPC source commit | `f5ffb68d8a2fd603dff16287e90a4ac571e1fec6` |
| GPU code target | H100, `-gencode arch=compute_90,code=sm_90` |

The PIRATE commit belongs to `feature/controlled-chord-experiment`. Checking
out a fixed commit intentionally produces a detached HEAD. A later branch
head may change dependencies, patch applicability, or tests; it is not covered
by this version baseline. The build targets H100 and is not a universal binary
for older GPU architectures.

The Python commands pin the direct dependencies, including pip, pybind11,
editables, PyYAML, and argcomplete. They do not fully lock every transitive
dependency. Alliance wheel versions may have a `+computecanada` suffix.
`pipmake` is installed from PyPI because no Alliance wheel was available during
the recorded installation.

The guide explicitly installs `setuptools>=70.1` and `wheel` before building
`pipmake` from its source archive. With `--no-build-isolation`, pip expects the
build tools to be present in the active environment. Older setuptools needs
the separate `wheel` package to supply `bdist_wheel`; without it, installation
can fail with `invalid command 'bdist_wheel'`. Setuptools includes that command
itself starting with [version 70.1.0](https://setuptools.pypa.io/en/stable/history.html#v70-1-0).
These build tools are selected from the Alliance wheelhouse rather than pinned
to versions from the original validation record.

## Why these build and runtime settings are needed

### Native libraries and CPU compilation

The Python `grpcio` package does not provide the C++ gRPC libraries and tools
needed to link PIRATE. The guide builds shared gRPC/Protobuf libraries and
installs them under `$HOME/opt/pirate/grpc-1.76.0`, including `protoc` and
`grpc_cpp_plugin`.

All gRPC submodules are downloaded on the login node. Setting
`gRPC_DOWNLOAD_ARCHIVES=OFF` then avoids configure-time archive downloads and
the cross-filesystem rename failure encountered when building in scratch from
sources in home. The option is defined in
[gRPC v1.76.0's CMake configuration](https://github.com/grpc/grpc/blob/v1.76.0/CMakeLists.txt).

CUDA compilation does not require an allocated GPU when the architecture is
specified explicitly. PIRATE's `build_wheel` target builds the libraries and
Python extension without the GPU-dependent configuration generation reached
through the default `all`/`lib` targets at this commit. Generate
`asdf-cxx/include/asdf/config.hxx` before the parallel build because its
dependency ordering is incomplete for other PIRATE source files.

The `clang/17.0.6` module supplies `clang-format`, used during kernel
generation. GCC remains the host compiler. `build_wheel` generates
`wheel_files.txt`; the separate `sdist_files.txt` target prepares the second
manifest inspected by grouper packaging checks.

### CUDA, MathDx, and CuPy

The inspected `libmathdx/0.3.2` module required CUDA 13.2. The guide therefore
uses the CUDA 12 MathDx archive with CUDA 12.6. The archive root is nested under
the extraction directory; both levels in `PIRATE_MATHDX_ROOT` are intentional.

The CuPy 14.1.0 wheel available during setup was built for CUDA 13.2. With nvcc
12.6 it failed during generated GPU-code compilation, including incomplete
`__nv_fp8_e8m0` errors. CuPy 13.3 supports CUDA 12.6 and requires NumPy `<2.3`;
the guide pins NumPy 2.2.2. See the
[CuPy 13.3 installation documentation](https://docs.cupy.dev/en/v13.3.0/install.html)
and [versioned dependency requirements](https://github.com/cupy/cupy/blob/v13.3.0/setup.py).

The Alliance CuPy 13.3 wheel reported a CUDA build version of `12020`.
Prepending `$CUDA_HOME/lib64` to `LD_LIBRARY_PATH` selected the CUDA 12.6
runtime, but NVRTC still resolved to 12.2. Preloading the absolute path
`$CUDA_HOME/lib64/libnvrtc.so.12` selected NVRTC 12.6. Both settings are included
in `env-fir.sh`, before Python starts.

The relevant runtime checks are:

```python
import cupy
assert cupy.cuda.get_local_runtime_version() == 12060
assert cupy.cuda.nvrtc.getVersion() == (12, 6)
```

`CUDA Build Version 12020` can still appear in `cupy.show_config()` and describes
the wheel's build, not a failure of these runtime checks. A driver version of
CUDA 13 does not require changing the toolkit used by this recipe.

### ASDF catalog patch

At the pinned commit, `TriggerCatalog.py` passes `write_checksums=True` to
`AsdfFile.write_to()`. ASDF 5.1 does not accept this keyword. Its writer already
computes checksums for ordinary array blocks. The patch removes the unsupported
argument and adds `validate_checksums=True` when reopening the temporary
catalog before publication. It preserves checksum validation. See the
[ASDF 5.1 writer API](https://github.com/asdf-format/asdf/blob/5.1.0/asdf/_asdf.py)
and [block writer](https://github.com/asdf-format/asdf/blob/5.1.0/asdf/_block/io.py).

The guide saves the patch as `$HOME/opt/pirate/asdf-compatibility.patch` and
runs `git apply --check` before applying it. The patch is an intentional local
source modification. Applying it twice will fail; that is not a reason to
discard the already-applied fix. Inspect the diff if resuming an installation:

```bash
git -C "$HOME/src/pirate" diff -- pirate_frb/TriggerCatalog.py
```

## File and log locations

| Path | Contents |
| --- | --- |
| `$HOME/venvs/pirate` | Personal Python environment |
| `$HOME/src/pirate`, `$HOME/src/ksgpu` | Source and native libraries used by the editable installations; keep these directories |
| `$HOME/opt/pirate/grpc-1.76.0` | Installed C++ gRPC/Protobuf libraries and tools |
| `$HOME/opt/pirate/mathdx-25.12.1-cuda12` | Extracted MathDx SDK |
| `$HOME/opt/pirate/env-fir.sh` | Modules, environment activation, compiler and runtime paths |
| `$HOME/opt/pirate/install-settings.sh` | Student's CPU and GPU project accounts |
| `$HOME/scratch/pirate-build/grpc-1.76.0/*.log` | gRPC configure, build, and install logs |
| `$HOME/src/ksgpu/build-fir.log` | ksgpu compilation log |
| `$HOME/src/pirate/build-fir.log` | PIRATE compilation log |
| `$HOME/opt/pirate/testjob.sh` | GPU validation job |
| `$HOME/opt/pirate/last-validation-job.txt` | Most recently submitted validation job ID |
| `$HOME/opt/pirate/validate-JOBID.log` | Validation output; replace `JOBID` with the job number |

## Troubleshooting

Use the complete error and job state to identify the failing stage before
changing packages. Preserve the source directories and completed build files.

| Symptom | Check or correction |
| --- | --- |
| Python path is outside `venvs/pirate` | Load the Python module and activate the environment. After student-guide step 4, source `env-fir.sh`. |
| Pinned Python wheel is unavailable | Use `avail_wheels PACKAGE` to inspect availability; choose and validate a revised version set before changing the course instructions. |
| Installing `pipmake` fails with `invalid command 'bdist_wheel'` | In the active `pirate` environment, run `python -m pip install --no-index --only-binary=:all: "setuptools>=70.1" wheel`, then retry the `pipmake` command in step 2. |
| Git destination already exists | Inspect the checkout and local changes. Resume the appropriate stage rather than deleting a working or partial installation. |
| ASDF patch does not apply | Check the PIRATE commit and its diff. The patch may already be applied, or the source may differ from the pinned version. |
| `asdf/config.hxx: No such file or directory` | Run the header-generation command before the parallel PIRATE build in student-guide step 7. |
| Kernel generation cannot find `clang-format` | Load `clang/17.0.6`; verify GCC and nvcc still report the baseline versions. |
| gRPC configure downloads an archive or reports `Invalid cross-device link` | Complete recursive submodule downloads on the login node and configure with `gRPC_DOWNLOAD_ARCHIVES=OFF`. |
| CuPy reports incomplete `__nv_fp8_e8m0` types | Check CuPy and CUDA versions. Use the pinned CuPy 13.3.0/NumPy 2.2.2 packages and the CUDA 12.6 environment. |
| `libcudart.so.12` cannot be loaded | Source `env-fir.sh` in the shell/job launching Python; verify `$CUDA_HOME/lib64` is in `LD_LIBRARY_PATH`. |
| NVRTC reports `(12, 2)` | Verify `LD_PRELOAD` selects `$CUDA_HOME/lib64/libnvrtc.so.12` before starting Python. |
| `write_to()` rejects `write_checksums` | Check the ASDF source patch in student-guide step 4. This Python change does not require recompiling C++. |
| Grouper packaging check reports a missing manifest | Generate `sdist_files.txt`; `build_wheel` supplies `wheel_files.txt`. |
| Job is `PENDING` / `PD` | Wait or inspect the scheduling reason with `scontrol show job JOBID`. No tests have run yet. |
| GPU job is `TIMEOUT` | Increase `#SBATCH --time` in `testjob.sh` and resubmit. Distinguish this from a Python exception. |
| `srun` exits with code 1 | Find the first traceback/error in the log; the exit code alone does not identify the cause. |

## Optional CPU catalog check

If diagnosing the ASDF change before obtaining a GPU, this small check uses
the fixture from the pinned test suite to write and reopen an empty catalog.
Run it after installation on allocated CPU resources:

```bash
source "$HOME/opt/pirate/env-fir.sh"
python - <<'PY'
from pathlib import Path
from tempfile import TemporaryDirectory
from pirate_frb.TriggerCatalog import write_trigger_catalog
from pirate_frb.tests.test_trigger_catalog import _metadata

with TemporaryDirectory() as directory:
    write_trigger_catalog(
        Path(directory) / "check.asdf", (),
        coverage=((100, 2), (100, 3), (101, 7)), metadata=_metadata(),
    )
print("CPU catalog write/reopen/checksum check passed.")
PY
```

## Record a validated installation

After a successful GPU job, an instructor can save the installed versions,
modules, and local source adjustment using these commands on a login node:

```bash
source "$HOME/opt/pirate/env-fir.sh"
python -m pip freeze > "$HOME/opt/pirate/requirements-validated.txt"
module list > "$HOME/opt/pirate/modules-validated.txt" 2>&1
git -C "$HOME/src/pirate" diff -- pirate_frb/TriggerCatalog.py \
    > "$HOME/opt/pirate/asdf-compatibility-record.patch"
{
    git -C "$HOME/src/pirate" rev-parse HEAD
    git -C "$HOME/src/pirate" submodule status
    git -C "$HOME/src/ksgpu" rev-parse HEAD
    git -C "$HOME/src/grpc-1.76.0" rev-parse HEAD
} > "$HOME/opt/pirate/source-versions.txt"
```

Keep these files with `testjob.sh` and the successful validation log. Editable
entries in `pip freeze` refer to personal source paths; the source version
record and patch are also needed to reproduce the installation. Copying the
virtual environment to another account is not a portable distribution method.

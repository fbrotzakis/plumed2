[![Homepage](https://img.shields.io/badge/Home-plumed.org-green.svg)](https://www.plumed.org)
[![Google group](https://img.shields.io/badge/Google_group-plumed--users-green.svg)](https://groups.google.com/g/plumed-users)
[![codecov](https://codecov.io/gh/plumed/plumed2/branch/master/graph/badge.svg)](https://codecov.io/gh/plumed/plumed2)
[![License: LGPL v3](https://img.shields.io/badge/License-LGPL%20v3-blue.svg)](https://www.gnu.org/licenses/lgpl-3.0)
[![Github Releases](https://img.shields.io/github/release/plumed/plumed2.svg)](https://github.com/plumed/plumed2/releases)
[![MacPorts package](https://repology.org/badge/version-for-repo/macports/plumed.svg)](https://repology.org/project/plumed/versions)
[![Anaconda-Server Badge](https://anaconda.org/conda-forge/plumed/badges/version.svg)](https://anaconda.org/conda-forge/plumed)
[![AUR package](https://repology.org/badge/version-for-repo/aur/plumed.svg)](https://repology.org/project/plumed/versions)
[![DPorts package](https://repology.org/badge/version-for-repo/dports/plumed.svg)](https://repology.org/project/plumed/versions)
[![FreeBSD port](https://repology.org/badge/version-for-repo/freebsd/plumed.svg)](https://repology.org/project/plumed/versions)
[![Spack package](https://repology.org/badge/version-for-repo/spack/plumed.svg)](https://repology.org/project/plumed/versions)
[![Bluesky](https://img.shields.io/bluesky/followers/plumed.org?label=Follow)](https://bsky.app/profile/plumed.org)

MEMMI: EMMI with the uncertainty in the mean estimate (this fork)
-----------------------------------------------------------------

This repository (`fbrotzakis/plumed2`) is PLUMED master plus the uncertainty in the mean estimate
(standard error of the mean, SEM) for [EMMI](https://www.plumed.org/doc-master/user-doc/html/EMMI/),
as used in Metainference/MEMMI simulations of cryo-EM maps with Parallel-Bias Metadynamics and reweighting
(see [fbrotzakis/MEMMI](https://github.com/fbrotzakis/MEMMI) for the input files).

It ports the original implementation by T. Löhr (`tlhr/plumed2`, branch `emmi-bias-sigmasem`) and corrects it:

- the SEM of a group of GMM components is the largest SEM of its components (as in METAINFERENCE with a single sigma);
  the original used the overlap of GMM component *i* for group *i*;
- sigma and SEM are added in quadrature, `sqrt(sigma^2 + scale^2 SEM^2)` (the original added them linearly);
- the Monte Carlo of sigma uses the same total uncertainty as the energy, and OUTLIERS is normalized with it;
- the SEM is not updated when the effective number of replicas is one.

New EMMI keywords: `OPTSIGMAMEAN`, `SIGMA_MEAN0` and `SEM_LEGACY` (reproduces the original implementation exactly,
only to compare with old simulations); output component `sigmaMean-<group>`.
Compared with the old branch, the forces with `REWEIGHT` and different replica weights, `biasDer`, the neighbor list and
PBMETAD (stretched Gaussians) also follow the corrections of current PLUMED master: do not restart from HILLS/GRID files of old runs.

### Tested software

| Component | Version used for testing |
|---|---|
| OS | Ubuntu 26.04 (glibc 2.43), x86_64 |
| Compilers | GCC/G++ 15.2 |
| MPI | Open MPI 5.0 (`openmpi-bin`, `libopenmpi-dev`) |
| Libraries | zlib (`zlib1g-dev`), FFTW 3 (`libfftw3-dev`, optional for PLUMED) |
| Build tools | GNU make, `patch`, CMake 4.2 (GROMACS needs >= 3.28) |
| CUDA toolkit | 13.5 from conda-forge (see below), NVIDIA driver 580 (CUDA 13.0); GPU RTX 4090 (sm_89) |
| GROMACS | 2026.4 (PLUMED patches are also provided for 2025.0 and 2024.3) |

On Ubuntu/Debian:

```bash
sudo apt install build-essential gfortran cmake openmpi-bin libopenmpi-dev zlib1g-dev libfftw3-dev patch git
```

### 1. Build PLUMED

```bash
git clone -b emmi-sem-master https://github.com/fbrotzakis/plumed2.git plumed2-memmi
cd plumed2-memmi
./configure --prefix=$HOME/opt/plumed-memmi --enable-mpi CXX=mpic++
make -j 16 && make install
# environment (add to a file you source, NOT to ~/.bashrc if other PLUMED versions are set there)
export PATH=$HOME/opt/plumed-memmi/bin:$PATH
export LD_LIBRARY_PATH=$HOME/opt/plumed-memmi/lib:$LD_LIBRARY_PATH
export PLUMED_KERNEL=$HOME/opt/plumed-memmi/lib/libplumedKernel.so
```

EMMI is in the `isdb` module, which is compiled by default.

### 2. CUDA toolkit

GROMACS must be built with CUDA: it refuses OpenCL on NVIDIA Volta and newer GPUs. CUDA 12.4 and older
cannot compile with glibc 2.41 or newer. If the system CUDA is too old, install a recent toolkit in an
isolated conda environment (nothing is installed system-wide; the toolkit minor version may be newer than
the one the driver reports, e.g. CUDA 13.5 runs on a CUDA 13.0 driver):

```bash
conda create -y -p $HOME/opt/cuda13 -c conda-forge --override-channels \
      cuda-nvcc=13.5 cuda-cudart-dev=13.5 cuda-profiler-api libcufft-dev cuda-version=13.5
```

(`cuda-profiler-api` provides `cuda_profiler_api.h`, needed by GROMACS.)

### 3. Patch and build GROMACS 2026.4 with CUDA

```bash
wget https://ftp.gromacs.org/gromacs/gromacs-2026.4.tar.gz
tar xzf gromacs-2026.4.tar.gz && cd gromacs-2026.4
plumed patch -p -e gromacs-2026.4          # plumed from step 1 must be in PATH
mkdir build && cd build
C=$HOME/opt/cuda13
cmake .. -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++ -DGMX_MPI=ON -DGMX_GPU=CUDA \
      -DCMAKE_CUDA_COMPILER=$C/bin/nvcc -DCMAKE_CUDA_HOST_COMPILER=$(which g++) -DCUDAToolkit_ROOT=$C \
      -DCMAKE_CUDA_ARCHITECTURES=89 -DGMX_BUILD_OWN_FFTW=ON -DGMX_HWLOC=OFF \
      -DCMAKE_INSTALL_RPATH=$C/lib -DCMAKE_INSTALL_PREFIX=$HOME/opt/gromacs-2026.4-plumed-cuda
make -j 16 && make install
```

Set `CMAKE_CUDA_ARCHITECTURES` for your GPU (89 = RTX 40xx/L40, 80 = A100, 90 = H100). Check with
`gmx_mpi --version` that it reports `GPU support: CUDA`.

GROMACS 2025 and newer load PLUMED at run time from `PLUMED_KERNEL`: always set it explicitly, otherwise
another PLUMED (e.g. one set in `~/.bashrc`) is loaded silently. Check the first lines of `PLUMED.OUT`.

### 4. Run MEMMI

Run with MPI ranks and **one OpenMP thread per rank**, e.g. 8 replicas with 4 ranks each
(GROMACS puts 3 PP ranks and 1 PME rank per replica; nonbonded and PME on the GPU):

```bash
export OMP_NUM_THREADS=1
mpirun -np 32 gmx_mpi mdrun -s topol.tpr -multidir r{0..7} -plumed ../plumed.dat \
       -ntomp 1 -npme 1 -nb gpu -pme gpu -update cpu
```

Important:

- **`-ntomp 1`**: with PLUMED master and more than one OpenMP thread per rank, EMMI runs crash with a
  segmentation fault in the GROMACS pair search (memory corruption; reproduced with unmodified upstream PLUMED,
  on CPU and GPU, so it is not caused by the SEM code). The GROMACS 2026.4 patch passes the GROMACS thread count
  to PLUMED; with one thread per rank PLUMED runs serially, as with older PLUMED versions.
- **`-update cpu`**: PLUMED forces are applied on the CPU at every step.
- **No energy groups** (`energygrps`) in the `.mdp`: GROMACS does not run nonbonded interactions on the GPU with energy groups.
- **Alignment with the cryo-EM map**: EMMI with `NOPBC` compares absolute coordinates, so the protein in the `.tpr`,
  the MOLINFO/`ADDREFERENCE` structure and the GMM file must be in the same (map) frame. Build the box without moving
  the protein (`gmx editconf -box ... -noc`, not `-c`, which shifts it). `WHOLEMOLECULES ... ADDREFERENCE` only picks
  the periodic image of each chain closest to the reference, it cannot undo a shift that is not a box vector.
- Do not restart from HILLS/GRID files written by the old `emmi-bias-sigmasem` branch (see above).

Branches and releases
---------------------

Several branches and tags are stored on the git repository.

Branches named `v2.X` correspond to release branches.

Master branch may contain non tested features and is not expected to be used by non-developers.
It typically contains features that will be available on the next release.

Tags named `v2.XbY` correspond to beta releases, use it with care.
Tags named `v2.X.Y` correspond to official releases, use the latest available.

In addition, the repository contains a number of other branches related to specific features.
Please contact the developers that are committing on those branches before basing your work
there, since they might contain temporary work and might be rebased later.
For instance, branch `testdoc` is setup so as to push a test copy of the manual
and is often force pushed.

To report problems found on beta or official releases, use the normal
[plumed-users@googlegroups.com](mailto:plumed-users@googlegroups.com)
mailing list. Please state exactly which version you are using.
To report problems found on `master` branch, use the
[plumed2-git@googlegroups.com](mailto:plumed2-git@googlegroups.com) mailing list.
This is also the correct place for discussions about new features etc.
When reporting please provide the git hash (you can obtain it with `git rev-parse HEAD`).

Status
------

Below you find the status on [GitHub Actions](https://github.com/plumed/plumed2/actions) for the release branches.

| Branch   |      Status   | First stable release (year) | Still supported |
|:--------:|:-------------:|:--------:|:------:|
| master   | [![CI](https://github.com/plumed/plumed2/actions/workflows/ci.yml/badge.svg?branch=master)](https://github.com/plumed/plumed2/actions/workflows/ci.yml) | 2026 (expected) | / |
| v2.10    | [![CI](https://github.com/plumed/plumed2/actions/workflows/ci.yml/badge.svg?branch=v2.10)](https://github.com/plumed/plumed2/actions/workflows/ci.yml)| 2025 | yes |
| v2.9     | [![CI](https://github.com/plumed/plumed2/actions/workflows/ci.yml/badge.svg?branch=v2.9)](https://github.com/plumed/plumed2/actions/workflows/ci.yml)   | 2023 | yes |
| v2.8     | [![CI](https://github.com/plumed/plumed2/actions/workflows/ci.yml/badge.svg?branch=v2.8)](https://github.com/plumed/plumed2/actions/workflows/ci.yml) | 2022 | no |
| v2.7     | [![CI](https://github.com/plumed/plumed2/actions/workflows/ci.yml/badge.svg?branch=v2.7)](https://github.com/plumed/plumed2/actions/workflows/ci.yml) | 2020 | no |
| v2.6     | [![CI](https://github.com/plumed/plumed2/actions/workflows/ci.yml/badge.svg?branch=v2.6)](https://github.com/plumed/plumed2/actions/workflows/ci.yml) | 2019 | no |
| v2.5     | [![CI](https://github.com/plumed/plumed2/actions/workflows/ci.yml/badge.svg?branch=v2.5)](https://github.com/plumed/plumed2/actions/workflows/ci.yml) | 2018 | no |
| v2.4     | [![CI](https://github.com/plumed/plumed2/actions/workflows/ci.yml/badge.svg?branch=v2.4)](https://github.com/plumed/plumed2/actions/workflows/ci.yml) | 2017 | no |
| v2.3     | Not available | 2016 | no |
| v2.2     | Not available | 2015 | no |
| v2.1     | Not available | 2014 | no |
| v2.0     | Not available | 2013 | no |

Content
-------

Here's a description of the content of each file and directory in the root PLUMED directory.

    .ci              : helper scripts used by continuous integration
    CHANGES          : change log
    CMakeLists.txt   : experimental CMake build for MSVC/Windows (not the official build system)
    COPYING.LESSER   : license
    Makefile         : makefile
    Makefile.conf.in : template configuration makefile
    PEOPLE           : list of authors
    PLATFORMS.md     : features available on each platform and in the Python package
    README.md        : this file
    VERSION.txt      : version file
    astyle           : a local version of astyle, used to format code
    cmake-msvc       : support files for the experimental CMake build
    conda            : conda packaging scripts
    configure        : configuration script
    configure.ac     : configuration script (autoconf)
    developer-doc    : developer documentation
    docker           : directory where Docker is generated
    fortran          : Fortran interfaces
    json             : tools to generate the JSON syntax file
    macports         : directory where Portfiles are generated
    new-manual       : user manual (MkDocs)
    patches          : patch scripts
    plugins          : optional plugins (e.g. pycv)
    python           : python package (wrappers and pip packaging)
    pythonformatter.sh : formats the python code (as astyle for C++)
    regtest          : regression tests, including reference results
    release.sh       : developer utility to publish releases
    ruff.toml        : configuration for the ruff Python linter
    scripts          : shell tools
    sourceme.sh.in   : template configuration script
    src              : source code
    stamp-h.in       : timestamp file used by autoconf
    vim              : directory where vim syntax is generated

Required software
-----------------

The following applies to the standard build from the sources. The features
available on each platform, and with the Python and conda packages, are
described in [PLATFORMS.md](PLATFORMS.md).

Required software on Linux and macOS:

* GNU make.
* C/c++ compiler (c++17 support is required as of version 2.10).
* A modern version of the `patch` command line tool.

Required software on Windows (experimental CMake build):

* [CMake](https://cmake.org) 3.18 or later.
* Visual Studio 2022, or the Build Tools for Visual Studio, with the
  "Desktop development with C++" workload (MSVC with C++17 support).

No other library is needed on Windows: BLAS and LAPACK are replaced by the
internal version, and parallel runs use the embedded thread-MPI. Running the
regression tests also needs a bash shell (e.g. Git Bash).

Suggested software on Linux and macOS (libraries are checked by `./configure`
and enabled if available):

* An external MPI library for distributed-memory parallel simulations, matching the MPI implementation used by your MD code. Serial builds can instead use embedded thread-MPI for local ranks; see [MPI runtime selection](src/mpiwrap/README.md).
* Optimized blas and lapack libraries (e.g. OpenBLAS or MKL). They are automatically replaced by an internal version if not available. They only make a difference for actions working on large matrices (e.g. DIAGONALIZE, INVERT_MATRIX or dimensionality reduction): the internal version is as fast as an optimized library for small matrices and faster than the reference netlib implementation.
* [Zlib library](https://zlib.net/) to use compressed data files.
* [HDF5 library](https://www.hdfgroup.org/solutions/hdf5/) to read and write tables in HDF5 files (`.h5`).
* [FFTW library](https://www.fftw.org/) for FOURIER_TRANSFORM (`fourier` module).
* [Chemfiles library](https://chemfiles.org/) to read more file formats in `plumed driver` and in the MOLINFO action (only used if requested with `--enable-libchemfiles`).
* [MkDocs](https://www.mkdocs.org) to build the user manual (Python requirements are listed in `new-manual/requirements.txt`).
* [Doxygen](https://www.doxygen.nl) to build the developer manual. Doxygen might need:
  * [Graphviz](https://www.graphviz.org) to show class hierarchy in
    developer manual.

MPI runtimes and optional packages
---------------------------------

A serial-built PLUMED kernel can use the MD code's MPI through injection or a
runtime provider, without rebuilding PLUMED. Embedded thread-MPI is available
through the explicit `--tmpi-threads N` launcher. See the
[MPI wrapper documentation](src/mpiwrap/README.md) for runtime selection,
`plumed mkmpi`, compatibility requirements and current limitations.

This also allows one base `plumed` conda package with small optional
`plumed-mpi` variants for MPICH and Open MPI, rather than separate full PLUMED
builds. Our [conda recipes](conda/README.md) prototype and test that layout;
this does not change conda-forge's current packages.

Modules and plugins
-------------------

The source code is organized in modules, one for each directory of `src`.
The `module.type` file of each module says when it is compiled:

* `always`: the basic infrastructure (e.g. `core`, `tools`), always compiled;
* `default-on`: the common actions (e.g. `bias`, `colvar`, `function`,
  `generic`, `multicolvar`, `opes`, `secondarystructure`), compiled unless they are
  disabled;
* `default-off`: specialized methods and modules that need external libraries
  (e.g. `ves`, `funnel`, `drr`, `sasa`, `pytorch`, `metatomic`), compiled only
  when they are requested.

Modules are selected when PLUMED is configured, with a `:`-separated list:

    ./configure --enable-modules=+ves:+drr    # add modules to the default ones
    ./configure --enable-modules=-isdb        # remove a default-on module
    ./configure --enable-modules=all          # all the modules
    ./configure --enable-modules=none:+bias   # only the always modules and bias

A module that needs an external library provides its actions only if PLUMED
is also configured with that library (see `./configure --help`, e.g.
`--enable-libtorch` for `pytorch`). The modules of an installed PLUMED are listed by
`plumed config show`, and `plumed config module ves` tells if a module is
enabled. The Python wheels (`pip install plumed`) include only the `always`
and `default-on` modules, and so does the CMake build for Windows unless
`-DPLUMED_MODULES` says otherwise (see [PLATFORMS.md](PLATFORMS.md)).

Plugins are compiled separately, against an installed PLUMED, and loaded at
run time with the `LOAD` action (`LOAD FILE=plugin.so` in the input, or
`plumed --load plugin.so` for the command line tools). This also works with the
Python wheels on Linux and macOS, but not on Windows. Plugins can be:

* your own actions: a C++ file with an action can be compiled with
  `plumed mklib MyCV.cpp`, or loaded directly with `LOAD FILE=MyCV.cpp`, which
  compiles it when PLUMED starts (a C++ compiler is needed in both cases);
* the plugins in the [plugins](plugins) directory, each with its own README and
  build instructions: [pycv](plugins/pycv/README.md) (collective variables and
  functions written in Python, installed with `pip`),
  [cudaCoord](plugins/cudaCoord/README.md) (coordination number on GPUs with
  CUDA) and [openaccPTM](plugins/openaccPTM/README.md) (collective variables on
  GPUs with OpenACC, including SAXS and SANS). cudaCoord and openaccPTM are
  compiled with `./configure` and `make` in their directory.

Python package and other packages
---------------------------------

The Python package, including a basic PLUMED kernel and the `plumed` command,
can be installed with

    python -m pip install plumed

Wheels are available for Linux (x86_64 and aarch64), macOS (Intel and Apple
Silicon) and Windows (x86_64). They are compiled without MPI libraries and
OpenMP, and include HDF5 and chemfiles; see [PLATFORMS.md](PLATFORMS.md) for the features
available on each platform and [python/README.rst](python/README.rst) for the
package itself.
PLUMED is also distributed by several package managers (see the badges above),
e.g. `conda install -c conda-forge plumed` on Linux and macOS; the features of
the conda packages are also described in [PLATFORMS.md](PLATFORMS.md).

Quick compilation instructions
------------------------------

Extensive installation instructions are in the [user documentation](https://www.plumed.org/doc).
Quick instructions for Linux and macOS (for Windows, see [below](#windows)):

    ./configure --prefix=$HOME/opt
    make
    make doc # optional
    make test # optional

User documentation can be found at `new-manual/site/index.html`.
Developer documentation can be found at `developer-doc/html/index.html`.
[Pre-compiled documentation](https://www.plumed.org/doc) is available online, so this is only required
if you are working with a modified version of the code!

In order to run PLUMED without installing it you should type `source sourceme.sh`. However,
we recommend installing PLUMED.
To install it in `$HOME/opt` (directory should be set during `./configure`):

    umask 022
    make install

Now you will be able to run plumed using e.g.

    plumed help

If you compiled your own documentation, paths to the installed documentation can be found with command `plumed info --user-doc`.

A sample modulefile with environment variable will be placed in
`$HOME/opt/lib/plumed/src/lib/modulefile`. This can be useful if you want to
install multiple PLUMED versions side by side and select them with env modules.

### Windows

On Windows, PLUMED is compiled with the experimental CMake build
(`CMakeLists.txt`), from a command prompt or PowerShell where `cmake` is
available (e.g. the Developer PowerShell for Visual Studio):

    cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
    cmake --build build --config Release --parallel
    cmake --install build --prefix <prefix>

Then `<prefix>\bin\plumed.exe help` runs PLUMED. Useful options, passed to the
first command:

* `-DPLUMED_SHARED_KERNEL=ON`: build a shared kernel (`plumedKernel.dll`) that
  can be loaded at run time (`PLUMED_KERNEL`, Python package), instead of a
  static one to be linked into an MD code;
* `-DPLUMED_MODULES=...`: the modules to compile, with the syntax of
  `./configure --enable-modules` (e.g. `+ves:+funnel`, or `all`). The modules
  that need a library not supported by this build (`drr`, `fourier`,
  `metatomic`, `pytorch`, `ttsketch`) are compiled without it, and their
  actions are not available; `kenref` is not compiled;
* `-DPLUMED_MSVC_DEBUG_INFO=ON`: function names in the stack trace of error
  messages.

A subset of the regression tests is run from Git Bash with
`.ci/windows-regtests serial` (and `mpi`, with the embedded thread-MPI, and
`modules`, for the `default-off` modules of a build with `-DPLUMED_MODULES=all`). The
script expects `build/plumed.exe`, as produced by the Ninja generator (add
`-G Ninja` to the first command, in the Developer PowerShell). If you only need
PLUMED from Python or the `plumed` command line tools, `python -m pip install plumed`
is simpler.

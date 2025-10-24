---
title: User Software
layout: default
parent: About Artemis
nav_order: 2
---

# Software
ARC provides pre-built software as [lmod](https://lmod.readthedocs.io/en/latest/index.html) modules for the *Intel* CPUs used on Great Lakes., However, Artemis uses *AMD* CPUs. Thus the provided modules **may or may not work** on Artemis.
- Software built for a generic x86/x64 architecture should work
- CPU-heavy codes should target the *specific microarchitecture* they run on

**ARC will not provide pre-built software for our CPUs**

Instead, you'll need to build your own software. I'd recommend using [spack](https://spack.io)
- [UM ARC's Spack Documentation](https://arc.umich.edu/spack/)
- [Spack's Documentation](https://spack.readthedocs.io/en/latest/)
- [Spack's Tutorial](https://spack-tutorial.readthedocs.io/en/latest/)
- [List of Available Spack Packages](https://packages.spack.io)
- [Public Build Cache](https://cache.spack.io)

> To get spack: `module load spack` to use the ARC provided instance or [install your own](https://spack.readthedocs.io/en/latest/getting_started.html#installation)

## Spack
The following is a suggested configuration for using Spack on Artemis. See spack documentation for more information.

In your `~/.bashrc` or equivalent:
```bash
# Load the spack module and unset SPACK_DISABLE_LOCAL_CONFIG
# to enable user configuration files in ~/.spack/
module load spack/0.21
unset SPACK_DISABLE_LOCAL_CONFIG
```

In `~/.spack/concretizer.yaml`
```yaml
concretizer:
  targets:
    # Lighthouse Login Nodes are Haswell, not Zen4
    # Allow host to be incompatible so concretizing on the login
    # nodes is possible
    host_compatible: false
```

In `~/.spack/packages.yaml`
```yaml
packages:
  openssl:
    # Don't build your own OpenSSL.
    # See: https://spack.readthedocs.io/en/latest/getting_started.html#openssl
    externals:
    - spec: openssl@1.1.1k
      prefix: /usr
    buildable: False

  # Restrict to CUDA provided by ARC
  # Realistically, you could install your own version of CUDA, but it's pretty heavy
  # Note: 'cuda::' overrides the ARC provided defaults, this is needed cause cuda@12.3.0
  # is not currently installed, but listed, causing issues.
  # Note: This should be updated as new versions of CUDA are installed
  cuda::
    buildable: False
    externals:
    - spec: cuda@10.2.89
      modules:
      - cuda/10.2.89
    - spec: cuda@11.2.2
      modules:
      - cuda/11.2.2
    - spec: cuda@11.3.0
      modules:
      - cuda/11.3.0
    - spec: cuda@11.5.1
      modules:
      - cuda/11.5.1
    - spec: cuda@11.6.2
      modules:
      - cuda/11.6.3
    - spec: cuda@11.7.1
      modules:
      - cuda/11.7.1
    - spec: cuda@11.8.0
      modules:
      - cuda/11.8.0
    - spec: cuda@12.1.1
      modules:
      - cuda/12.1.1
    - spec: cuda@12.6.3
      modules:
      - cuda/12.6.3
    - spec: cuda@12.8.1
      modules:
      - cuda/12.8.1

  # Versions of cuDNN provided by ARCH
  # Again, could install but a) space and b) getting the actual files is tricky (install is easy)
  # Note: This should be updated as new versions of cuDNN are installed
  cudnn:
    buildable: False
    externals:
    - spec: cudnn@8.3.1-10.2
      modules:
      - cudnn/10.2-v8.3.1
    - spec: cudnn@8.1.1-11.2
      modules:
      - cudnn/11.2-v8.1.1
    - spec: cudnn@8.2.1-11.3
      modules:
      - cudnn/11.3-v8.2.1
    - spec: cudnn@8.3.1-11.5
      modules:
      - cudnn/11.5-v8.3.1
    - spec: cudnn@8.4.1-11.6
      modules:
      - cudnn/11.6-v8.4.1
    - spec: cudnn@8.7.0-11.7
      modules:
      - cudnn/11.7-v8.7.0
    - spec: cudnn@8.7.0-11.8
      modules:
      - cudnn/11.8-v8.7.0
    - spec: cudnn@8.9.0-12.1
      modules:
      - cudnn/12.1-v8.9.0
    - spec: cudnn@9.6.0-12.6
      modules:
      - cudnn/12.6-v9.6.0
    - spec: cudnn@9.10.0-12.8
      modules:
      - cudnn/12.8-v9.10.0

  # The following applies for all packages
  all:
    # Check cuda_arch specification
    require:
    - one_of: ["cuda_arch=80,90", cuda_arch=80, cuda_arch=90]
      when: +cuda
      message: cuda_arch should be 80 (A100) or 90 (H100)

    # Enable certain variants by default
    variants:
    - +cuda             # Enable cuda support when possible

    # Artemis Worker Nodes are Zen4, keep x86_64 as a fallback
    target: [zen4, x86_64]

    # Sets preference for compilers
    compiler:
    - gcc
    - clang
    - aocc

    # The following biases Spack towards AMD specific providers
    providers:
      blas: [amdblis, openblas]
      fftw-api: [amdfftw, fftw]
      flame: [amdlibflame, libflame]
      lapack: [amdlibflame, openblas]
      scalapack: [amdsclapack, netlib-scalapack]

```

## Julia

## Minimizing Redundant Precompilation

As Artemis is a [heterogeneous cluster](./hardware.md#cluster-architecture), switching between different partitions can trigger frequent precompilation, as Julia attempts to recompile for the current microarchitecture. To avoid this set [`JULIA_CPU_TARGET`](https://docs.julialang.org/en/v1/manual/environment-variables/#JULIA_CPU_TARGET) to an appropriate setting.

For example, within your `~/.bashrc` or equivalent, put:

```shell
export JULIA_CPU_TARGET="generic;znver4,clone_all;znver3,clone_all;haswell"
```

This will instruct Julia to compile for Zen4 (Most Worker Nodes), Zen3 (A100 Nodes) and haswell (Head node), with a generic CPU target as a fallback. This will increase the size of `~/.julia/compiled` as 3 sets of system and package images will now need to be stored. For more information see the [Julia docs](https://docs.julialang.org/en/v1/manual/environment-variables/#JULIA_CPU_TARGET).


### Example Precompile Job Script
```shell
#!/bin/bash
#SBATCH -p venkvis-cpu
#SBATCH --cpus-per-task 4
#SBATCH --mem-per-cpu 1800M
#SBATCH --time=0:20:00
set -x
my_job_header

# Configure Julia
export JULIA_CPU_TARGET="generic;znver4,clone_all;znver3,clone_all;haswell"
export JULIA_PKG_PRECOMPILE_AUTO=0                        # Disable automatic precompilation so we control when it happens (Mainly for MPI)
export JULIA_NUM_PRECOMPILE_TASKS=$SLURM_CPUS_ON_NODE     # Limit precompilation tasks to the number of CPUs

# Uncomment if using MPI.jl
# module --ignore_cache load openmpi/4.1.6
# julia --color=no --startup-file=no --project -e 'using MPIPreferences; MPIPreferences.use_system_binary()'

# Precompile the Project
julia --color=no --startup-file=no --project -e 'using Pkg; Pkg.resolve(); Pkg.instantiate(); Pkg.precompile(timing=true)'
```

The job script would then look like:

```shell
# [Job Header Goes Here]

export JULIA_CPU_TARGET="generic;znver4,clone_all;znver3,clone_all;haswell"

julia --startup-file=no --project ...
```

> Note: `--startup-file=no` is needed to avoid loading packages from your base environment (i.e. Revise) and

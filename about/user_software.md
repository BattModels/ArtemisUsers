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

  # Versions of cuDNN provided by ARCH
  # Again, could install but a) space and b) get the actual files is tricky (install is easy)
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

  # The following applies for all packages
  all:
    # Check cuda_arch specification
    require:
    - one_of: ["cuda_arc=80,90", cuda_arch=80, cuda_arch=90]
      when: +cuda
      message: cuda_arch should be 80 (A100) or 90 (H100)

    # Enable certain variants by default
    variants:
    - +cuda             # Enable cuda support when possible

    # Artemis Worker Nodes are Zen4, keep x86_64 as a fallback
    target: [zen4, x84_64]

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

![[_a248c30b-4f63-4f2b-bcf1-5da143d07303 (1]].jpg)

---
## What's New
- Bigger Nodes: Higher Core counts, More Memory, TBs of NVMe scratch
- Faster GPUs: Between 2.5x and 36x faster
- More Storage: Up to 100TB of Archival Storage
- No pre-built modules will need to use [spack](https://spack.io )
- 4 Tier Storage System: Node, Scratch, Turbo, and DataDen
- Short queues, limited wall times

---
## Artemis by the Numbers

| Node     |  #  | CPU      | GPU         | RAM    | Disk      |    $    |
| -------- | :-: | -------- | ----------- | ------ | --------- | :-----: |
| H100     |  3  | AMD 9654 | 4x H100 SXM | 768 GB | 1.9TB     | 117,950 |
| A100     |  2  | AMD 7513 | 4x A100 SXM | 512 GB | 1.6TB (?) | 58,597  |
| Largemem |  3  | AMD 9654 |             | 768 GB | 1.9TB     | 13,989  |
| CPU      | 25  | AMD 9654 |             | 368 GB | 1.9TB     | 12,998  |

| CPU                                                                    | Cores | Threads | Base   | Boost              | L3 Cache |
| ---------------------------------------------------------------------- | ----- | ------- | ------ | ------------------ | -------- |
| [AMD Epyc 9654 CPU](https://www.amd.com/en/products/cpu/amd-epyc-9654) | 96    | 192     | 2.6GHz | 3.55GHz (All Core) | 384MB    |
| [AMD Epyc 7513 CPU](https://www.amd.com/en/products/cpu/amd-epyc-7513) | 32    | 64      | 2.6GHz | 3.65GHz (Max)      | 128MB    |

> Nodes are partitioned by *threads*, not *cores*. Picking 1 or a multiple of 2 is advisable; see [sbatch's](https://slurm.schedmd.com/sbatch.html) `--distribution` flag

| GPU                                                        | VRAM | GPU Mem Bandwidth | FP64 | FP64 TC | FP32 - TC | BF16 TC |
| ---------------------------------------------------------- | :--: | :---------------: | :--: | :-----: | :-------: | :-----: |
| [A100 SXM](https://www.nvidia.com/en-us/data-center/a100/) | 80GB |     2,039GB/s     | 9.7  |  19.5   |    156    |   312   |
| [H100 SXM](https://www.nvidia.com/en-us/data-center/h100/) | 80GB |     3.34TB/s      |  34  |   67    |    989    |  1989   |
> FLOPs are listed in teraFLOPs ($10^{12}$ floating point operations per second). [[TC)](https://www.nvidia.com/en-us/data-center/tensor-cores/|Tensor Cores (TC)]] are specialized for general matrix multiplications (GEMM)

---
# Partitions
| Partition        |   Nodes   | Max Wall Time | Priority | Max Jobs | Max Nodes |
| ---------------- | :-------: | :-----------: | :------: | :------: | :-------: |
| venkvis-cpu      |    CPU    |     48hrs     |          |          |           |
| venkvis-largemem | Large Mem |     48hrs     |          |          |           |
| venkvis-a100     |   A100    |     8hrs      |          |          |           |
| venkvis-h100     |   H100    |     8hrs      |          |          |           |
| debug            |    all    |  30 minutes   |   100    |    1     |     4     |
- Usage is proportionate to the cost of the nodes you use
- Usage (currently) does not reset; thus, your fair share priority *will not recover*
- The *Max Jobs* limit is enforced using [[https://slurm.schedmd.com/resource_limits.html#qos_maxsubmitjobspu]]

___
## A Note on Fairshare

Nodes are priced proportionate to their cost and the fraction that you use.
- 1 H100 and 1 - 48 CPUs is charged at 1/4th the H100 cost, or 29,487.5 per hour
- 1 A100 and 1 - 16 CPU is charged at 1/4th the A100 cost, or 14,649.25 per hour
- 1 CPU and 0 - 1.9GB is charged at 1/192 the CPU cost, or 67.7 per hour

Using the most appropriate resource for the job is the *best way* to spend less time in the queue
- Fair Share is based on your usage and the primary driver of priority.
- Efficiently using the nodes will minimize your usage.
- Submitting to the wrong partition *will kill your fair share*
- Fairshare, does not (currently) reset, but long term will have a half-life of ~2 weeks

---
## Debugging
The Debug partition is explicitly designed to get you on a node *fast* for *debugging* or development. It's priced at the average cost per CPU/GPU/Memory unit

- Use `--gres` to target particular GPUs (i.e. `--gres=gpu:h100:1` to get 1 H100 GPU or any gpu `--gres=gpu:1`)
    - Being flexible let's slurm schedule you sooner
    - 
- You can end up on any node that meets your requirements
    - CPU-only jobs will get routed to GPU nodes if all the CPU nodes are taken
    - You still only pay the debug rate and you only get what you asked for
- You can only have one debug job *running or in the queue* at a time

---
## Priority on Lighthouse

Priority is how jobs are sorted in the queue, jobs with a higher priority run first

```
Job_priority =
	(10000) * min(Time In Queue / 28 Days, 1) + 
	(10000) * (fair-share_factor) +
	(1000000) * (0 or 100 if venkvis-debug) +
	... # Other stuff (Assoc Factor)
```

The `fair-share_factor` is (roughly) $U_{total} / (N U_{you})$, where $N$ is the size of the group, $U_{total}$ is the total usage of the group and $U_{you}$ is your usage. 
- You can get a report with `sshare -lU` with `LevelFS` being your `fair-share_factor`
- It's greater than 1 for under-served users
- Between 0 and 1 for over-served users

[Slurm's Multifactor Priority Plugin](https://slurm.schedmd.com/priority_multifactor.html)
[The Fair Tree Fairshare Algorithm](https://slurm.schedmd.com/fair_tree.html)

---
## Why not a longer Max Wall Time?
> tl;dr: To keep the queue short. Use checkpointing for longer runs. [[#Checkpointing and Requeing Jobs|Details below]]

Let's assume a [M/M/1 queue](https://en.wikipedia.org/wiki/M/M/1_queue)
- Jobs arrive every $\lambda$ time units (Poisson Process)
- Run times take on average $1/\mu$ time units and are exponentially distributed
- First-come, first-served queue (so no priority, fair share, or partitions)

On average, the time from submission to job completion is:
$$
\frac{1}{\mu - \lambda}
$$
The utilization is $\rho = \lambda/\mu$, if $\rho > 1$ the queue will grow unbounded. Otherwise, it's expected length is:
$$ \frac{\rho}{1-\rho}$$
With a variance of:
$$\frac{\rho}{(1-\rho)^2}$$

---

## Expected Queue Lengths
<iframe src="https://www.desmos.com/calculator/wtsunq4a2w?embed" width="500" height="250" style="border: 1px solid #ccc" frameborder=0></iframe>

Plot of the queue length (red), it's variance (blue) and $+3\sigma$ band (green)
- The expected queue length (red) rapidly increases as $\rho \to 1$
- The *variance* in queue length (blue) increases even faster

The expected wait for a one-off job is ~1/2 the max wall time divided by the number of nodes
- ~0:40 for a H100 gpu
- 1 hr for an A100 gpu
- ~2hrs for an *entire* CPU node

--- 
# Checkpointing and Requeing Jobs 
Have a really long job that you want to run? Here's how you do it:
1. Submit the job to the queue
2. Run for almost  the full max wall time
3. Send a kill signal to your code using [`timeout`](https://manpages.org/timeout)
4. Your code saves a checkpoint
5. Requeue the job with [scontrol](https://slurm.schedmd.com/scontrol.html)
6. Repeat 2-5 until your job finishes 

```bash
#!/bin/bash
#SBATCH --open-mode=append # Needed to append, instead of overwriting the log
#SBATCH --time 2-0:0:) # Run for 2 days (or --time 8:0:0 for gpus)
#... other slurm flags

# Possible checkpoint file name to look for
# ie. `awesome_script --resume "$CHECKPOINT" ...other args...`
CHECKPOINT="${SLURM_JOB_ID}.ckpt"

# Run your code, timing out 30 minutes before closing
timeout 47.5h awesome_script ...
time awesome_script

# Requeue if your code timed out
if [[ $? == 124 ]]; then 
    scontrol requeue $SLURM_JOB_ID
fi
```
---
## Catching Signals

Instead of using [`timeout`](https://manpages.org/timeout)  slurm can send your script a signal ([blog](https://www.makeuseof.com/linux-signal-generation-handling/), [man](https://manpages.org/signal/7)) just before the job ends. Here, I'm sending [USR1](https://manpages.org/signal/7) (User Signal 1) 300 seconds before the wall time, see the [SLURM documentation](https://slurm.schedmd.com/sbatch.html) for more.

```bash
#!/bin/bash
#SBATCH --signal=USR1@300
#... other slurm flags

awesome_script
if [[ $? == 124 ]]; then
    scontrol requeue $SLURM_JOB_ID
fi
```

> Typically, a non-zero exit code in Linux means "something went wrong".  Because we don't want to requeue a job that failed indefinetly, we need to be able to distighish between "Something went wrong" and "I need more time".
> 
> Here we're checking if the exit code is 124 (`timeout` uses 124 to indicate the command timed out), but any non-zero exit code could work. Check your code's docs to see what's normal, what's an error, and how to throw a different signal


---
## Tool Specific Support

| Program                                                                                                                                                                                   | Restart | Signal Catching/Handling |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :-----: | :----------------------: |
| [[https://lightning.ai/docs/pytorch/stable/api/lightning.pytorch.plugins.environments.SLURMEnvironment.html#lightning.pytorch.plugins.environments.SLURMEnvironment.params.auto_requeue]] |   yes   |           yes            |
| [Quantum Espresso](https://www.quantum-espresso.org/Doc/pw_user_guide/node20.html)                                                                                                        |   yes   |       experimental       |
| [LAMMPs](https://docs.lammps.org/restart.html)                                                                                                                                            |   yes   |      no, see Python      |
| [GPAW](https://wiki.fysik.dtu.dk/gpaw/documentation/restart_files.html)                                                                                                                   |   yes   |      no, see Python      |
| [Python](https://docs.python.org/3/library/signal.html)                                                                                                                                   |    ?    |           Yes            |
| [Julia](https://github.com/JuliaLang/julia/issues/14675)                                                                                                                                  |    ?    |            No            |

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

> To get spack: `module load spack` to use the ARC provided instance or [[https://spack.readthedocs.io/en/latest/getting_started.html#installation]]

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

___
# Storage

| Name                                                           | Path                                                                                                    | Base Size | Fair Share | $/TB/month | Notes                                                                         |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- | --------- | ---------- | ---------- | ----------------------------------------------------------------------------- |
| Node Local                                                     | `/tmp`                                                                                                  | 1.9TB     |            |            | Fast, On-Node NVMe storage                                                    |
| [Turbo](https://arc.umich.edu/turbo/), replicated              | `/nfs/turbo/coe-venkvis/`                                                                               | 10TB      | 500GB      | 13.02      | Fast, Automated regular backups                                               |
| [scratch](https://arc.umich.edu/document/lighthouse-policies/) | `/scratch/venkvis_root/venkvis/`                                                                        | 10TB      | 500GB      |            | Fast, Auto-purged 60 days after last use                                      |
| [DataDen](https://arc.umich.edu/data-den/)                     | Access via [Globus](https://app.globus.org/file-manager?origin_id=ab65757f-00f5-4e5b-aa21-133187732a01) | 100TB     | 5TB        | 1.67       | Tape Storage; Files should be between 10 - 200 GB, accessible only via Globus |
| [/home](https://arc.umich.edu/document/greatlakes-policies/)   | `/home/<user>`                                                                                          | 80GB      |            |            | Fast, mounted on Turbo                                                        |
- Node Local: Scratch files, temporary checkpoints
- Turbo/ Home: Software, Environments, Code
- scratch: Large Datasets *actively* being used, multi-node checkpoints
- DataDen: Large Datasets not *actively* being used

> Please manage your storage responsibly and clean up after yourself
> In particular, node-local storage **is not automatically cleaned up**, it's on you to clean up

## Cleanup with trap

Use [trap](https://manpages.org/trap) to create a job specific TMPDIR and clean it up on exit

```bash
#!/bin/bash
# SBATCH directives

# Create job-specific tmp directory
export TMPDIR=$(mktemp --directory --tmpdir)

# Ensure Cleanup on exit
trap 'rm -rf -- "$TMPDIR"' EXIT

# ... do job stuff

```

---
# Miscellaneous

## Getting Help
- The [Artemis slack channel](https://eeg-group.slack.com/archives/C070HCDCY9F)
- [UM CoderSpaces Slack](https://umich.enterprise.slack.com/archives/C02T1M5QNH3) ([[https://documentation.its.umich.edu/node/352#JoinResign]])
-  [UM Lighthouse User Guide](https://arc.umich.edu/lighthouse/user-guide/)
- [UM Great Lakes User Guide](https://arc.umich.edu/greatlakes/user-guide/)
- [UM Cheat Sheet](https://arc.umich.edu/wp-content/uploads/sites/4/2020/05/Great-Lakes-Cheat-Sheet.pdf)
## Tmux
Lighthouse and GreatLakes use multiple login nodes for load balancing/redundancy. To persist a session across login nodes, change where tmux creates its sockets:
```bash
# Goes in your ~/.bashrc
export TMUX_TMPDIR=~/.local/state/tmux
mkdir -p -m 700 $TMUX_TMPDIR
```

> This won't apply until you've killed tmux on *all login nodes* with `tmux kill-server` and loaded your shell (Log out and back in again)
## Setting up Turbo / Scratch
```bash
# Create a folder on turbo
mkdir "/nfs/turbo/coe-venkvis/$(whoami)"

# Symlinks to make things nice
ln -s "/nfs/turbo/coe-venkvis/$(whoami)" ~/turbo
ln -s "/scratch/venkvis_root/venkvis/$(whoami)" ~/scratch
```

## Globus
If you're moving data between clusters, use [Globus](https://www.globus.org):
- It's way faster than scp/rclone/rsync
- On Arjuna use [Globus Connect Personal](https://www.globus.org/globus-connect-personal)

[[https://arc.umich.edu/globus/#document-4]] (don't go using some rando endpoint)
- [DataDen](https://app.globus.org/file-manager?origin_id=ab65757f-00f5-4e5b-aa21-133187732a01)
- [Turbo](https://app.globus.org/file-manager?origin_id=8c185a84-5c61-4bbc-b12b-11430e20010f&origin_path=%2F)
- [/home on Lighthouse](https://app.globus.org/file-manager?origin_id=3242c149-a2b9-4dba-9406-ae3717981621)
- [/home on Great Lakes](https://app.globus.org/file-manager?origin_id=454f457e-a41b-4807-8775-d132f15a228f)

## Brock's Guide to Data Archiving
[Brock Palen](https://its.umich.edu/arc/people/brock-palen) created a great [guide to data archiving](https://docs.google.com/document/d/1xkVPjkqge4BCgNMfNKJHCbBj2B-6t1_r0jJSi3nphwE/edit?usp=sharing), highly recommend for anyone with lots of data that needs to be saved somewhere:
- Tools for finding files
- Various compression tools (gzip, bzip2, xz, lz4) and their trade offs
- Tar: What it is and why you must use it on DataDen
- [Archivetar](https://github.com/brockpalen/archivetar/) - An all-in-one tool for archiving data and uploading it to DataDen
    - Parallel compression/tar tools built-in
    - Can module load it on GreatLakes or Artemis `module load archivetar`

## OnDemand
Provided by ARC [OnDemand](https://arc.umich.edu/open-ondemand/) let's you interact with Artemis (Or Great Lakes) via a web browser to:
- Launch a Jupyter Notebook (Even one with a GPU)
- Launch a Virtual Remote Desktop (You can even connect to it with a native app)

To start visit: [lighthouse.arc-ts.umich.edu](http://lighthouse.arc-ts.umich.edu/)

> You're still billed for the resources you use, it's just a different way of accessing them

# Performance Tools
## [NVTOP](https://github.com/Syllo/nvtop)
Like `top` but for the GPU. Great little tool for getting nice usage traces from your GPU jobs. It doesn't do much but for questions like:
- How much memory am I using?
- Is memory usage churning?
- Could I fit more of these into a single job?

![](https://github.com/Syllo/nvtop/blob/master/screenshot/NVTOP_ex1.png?raw=true)

You can install this with spack or just:
```shell
# Download the AppImage from GitHub (update release as needed)
wget https://github.com/Syllo/nvtop/releases/download/3.1.0/nvtop-x86_64.AppImage
# Mark it as executable 
chmod u+x nvtop-x86_64.AppImage

# Profit (ssh to your job's node first)
./nvtop-x86_64.AppImage
```
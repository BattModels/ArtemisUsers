---
layout: default
title: Cluster Architecture
parent: About Arjuna
nav_order: 1
---

# Cluster Architecture

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
## Partitions
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
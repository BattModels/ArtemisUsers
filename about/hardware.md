---
layout: default
title: Cluster Architecture
parent: About Artemis
nav_order: 1
math: katex
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

> FLOPs are listed in teraFLOPs ($10^{12}$ floating point operations per second). Tensor Cores ([TC](https://www.nvidia.com/en-us/data-center/tensor-cores/)) are specialized for general matrix multiplications (GEMM).

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
- The *Max Jobs* limit is enforced using [MaxSubmitJobsPerUser](https://slurm.schedmd.com/resource_limits.html#qos_maxsubmitjobspu)

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
> tl;dr: To keep the queue short. Use checkpointing for longer runs. [Details below](checkpoint.md)

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

# Storage

| Name                                                           | Path                                                                                                    | Base Size | Fair Share | $/TB/month | Notes                                                                         |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- | --------- | ---------- | ---------- | ----------------------------------------------------------------------------- |
| Node Local                                                     | `/tmp`                                                                                                  | 1.9TB     |            |            | Fast, On-Node NVMe storage                                                    |
| [Turbo](https://arc.umich.edu/turbo/), replicated              | `/nfs/turbo/coe-venkvis/`                                                                               | 10TB      | 500GB      | 13.02      | Fast, Automated regular backups                                               |
| [scratch](https://arc.umich.edu/document/lighthouse-policies/) | `/scratch/venkvis_root/venkvis/`                                                                        | 10TB      | 500GB      |            | Fast, Auto-purged 60 days after last use                                      |
| [DataDen](https://arc.umich.edu/data-den/)                     | Access via [Globus](https://app.globus.org/file-manager?origin_id=ab65757f-00f5-4e5b-aa21-133187732a01) | 100TB     | 5TB        | 1.67       | Tape Storage; Files should be between 10 - 200 GB, accessible only via Globus |
| [/home](https://arc.umich.edu/document/greatlakes-policies/)   | `/home/<user>`                                                                                          | 80GB      |            |            | Fast, mounted on Turbo |

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


---
title: What is Slurm?
layout: default
parent: Getting Started
nav_order: 1
---

Slurm or, the "Simple Linux Utility for Resource Management," is a software package for
submitting computing workloads to a shared computer cluster. Additionally,
Slurm provides tools for managing job priority so that computational resources are split fairly among users.

![SLURM Architecture](https://slurm.schedmd.com/arch.gif)

{: .text-center .lh-0 .fw-300}
Source: [Slurm's Quick Start User Guide](https://slurm.schedmd.com/quickstart.html)

Slurm consists of the following main components:

<dl>
  <dt>Client Commands</dt>
  <dd>CLI tools for interacting with the Slurm. Such as <code>sbatch</code>, <code>srun</code>, <code>squeue</code>, etc.</dd>
  <dt>Compute Node Daemons</dt>
  <dd>On each node <code>slurmd</code> is used to monitor jobs and allocate nodes between multiple jobs</dd>
  <dt>Slurm Controller</dt>
  <dd>The <code>slurmctld</code> controller orchestrates moving jobs from the queues to the compute nodes </dd>
  <dt>Databases</dt>
  <dd>Various databases are maintained to track the current queue and historical usage</dd>
</dl>

## Allocating Resources

Slurm manages resources (cpus, memory, gpus, etc.) using a tiered system:

<dl>
    <dt>Nodes</dt>
    <dd>A single physical computer with CPUs, memory and possibly GPUs</dd>
    <dt>Partition</dt>
    <dd>Group nodes into logical (But possibly overlapping) groups. For example, the <code>gpu</code> partition, contains nodes with a gpu</dd>
    <dt>Jobs</dt>
    <dd>An allocation of resources for a period of time. For example, 6 CPUs and 1 Gb of memory for 1 hour</dd>
    <dt>Job Step</dt>
    <dd>Within a Job, Job Steps requests resources for a particular task</dd>
</dl>

While Slurm manages this from the top-down, most users will interact from the bottom up. For example, a typical compute workload might have several *job steps* such as:

1. Pre-processing data from raw files into a convenient format
2. Analyzing the data and generating results
3. Post-processing the results into graphs or generating summary statistics

Collectively, these tasks form a single *Job* that requires a certain amount of computing resources to complete. For example, a job might request 8 CPUs, 10 Gb of memory, and 1 GPU for 3 hours.

> We could also split these tasks across several jobs, or use a [Heterogeneous Job](https://slurm.schedmd.com/heterogeneous_jobs.html), to tailor our requests for each job step.

The user submits this *job* to a particular *partition* to satisfy the job's computing needs. This job requires a GPU, and it needs a partition with GPUs (i.e., `venkvis-h100`). Other jobs may need lots of memory or a longer run time and should be submitted accordingly. For more information about Artemis's partitions see [Cluster Architecture]({% link about/hardware.md %}).

Once submitted to a *partition*, the *Job* waits in the Slurm queue until Slurm releases it to run on one or more *nodes*. Slurm releases jobs based on their priority computed from the user's [fair share](https://slurm.schedmd.com/fair_tree.html).  Users using less than their "fair share" have a higher priority than users using more than their "fair share."

Carefully crafting the resources requested by a job ensures that it is submitted as soon as possible and does not get delayed waiting for resources that it does not need.

## Accounting

Usage on Slurm is billed based on [Trackable RESources (TRES)](https://slurm.schedmd.com/tres.html). Currently, using 1 CPU for 1 hour consumes 1 TRES-hour. A job that used 32 CPUs for 2 hours is billed for 64 TRES-hours (32 * 2 = 64).

Memory is billed based on the number of CPUs on the node divided by the total memory. Jobs may use up to this amount at no additional cost but are billed additional TRES above this limit. For example, on the `cpu` partition, each CPU gets 0.435GiB of memory. A Job that requests 2 CPUs and 512MiB of memory is billed 2 TRES per hour (max(2, 0.5 * 0.435) = 2).

> Slurm uses a system of units based on a power of 2, not 10. Thus 1G of memory is 1,073,741,824 bytes, not 1,000,000,000 bytes

This usage is billed to the user's default account. To see your accounts run `sacctmgr show user $(whoami) WithAssoc`

```
sacctmgr show user $(whoami) WithAssoc WOPLimits
      User   Def Acct     Admin    Cluster    Account  Partition     Share  Priority MaxJobs MaxNodes  MaxCPUs MaxSubmit     MaxWall  MaxCPUMins                  QOS   Def QOS
---------- ---------- --------- ---------- ---------- ---------- --------- ------- -------- -------- --------- ----------- ----------- -------------------- ---------
      jdoe    venkvis0    None   greatlakes    engin1                    1                                      5000
      jdoe    venkvis0    None   greatlakes  venkvis0                    1                                      5000
      jdoe     venkvis    None    ighthouse   venkvis                    1                                      5000
```

Here we can see that `jdoe` has `venkvis0` and `venkvis` as their default account. See [sbatch](https://slurm.schedmd.com/sbatch.html) for more information.

## Additional Resources

See the [High-Performance Computing]({% link getting_started/linux.md %}#high-performance-computing) section for more information.

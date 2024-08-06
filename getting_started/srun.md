---
title: Submitting Interactive Jobs
layout: default
parent: Getting Started
nav_order: 7
---

# Submitting Interactive Jobs with `srun`
`srun` is a command-line tool used for submitting jobs to the [Slurm workload manager]({% link getting_started/slurm_intro.md %}), offering a wide range of options for customizing job submission, and essentially letting you run parallel jobs on a cluster managed by Slurm.


### `srun` examples

The basic syntax for using `srun` is as follows:

```
srun [options] command
```

where `command` is the command or script to be executed.

To start an interactive job requesting one node, 2GB of memory, for 30 minutes, on the `venkvis-debug` partition, type:

```
srun --partition=venkvis-debug --nodes=1 --mem=2G --time=30 --pty bash
```

> The option `--pty` here is required to start an interactive job.

If you want, you can get a notification by email when the interactive environment is ready.
If your email address is <uniquename>@umich.edu, to start an interactive job requesting one node, 1 task, 1GB of memory, for 2 minutes, on the `venkvis-cpu` partition, type:

```
srun --partition=venkvis-cpu --ntasks=1 --nodes=1 --mem=1G --time=2 --mail-type=BEGIN --mail-user=uniquename@umich.edu --pty bash
```

You can also submit a job using `srun`.
To run a Python script named `my_script.py`, using 1 node, 1GB memory, 1 minute, redirecting stdout to 'out', redirect stderr to 'error', on the `debug` partition, type:

```
srun --partition=venkvis-debug --node=1 --mem=1G --time=1 --output=out --error=error python my_script.py
```

> `--out` and `--error` are incompatible with `--pty`.


### Options

`srun` provides a wide range of options for customizing job submission.
Some of the most commonly used are shown above as examples.
You can get further instruction on these options by typing the following:

```
srun -h
```

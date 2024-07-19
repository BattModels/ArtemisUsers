---
layout: default
title: Checkpointing and Requeing Jobs
parent: About Artemis
nav_order: 4
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
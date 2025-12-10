---
layout: default
title: Miscellaneous
parent: About Artemis
nav_order: 5
---

# Miscellaneous

## Getting Help
- The [Artemis slack channel](https://eeg-group.slack.com/archives/C070HCDCY9F)
- [UM CoderSpaces Slack](https://umich.enterprise.slack.com/archives/C02T1M5QNH3) ([join](https://documentation.its.umich.edu/node/352#JoinResign))
-  [UM Lighthouse User Guide](https://documentation.its.umich.edu/arc-hpc/lighthouse/user-guide)
- [UM Great Lakes User Guide](https://documentation.its.umich.edu/arc-hpc/greatlakes/user-guide)
- [UM Great Lakes Cheat Sheet](https://docs.google.com/document/d/1wsr3yzkkojUMBCCneCz-l413xBzU-SZFAqcFrAAjttk/edit?usp=sharing)

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

[UM ARC Endpoints](https://coerc.engin.umich.edu/globus/) (don't go using some random endpoint)
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

To start visit: [lighthouse.arc-ts.umich.edu](https://lighthouse.arc-ts.umich.edu/)

> You're still billed for the resources you use, it's just a different way of accessing them

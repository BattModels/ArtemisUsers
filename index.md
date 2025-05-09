---
title: Home
layout: default
nav_order: 1
---

# Welcome!

Artemis is the EEG's group CPU and GPU computing cluster within [UM ARC's Lighthouse](https://arc.umich.edu/lighthouse/) cluster.
Here you can find documentation about what Artemis is and how to use it.

## User Accounts

Getting access to Artemis is a multi-step process:

1. Ask the cluster admin to add you to the group via the [Resource Management Portal](https://portal.arc.umich.edu/projects)
   and send them (via Slack) your UMich uniqname.
3. Request an [HPC User Login]

### HPC User Login Request

> Do not request an HPC User login until **after** the admin has added you to the group. Your request for access to Artemis will
> be rejected.

- College or Department: Engineering
- Please describe … : Access to the Viswanathan’s cluster for ….
- The ARC Services I need: “Great Lakes” and “Lighthouse”

### Bulk Adding Users

The [RMP](https://portal.arc.umich.edu/projects) is painfully slow, especially when adding multiple students. Instead,
the admin should email [ARC-support](mailto:arc-support@umich.edu) with the following:

- User uniqnames to add
- Which resource to add them too (Lighthouse, DataDen, Turbo and Great Lakes)
- Which account to add them too (typically, root_account: `venkvis_root` subaccount; `coe-venkvis`)
- Which storage accounts to add them to (`coe-venkvis)

Users will still need to complete the [HPC User Login] form to acknowledge ARC's terms of service.

[HPC User Login]: https://its.umich.edu/advanced-research-computing/high-performance-computing/login

## Support

If you encounter an issue using Artemis, please refer to the following resources:

### Where to ask questions

- [UM CoderSpaces Slack](https://umich.enterprise.slack.com/archives/C02T1M5QNH3) ([join](https://documentation.its.umich.edu/node/352#JoinResign))
- EEG's [#artemis-cluster](https://umich.enterprise.slack.com/archives/C070HCDCY9F) Slack Channel

### Documentation

- [UM Lighthouse User Guide](https://arc.umich.edu/lighthouse/user-guide/)
- [UM Lighthouse Support Page](https://its.umich.edu/advanced-research-computing/high-performance-computing/lighthouse/support)
- [UM Great Lakes User Guide](https://arc.umich.edu/greatlakes/user-guide/)
- [UM Cheat Sheet](https://arc.umich.edu/wp-content/uploads/sites/4/2020/05/Great-Lakes-Cheat-Sheet.pdf)
- [Slurm](https://slurm.schedmd.com/documentation.html)
- [Spack](https://spack.readthedocs.io/en/latest/) ([UM](https://arc.umich.edu/spack/))

### Tutorials

- [Spack](https://spack-tutorial.readthedocs.io/en/latest/)
- [HPC 101](https://www.dropbox.com/scl/fo/8b54mv1hcl3tovft1tz54/h?rlkey=lfo2mgcg9fi563p0fnpkesmrj&dl=0)

## Acknowledgement in Publications

All publications resulting from the usage of Artemis should include an acknowledgement.
We recommend the following text:

> This research was supported in part through computational resources and services provided by Advanced Research Computing at the University of Michigan, Ann Arbor.

## Contributing

Thank you for wanting to contribute to Artemis's Documentation! We recommend you
first [open an issue][issue], but welcome [pull requests] to improve our documentation.

[pull requests]: https://github.com/BattModels/ArtemisUsers/pulls
[issue]: https://github.com/BattModels/ArtemisUsers/issues/

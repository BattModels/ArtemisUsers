---
layout: default
title: Performance Tools
parent: About Artemis
nav_order: 6
---

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

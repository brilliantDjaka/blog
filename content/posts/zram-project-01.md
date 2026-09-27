---
title: "ZRAM: Compressing RAM Into Its Limit For Daily Use"
subtitle: "This is the very first post on this blog."
date: 2026-09-27T10:23:56.166Z
lastmod: 2026-09-27T10:23:56.166Z
draft: false
authors: []
description: ""

tags: ["zram", "weekend", "linux"]
categories: ["linux"]
series: []

featuredImage: "/blog/images/zram/zram-featured.jpg"
featuredImagePreview: "/blog/images/zram/zram-featured.jpg"

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---
## Background
My laptop has very regular amount of ram (16G), but sometimes my workload force me into pushing my laptop into it's limit. Opening so many chrome tabs, multiple agents, code editors, and running several project at once. Resulting my laptop's ram goes 99%, it will become so laggy and leads to completely freezes. <!--more--> I cannot even reboot it. I was so frustrated at that time.
So I've come out with an idea to use ZRAM to compress my ram. So my ram usage will become smaller. Resulting bigger ram usage overall.
Btw, ZRAM is not a new thing. It is feature of linux kernel [https://docs.kernel.org/admin-guide/blockdev/zram.html](https://docs.kernel.org/admin-guide/blockdev/zram.html). So it is already available in pretty much all machine powered by linux.
Heck I was even remembered installing it on my android phone on 2015. But cannot prove its benefit, since I was a high school student at that time haha.

## The Concept

### What we are going to do?
Install zram and configure zram on my laptop.
Try all available compression algoritm.
Prove it with benchmark.
Prove it with manual testing (Open many programs until my ram usage bigger than my physical ram amount)

### Goals
Total used ram should be more than physical ram amount. My laptop has 16GB RAM installed. So later we need to fill it until the usage is more than 16GB

### Environment
My laptop currently runs NixOs with GNOME.
```sh
❯ fastfetch -l none
brian@nixos
-----------
OS: NixOS 26.11 (Zokor) x86_64
Host: Swift SFG14-41 (V1.15)
Kernel: Linux 7.2.7
Uptime: 11 hours, 7 mins
Packages: 5 (appimage), 52 (flatpak), 1228 (nix-system), 510 (nix-user)
Shell: bash 5.3.15
Display (CMN1408): 1920x1080 in 14", 60 Hz [Built-in]
Desktop Environment: GNOME 50.4
Window Manager: Mutter (Wayland)
WM Theme: Adwaita
Theme: Adwaita [GTK2/3/4]
Icons: Adwaita [GTK2/3/4]
Font: Adwaita Sans (11pt) [GTK2/3/4]
Cursor: Adwaita (24px)
Terminal: ghostty 1.3.1
Terminal Font: FiraCode Nerd Font Mono (12pt)
CPU: AMD Ryzen 5 7530U (12) @ 4.55 GHz
GPU: AMD Barcelo [Integrated]
Memory: 3.80 GiB / 14.98 GiB (25%)
Swap: 455.79 MiB / 22.46 GiB (2%)
Disk (/): 259.46 GiB / 475.92 GiB (55%) - btrfs
Local IP (wlp1s0): 192.168.1.38/24
Battery (AP19B5L): 80% [AC Connected]
Locale: en_US.UTF-8
```
NOTE: Please ignore the swap space (22GB), i will explain later

## ZRAM Explanation (Quick Version)
We will use [zram generator](https://github.com/systemd/zram-generator) to configure zram. We will basically change few values:
compression algorithm => I'm currently using `zstd level 1`, but we will try all compression algoritm on the benchmark
zram size => set to magic formula "your ram * 3 / 2"
zram resident limit => set to magic formula "your ram / 1.5"
The way how zram compress ram it will spin up new swap device. So if i set zram size to 24 GB it will creates 1 new swap with 24GB size. The size will not take actual space on disk or ram. It just a maximum limit **how much compressed ram can be stored on zram**
This is my current configuration:
```
NixConf on  main
❯ free -hm
               total        used        free      shared  buff/cache   available
Mem:            14Gi       4.8Gi       5.2Gi        23Mi       5.1Gi        10Gi
Swap:           22Gi       428Mi        22Gi
```
I'm using 22GB swap space, it means i can take maximum 22GB of compressed ram.
Then what is the **zram resident limit**? It is a physical guardian of zram. So it states how much physical ram zram can use. So think it like safety net for your physical ram.

### NOTE
**But**, please don't think this seriously yet. Just use magic number. Because at the end the most important thing is the **compression ratio** we can achieve. Which determined by compression algoritm

## How to benchmark

### Syntetic Benchmark
With help of AI Agent, I've created benchmark script. You can view it on my [github](https://github.com/brilliantDjaka/zram-workbench).
This is how the script works in very simple list
Configure the zram
Fill up ram
Measure it
check ram lags and speed
try open firefox and check the startup times
You can read [flowchart.md](https://github.com/brilliantDjaka/zram-workbench/blob/main/flowchart.md) for more detailed flow

### Manual Test
I will try to open so many programs until the total of my ram usage surpass 16GB. I will check if my laptop still snappy and measure is there a noticeable lags occur

## The Result (Sythetic Benchmark)
The run is archived at `results/2026-09-26_1906_run` (11 algorithms × 2 repeats = 22 rows).

### What was actually run
```
hog:      stress-ng --vm 1 --vm-bytes 13G --vm-keep --timeout 35s   (settle 8s)
device:   /dev/zram0, 22.46 GiB, swapon -p 100
matrix:   lzo-rle, lzo, lz4, lz4hc, zstd:1, zstd:3, zstd:8, zstd:15, zstd:19, deflate, 842
box:      Ryzen 5 7530U (12 threads), 14.98 GiB RAM, NixOS 26.11, kernel 7.2.7
```

### Goal check: did we actually go past physical ram?
Yes. `stress-ng` logged `using 13GB per stressor instance (total 13GB of 12.00GB available memory)`.
13G of hog plus the GNOME session on a 14.98 GiB box is more memory than exists.

### The Comparison table
| Algorithm | Data held in zram | Real RAM it cost | Compression gained | Firefox startup time | Worst single hitch | CPU spent compressing |
|---|---|---|---|---|---|---|
| lz4 | 1.14 GiB | 402 MiB | 2.98x | 1.33 s | 0.03 ms | 14.4% |
| lzo-rle | 0.95 GiB | 323 MiB | 3.13x | 1.33 s | 0.03 ms | 23.1% |
| zstd:1 | 0.86 GiB | 213 MiB | 4.39x | 1.36 s | 0.52 ms | 22.0% |
| lzo | 0.87 GiB | 284 MiB | 3.25x | 1.29 s | 0.19 ms | 23.1% |
| zstd:3 | 0.98 GiB | 239 MiB | 4.35x | 1.68 s | 0.37 ms | 25.1% |
| 842 | 0.76 GiB | 290 MiB | 2.80x | 1.40 s | 2.01 ms | 25.4% |
| lz4hc | 0.85 GiB | 268 MiB | 3.41x | 1.95 s | 0.04 ms | 26.3% |
| deflate | 0.88 GiB | 212 MiB | 4.41x | 1.41 s | 0.03 ms | 25.1% |
| zstd:8 | 0.72 GiB | 176 MiB | 4.50x | 1.58 s | 1.14 ms | 26.6% |
| zstd:19 | 0.53 GiB | 106 MiB | 5.33x | 0.96 s | 12.21 ms | 30.8% |
| zstd:15 | 0.69 GiB | 151 MiB | 4.91x | 1.70 s | 8.66 ms | 37.5% |

**Worst single hitch** — the longest single pause of a small competing workload (200 samples) during the ~27s window. Anything around 0.03 ms is ordinary scheduler noise; the two entries in the 8–12 ms range are exactly the two algorithms that pay the most CPU.

## The Result (Manual Test)
Besides synthetic benchmark, I also try to prove zram with manual testing. I tried open 70 firefox tabs that opens youtube to fill ram as much as i can.
Here's the screenshot
![Screenshot From 2026-09-27 12-53-36.png](/blog/images/zram/Screenshot_From_2026-09-27_12-53-36_1790490127006_0.png)

### The Result
![Screenshot From 2026-09-27 12-50-39.png](/blog/images/zram/Screenshot_From_2026-09-27_12-50-39_1790490159297_0.png)
![Screenshot From 2026-09-27 12-54-34.png](/blog/images/zram/Screenshot_From_2026-09-27_12-54-34_1790490165208_0.png)
My total ram usage was 13.1 GB out of 16.1 GB, this may seem like there's still ram available, but if you see on the second screenshot. There's actually 4.5GB ram compressed on the zram. **So total of my RAM usage was 13.1 + 4.5 = 17.6 GB proofing that zram works. 4.5GB ram was compressed into 1.3G swap space.** So we achieved 3.46 compression ratio (worst than the benchmark says 5.33 ratio, but after all it's still good thing ram could be compressed)

## How you can get benefit of this article
If you are using linux right now you can enable zram. Use [zram-generator](https://github.com/systemd/zram-generator) for easy configuration. You can download the package with your distro package manager
[Ubuntu 26.04 LTS](https://packages.ubuntu.com/resolute/systemd-zram-generator)
[Debian Stable (trixie)](https://packages.debian.org/trixie/systemd-zram-generator)
[Arch Linux](https://archlinux.org/packages/extra/x86_64/zram-generator/)
You just need to create a configuration file on `/etc/systemd/zram-generator.conf` with this value
```
[zram0]
compression-algorithm=zstd(level=1)
swap-priority=100
zram-size=ram * 3 / 2
```
After that just reboot your system and zram should be enabled. You can check by running command `zramctl`
![image.png](/blog/images/zram/image_1790503551933_0.png)

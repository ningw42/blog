---
title: "DIY 10GbE NAS"
date: 2022-12-18T16:43:07+08:00
tags: [NAS, Linux, Networking]
---

- [需求](#需求)
- [硬件选择](#硬件选择)
  - [NAS](#nas)
  - [Networking](#networking)
- [Host OS 选择](#host-os-选择)
- [ZFS](#zfs)
  - [安装](#安装)
  - [创建存储池](#创建存储池)
  - [其他配置](#其他配置)
    - [Import on Boot](#import-on-boot)
    - [ARC Size](#arc-size)
    - [Automatic Scrubbing](#automatic-scrubbing)
- [Intel DG2](#intel-dg2)
- [10GbE](#10gbe)
  - [Windows NIC Driver](#windows-nic-driver)
  - [Linux NIC Driver](#linux-nic-driver)
  - [测速和排查](#测速和排查)
  - [Samba](#samba)


## 需求

1. 容量越大越好
2. 1-2 块硬盘的冗余
3. 硬件转码能力，越强越好
4. 至少能跑 Container

## 硬件选择

### NAS

| Component  | Model                      | Remarks                                      | Cost (CNY) |
| :--------- | :------------------------- | :------------------------------------------- | :--------- |
| Chassis    | 星之海 半人马座            | 8 盘位 7 半高 PCIe 槽                        | 1299       |
| CPU        | AMD Ryzen 3900X            | 主力机淘汰                                   | 0          |
| CPU Cooler | Themalright AXP90 X53      | 机箱限高，只能用下压式                       | 159        |
| MB         | MSI X570 Unify             | 主力机淘汰                                   | 0          |
| Memory     | Kingston 32GB DDR4@3200MHz | 主力机淘汰                                   | 0          |
| GPU        | MSI Intel Arc A380 (DG2)   | 半高双槽，最强的编解码能力，没有之一（应该） | 899        |
| HBA Card   | LSI 2308                   | 刷 IT Mode，便宜                             | 75         |
| Data Drive | Seagate EXOS X18 16TB * 8  | 比 HC550 “安静”                              | 1670 * 8   |
| Boot Drive | KIOXIA RC20 1TB            | 便宜可靠                                     | 429        |
| PSU        | Themaltake SFX 450W        | 便宜                                         | 429        |

### Networking

| Component | Model              | Remarks                                     | Cost (CNY) |
| :-------- | :----------------- | :------------------------------------------ | :--------- |
| NIC       | Intel X520 DA1 * 2 | 最便宜的 10GbE NIC                          | 130 * 2    |
| Switch    | QNAP QSW-M408S     | 至少三个 SFP+ 和至少 8 个 RJ45 中最便宜的？ | 1550       |

## Host OS 选择

| OS (Distribution)      | Pros                      | Cons                        |
| :--------------------- | :------------------------ | :-------------------------- |
| Windows Server         | 最好的驱动支持            | Windows, Proprietary        |
| Proxmox VE             | 听说靠谱，Open Source     | 没用过，内核旧              |
| ESXi                   | 听说靠谱                  | 没用过，Proprietary         |
| UnRAID                 | Linux，简单，社区好像不错 | 性能受限于单盘，Proprietary |
| TrueNAS Core (FreeBSD) | 听说靠谱，OOTB ZFS，WebUI | FreeBSD                     |
| TrueNAS Scale (Debian) | Linux，OOTB ZFS，WebUI    | 内核旧                      |
| Arch Linux             | 常用 Distro，文档全       | 配置成能用相对麻烦          |

最后上面的一个也没有选，用了从没用过的 Distro，[NixOS](https://nixos.org/)。选它的原因有两个：

1. Unstable Channel 可以 Rolling Upgrade，Stable Channel 又可以像普通 Distro 一样冻结版本。
2. NixOS 或者说 Nix 的一个特点是，它是 Declarative 的。只要配置一样，可以产出同样的 OS 或者 Package。因为我不打算给 Boot Drive 做任何 RAID，快速恢复 Host OS 的快速恢复（配置）能力很重要。使用 NixOS 并备份配置即可达成目的，[Ansible](https://www.ansible.com/) + 其他 Distro 应该也是可以的。


另外不能小看 TrueNAS 的 WebUI 的含金量，毕竟 ZFS 的 Management WebUI 少的可怜。除了 TrueNAS 自带的这个之外，我仅仅搜到两个基于 [Cockpit](https://cockpit-project.org/) 的 [cockpit-zfs-manager](https://github.com/45Drives/cockpit-zfs-manager) 和一个 Early Access 还卖 $49/year 起的 [Poolsman](https://www.poolsman.com/)。前者是开源免费的，但是看起来功能比较简陋；后者看起来不错，接近 TrueNAS 的那个，但是有点贵。TrueNAS 整个 OS 可都是免费的。我选择放弃 Management WebUI，用 [Netdata](https://github.com/netdata/netdata) 做一个 Readonly 的 Dashboard 就行了。

## ZFS

### 安装

ZFS 在 Linux 上已经有比较好的支持了，一些面向存储的 Distro 基本都有开箱即用的 ZFS，比如 TrueNAS Scale 和 Proxmox VE。其他的 Distro 可以参考 [OpenZFS Doc](https://openzfs.github.io/openzfs-docs/) 或者 [ZFS - ArchWiki](https://wiki.archlinux.org/title/ZFS)。

我不需要 Root On ZFS，所以复杂的部分就跳过了 :)。

### 创建存储池

想着有 8 * 16TB 的 Raw Capacity，就多一点冗余，决定用 RAIDZ2 做一个大的 Pool。

ZFS 有别于一般的文件系统，它自己就是一个 Volume Manager。所以创建存储池的时候只需要告诉 ZFS 磁盘就行。磁盘的 ID 一般使用 `/dev/disk/by-id/` 即可。有 WWN 也有型号+序列号，我选择了后者。

`-o ashift=12` 是因为绝大多数现代的 HDD 都是 4KB 的 Sector，具体可以参考 [Advanced Format Disks](https://openzfs.github.io/openzfs-docs/Project%20and%20Community/FAQ.html#advanced-format-disks)

```bash
zpool create -f -o ashift=12 -m /mnt/data data raidz2 \
    ata-ST16000NM000J-{DISK_SERIAL_NUMBER} \
    ata-ST16000NM000J-{DISK_SERIAL_NUMBER} \
    ata-ST16000NM000J-{DISK_SERIAL_NUMBER} \
    ata-ST16000NM000J-{DISK_SERIAL_NUMBER} \
    ata-ST16000NM000J-{DISK_SERIAL_NUMBER} \
    ata-ST16000NM000J-{DISK_SERIAL_NUMBER} \
    ata-ST16000NM000J-{DISK_SERIAL_NUMBER} \
    ata-ST16000NM000J-{DISK_SERIAL_NUMBER}

zpool status -v
  pool: data
 state: ONLINE
  scan: scrub repaired 0B in 00:00:01 with 0 errors on Sun Dec 18 10:45:27 2022
config:

        NAME                                        STATE     READ WRITE CKSUM
        data                                        ONLINE       0     0     0
          raidz2-0                                  ONLINE       0     0     0
            ata-ST16000NM000J-{DISK_SERIAL_NUMBER}  ONLINE       0     0     0
            ata-ST16000NM000J-{DISK_SERIAL_NUMBER}  ONLINE       0     0     0
            ata-ST16000NM000J-{DISK_SERIAL_NUMBER}  ONLINE       0     0     0
            ata-ST16000NM000J-{DISK_SERIAL_NUMBER}  ONLINE       0     0     0
            ata-ST16000NM000J-{DISK_SERIAL_NUMBER}  ONLINE       0     0     0
            ata-ST16000NM000J-{DISK_SERIAL_NUMBER}  ONLINE       0     0     0
            ata-ST16000NM000J-{DISK_SERIAL_NUMBER}  ONLINE       0     0     0
            ata-ST16000NM000J-{DISK_SERIAL_NUMBER}  ONLINE       0     0     0
```

### 其他配置

均以 NixOS 的 `/etc/nixos/configuration.nix` 为例，其他 Distro 需参考各自的文档。

#### Import on Boot

在 NixOS 上，只需要在 `/etc/nixos/configuration.nix` 里面加上一个 Option 就可以在 Boot 的时候 Import 上面创建的存储池。

```nix
boot.zfs.extraPool = [ "YOUR_POOL_NAME" ]
```

#### ARC Size

```nix
boot.kernelParams = [ "zfs.zfs_arc_max=ARC_MAX_SIZE_IN_BYTES" ];
```

or 

```nix
boot.extraModprobeConfig = ''
  options zfs zfs_arc_max=ARC_MAX_SIZE_IN_BYTES
'';
```

#### Automatic Scrubbing

```nix
services.zfs.autoScrub.enable = true;
```

## Intel DG2

Intel DG2 目前需要 Linux 6.0 和 Mesa 22.2 才能支持。这也是没有选择 TrueNAS 和 UnRAID 的原因。

NixOS 22.11 自带的内核是 5.15 LTS，跟换内核只需要在 `/etc/nixos/configuration.nix` 里面设置一下 `boot.kernelPackages` 就可以了，但是之前用到的 ZFS 并不是 DKMS 的，所以这里最高只能安装 Nixpkgs 中的 ZFS 2.1.7 所支持的最新 Kernel。[NixOS ZFS](https://nixos.wiki/wiki/ZFS) 的文档也描述了如何设置，`boot.kernelPackages = config.boot.zfs.package.latestCompatibleLinuxPackages;` 即可。所幸，目前 [ZFS 2.1.7 @ Nixpkgs](https://github.com/NixOS/nixpkgs/blob/master/pkgs/os-specific/linux/zfs/default.nix) 能支持的最新的 Kernel 是 `linuxPackages_6_0`。

仅仅是 Linux 6.0 并不能使 Intel DG2 被正确驱动。`dmesg` 可以看到如下信息，其中 `56a5` 是 Intel Arc A380 的 ID，需要 `force_probe` 才行。

```
[   10.171063] i915 0000:30:00.0: Your graphics device 56a5 is not properly supported by the driver in this kernel version. To force driver probe anyway, use i915.force_probe=56a5 module parameter or CONFIG_DRM_I915_FORCE_PROBE=56a5 configuration option, or (recommended) check for kernel updates.
```

按照提示，设置 `modprobe.conf`，在一般的 Distro 上，应该在 `/etc/modprobe.d/` 下新建一个文件写入 `options i915 force_probe=56a5`。在 NixOS 上，则是在 `/etc/nixos/configurations.nix` 中添加如下设置。

```nix
boot.extraModprobeConfig = ''
  options i915 force_probe=56a5
''
```

重启后，查看 `/dev/dri`，相比之前多了 `renderD128` 即可。

```bash
ls /dev/dri/
by-path  card0  renderD128
```

## 10GbE

原以为 10GbE 也是像 1GbE 一样即插即用的，毕竟现在 DataCenter 都已经 40Gbps+ 了，然而事情并没有那么简单。

### Windows NIC Driver

Intel X520 DA1 在 Windows 上是需要手动安装驱动的，参考[这篇文章](https://malash.me/202207/intel-x520-drivers-on-windows-and-macos/)安装驱动。简单来说就是从官网下载驱动并手动安装 `PROXGB/Winx64/NDIS68/ixn68x64.inf` 即可。

### Linux NIC Driver

Intel X520 DA1 (82599) 在 Linux 上是开箱即用的，驱动是 `ixgbe`，即使它是 PCIe 2.0 * 8，而我的主板只剩 PCIe * 4 可用了。

```bash
lspci | grep 82599
24:00.0 Ethernet controller: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection (rev 01)

dmesg | grep ixgbe
[    9.702824] ixgbe: Intel(R) 10 Gigabit PCI Express Network Driver
[    9.702827] ixgbe: Copyright (c) 1999-2016 Intel Corporation.
[    9.886350] ixgbe 0000:24:00.0: Multiqueue Enabled: Rx Queue count = 24, Tx Queue count = 24 XDP Queue count = 0
[    9.886640] ixgbe 0000:24:00.0: 16.000 Gb/s available PCIe bandwidth, limited by 5.0 GT/s PCIe x4 link at 0000:21:02.0 (capable of 32.000 Gb/s with 5.0 GT/s PCIe x8 link)
[    9.886952] ixgbe 0000:24:00.0: MAC: 2, PHY: 19, SFP+: 5, PBA No: G30771-004
[    9.886955] ixgbe 0000:24:00.0: 24:1c:04:f3:72:8b
[    9.888269] ixgbe 0000:24:00.0: Intel(R) 10 Gigabit Network Connection
[    9.896611] ixgbe 0000:24:00.0 enp36s0: renamed from eth0
[   13.283241] ixgbe 0000:24:00.0: registered PHC device on enp36s0
[   13.453178] ixgbe 0000:24:00.0 enp36s0: detected SFP+: 5
[   13.597726] ixgbe 0000:24:00.0 enp36s0: NIC Link is Up 10 Gbps, Flow Control: RX/TX
[   16.304564] ixgbe 0000:24:00.0 enp36s0: NIC Link is Down
[   17.861740] ixgbe 0000:24:00.0 enp36s0: NIC Link is Up 10 Gbps, Flow Control: RX/TX
```

### 测速和排查

使用 iperf3 测速发现，任何一端作为 Server，**单线程**都无法跑满 10GbE。观察测速时两边的 CPU 占用，发现并没有吃满任何一个 CPU 核心。要跑满 10GbE，至少需要四个线程 `-P4`。搜索发现 V2EX 上的一个帖子 [Windows 10 的 10G 网络性能非常糟糕，有什么办法优化吗？](https://www.v2ex.com/t/844789)，OP 的 Root Cause 是火绒。我并没有除 Windows Defender 之外的任何安全软件，尝试卸载 Windows 网络相关的软件，并再次测速，并没有任何变化。另外尝试设置两边的 TCP Send/Receive Buffer 和 Jumbo Packet(Frame)，有一些提升，但是距离 10Gbps 还很远。

```bash
~ took 4s
❯ iperf3 -c 192.168.2.6
Connecting to host 192.168.2.6, port 5201
[  4] local 192.168.2.11 port 56650 connected to 192.168.2.6 port 5201
[ ID] Interval           Transfer     Bandwidth
[  4]   0.00-1.00   sec   556 MBytes  4.66 Gbits/sec
[  4]   1.00-2.00   sec   562 MBytes  4.71 Gbits/sec
[  4]   2.00-3.00   sec   568 MBytes  4.77 Gbits/sec
[  4]   3.00-4.00   sec   564 MBytes  4.73 Gbits/sec
[  4]   4.00-5.00   sec   574 MBytes  4.81 Gbits/sec
[  4]   5.00-6.00   sec   564 MBytes  4.73 Gbits/sec
[  4]   6.00-7.00   sec   565 MBytes  4.74 Gbits/sec
[  4]   7.00-8.00   sec   567 MBytes  4.75 Gbits/sec
[  4]   8.00-9.00   sec   574 MBytes  4.81 Gbits/sec
[  4]   9.00-10.00  sec   568 MBytes  4.77 Gbits/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth
[  4]   0.00-10.00  sec  5.53 GBytes  4.75 Gbits/sec                  sender
[  4]   0.00-10.00  sec  5.53 GBytes  4.75 Gbits/sec                  receiver

iperf Done.

~ took 10s
❯ iperf3 -c 192.168.2.6 -R
Connecting to host 192.168.2.6, port 5201
Reverse mode, remote host 192.168.2.6 is sending
[  4] local 192.168.2.11 port 56652 connected to 192.168.2.6 port 5201
[ ID] Interval           Transfer     Bandwidth
[  4]   0.00-1.00   sec   680 MBytes  5.70 Gbits/sec
[  4]   1.00-2.00   sec   687 MBytes  5.77 Gbits/sec
[  4]   2.00-3.00   sec   685 MBytes  5.74 Gbits/sec
[  4]   3.00-4.00   sec   687 MBytes  5.76 Gbits/sec
[  4]   4.00-5.00   sec   686 MBytes  5.75 Gbits/sec
[  4]   5.00-6.00   sec   688 MBytes  5.77 Gbits/sec
[  4]   6.00-7.00   sec   686 MBytes  5.75 Gbits/sec
[  4]   7.00-8.00   sec   683 MBytes  5.73 Gbits/sec
[  4]   8.00-9.00   sec   684 MBytes  5.73 Gbits/sec
[  4]   9.00-10.00  sec   687 MBytes  5.76 Gbits/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  6.69 GBytes  5.75 Gbits/sec    0             sender
[  4]   0.00-10.00  sec  6.69 GBytes  5.75 Gbits/sec                  receiver

iperf Done.
```

在原本的 Windows 机器上使用 ArchLiveISO 测速，可以跑到约 9.4Gbps。只好暂时认为 Windows，Intel 的驱动或者 iperf3 on Windows 三者中至少有一个有性能问题。

### Samba

最开始 Samba 的性能（Linux Server, Windows Client），符合测速，仅能跑 5Gbps 左右。在开启 Jumbo Packet(Frame) 和修改 TCP Buffer Size 之后，Windows Client 向 Linux Server 写入可以接近 1GBps，但是读取仅有一半。
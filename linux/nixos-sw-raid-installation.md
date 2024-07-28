# Installing NixOS with root on software RAID

- [Installing NixOS with root on software RAID](#installing-nixos-with-root-on-software-raid)
  - [Introduction](#introduction)
  - [Installation Media](#installation-media)
  - [Network](#network)
  - [Partitioning and Formatting](#partitioning-and-formatting)
    - [Disk preparation](#disk-preparation)
    - [Partitioning](#partitioning)
    - [Encryption](#encryption)
    - [Formatting](#formatting)
      - [ext4](#ext4)
        - [LVM](#lvm)
      - [XFS](#xfs)
  - [Installing](#installing)
    - [Configuring `/etc/mdadm.conf`](#configuring-etcmdadmconf)
  - [Common errors \& warnings](#common-errors--warnings)
    - [trace: warning: mdadm: Neither MAILADDR nor PROGRAM has been set. This will cause the `mdmon` service to crash.](#trace-warning-mdadm-neither-mailaddr-nor-program-has-been-set-this-will-cause-the-mdmon-service-to-crash)

## Introduction

This is _not_ a comprehensive step by step guide, but an unofficial extention to the official docs for where they fall short. Throughout this document I often refer you to official docs, while aiming to only cover deviations and tips & tricks.

I go over:

- Setting up RAID1 with `mdadm`
- Setting up these filesystems
  - ext4
  - ext4 and LVM
  - XFS
- Encryption with LUKS2

The [Arch Linux Wiki](https://wiki.archlinux.org) is used in addition throughout this guide. This is perfectly fine as these steps aren't specific to a distro, but general steps with many Linux systems.

URLs prefixed with "📄" link to relevant docs and are not a part of this guide, but serve as a quality of life feature to save you from a Google search to find relevant info.

This is a WIP and will be updated as I learn more.

## Installation Media

Download and boot the [minimal ISO image](https://nixos.org/download/#nixos-iso).

## Network

Setup your network as usual or rely on DHCP if possible.

> 📄 https://nixos.org/manual/nixos/stable/#sec-installation-manual-networking

## Partitioning and Formatting

### Disk preparation

I'm assuming a new device, but if the device is being reused or re-purposed from an existing array, [erase any old RAID configuration information](https://wiki.archlinux.org/title/RAID#Prepare_the_devices). Otherwise move on.

### Partitioning

Partition according to [this section in Arch Wiki](https://wiki.archlinux.org/title/RAID#Partition_the_devices).

For your ease here are the `fdisk` type numbers:

| Part. Type | Type Number |
| ---------- | ----------- |
| Linux EFI  | 1           |
| Linux swap | 19          |
| Linux RAID | 43          |

Then after writing the partition scheme [build the RAID array](https://wiki.archlinux.org/title/RAID#Build_the_array).

An example for a 2-disk RAID1 array (⚠️ don't just blindly copy and paste):

```bash
sudo mdadm --create --verbose --level=1 --metadata=1.2 --raid-devices=2 /dev/md/<name> /dev/sdaX /dev/sdbY # (Replace X and Y with the partition number of your root partition)
```

A common name and path for the RAID device is `/dev/md0`, but `/dev/md/<name>` works as well.

> 💡 Tip: use `cat /proc/mdstat` to view the syncing progress of the RAID array. You don't have to wait for this to finish before using it though.

- 📄 https://nixos.org/manual/nixos/stable/#sec-installation-manual-partitioning
- 📄 https://wiki.archlinux.org/title/RAID

### Encryption

TODO

### Formatting

Then [format](https://nixos.org/manual/nixos/stable/#sec-installation-manual-partitioning-formatting) your boot partitions, making sure only to label the primary boot partition.

Next you format the RAID device with your desired filesystem.

#### ext4

TODO

##### LVM

TODO

#### XFS

```bash
mkfs.xfs -L <label> /dev/md/<name>
```

> 📄 https://nixos.org/manual/nixos/stable/#sec-installation-manual-partitioning-formatting

## Installing

Continue the installation as usual, as per the [NixOS docs](https://nixos.org/manual/nixos/stable/#sec-installation-manual-installing). But make sure to mount the RAID device instead of the raw block device:

```bash
sudo mount -L <label> /mnt # (The label you've set in `mkfs.xfs` above)
```

`nixos-generate-config` will automatically detect the RAID array and add [`boot.swraid.enable = true;`](https://search.nixos.org/options?channel=24.05&show=boot.swraid.enable&from=0&size=50&sort=relevance&type=packages&query=mdadm) to `/mnt/etc/nixos/hardware-configuration.nix`

### Configuring `/etc/mdadm.conf`

Next, before running `nixos-install`, read [3.4 Update Configuration File](https://wiki.archlinux.org/title/RAID#Update_configuration_file) in full to understand the problem and decide whether you want to do this. If so, follow along, otherwise you can skip this and complete installing NixOS now.

Since `/etc/mdadm.conf` is managed by NixOS you can't imperatively update it according to how the Arch Wiki explains it, therefore you must do it declaratively.

Print your current RAID arrays and copy the output:

```bash
sudo mdadm --detail --scan --verbose
```

Edit `/mnt/etc/nixos/hardware-configuration.nix` and add the option [`boot.swraid.mdadmConf`](https://search.nixos.org/options?channel=24.05&show=boot.swraid.mdadmConf&from=0&size=50&sort=relevance&type=packages&query=mdadm) like so, pasting the output inbetween the quotes.

```nix
{
    boot.swraid.mdadmConf = ''
        ARRAY /dev/md0 level=raid1 num-devices=2 metadata=1.2 UUID=acddd0cf:d8e65dab:7c29ff58:fab09463 devices=/dev/vda2,/dev/vdb2
    '';
}
```

Now you can `nixos-install`.

## Common errors & warnings

### trace: warning: mdadm: Neither MAILADDR nor PROGRAM has been set. This will cause the `mdmon` service to crash.

See this issue: https://github.com/nix-community/disko/issues/451#issuecomment-1824049815

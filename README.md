# Disclaimer

This repository contains experimental patches and notes created as part of a research and bring-up effort to run a Mellanox Innova 2 Flex FPGA card on a Raspberry Pi 5 using modern Mellanox OFED (24.10) and the Mellanox innova_2_flex_open_18_12 Open Bundle.

The primary goal of the project was to restore FPGA-related functionality that existed in older Mellanox OFED releases (around OFED 5.2) and make it work on a modern kernel and ARM platform. This goal has been achieved: the Mellanox userspace tools build and run, and the FPGA on the Innova 2 Flex card can be successfully programmed using standard Mellanox utilities.

The work is based on analysis of publicly available source code from Mellanox open repositories, historical OFED versions, and limited reverse-engineering. No proprietary or confidential information is used.

This project is not an official solution and is not affiliated with, endorsed by, or supported by NVIDIA, Mellanox, AMD, or Xilinx.

The code is shared in the spirit of open research and community collaboration. Testing coverage is limited, and behavior may vary across kernel versions, distributions, and hardware platforms.

This repository is intended for research, educational, and experimental use only. Production use is strongly discouraged.

Use at your own risk.

# Overview

Modern Mellanox OFED releases no longer include FPGA-related functionality for Innova 2 Flex cards. This repository restores that support by backporting relevant driver components from OFED 5.2 into a modern OFED 24.10 codebase.

The primary goal of this project was to:
 - Run a Mellanox Innova 2 Flex card on a Raspberry Pi 5
 - Use Mellanox innova_2_flex_open_18_12 Open Bundle
 - Successfully build and run Mellanox userspace utilities
 - Program the FPGA using standard Mellanox tools

This goal has been achieved.

# Motivation

Innova 2 Flex boards (Kintex UltraScale+ XCKU15P) are still available on the secondary market and offer a powerful FPGA platform with PCIe and high-speed transceivers.

However, official FPGA support was removed from newer OFED releases.

This project restores that functionality for experimentation and research on modern kernels and ARM platforms.

# Tested Configuration

 - Board: Mellanox Innova 2 Flex (Morse)
 - Host: Raspberry Pi 5
 - Architecture: ARM64
 - Kernel: 6.8.0-1032-raspi
 - OFED base: MLNX OFED 24.10
 - Open Bundle: innova_2_flex_open_18_12
Other configurations may work but are untested.

# Prerequisites

1. Install stock MLNX OFED 24.10 using the official installer from NVIDIA.

2. Ensure kernel headers are installed:
```
sudo apt install linux-headers-$(uname -r)
```
# Build Instructions

Clone repository:
```
git clone https://github.com/Kcctech-git/Innova2Flex_OFED_Support.git
cd Innova2Flex_OFED_Support
```

Set kernel version:
```
KVER=$(uname -r)
```
Run configure:
```
./configure \
    --with-mlx5_core-mod \
    --with-innova-flex \
    --with-mlxfw-mod \
    --without-debug-info \
    --with-linux=/lib/modules/$KVER/build \
    --kernel-version=$KVER
```
Build:
```
make
```
Build should complete without errors.

# Installing Patched Modules

After build, required kernel modules are located under patched OFED folder:
```
compat/mlx_compat.ko
net/mlxdevm/mlxdevm.ko
drivers/net/ethernet/mellanox/mlxfw/mlxfw.ko
drivers/net/ethernet/mellanox/mlx5/core/mlx5_core.ko
drivers/net/ethernet/mellanox/mlx5/fpga/mlx5_fpga_tools.ko
```

Copy them to a custom updates folder (example):
```
sudo mkdir -p /lib/modules/$KVER/updates/patched_ofed
sudo cp <module>.ko /lib/modules/$KVER/updates/patched_ofed/
```
Run:
```
sudo depmod -a
```
### Unload Stock Mellanox Modules
```
sudo modprobe -r mlx5_fpga_tools mlx5_core mlxfw mlxdevm mlx_compat
```

### Load Patched Modules
```
sudo insmod /lib/modules/$KVER/updates/patched_ofed/mlx_compat.ko
sudo insmod /lib/modules/$KVER/updates/patched_ofed/mlxdevm.ko
sudo insmod /lib/modules/$KVER/updates/patched_ofed/mlxfw.ko
sudo insmod /lib/modules/$KVER/updates/patched_ofed/mlx5_core.ko
sudo insmod /lib/modules/$KVER/updates/patched_ofed/mlx5_fpga_tools.ko
```

After reboot, modules may need to be reloaded unless permanently replaced.

# Verification

Check dmesg:
```
dmesg | grep FPGA
```
Expected output:
```
[    1.040638] mlx5_core 0000:04:00.0: FPGA: Status 0; Admin image 0; Oper image 0
[    1.040656] mlx5_core 0000:04:00.0: FPGA: FPGA card Morse:2
[    1.602211] mlx5_core 0000:04:00.1: FPGA: Status 0; Admin image 0; Oper image 0
[    1.602218] mlx5_core 0000:04:00.1: FPGA: FPGA card Morse:2
[   11.378737] Call normal FPGA init
[   11.378741] Initializing FPGA
[   11.385147] FPGA: Status 0; Admin image 0; Oper image 0
[   11.385157] FPGA: FPGA card Morse:2
```

You should see FPGA initialization messages.

# Open Bundle Setup

Install and build Mellanox Open Bundle (example path):
```
~/Distrib/Open_bundle/Innova_2_Flex_Open_18_12/
```
I used a manual from Mwrnd to setup Open Bundle:
```
https://github.com/mwrnd/innova2_flex_xcku15p_notes?tab=readme-ov-file#set-up-innova-2-flex-application
```

### Initialize MST:
```
sudo mst start
sudo flint -d /dev/mst/mt4119_pciconf0 q
```

### Build device driver:
```
cd Innova_2_Flex_Open_18_12/driver
sudo ./make_device
```

You should see:
```
chardev /dev/mlx_fpga_bope0 created
```
I made a snippet to initialize it by single command:
```
sudo mst start && sudo flint -d /dev/mst/mt4119_pciconf0 q &&
cd ~/Distrib/Open_bundle/Innova_2_Flex_Open_18_12/driver/ && sudo ./make_device &&
sudo insmod /usr/lib/modules/`uname -r`/updates/patched_ofed/mlx5_fpga_tools.ko
```
# Running Userspace Application
```
sudo ./innova2_flex_app -v
```

Expected result:
 - FPGA accessible
 - Board type: Morse
 ```
 Verbosity:        1
 BOPE device:      None
 ConnectX device:  None
 ConnectX device: /dev/0000:04:00.0_mlx5_fpga_tools
 BOPE device:     None
 Scheduled image:  User Image
 Running image:    User Image(Success)
 Type of board: Morse

Jump-to-Innova2-User menu
------------------
[ 1 ] Set Innova2_Flex image active (reboot required)
[ 2 ] Set User image active
[ 3 ] Enable JTAG Access - no thermal status
[ 4 ] Read FPGA thermal status
[ 5 ] Reload User image
[99 ] Exit
Your choice:
```

# Known Limitations
 - Tested only on Raspberry Pi 5 (ARM64)
 - Limited validation across kernel versions
 - No production-level stress testing performed
 - Module load order matters

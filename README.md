
PVE 8.2 Linux pve 6.8.12-1-pve #1 SMP PREEMPT_DYNAMIC PMX 6.8.12-1 (2024-08-05T16:17Z) x86_64 GNU/Linux

1、安装 dkms 及头文件
apt update && apt install -y pve-headers proxmox-headers-$(uname -r) dkms 

2、确保有github的连通性，克隆i915-sriov-dkms
git clone https://github.com/strongtz/i915-sriov-dkms.git

3、进入i915-sriov-dkms文件夹,纳入模块
dkms add .

4、安装i915 dkms模块
dkms install -m i915-sriov-dkms -v $(cat VERSION) --force

5、安装sysfsutils
apt install sysfsutils -y

6、sysfs设置属性，一般intel核显总线位置都是在0000:00:02.0位置，不清楚的话lspci | grep VGA查找GPU总线位置
echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7" > /etc/sysfs.conf

7、以下分为grub和efi引导，如果pve系统使用lvm情况下执行grub方法，如果pve系统使用zfs按efi执行

GRUB
nano /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on i915.enable_guc=3 i915.max_vfs=7" 
之后执行
update-grub
update-initramfs -u -k all

EFI
nano /etc/kernel/cmdline
root=ZFS=rpool/ROOT/pve-1 boot=zfs quiet intel_pstate=passive intel_iommu=on 
最后面加入 i915.enable_guc=3 i915.max_vfs=7
之后执行
update-initramfs -u -k all
pve-efiboot-tool refresh


8、win10\11虚拟机，增加PCI设备
设备选择 0000:00:02.1~7都可以，勾选主GPU、ROM-BAR 、PCIE，所有功能一定不要勾，勾了之后是该虚拟机独占GPU,虚拟机显示选择  VirtIO-GPU，我试了下，如果选择默认、VGA等等，WIN10系统的核显会有43代码错误，无法运行
![image](https://github.com/user-attachments/assets/c8429868-15c9-481d-ab3f-028214289ca9)





# Linux i915 driver (dkms module) with SR-IOV support for linux 6.1 ~ linux 6.11

Originally from [linux-intel-lts](https://github.com/intel/linux-intel-lts/tree/lts-v5.15.49-adl-linux-220826T092047Z/drivers/gpu/drm/i915)
Update to [6.1.12](https://github.com/intel/linux-intel-lts/tree/lts-v6.1.12-linux-230415T124447Z/drivers/gpu/drm/i915)

## Update Notice

The i915 module parameter to enable SR-IOV functionality has changed since [commit #092d1cf](https://github.com/strongtz/i915-sriov-dkms/commit/092d1cf126f31eca3c1de4673e537c3c5f1e6ab4). If you are updating from previous version, please modify `i915.enable_guc=7` to **`i915.enable_guc=3 i915.max_vfs=7`** in your kernel command line or in the corresponding modprobe config file.

## Warning

This package is **highly experimental**, you should only use it when you know what you are doing.

You need to install this dkms module in **both host and guest!**

For Arch Linux users, it is available in AUR. [i915-sriov-dkms-git](https://aur.archlinux.org/packages/i915-sriov-dkms-git)

Tested kernel versions: 

* Proxmox VE Host: `pve-kernel-6.1.0-1-pve ~ 6.2.9-1-pve`, `proxmox-kernel-6.5.13-3-pve ~ 6.8.12-1-pve`
* Debian 12 VM Guest: `linux-image-6.5.0-0.deb12.4-amd64 ~ 6.7.12+bpo` (6.1 requires custom kernel, see below)
* Ubuntu 22.04 VM Guest: `linux-image-6.2.0-39-generic ~ 6.5.0-44-generic`
* Gentoo VM Guest: `gentoo-sources-6.1.19-gentoo ~ 6.2.11-gentoo`

Tested usages:

- VA-API video acceleration in VM (need to remove any other display device such as virtio-gpu)


## Required Kernel Parameters
```
intel_iommu=on i915.enable_guc=3 i915.max_vfs=7
```

## Creating Virtual Functions (VF)

```
echo 2 > /sys/devices/pci0000:00/0000:00:02.0/sriov_numvfs
```

You can create up to 7 VFs on Intel UHD Graphics 

## PVE Host Installation Steps (Tested Kernel 6.5 and 6.8)
1. Clone this repo
1. Install build tools: `apt install build-* dkms`
1. Install the kernel and headers for desired version: `apt install proxmox-headers-6.8.8-2-pve proxmox-kernel-6.8.8-2-pve` (for unsigned kernel).
1. Change into the root of the cloned repository and run `dkms add .`.
1. Execute the command `dkms install -m i915-sriov-dkms -v 2024.09.21 --force` or `dkms install -m i915-sriov-dkms -v $(cat VERSION) --force` for a version-independent command.
1. Once finished, the kernel commandline needs to be adjusted: `nano /etc/default/grub` and change `GRUB_CMDLINE_LINUX_DEFAULT` to `intel_iommu=on i915.enable_guc=3 i915.max_vfs=7`, or add to it if you have other arguments there already.
1. Optionally pin the kernel version and update the boot config via `proxmox-boot-tool`.
1. In order to enable the VFs, a `sysfs` attribute must be set. Install `sysfsutils`, then do `echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7" > /etc/sysfs.conf`, assuming your iGPU is on 00:02 bus. If not, use `lspci | grep VGA` to find the PCIe bus your iGPU is on.
1. Reboot the system.
1. When the system is back up again, you should see the number of VFs under 02:00.1 - 02:00.7. Again, assuming your iGPU is on 00:02 bus.
1. You can passthrough the VFs to LXCs or VMs. However, never touch the PF which is 02:00.0 under any circumstances.

## PVE Host Installation Steps (Tested Kernel 6.1 and 6.2) 
1. Clone this repo
1. Install some tools. `apt install build-* dkms`
1. Go inside the repo, edit the `dkms.conf`file, change the `PACKAGE_NAME` to `i915-sriov-dkms`, and change the `PACKAGE_VERSION` to `6.1`. Save the file.
1. Move the entire content of the repository to `/usr/src/i915-sriov-dkms-6.1`. The folder name will be the DKMS package name.
1. Execute command `dkms install -m i915-sriov-dkms -v 6.1 --force`. `-m` argument denotes the package name, and it should be the same as the folder name which contains the package content. `-v` argument denotes the package version, which we have specified in the `dkms.conf` as `6.1`. `--force` argument will reinstall the module even if a module with same name has been already installed.
1. The kernel module should begin building.
1. Once finished, we need to make a few changes to the kernel commandline. `nano /etc/default/grub` and change `GRUB_CMDLINE_LINUX_DEFAULT` to 'intel_iommu=on i915.enable_guc=3 i915.max_vfs=7`, or add to it if you have other arguments there already.
1. Update `grub` and `initramfs` by executing `update-grub` and `update-initramfs -u`
1. In order to enable the VFs, we need to modify some variables in the `sysfs`. Install `sysfsutils`, then do `echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7" > /etc/sysfs.conf`, assuming your iGPU is on 00:02 bus. If not, use `lspci | grep VGA` to find the PCIe bus your iGPU is on.
1. Reboot the system.
1. When the system is back up again, you should see the number of VFs you specified show up under 02:00.1 - 02:00.7. Again, assuming your iGPU is on 00:02 bus.
1. You can passthrough the VFs to LXCs or VMs. However, never touch the PF which is 02:00.0 under any circumstances.

## Linux Guest Installation Steps (Tested Kernel 6.2)
We will need to run the same driver under Linux guests. We can repeat the steps for installing the driver. However, when modifying command line defaults, we use `i915.enable_guc=3` instead of `i915.enable_guc=3 i915.max_vfs=7`. Furthermore, we don't need to use `sysfsutils` to create any more VFs since we ARE using a VF.
Once that's done, update `grub` and `initramfs`, then reboot. Once the VM is back up again, do `dmesg | grep i915` to see if your VF is recognized by the kernel.
Optionally, install `vainfo`, then do `vainfo` to see if the iGPU has been picked up by the VAAPI.

## Windows Guest
It is required to set the host CPU type in Proxmox to "host". I was able to get it working without further fiddling in the config files but your mileage may vary (i5-12500T with UHD 770).
I've used Intel gfx version 4316 to get it working. Here's a link to download it.
(https://www.intel.com/content/www/us/en/download/741626/780560/intel-arc-pro-graphics-windows.html)

## Debian Guest Installation

Debian poses some additional challenges because not all of the necessary modules are compiled in by default. That leads us to building a custom kernel with a modified version of their configuration. The following steps were tested on Debian 11.6 "bullseye" which includes the 5.10.0 kernel out-of-the-box. We need, at minimum, the 6.1 kernel

First, the VM configuration:
* BIOS: OVMF (UEFI)
* Display: Default
* Machine: q35
* Secure boot must be disabled in the UEFI BIOS, otherwise the new, unsigned, kernel will not start.

With the 6.1 kernel not being available in the stable repository, the testing repository must be added. The following steps are based on [these instructions](https://serverfault.com/questions/22414/how-can-i-run-debian-stable-but-install-some-packages-from-testing)

**NOTE:** All of these commands were run by the root user. Run as a regular
user by prepending `sudo`, if you prefer.

* Create `/etc/apt/preferences.d/stable.pref`:
```
cat <<EOT >> /etc/apt/preferences.d/stable.pref
# 500 <= P < 990: causes a version to be installed unless there is a
# version available belonging to the target release or the installed
# version is more recent
Package: *
Pin: release a=stable
Pin-Priority: 900
EOT
```
* Create `/etc/apt/preferences.d/testing.pref`:
```
cat <<EOT >> /etc/apt/preferences.d/testing.pref
# 100 <= P < 500: causes a version to be installed unless there is a
# version available belonging to some other distribution or the installed
# version is more recent
Package: *
Pin: release a=testing
Pin-Priority: 400
EOT
```
* Move `/etc/apt/sources.list` to `/etc/apt/sources.list.d/stable.list`
```
mv /etc/apt/sources.list /etc/apt/sources.list.d/stable.list
```
* Create `/etc/apt/sources.list.d/testing.list` as follows:
```
sed 's/bullseye/testing/g' /etc/apt/sources.list.d/stable.list > /etc/apt/sources.list.d/testing.list
```

Now the process of building and installing a new kernel begins

* Use `apt` to fully update the system.
```
apt update && apt -y dist-upgrade && apt -y autoremove
```
* Find the latest version of the 6.1 kernel and install it
```
apt search '^linux-image-6.*-amd64'
  linux-image-6.1.0-7-amd64/testing 6.1.20-1 amd64
    Linux 6.1 for 64-bit PCs (signed)
apt -y install linux-image-6.1.0-7-amd64
reboot
```
* Install the 6.1 kernel source and configure it.
```
apt -y install dkms dwarves git linux-source-6.1 pahole vainfo
cd /usr/src
tar xJvf linux-source-6.1.tar.xz
```
* Copy Debian's original build configuration into the source tree:
```
cp /boot/config-6.1.*-amd64 /usr/src/linux-source-6.1/.config
```
* Edit `/usr/src/linux-source-6.1/.config` and ensure the following parameters exist:
```
CONFIG_INTEL_MEI_PXP=m
CONFIG_DRM_I915_PXP=y
```
* Build and install the kernel
```
cd /usr/src/linux-source-6.1
make deb-pkg LOCALVERSION=-sriov KDEB_PKGVERSION=$(make kernelversion)-1

    ...four hours later...

dpkg -i /usr/src/*.deb
reboot
```
* Verify the new kernel is indeed running:
```
uname -r
6.1.15-sriov
```
* Build and install the i915-sriov module
```
cd /usr/src
git clone https://github.com/strongtz/i915-sriov-dkms i915-sriov-dkms-6.1

    edit /usr/src/i915-sriov-dkms-6.1/dkms.conf with the following:
    PACKAGE_NAME="i915-sriov-dkms"
    PACKAGE_VERSION="6.1"

dkms install --force -m i915-sriov-dkms -v 6.1

    edit /etc/default/grub with the following:
    GRUB_CMDLINE_LINUX_DEFAULT="quiet i915.enable_guc=3"

update-grub
update-initramfs -u
poweroff
```
* In Proxmox, add one of the 0000:00:02.x devices to the VM, then start the VM
* Log into the machine and verify that the module has loaded
```
# lspci | grep -i vga
06:10.0 VGA compatible controller: Intel Corporation Alder Lake-S GT1 [UHD Graphics 730] (rev 0c)

# lspci -vs 06:10.0
06:10.0 VGA compatible controller: Intel Corporation Alder Lake-S GT1 [UHD Graphics 730] (rev 0c) (prog-if 00 [VGA controller])
	Subsystem: ASRock Incorporation Alder Lake-S GT1 [UHD Graphics 730]
	Physical Slot: 16-2
	Flags: bus master, fast devsel, latency 0, IRQ 42
	Memory at c1000000 (64-bit, non-prefetchable) [size=16M]
	Memory at 800000000 (64-bit, prefetchable) [size=512M]
	Capabilities: [ac] MSI: Enable+ Count=1/1 Maskable+ 64bit-
	Kernel driver in use: i915
	Kernel modules: i915

# dmesg | grep i915
[    6.461702] i915: loading out-of-tree module taints kernel.
[    6.462463] i915: module verification failed: signature and/or required key missing - tainting kernel
[    6.591228] i915 0000:06:10.0: Running in SR-IOV VF mode
[    6.592001] i915 0000:06:10.0: GuC interface version 0.1.0.0
[    6.592212] i915 0000:06:10.0: [drm] VT-d active for gfx access
[    6.592225] i915 0000:06:10.0: [drm] Using Transparent Hugepages
[    6.593437] i915 0000:06:10.0: GuC interface version 0.1.0.0
[    6.593564] i915 0000:06:10.0: GuC firmware PRELOADED version 1.0 submission:SR-IOV VF
[    6.593565] i915 0000:06:10.0: HuC firmware PRELOADED
[    6.595883] i915 0000:06:10.0: [drm] Protected Xe Path (PXP) protected content support initialized
[    6.595886] i915 0000:06:10.0: [drm] PMU not supported for this GPU.
[    6.595951] [drm] Initialized i915 1.6.0 20201103 for 0000:06:10.0 on minor 1

# ls /dev/dri/render*
crw-rw---- 1 root render 226, 128 Jan 00 00:00 renderD128

# vainfo
libva info: VA-API version 1.17.0
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_17
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.17 (libva 2.12.0)
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 23.1.1 ()
vainfo: Supported profile and entrypoints

    ... List of all the compression formats and profiles ...
```
* The kernel DEB installation files can be copied to other, similar, Debian systems for use without recompiling.

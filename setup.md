# Giving a KVM/QEMU Virtual Machine direct access to a memory-mapped hardware device with VFIO (ARM)



## Building the environment

**Suitable guest and host device trees**

- **qemu-guest.dts**: Guest VM's device tree adapted to QEMU and the SDHC passed-through with VFIO .
- **host.dts**: Host's device tree adapted to properly pass-through the SDHC device to user-space/VM.

**For building host and guest Linux systems:**

- **host_config**: Host's buildroot menuconfig
- **host_kconfig**: Host's buildroot linux-menuconfig (kconfig)
- **guest_config**: Guest's buildroot menuconfig
- **guest_kconfig**: Guest's buildroot linux-menuconfig (kconfig)

**Once the guest and the host Linux systems have been built:**

- Load the host's Linux image and device tree on a first partition of the MMC. 

- Flash the host's device tree (rootfs.ext4) on a second partition of the MMC.

- Load the guest's image, device tree and rootfs (rootfs.cpio) on the host's root file systems.

  

## Setting up the environment

*Refer to readme.md for more explanation and details.*

### Booting the Host Linux system

- arm-smmu.disable_bypass must be disabled to ensure the device is behind the IOMMU

- vfio-platform.reset_required must be disabled if the host has no reset mechanisms

```
setenv bootargs  vfio-platform.reset_required=0 arm-smmu.disable_bypass=0 root="/dev/sda" rw rootwait; fatload mmc 0 0x200000 Image && fatload mmc 0 0x1e0000 host.dtb && booti 0x200000 - 0x1e0000
```

### Preparing VFIO device
```shell
# Unbind the physical driver from the device to be passed-through if necessary
echo ff170000.mmc > /sys/bus/platform/drivers/sdhci-arasan/unbind
echo vfio-platform > /sys/bus/platform/devices/ff170000.mmc/driver_override

# Required if interrupt remapping is not avalaible (likely on SoC platforms)
echo 1 > /sys/module/vfio_iommu_type1/parameters/allow_unsafe_interrupts
echo ff170000.mmc > /sys/bus/platform/drivers_probe
```

### Running the QEMU guest VM
```shell
qemu-system-aarch64 -nodefaults -machine virt,accel=kvm -cpu host -m 1024 -nographic -serial mon:stdio -kernel Image -dtb qemu-guest.dtb -initrd rootfs.cpio -append "mmc_core.debug_level=7 console=ttyAMA0 root=/dev/ram rw earlycon=pl011,0x09000000" -device vfio-platform,host=ff170000.mmc -d int,guest_errors,invalid_mem,page,mmu
```

### Running the native device driver in the VM

```shell
modprobe cqhci.ko
insmod /lib/modules/6.12.27/kernel/drivers/mmc/host/sdhci.ko
insmod /lib/modules/6.12.27/kernel/drivers/mmc/host/sdhci-pltfm.ko
insmod /lib/modules/6.12.27/kernel/drivers/mmc/host/sdhci-of-arasan.ko
```
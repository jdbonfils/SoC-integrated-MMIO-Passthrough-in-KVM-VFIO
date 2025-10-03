# Usefull Commands

#Boot the Host Linux vfio-platform.reset_required=0
setenv bootargs  vfio-platform.reset_required=0 arm-smmu.disable_bypass=0 root="/dev/sda" rw rootwait; fatload mmc 0 0x200000 Image && fatload mmc 0 0x1e0000 host.dtb && booti 0x200000 - 0x1e0000

#Prepare VFIO device
echo ff170000.mmc > /sys/bus/platform/drivers/sdhci-arasan/unbind

echo vfio-platform > /sys/bus/platform/devices/ff170000.mmc/driver_override
echo 1 > /sys/module/vfio_iommu_type1/parameters/allow_unsafe_interrupts
echo ff170000.mmc > /sys/bus/platform/drivers_probe

#Boot Linux with rootfs.cpio as initial RAM fs
qemu-system-aarch64 -nodefaults -machine virt,accel=kvm -cpu host -m 1024 -nographic -serial mon:stdio -kernel Image -dtb qemu-guest.dtb -initrd rootfs.cpio -append "mmc_core.debug_level=7 console=ttyAMA0 root=/dev/ram rw earlycon=pl011,0x09000000" -device vfio-platform,host=ff170000.mmc -d int,guest_errors,invalid_mem,page,mmu

qemu-system-aarch64 -nodefaults -machine virt,virtualization=on,iommu=smmuv3 -cpu cortex-a57 -m 1024 -nographic -serial stdio -kernel Image -dtb qemu-guest.dtb -initrd rootfs.cpio -append "mmc_core.debug_level=7 console=ttyAMA0 root=/dev/ram rw earlycon=pl011,0x09000000" -device vfio-platform,host=ff170000.mmc,sysfsdev=/sys/bus/platform/devices/ff170000.mmc/ -d guest_errors,unimp,invalid_mem,mmu


#DUmp the guest DTB (useful for QEMU generated DTB)
dtc -I fs /proc/device-tree -O dtb -o /tmp/qemu-runtime.dtb

#Insert physical device driver in the QEMU Guest
modprobe cqhci.ko
insmod /lib/modules/6.12.27/kernel/drivers/mmc/host/sdhci.ko
insmod /lib/modules/6.12.27/kernel/drivers/mmc/host/sdhci-pltfm.ko
insmod /lib/modules/6.12.27/kernel/drivers/mmc/host/sdhci-of-arasan.ko

sleep 8

busybox devmem 0xc000000 64

busybox devmem 0xc000000 64



















#bin/bash

modprobe cqhci.ko

insmod /lib/modules/6.12.27/kernel/drivers/mmc/host/sdhci.ko
insmod /lib/modules/6.12.27/kernel/drivers/mmc/host/sdhci-pltfm.ko
insmod /lib/modules/6.12.27/kernel/drivers/mmc/host/sdhci-of-arasan.ko

sleep 8

busybox devmem 0xc000000 64
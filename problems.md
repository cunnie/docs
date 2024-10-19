### GCP metal-intel-192-1536-noble

```
[ 4629.294448] DMAR: DRHD: handling fault status reg 3
[ 4629.299366] DMAR: [DMA Read NO_PASID] Request device [05:00.1] fault addr 0x1ffffffff0000 [fault reason 0x71] SM: Present bit in first-level paging entry is clear
[ 4629.322396] DMAR: DRHD: handling fault status reg 3
[ 4629.327309] DMAR: [DMA Read NO_PASID] Request device [05:00.1] fault addr 0x1fffffffc0000 [fault reason 0x71] SM: Present bit in first-level paging entry is clear
[ 4629.344250] DMAR: DRHD: handling fault status reg 3
...
[ 4640.181137] nvme0n2: I/O Cmd(0x1) @ LBA 4295251447, 64 blocks, I/O Error (sct 0x0 / sc 0x4) 
[ 4640.185310] DMAR: [DMA Read NO_PASID] Request device [05:00.1] fault addr 0x1ffffff718000 [fault reason 0x71] SM: Present bit in first-level paging entry is clear
[ 4640.193770] I/O error, dev nvme0n2, sector 4295251447 op 0x1:(WRITE) flags 0x9800 phys_seg 1 prio class 0
[ 4640.218145] XFS (nvme0n2p1): log I/O error -5
[ 4640.222577] XFS (nvme0n2p1): Filesystem has been shut down due to log error (0x2).
[ 4640.230234] XFS (nvme0n2p1): Please unmount the filesystem and rectify the problem(s).
[ 4640.238549] nvme0n2p1: writeback error on inode 596964943, offset 578813952, sector 603113720
...
nvme0n1: I/O Cmd(0x1) @ LBA 10628000, 16 blocks, I/O Error (sct 0x0 / sc 0x4)
...
[ 4918.251200] Buffer I/O error on device nvme0n1p1, logical block 31010816
[ 4918.257972] Buffer I/O error on device nvme0n1p1, logical block 31010817
...
[ 4940.837091] EXT4-fs (nvme0n1p1): Remounting filesystem read-only
```

Looks most like [this bug](https://lore.kernel.org/linux-iommu/20201231005323.2178523-4-baolu.lu@linux.intel.com/t/)
I'm turning setting [iommu=off](https://docs.kernel.org/admin-guide/kernel-parameters.html) to hopefully fix.

Reliably triggered by running the following:

```bash
ollama create Llama-3.1-70B -f <(echo FROM /mnt/work/Llama-3.1-70B)
```


### RISC-V

Solution: move custom 6.10.7 kernel out of `/boot/`

```
Processing triggers for flash-kernel (3.107ubuntu9~24.04.3) ...
Using DTB: sifive/hifive-unmatched-a00.dtb#####################################################################...]
Installing /etc/flash-kernel/dtbs/hifive-unmatched-a00.dtb into /boot/dtbs/6.10.7-majestic.old/sifive/hifive-unmatched-a00.dtb
Taking backup of hifive-unmatched-a00.dtb.
Installing new hifive-unmatched-a00.dtb.
dpkg-query: no packages found matching
dpkg: error processing package flash-kernel (--configure):
 installed flash-kernel package post-installation script subprocess returned error exit status 1
Processing triggers for linux-image-6.8.0-47-generic (6.8.0-47.47.1) ...
/etc/kernel/postinst.d/initramfs-tools:#########################################################################..]
update-initramfs: Generating /boot/initrd.img-6.8.0-47-generic
Using DTB: sifive/hifive-unmatched-a00.dtb
Installing /etc/flash-kernel/dtbs/hifive-unmatched-a00.dtb into /boot/dtbs/6.8.0-47-generic/sifive/hifive-unmatched-a00.dtb
Installing new hifive-unmatched-a00.dtb.
flash-kernel: deferring update (trigger activated)
/etc/kernel/postinst.d/zz-flash-kernel:
Using DTB: sifive/hifive-unmatched-a00.dtb
Installing /etc/flash-kernel/dtbs/hifive-unmatched-a00.dtb into /boot/dtbs/6.8.0-47-generic/sifive/hifive-unmatched-a00.dtb
Taking backup of hifive-unmatched-a00.dtb.
Installing new hifive-unmatched-a00.dtb.
flash-kernel: deferring update (trigger activated)
/etc/kernel/postinst.d/zz-update-grub:
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.10.7-majestic
Found initrd image: /boot/initrd.img-6.10.7-majestic
Found linux image: /boot/vmlinuz-6.10.7-majestic.old
Found initrd image: /boot/initrd.img-6.10.7-majestic
Found linux image: /boot/vmlinuz-6.8.0-47-generic
Found initrd image: /boot/initrd.img-6.8.0-47-generic
Found linux image: /boot/vmlinuz-6.8.0-44-generic
Found initrd image: /boot/initrd.img-6.8.0-44-generic
Found linux image: /boot/vmlinuz-6.8.0-41-generic
Found initrd image: /boot/initrd.img-6.8.0-41-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done
Processing triggers for libc-bin (2.39-0ubuntu8.3) ...
Errors were encountered while processing:
 flash-kernel
needrestart is being skipped since dpkg has failed
```

```bash
less /var/log/apt/{history,term}.log
sudo find / -name flash-kernel\*
sudo nvim /var/lib/dpkg/info/flash-kernel.postinst # insert -x to trace the install, also, echo the environment to stderr
 # sudo bash -x /var/lib/dpkg/info/flash-kernel.postinst configure
sudo /usr/share/debconf/frontend /var/lib/dpkg/info/flash-kernel.postinst configure 3.107ubuntu9~24.04.1
sudo debconf-show flash-kernel
  # flash-kernel/linux_cmdline: quiet splash
sudo dpkg-reconfigure flash-kernel
  #/usr/sbin/dpkg-reconfigure: flash-kernel is broken or not fully installed
sudo find / -name flash-kernel\*deb
cp /var/cache/apt/archives/flash-kernel_3.107ubuntu9~24.04.3_riscv64.deb /tmp/junk
sudo -i
mkdir /tmp/junk
cd /tmp/junk
sudo dpkg -e /var/cache/apt/archives/flash-kernel_3.107ubuntu9~24.04.3_riscv64.deb
export DEBCONF_PACKAGE=flash-kernel
bash -x DEBIAN/postinst configure
FLASH_KERNEL_NOTRIGGER=y sudo -E flash-kernel
```

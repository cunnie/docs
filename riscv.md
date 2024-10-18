### RISC-V

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
```

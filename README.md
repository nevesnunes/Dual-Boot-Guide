# Linux + Windows Dual-Boot Troubleshooting Guide

Each section describes an issue that you may experience, followed by possible solutions. 

We assume the installed Linux distribution is Ubuntu. Nevertheless, these procedures end up being very similar for other mainstream distributions.

If possible, avoid disabling `Secure Boot`. It should be your last option.

# Windows bootloader is run instead of GRUB

We have to set GRUB as the default bootloader.

### In Windows

```
bcdedit /set {bootmgr} path \EFI\ubuntu\shimx64.efi
```

### In Linux

```bash
sudo efibootmgr -v
```
Lists the available boot entries.

```bash
sudo efibootmgr -o 0001,0002,0003,...
```
Sets the order of each boot entry that will be attempted to run.

# Windows ignores the order I set

Unfortunately, some manufacturers hard-code the path to the bootloader.

We have to replace that path with GRUB's files. The Windows bootloader will need a custom entry to be run.

### Check boot order

Make sure the Windows bootloader is the first boot entry to be loaded:

```bash
sudo efibootmgr -v
```

### Switch bootloaders

Move each bootloader's files like this:

```
/EFI/Microsoft/Boot => /EFI/MicrosoftReal/Boot
/EFI/ubuntu => /EFI/Microsoft/Boot
/EFI/Microsoft/Boot/shimx64.efi => /EFI/Microsoft/Boot/bootmgfw.efi
```

### Create a new boot entry for Windows

Copy the following entry to `/etc/grub.d/40_custom`:

```
menuentry 'Windows Boot Manager' {
    insmod part_gpt
    insmod fat
    set root='hdX,gptX'
    chainloader /EFI/MicrosoftReal/Boot/bootmgfw.efi
}
```
Where `hdX` is your disk, `gptX` is your EFI partition.

Finally run `update-grub`. You should have the generated GRUB in `/boot`.

### [Optional] Override Linux GRUB config

In case the above changes are ignored, you can copy that menuentry to the GRUB config file that is present in the EFI partition. Since we moved them around, it should be something like `/EFI/Microsoft/Boot/grub.cfg`.

# GRUB doesn't appear as a boot entry

You need to add it as a trusted bootloader.

```
BIOS > Set Password
BIOS > Secure Boot > Select an UEFI file as trusted for executing
```

# Maybe GRUB isn't working?

Try to load it manually with a `Live CD` image. The procedure is called **chainloading**, since you will be running your installed GRUB from the image's GRUB:

```
set root='(hdX,gptX)'
chainloader /EFI/Microsoft/Boot/bootmgfw.efi
boot
```
Where `hdX` is your disk, `gptX` is your EFI partition. Confirm with `set`.

Here we assume you have the hard-coded Windows path replaced with GRUB.

If you can't run GRUB, you can try reinstalling it (see the next section).

# Fix a broken GRUB

Run as root:

```bash
# Check partition ids
fdisk -l

# Mount partitions
modprobe efivars
mount /dev/sdaX /mnt
mount /dev/sdaY /mnt/boot/efi
for i in /dev /dev/pts /proc /sys; do mount -B "$i" "/mnt$i"; done

# Make network available in chroot
cp /etc/resolv.conf /mnt/etc/

chroot /mnt
```
Where `sdaX` is your root partition, `sdaY` is your EFI partition.

Now reinstall GRUB:

```bash
apt-get install --reinstall grub-efi
update-grub
efibootmgr -c --disk /dev/sda --part Y
```
Where `Y` is your EFI partition's number.

Finally, exit the chroot, unmount everything and reboot:

```bash
exit
for i in /sys /proc /dev/pts /dev; do umount "/mnt$i"; done
umount /mnt/boot/efi
umount /mnt
reboot
```

# Maybe my menuentry isn't working?

Try to load the kernel image manually:

```
root (hd0,msdos3)
kernel /boot/vmlinuz-$KERNEL_RELEASE root=/dev/sda3
initrd /boot/initramfs-$KERNEL_RELEASE
boot
```

Where `$KERNEL_RELEASE` can be obtained with `uname -r`.

If your distro is using `blscfg` to generate menuentries, check:

- Template: `/boot/loader/entries/$MACHINE_ID-$KERNEL_RELEASE.conf`
- Image: `/boot/$MACHINE_ID/`

An example of a corresponding menuentry:

```
menuentry 'Linux (4.4.183)' --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-4.4.183' {
    load_video
    set gfxpayload=keep
    insmod gzio
    insmod part_msdos
    insmod ext2
    set root=(hd0,msdos3)
    linux16 /boot/$MACHINE_ID/$KERNEL_RELEASE/linux root=/dev/sda3 ro
    initrd16 /boot/$MACHINE_ID/$KERNEL_RELEASE/initrd
}
```

If you get dropped to an initramfs shell, you can make changes to the root partition:

```bash
mount -o remount,rw /sysroot
```

If you made any external changes that affect menuentries, rebuild the initramfs:

```bash
# For BIOS-based machines
grub2-mkconfig -o /boot/grub2/grub.cfg

# For UEFI-based machines
grub2-mkconfig -o /boot/efi/EFI/$DISTRO/grub.cfg
```

# Windows doesn't shutdown || I want to access my Windows partition in Linux

You need to disable `Fast Startup`.

```
Start > Settings > System > Power & Sleep > Additional Power settings >
    "Choose what the power buttons do" >
    "Change settings that are currently unavailable" >
    Untick "Turn on fast startup"
```

# Other issues

Feel free to leave an issue or pull request if you want some other cases covered.

# References

- https://www.gnu.org/software/grub/manual/grub/html_node/Command_002dline-and-menu-entry-commands.html
- https://help.ubuntu.com/community/Grub2/Troubleshooting
- https://superuser.com/questions/111152/whats-the-proper-way-to-prepare-chroot-to-recover-a-broken-linux-installation
- https://wiki.archlinux.org/index.php/GRUB

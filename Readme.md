# **DRAFT/INCOMPLETE** as of 11 July 20: Dual boot Manjaro Linux on Dell XPS 7590

This guide builds on the excellent earlier work done by Armin Coralic and documents my experience installing Manjaro Linux (encrypted) next to pre-installed Windows on my Dell XPS 7590. I have changed those items necessary for a US-based Manjaro installation.  

As with the earlier Arch guide, my XPS-15 also contains an Intel i9 CPU, 32 gigs of memory, a 1TB disk, and an OLED screen. 

## Key References: 

1. The Official Arch install guide for the [XPS 7590](https://wiki.archlinux.org/index.php/Dell_XPS_15_7590). 

2. The [Manjaro User Guide](https://manjaro.org/support/userguide/).

## Prepare Yourself:

1. Back up all your files and have a Windows rescue disk ready.

2. During safemode reboot, you may be presented with a login screen that uses a *local* account. Make sure you have the password!

3. Make some coffee (or drink of choice)! 

## Prepare Windows:

1. Update your Windows drivers and [BIOS](https://www.dell.com/support/home/en-us/product-support/product/xps-15-7590-laptop/drivers). 

2. Change SATA mode to "AHCI" in BIOS by [following this guide](https://triplescomputers.com/blog/uncategorized/solution-switch-windows-10-from-raidide-to-ahci-operation/).

3. Turn off Fast Start-Up by [following this guide](https://www.windowscentral.com/how-disable-windows-10-fast-startup).

4. Shrink the OS partition (usually C:\ drive) to make space for Manjaro Linux by following [this guide](https://docs.microsoft.com/en-us/windows-server/storage/disk-management/shrink-a-basic-volume). 

5. Turn off the SSD hardware encryption (BitLocker) by following this [guide](https://www.dell.com/support/article/en-us/sln302845/how-to-enable-or-disable-bitlocker-with-tpm-in-windows?lang=en).  Note: Full BitLocker may be "ON" or you may see "Used Space Only Encrypted".  You can verify the status of your disk encryption by running the following command as administrator from cmd: `manage-bde -status`.  If this is a brand new Dell and you only see "Used Space Only Encrypted" you can choose to continue without disabling BitLocker.  For additional information you may read more [from Microsoft](https://docs.microsoft.com/en-us/windows/security/information-protection/bitlocker/bitlocker-device-encryption-overview-windows-10). 

6. Turn off UEFI secure boot and change "Fastboot" to "Thorough". While booting your machine press `F2` when you see the Dell logo. When the BIOS loads up select "Secure Boot" -> "Secure Boot Enable" and deslect the box on that screen. Then select "POST Behaviour" -> "Fastboot" and change from "Minimal" to "Thorough".

7. Change Windows to use UTC time by [following this guide](https://wiki.archlinux.org/index.php/System_time#UTC_in_Windows).  

## Manjaro Pre-Install 

1. Please download the [Manjaro Linux iso](https://manjaro.org/download/) and put it on a USB stick using [Etcher](https://www.balena.io/etcher/)

2. Use the USB disk to boot into Manjaro. 

3. Verify partitions on your disk. Running the `fdisk -l` command will show you all the partitions on the SSD, it should be something as following:

* /dev/nvme0n1p1 -> EFI
* /dev/nvme0n1p2 -> Microsoft reserved (128mb)
* /dev/nvme0n1p3 -> Windows
* /dev/nvme0n1p4 -> Winretools
* /dev/nvme0n1p5 -> Recovery Image
* /dev/nvme0n1p6 -> Dell Support

Notice how you don't see the free space of 488gb for Manjaro. You can execute `cfdisk /dev/nvme0n1` to visually see if there is free space left on the disk, in my case I can see Free space of 488gb.

## Manjaro Install

We will reuse the existing EFI volume, create an unencrypted /boot volume and an encrypted LVM with Swap space and /root in it. We need /boot outside of the encrypted partition because it simplifies the boot process. It isn't that big of a security issue because it's only the kernel which is the /boot.

**Step 1**
Connect to the internet through WiFi, any ISO with kernel 5.2.2 and above already has support for the WiFi card. To check your kernel execute `uname -r`.

You can use the `wifi-menu` command to connect to your WiFi. After entering the password you will get a black screen for a couple of seconds, just wait a bit.

Verify you have internet by `ping google.com`. (Note, you might need to retry if you are too quick)

**Step 2**
Partition the free space on the disk for Arch Linux. Use `cfdisk /dev/nvme0n1` to start cfdisk and then create the following two partitions:

* 1gb -> which is going to be used for /boot. I am using 1gb because I might want to use multiple kernels, it can be smaller
* 487gb -> which is to be used for encrypted LVM

Execute `blkid` and remember the UUID's you need them later on.

**Step 3**
Format the 1gb partition with the command `mkfs.ext4 /dev/nvme0n1p7`. Be sure that nvme0n1p7 is actually the Arch /boot partition.

**Step 4**
Encrypt the disk by executing `cryptsetup luksFormat /dev/nvme0n1p8` Be sure that nvme0n1p8 is actually the Arch partition (the 487gb one). The default settings from cryptsetup are [good enough](https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#Encryption_options_for_LUKS_mode)

After the encryption, we need to decrypt the disk with the following command `cryptsetup luksOpen /dev/nvme0n1p8 Arch`. The "Arch" can be anything you wish, it is the name you will use to reference the volume after it is decrypted.

**Step 5**
Format the encrypted partition in the correct format. I am creating a swap space of 32gb because I want to be able to hibernate if needed. If you don't want that you can create a small swap space. Execute the following commands:

* `pvcreate /dev/mapper/Arch`
* `vgcreate Arch /dev/mapper/Arch`
* `lvcreate -L +32G Arch -n swap`
* `lvcreate -l +100%FREE Arch -n root`

Now we need to create filesystems on the encrypted partitions:

* `mkswap /dev/mapper/Arch-swap`
* `mkfs.ext4 /dev/mapper/Arch-root`

**Step 6**
Mount all the partitions:

* `mount /dev/mapper/Arch-root /mnt` -> Mount root
* `swapon /dev/mapper/Arch-swap` -> Enable swap
* `mkdir /mnt/boot` -> Create boot dir
* `mount /dev/nvme0n1p7 /mnt/boot` -> Make sure nvme0n1p7 is Arch /boot of 1gb
* `mkdir /mnt/boot/efi` -> Create efi dir
* `mount /dev/nvme0n1p1 /mnt/boot/efi` -> Make sure nvme0n1p1 is EFI partition

**Step 7**
Before installing Arch you can specify which mirrors you want to use but I leave that for later. To install Arch execute the following command `pacstrap /mnt base base-devel refind-efi dialog wpa_supplicant`.

You need "dialog wpa_supplicant" to be able to access wifi again with wifi-menu when you are done with the installation.

**Step 8**
We need to generate fstab with `genfstab -U /mnt >> /mnt/etc/fstab`. Now verify /de/mapper/Arch-root, /dev/nvme0n1p7 and /dev/nvme0n1p1 are in fstab with `cat /mnt/etc/fstab`.

**Step 9**
Chroot into your Arch `arch-chroot /mnt`

**Step 10**
Set the timezone, see [official guide](https://wiki.archlinux.org/index.php/Installation_guide#Time_zone) on how to do it.

**Step 11**
Set the locale, see [official guide](https://wiki.archlinux.org/index.php/Installation_guide#Localization) on how to do it.

**Step 12**
Set your network preferences, see [official guide](https://wiki.archlinux.org/index.php/Installation_guide#Network_configuration) on how to do it.

**Step 13**
Set your root password with the `passwd` command

**Step 14**
Generate mkinitcpio by first editing `nano /etc/mkinitcpio.conf` with the following HOOK -> (base udev autodetect modconf block keymap encrypt lvm2 resume filesystems keyboard fsck). The hooks are explained [here](https://wiki.archlinux.org/index.php/Mkinitcpio#Common_hooks)

Now generate mkinitcpio with the following command `mkinitcpio -p linux`

**Step 15**
To be able to run Arch we need REFInd boot manager. First execute `refind-install` to install REFInd to EFI. Because we use encrypted volume we need to edit it: `nano /boot/efi/EFI/refind/refind.conf`. Enable the "scanfor" option and then find the "menuentry Arch Linux" and change it to the following:

menuentry "Arch Linux" {
    icon    /EFI/refind/icons/os_arch.png
    volume  USE_THE_PARTUUID_OF_THE_BOOT_PARTITION
    loader  /vmlinuz-linux
    initrd  /initramfs-linux.img
    options "rw cryptdevice=/dev/nvme0n1p8:Arch root=/dev/mapper/Arch-root resume=/dev/mapper/Arch-swap"
    submenuentry "Boot using fallback initramfs" {
        initrd  /initramfs-linux-fallback.img
    }
    submenuentry "Boot to terminal" {
        add_options "systemd.unit=multi-user.target"
    }
}

Make sure you remove the word "disabled" which is by default there.

**Done**
It's time to reboot and see if you can start Windows and Arch. Please exit chroot and execute `umount -R /mnt` then `swapoff -av` and then `reboot now`.

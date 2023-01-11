# lvm-on-luks

Set up full partition encryption for Ubuntu using LUKS and GRUB2 and optionally LVM and Secure Boot for multi-boot systems.

These instructions bridge a gap left by Ubiquity, the Ubuntu installer, which supports either full disk encryption or no encryption. When dual/multi-booting or otherwise wanting to retain other disk partitions, these instructions can be used to set up an encrypted Ubuntu OS partition.

These instructions were most recently developed / tested with Kubuntu 22.10 on a UEFI system with an NVMe drive using LUKS and Secure Boot but not LVM. Previous iterations have been used with some combination of the aforementioned and: Ubuntu or Arch Linux, with MBR, and with SATA drives.

## Unencrypted boot

### Notes on terminology:

* in your Linux distro, your physical storage disk will be named similar to `nvme0n1` or `sda`, henceforth refered to as `<disk>`
* the partitions on that disk will be named similar to `nvme0n1p8` or `sda3`; `<partition>` will henceforth refer to the suffix of `<disk>`, i.e. `p8` or `3`
* therefore `<disk><parition>` means something like `nvme0n1p8`

### Instructions

1. Ensure there is space available for your Linux installation

    Resize your Windows / other partitions as needed; resize Windows partitions using the built in Windows tools.

1. Using your Linux live USB, create `/boot` and LUKS ext4 partitions

    1. boot partition - [recommended ~750MB](https://github.com/rdkr/lvm-on-luks/issues/4)), e.g `nvme0n1p3` 
    1. LUKS partition  - rest of the available space, e.g `nvme0n1p4`
    
    Set some variables for later:
    
    ```
    disk=<disk> # e.g. nvme0n1
    efi_partition=<efi partition> # e.g. p1
    boot_partition=<boot partition> # e.g. p7
    luks_partition=<luks partition> # e.g. p8
    ```

1. Optionally, fill the partitions with noise

    This could prevent an attacker from learning about the size or structure of the contents, and will destroy any data currently on disk where the new partition has been created. This may take a while for a large partition.
    
    ```
    openssl enc -pbkdf2 -pass pass:"$(dd if=/dev/urandom bs=128 count=1 2>/dev/null | base64)" -nosalt < /dev/zero > /dev/<disk><partition>
    ```

1. Configure the LUKS container

    ```
    # create the LUKS container
    cryptsetup luksFormat /dev/$disk$luks_partition

    # open the LUKS container and name it crypt
    cryptsetup luksOpen /dev/$disk$luks_partition crypt
    
    # format the LUKS container
    mkfs.ext4 /dev/mapper/crypt
    ```

1. Optionally, configure LVM

    ```
    # create physical volume on in new luks container
    pvcreate /dev/mapper/crypt

    # create virtual group called lvm in physical volume
    vgcreate lvm /dev/mapper/crypt
    ```

    Now using a `lvcreate` or a partition manager, create the volumes you would like. For example, `/` and `/home`.
    
    The rest of the instructions will assume at minimum that you have created `/` with the name `root`, e.g. `lvcreate -n root -l <size> lvm` 

1. Run the Ubuntu installer
    
    1. you can enable Secure Boot - just enroll the key using the password you set upon first boot
    1. manually configure the partitions:
        - if not LVM: `/dev/mapper/crypt` -> `/`, don't format
        - if LVM: `/dev/mapper/root` -> `/`, format as ext4 + any other volumes as desired
        - `/dev/<disk><boot paritition>` -> `/boot`, format as ext4
        - `/dev/<disk><efi partition>` -> bootloader device

1. Run the `lvm-on-luks` script
    
    ```
    bash lvm-on-luks $disk $efi_partition $boot_partition $luks_partition clearboot
    ```

    This will update the crpyttab and initramfs to work with LUKS.
    
## Encrypted boot (deprecated)

 - https://help.ubuntu.com/community/Full_Disk_Encryption_Howto_2019 is a more recent version of the same idea.
 - If you are interested in experimenting with this information, refer to an older version of this readme instead: https://github.com/rdkr/lvm-on-luks/tree/15c0e1c3ddaaa576139029975cb79f5f09aac8db

⚠️ **The encrypted boot section is out of date and untested as of 2023!** ⚠️
 
It is possible to also encrypt /boot on LVM on LUKS, as decribed by Pavel Kogan (see References). Note that this is a less common option and does have potential drawbacks for dual boot use, as the LUKS password must be entered regardless of which OS is booted. Importantly, it does NOT protect against an evil maid type attacker with physical access to the machine, as the bootloader is unencrypted and could be modified to capture the password. An external / encrypted bootloader, for example running from / decrypting from USB, could mitigate this and improve security, but this script does not yet accommodate that.

As /boot is on LVM on LUKS, the LUKS container must first be unencrypted and loaded by GRUB; this is done with the LUKS password before the GRUB menu. GRUB does not pass the key on to the ramdisk so a second unencryption must be performed by the ramdisk in order to mount the other logical volumes. Rather than enter the password twice, the script sets up a keyfile for LUKS which is contained within initramfs (which is encrypted until the password is entered to GRUB) and is used automatically instead of a duplicate password entry.

As mentioned above, this setup means the LUKS password must be entered before GRUB, even to access the GRUB menu; if the password is not supplied, GRUB will enter rescue mode. For a dual boot system this may be inconvenient. It may be possible to address this for other encrypted dual boot OSes sharing a common password, and is something for future research. Overall, this method does not have much benefit over the more common first method for the reason described in the first paragraph of this section.

### Instructions

0. As with unencrypted /boot

1. Create one partitions: sdxy for the LUKS container

2. As with unencrypted /boot

3. As with unencrypted /boot
    
4. Install Ubuntu as usual using Ubiquity to format the new logical volumes: / on lvm-root (ext4), /home on lvm-home (ext4), and swap space on lvm-swap; bootloader on /sdx.  I found using this method that the bootloader installation failed every time (select continue without bootloader) and sometimes crashed, but this doesn't matter as the script must re-install GRUB with modified config anyway. 

5. Return to the script and press [Enter] once, then answer 'y' to the question and press [Enter] again. This will run the remainder of the script to set up GRUB, the keyfile, and crypttab before running update-initramfs. It also runs an apt-get remove for Ubiquity as I found that it persists after the installation with 'Install REVISION' on the launcher, which I believe may be due to the bootloader installation and consequently Ubiquity failing. Upon exit of the script, restart the computer.

### Limitations

The script does not set up `/swap` for use with hibernation. The `/boot` encryption provided in this is of questionable benefit without further work to run an encrypted or external bootloader.

## Issues and Contributing

If you encounter problems running the script I'm happy to help try to work through them with you - please open an issue! I'm also happy to review improvements and updates - please open a PR!

## References

Many thanks to Pavel Kogan for his work which formed the basis of this script, and to the Ubuntu community at large.

* http://www.pavelkogan.com/2014/05/23/luks-full-disk-encryption/
* http://www.pavelkogan.com/2015/01/25/linux-mint-encryption/
* https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS
* http://manpages.ubuntu.com/manpages/hardy/man8/initramfs-tools.8.html
* https://unix.stackexchange.com/questions/107810/why-my-encrypted-lvm-volume-luks-device-wont-mount-at-boot-time

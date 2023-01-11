# linux-lvm-on-luks

Sets up full partition encryption for Linux (optionally including /boot) using LUKS and optionally LVM.

Following these instructions whilst installing your Linux distro, you can set up working LUKS encryption, optional LVM volume management, and GRUB boot manager alongside your existing partitions / OS.

These instructions are motivated by Ubiquity, the Ubuntu installer, not being flexible enough to support encrypted installations from a drive with other partitions, such as when dual booting. They were developed with (K)ubuntu, but should work / be adaptable for any disto with the appropriate tools installed.

## Unencrypted boot

This is a common method for full system encryption, and uses two disk partitions: one for /boot and one for LVM on LUKS which contains: `/`, and optionally `/home`.

### Note on terminology:

* in your Linux distro, your physical storage disk will be named similar to `nvme0n1` or `sda`, henceforth refered to as `<disk>`
* the partitions on that disk will be named similar to `nvme0n1p3` or `sda3`; `<partition>` will henceforth refer to the suffix of `<disk>`, i.e. `p3` or `3`
* therefore `<disk><parition>` is something like `nvme0n1p3` and `<disk> <parition>` is something like `nvme0n1 p3`

### Instructions

1. Ensure there is space available for your Linux installation

    Resize your Windows / other partitions as needed; resize Windows partitions using the built in Windows tools.

1. Using your Linux live USB, create `/boot` and LUKS ext4 partitions

    1. boot partition - [recommended ~750MB](https://github.com/rdkr/lvm-on-luks/issues/4)), e.g `nvme0n1p3` 
    1. LUKS partition  - rest of the available space, e.g `nvme0n1p4`
    
    Set some variables for later:
    
    ```
    disk=<disk>
    efi_partition=<efi partition>
    boot_partition=<boot partition>
    luks_partition=<luks partion>
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

3. Run the install script
    
    ```
    bash lvm-on-luks $disk $efi_partition $boot_partition $luks_partition clearboot
    ```
    
4. Partitions and bootloader should be set up as follows:

    ```
    /          on lvm-root (ext4)
    /home      on lvm-home (ext4)
    /boot      on /sdxz    (ext4)
    bootloader on /sdx
    ```

    Then proceed with the installation and choose 'continue testing' upon completion.

5. Return to the script and press [Enter] twice to continue running the script for a non-encrypted /boot. This will run the remainder of the script to set up crypttab and will ask for the location of the /boot partition (sdxz, created earlier) before running update-initramfs. Upon exit of the script, restart the computer.

## Encrypted boot

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

## Limitations

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

 # from http://serverfault.com/a/415962

# ubuntu-lvm-on-luks
Script to set up full partition encryption for Ubuntu (optionally including /boot) using LVM on LUKS, for use with dual boot

Ubiquity (the Ubuntu installer) will set up encryption and LVM automatically but only using for an entire drive; this is not appropriate for dual booting with an existing OS. This script automates the installation of LUKS encryption, LVM volume management, and GRUB boot manager during an Ubuntu installation.

Please note that I am not an expert and this was written as a learning exercise. Many thanks to Pavel Kogan and the Ubuntu community (see References below). Any contributions or advice are more than welcome.

## Unencrypted boot
This is a more common method for full system encryption, and uses two disk partitions: one for /boot and one for LVM on LUKS which contains / /home and /swap.

### Instructions
0. Refer to https://help.ubuntu.com/community/DiskSpace regarding partition sizes, including swap and boot, if unsure.
1. Create two partitions: sdxy for the LUKS container, and sdxz for the /boot partition
2. Optionally fill the LUKS container partition with enecrypted noise to prevent an attacker from potentially learning about the size or structure of the contents, and to destroy any data currently on the partition.
    
    `sudo bash lvm-on-luks sdx y noise`

3. Run the install script:

    `sudo bash lvm-on-luks <disk: sdx> <partition: y> install <swap size in G> <root size in G or 0 for all remaining> [optionally, home size in G. default: all remaining. if previous argument is 0 no /home is created]`

    This command will ask for an encryption password three times (twice to create and once to open the encypted volume) in order to set up the the LUKS container and LVM volumes ready for installation. It will then pause and prompt for Ubuntu to be installed.
    
4. Install Ubuntu as usual using Ubiquity to format the new logical volumes: / on lvm-root (ext4), /home on lvm-home (ext4), and swap space on lvm-swap; /boot on /sdxz; bootloader on /sdx. Click 'continue testing' upon completion.

5. Return to the script and press [Enter] twice to continue running the script for a non encrypted /boot. This will run the remainder of the script to set up crypttab and will ask for the location of the /boot partition (sdxz, created earlier) before running update-initramfs. Upon exit of the script, restart the computer.

## Encrypted boot
It is possible to also encrypt /boot on LVM on LUKS, as decribed by Pavel Kogan (see References). Note that this is a less common option and does have potential drawbacks for dual boot use, as the LUKS password must be entered regardless of which OS is booted. Importantly, it does NOT protect against an evil maid type attacker with physical access to the machine, as the bootloader is unencrypted and could be modified to capture the password. An external / encrypted bootloader, for example running from / decrypting from USB, could mitigate this and improve security, but this script does not yet accomodate that.

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
The script is currently untested with GPT partitioned disks, and does not set up /swap for use with hibernation. The /boot encryption provided in this is of questionable benefit without further work to run an encrypted or external bootloader.

##References

http://www.pavelkogan.com/2014/05/23/luks-full-disk-encryption/

http://www.pavelkogan.com/2015/01/25/linux-mint-encryption/

https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS

http://manpages.ubuntu.com/manpages/hardy/man8/initramfs-tools.8.html

https://unix.stackexchange.com/questions/107810/why-my-encrypted-lvm-volume-luks-device-wont-mount-at-boot-time

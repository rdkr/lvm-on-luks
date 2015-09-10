# ubuntu-lvm-on-luks
Script to set up full partition encryption for Ubuntu (including /boot) using LVM on LUKS, for use with dual boot

Ubiquity (the Ubuntu installer) will set up encryption and LVM automatically but only for a blank drive. This script automates the installation of LUKS encryption, LVM volume management, and GRUB boot manager for an Ubuntu installation which will be entirely contained on one hard drive partition (including /boot). This is useful for a fully encrypted Ubuntu installation on a dual boot machine.

As /boot is encrypted it must first be unencrypted to be loaded by GRUB; this is done with a password before GRUB. GRUB does not pass the key on so a second unencryption is needed such that /boot can launch Ubuntu; the script sets up a keyfile which is used for this purpose and the password is therefore only entered once. Note that this means the password must be entered before GRUB, even to access the main GRUB page to boot into another OS. If this is undesirable, an alternative and more common approach is to have /boot unencrypted on second hard drive partition with the rest of the installation on a LUKS/LVM partition as here; the key (in the form of a password) is needed only after GRUB with the tradeoff that /boot is unencrypted. 
Please note that I am not an expert and this was written as a learning exercise. Many thanks to Pavel Kovan and the Ubuntu community (see References below). Any contributions or advice are more than welcome.

1. create partition sdxy for the LUKS container
2. `sudo bash lvm-on-luks sdx y noise` - this will fill the new partition with encrypted noise so nothing about the size or structure of the encrypted content can be learnt from examining the hard drive partition
3. `sudo bash lvm-on-luks sdx y pre` - this sets up LUKS and LVM volumes for installation. It will ask for a password three times (twice to create and once to open the encypted volume).
4. install ubuntu as usual using Ubiquity. / on lvm-root (ext4), /home on lvm-home (ext4), and swap-space on lvm-swap (swap). /sdx for bootloader; I found that the bootloader installation failed everytime (select continue without bootloader) and sometimes crashed, but this doesn't matter as the script must re-install GRUB with modified config anyway. 
5. `sudo bash lvm-on-luks sdX Y post` - this configures GRUB and sets up initramfs to automatically load a keyfile rather than requires the password a second time. It also runs an apt-get remove for Ubiquity as I found that it persists after the installation with 'Install REVISION' on the launcher, which I believe may be due to the bootloader installation and consequently Ubiquity failing.

##References

http://www.pavelkogan.com/2014/05/23/luks-full-disk-encryption/

http://www.pavelkogan.com/2015/01/25/linux-mint-encryption/

https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS

http://manpages.ubuntu.com/manpages/hardy/man8/initramfs-tools.8.html

https://unix.stackexchange.com/questions/107810/why-my-encrypted-lvm-volume-luks-device-wont-mount-at-boot-time

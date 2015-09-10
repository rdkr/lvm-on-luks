# ubuntu-lvm-on-luks
Script to set up full partition encryption for Ubuntu (including /boot) using LVM on LUKS, for use with dual boot

manually create sdXY partition for luks to live on
run 'sudo bash lvm-on-luks sdX Y noise'
run 'sudo bash lvm-on-luks sdX Y pre'
install ubuntu as usual. format and use the new lvm partitions
choose no bootloader if grub install fails. ubiquity may crash
run 'sudo bash lvm-on-luks sdX Y post'

http://www.pavelkogan.com/2014/05/23/luks-full-disk-encryption/
http://www.pavelkogan.com/2015/01/25/linux-mint-encryption/
https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS
http://manpages.ubuntu.com/manpages/hardy/man8/initramfs-tools.8.html
https://unix.stackexchange.com/questions/107810/why-my-encrypted-lvm-volume-luks-device-wont-mount-at-boot-time
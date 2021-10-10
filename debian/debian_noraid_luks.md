## Overview
Install Debian on a bare metal server on hetzer with an encrypted disk.
**!! This doesnt use LVM !!**
### Install Image
1. Set the server in rescue mode and log in via ssh.
2. Create a "setup.conf" file, with this content and edit to your preferences
```
CRYPTPASSWORD <put your luks password here>
DRIVE1 /dev/nvme0n1
BOOTLOADER grub
HOSTNAME myserver
PART /boot ext4 5G
PART /     ext4 all crypt
IMAGE /root/images/Debian-1100-bullseye-amd64-base.tar.gz
```
Don't forget to edit the IMAGE here, if hetzner provides newer ones. You can get them from the ~/configs/ folder, where different default configs are.

3. Start install with
```
ìnstallimage -a -c setup.conf
```

### Post Install Setup 
1. Mount partitions
```
mount /dev/mapper/luks-abcdefgh-1234-5678-9012-ijklmnopqrst /mnt
mount /dev/nvme0n1p1 /mnt/boot/
```
2. Mount additional directories
```
mount --bind /dev /mnt/dev/
mount --bind /sys /mnt/sys
mount --bind /proc /mnt/proc
mount --bind /run/udev /mnt/run/udev
```
3. chroot into your installation
```
chroot /mnt
```
4. Install additional packages
```
apt update && apt upgrade -y
apt install dropbear-initramfs cryptsetup-initramfs
```
5. Configure Dropbear
add to /etc/dropbear-initramfs/authorized_keys
```
no-port-forwarding,no-agent-forwarding,no-X11-forwarding,command="/bin/cryptroot-unlock" <paste the public key of the keypair you want to use for login>
```
in /etc/dropbear-initramfs/config change
```
#DROPBEAR_OPTIONS="
```
to
```
DROPBEAR_OPTIONS='-c cryptroot-unlock'
```
finally update initramfs
```
update-initramfs -u
```

Then you can ```exit``` and ```reboot``` your server and then login with your private ssh key to unlock the disk.

Tested succesfully on 10.10.2021

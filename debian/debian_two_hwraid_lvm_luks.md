## Overview

This tutorial is a TLDR version of my [blogpost](https://www.sysorchestra.com/2018/06/14/hetzner-root-server-with-dual-hardware-raid-1-and-lvm-on-luks-on-debian-9/) that explains everything in much more detail. Check it out if you have questions.

* Hetzner Serverbörse Root-Server "SB48"
* Intel Xeon 4C/8T E3-1275V2
* 32GB DDR3 ECC Reg. RAM
* LSI MegaRAID SAS 9260-4i Hardware-RAID-Controller
* 2x 300GB SAS 15k
* 2x 3TB SATA 7.2k
* Redundant PSU
* LVM on LUKS on Debian 9

### Setup Disks

Overview of all physical disks and virtual drives (raids):

```
~$ megacli -PDList -a0 # Lists all physical disks on the first adapter (controller)
Adapter #0

Enclosure Device ID: 252
Slot Number: 0
...
Enclosure Device ID: 252
Slot Number: 1
...
Enclosure Device ID: 252
Slot Number: 2
...
Enclosure Device ID: 252
Slot Number: 3
...

~$ megacli -LDInfo -Lall -a0 # Lists all logical devices configured (e.g. RAID configurations)

Adapter 0 -- Virtual Drive Information:
Virtual Drive: 0 (Target Id: 0)
Name                :
RAID Level          : Primary-1, Secondary-0, RAID Level Qualifier-0
Size                : 278.875 GB
...
Adapter 0 -- Virtual Drive Information:
Virtual Drive: 1 (Target Id: 1)
Name                :
RAID Level          : Primary-1, Secondary-0, RAID Level Qualifier-0
Size                : 2.728 TB
...
```

Save actual RAID-config and delete it for later recreation.

```
megacli -CfgSave -f raidcfg.txt -a0 # Saves the current config to raidcfg.txt
megacli -CfgClr -a0 # Removes all available RAID configurations
```

Now build up the new RAID config. We will use the faster SAS disks as a RAID-1 for the System and the "slower" 3TB RAID-1 for other stuff. You could also build two RAIDs and then use both as physical storage in a single LVM. See below on a comment when you have the choice for either.

Let's build the new RAID. First, check the health:

```
~$ megacli -PDList -a0 | grep "Firmware state"
Firmware state: Unconfigured(good), Spun Up
Firmware state: Unconfigured(good), Spun Up
Firmware state: Unconfigured(good), Spun Up
Firmware state: Unconfigured(good), Spun Up
```

If one is not "good" make it "good".

```
megacli -PDMakeGood -PhysDrv[252:2] -a0 # [x:y] - x = enclosure-id, y = drive slot from PDList command
```

Configure both RAID-1. Since Hetzner has redundant Power supply and this machine has redundant PSU, we will use Caching without BBU (Battery).

```
megacli -CfgLdAdd -r1 [252:0,252:1] WB RA Direct CachedBadBBU -a0
megacli -CfgLdAdd -r1 [252:2,252:3] WB RA Direct CachedBadBBU -a0
```

RAID-(Re)build should be really fast but you can check status with:

```
~$ megacli -PDRbld -ShowProg -PhysDrv [252:0,252:1,252:2,252:3] -a0

Device(Encl-252 Slot-0) is not in rebuild process
Device(Encl-252 Slot-1) is not in rebuild process
Device(Encl-252 Slot-2) is not in rebuild process
Device(Encl-252 Slot-3) is not in rebuild process
```

Check first RAID for bootable-status and set it if it's not bootable.

```
megacli -AdpBootDrive -get -a0 # Get bootable RAID's
megacli -AdpBootDrive -set -L0 -a0 # Set our first RAID config as bootable
```

## Install the system

```
installimage -a -n myhostname.com \
-b grub \
-r no \
-i root/.oldroot/nfs/images/Debian-94-stretch-64-minimal.tar.gz \
-p /boot:ext4:512M,lvm:vg0:all \
-v vg0:swap:swap:swap:8G,vg0:root:/:ext4:16G,vg0:var-log:/var/log:ext4:8G,vg0:tmp:/tmp:ext4:8G,vg0:var-lib-postgresql:/var/lib/postgresql:ext4:150G \
-f yes -s de -K /root/.ssh/robot_user_keys
```

Reboot to new system and install updates and necessary tools for headless boot-time-decryption.

```
apt update && apt-get -y upgrade
apt -y install busybox dropbear dropbear-initramfs lvm2
```

Ignore the following error as we will configure (fix) that later.

```
dropbear: WARNING: Invalid authorized_keys file, remote unlocking of cryptroot via SSH won't work!
```

## Adding first LUKS Device

Reboot to Rescue System again! Now we will backup our new system and install LUKS.

```
mkdir /rootbackup
mount /dev/vg0/root /mnt
mount /dev/vg0/var-log /mnt/var/log
mount /dev/vg0/tmp /mnt/tmp
mount /dev/vg0/var-lib-postgresql /mnt/var/lib/postgresql
rsync -a /mnt/ /rootbackup/ # takes up to 30sec
```

Unmount our system and remove it.

```
umount /mnt/var/lib/postgresql && umount /mnt/tmp && umount /mnt/var/log && umount /mnt
vgchange -a n vg0 # Otherwise recreation of LUKS on sda2 will fail later
```

Recreate partition.

```
parted /dev/sda
print free # check which partition contains the LVM. Mind the "Start" value of that partition which should be 538MB
rm 2 # Remove the partition
mkpart primary 538MB -1s # Change the "Start" value of sda2 if it differs
quit
```

Generate password and create LUKS device on first RAID.

```
pwgen 64 1 # Generates long secure password
# Now create LUKS device. See my blogpost for details!
cryptsetup --cipher aes-xts-plain64 --key-size 512 --hash sha256 --iter-time 6000 luksFormat /dev/sda2
```

## Add second LUKS device

Lets configure the second LUKS device for our 3TB RAID-1.

```
parted /dev/sdb
mklabel gpt # For disks larger than 2TB! Otherwise mklabel msdos.
mkpart primary 2048s 100%
```

Generate a new password or use the same (both is ok) and add LUKS device.

```
cryptsetup --cipher aes-xts-plain64 --key-size 512 --hash sha256 --iter-time 6000 luksFormat /dev/sdb1
```

## Restore system!

Mount first LUKS device and recreate the initial LVM with its filesystems.

```
cryptsetup luksOpen /dev/sda2 cryptroot # Open LUKS device
pvcreate /dev/mapper/cryptroot # Create physical LVM volume on LUKS
vgcreate vg0 /dev/mapper/cryptroot # Create our volume group again
# Create our logical volumes now
lvcreate -L 8G -n swap vg0
lvcreate -L 16G -n root vg0
lvcreate -L 8G -n var-log vg0
lvcreate -L 8G -n tmp vg0
lvcreate -L 150G -n var-lib-postgresql vg0
# Format the volumes and create swap again
mkfs.ext4 /dev/vg0/root
mkfs.ext4 /dev/vg0/var-log
mkfs.ext4 /dev/vg0/tmp
mkfs.ext4 /dev/vg0/var-lib-postgresql
mkswap /dev/vg0/swap
```

Add the second LUKS to the party! In my blogpost I explain why I mount it on /var/lib/barman. Use your preferred mountpoint here.

```
cryptsetup luksOpen /dev/sdb1 cryptmount
pvcreate /dev/mapper/cryptmount
vgcreate vg1 /dev/mapper/cryptmount
lvcreate -l 100%FREE -n var-lib-barman vg1
mkfs.ext4 /dev/vg1/var-lib-barman
```

Mount all initial folders again and sync your system back to the disk.

```
mount /dev/vg0/root /mnt
mkdir -p /mnt/var/log
mount /dev/vg0/var-log /mnt/var/log
mkdir /mnt/tmp
mount /dev/vg0/tmp /mnt/tmp
mkdir -p /mnt/var/lib/postgresql
mount /dev/vg0/var-lib-postgresql /mnt/var/lib/postgresql
rsync -a /rootbackup/ /mnt/
```

## Chroot and configure boot-time-decryption!

Now let's chroot to our new system from the Rescue system and configure it.

```
mount /dev/sda1 /mnt/boot
mount --bind /dev /mnt/dev
mount --bind /sys /mnt/sys
mount --bind /proc /mnt/proc
chroot /mnt
```

Create the mountpoint for our second RAID. Use your preferred one here as well and change the pats accordingly.

```
mkdir /var/lib/barman # Create new mountpoint
mount /dev/vg1/var-lib-barman /var/lib/barman
# Open /etc/fstab and add the new mount point at the end
vim /etc/fstab
/dev/vg1/var-lib-barman  /var/lib/barman  ext4  defaults 0 0
```

Add both LUKS device to the crypttab list, so we can decrypt both at boot time. Below are the contents of `/etc/crypttab`.

```
# <target name> <source device>     <key file>  <options>
cryptroot /dev/sda2 none luks,initramfs
cryptmount /dev/sdb1 none luks,initramfs
```

Fix the interfaces after unlocking the LUKS devices and booting to the main system. Add this below to `/etc/rc.local`

```
# Content of rc.local
/sbin/ifdown --force eth0
/sbin/ifup --force eth0
```

Since the `cryptroot-unlock` is quite old and has a bug with unlocking multiple devices at boot time, fix it by overwriting the current one with a fixed version with the following command.

```
curl https://salsa.debian.org/mjg59-guest/cryptsetup/raw/e0ad47dc25281372c01798dce41a8786f052057c/debian/initramfs/cryptroot-unlock -o /usr/share/cryptsetup/initramfs/bin/cryptroot-unlock
```

Now add your SSH RSA public key(s) to dropbear-initramfs so you can login securely for decrypting at boot-time.

```
# Copy the content of your public key
cat ~/.ssh/id_rsa.pub # For macOS you can add "|pbcopy" to directly copy it to your clipboard
On the server
vim /etc/dropbear-initramfs/authorized_keys
# Paste the key here and save the file
# You can add as much keys for multiple persons/machines as you like
```

Now everything is configured. Let's update the initramfs and install grub to be sure, everythings works just fine.

```
update-initramfs -u -k all
update-grub
grub-install /dev/sda
```

## Cleanup and first boot to final secured system

Now we're basically done. Let's save and unmount everything and reboot to our new system.

```
exit # Exit Chroot environment
umount /mnt/var/lib/barman/
umount /mnt/var/lib/postgresql
umount /mnt/var/log
umount /mnt/tmp
umount /mnt/boot
umount /mnt/dev
umount /mnt/sys
umount /mnt/proc
umount /mnt
sync
shutdown -r now
```

After reboot, you should be able to login to dropbear with your SSH keys. Now unlock your system.

```
~ # cryptroot-unlock
Please unlock disk cryptroot (/dev/sda2):
cryptsetup: cryptroot set up successfully
Please unlock disk cryptmount (/dev/sdb1):
cryptsetup: cryptmount set up successfully
# ...continues with booting here!
```

Done!

## Contributions

Contributions are welcome if something does not work as expected.

## Comments
- Tested by [@martinseener](https://github.com/martinseener)
- There are two Bonus Points on my blogpost! Enhance your servers security by disabling USB and install megacli tool on your server to manage your RAID without the Rescue System!

## overview


Debian 9.4
No LVM
Hardware Raid Controller



### Setup Disks

I'm running Debian with a hardware raid (3ware) controller and 2 physical disks).

First boot to rescue, then take a look at the raid controller.
 
```
root@rescue ~ # tw_cli show

Ctl   Model        (V)Ports  Drives   Units   NotOpt  RRate   VRate  BBU
------------------------------------------------------------------------
c6    9650SE-2LP   2         2        0       0       1       1      -

root@rescue ~ # tw_cli /c6 show

Unit  UnitType  Status         %RCmpl  %V/I/M  Stripe  Size(GB)  Cache  AVrfy
------------------------------------------------------------------------------

VPort Status         Unit Size      Type  Phy Encl-Slot    Model
------------------------------------------------------------------------------
p0    OK             -    2.73 TB   SATA  0   -            ST33000651AS
p1    OK             -    2.73 TB   SATA  1   -            ST33000651AS
```

create new raid 0 with `tw_cli maint createunit c6 rraid0 p0:1` (see [hetzner 3ware page](https://wiki.hetzner.de/index.php/3Ware_RAID_Controller/en) for reference)

Now run `installimage` don't forget to set your `HOSTNAME`, then choose partitions. My config is:

```
PART /boot ext3 512M
PART lvm vg0 all
LV vg0 root / ext4 100G
LV vg0 swap swap swap 8G
LV vg0 srv /srv ext4 all
```

then you can `shutdown -r now`

### Configure Debian

- ssh into new system
- I always get perl locale errors so `dpkg-reconfigure locales`, choose & generate your (UTF-8) locale
- install all the things `apt update && apt install busybox dropbear dropbear-initramfs`
- I get an error like this in the last few lines:

```
update-initramfs: Generating /boot/initrd.img-4.9.0-6-amd64
dropbear: WARNING: Invalid authorized_keys file, remote unlocking of cryptroot via SSH won't work!
```

Don't worry about it for now, we will fix it when we configure initramfs

If you're using mdadm, you should wait until initial md replication is complete. check `cat /proc/mdstat`

### Boot 3: Rescue Again

```
mkdir /oldroot
mount /dev/mapper/vg0-root /mnt
mount /dev/mapper/vg0-srv /mnt/srv
rsync -a /mnt/ /oldroot/
umount /mnt/srv /mnt
```

Now delete the lvm partition and create a new one (with no fs), open parted with `parted /dev/sda` then in the parted cli:

```
parted /dev/sda
print free
rm 2
mkpart primary 539MB -1s
quit
```

Before encrypting that new partition, `lsblk` to identify what device it is, mine's `/dev/sda2`

use `pwgen 64 1` or something to generate a super secret passphrase, save that somewhere.

cryptsetup magic
```
cryptsetup --cipher aes-xts-plain64 --key-size 512 --hash sha256 --iter-time 6000 luksFormat /dev/sda2
```

Things to know:

 - it's an uppercase YES
 - you can check luks headers with `cryptsetup luksDump /dev/sda3`
 - I corrupted my partition table by referring to devices by uuid the first time round. LUKS will change those ids.


this "activates the luks device" ?!
```
cryptsetup luksOpen /dev/sda3 cryptroot
```


create the physical volume, the volume group, and the partitions we want, create file systems
```
pvcreate /dev/mapper/cryptroot
vgcreate vg0 /dev/mapper/cryptroot
lvcreate -L 8G -n swap vg0
lvcreate -L 100G -n root vg0
lvcreate -L 5.3T -n srv vg0
mkfs.ext4 /dev/vg0/srv
mkfs.ext4 /dev/vg0/root
mkswap /dev/vg0/swap
```

copy data back in (leave partitions mounted for now)

```
mount /dev/vg0/root /mnt
mount /dev/vg0/srv /mnt/srv
rsync -a /oldroot/ /mnt/
```

## configure initramfs & boot magic


To unlock remotely you need to configure cryptsetup to 'pause' during boot, open an ssh server (dropbear) and wait for you to log in and provide the password.

Now chroot into the boot partition and configure it.

```
mount /dev/sda1 /mnt/boot && \
mount --bind /dev /mnt/dev && \
mount --bind /sys /mnt/sys && \
mount --bind /proc /mnt/proc && \
chroot /mnt
```

edit `/etc/crypttab` to look like this:

```
# <target name> <source device>         <key file>      <options>
cryptroot /dev/sda2 none luks
```

make sure you'll have a network interface during boot, edit `/etc/rc.local`

```
/sbin/ifdown --force eth0
/sbin/ifup --force eth0
```

```
update-initramfs -u
update-grub
grub-install /dev/sda
```



close out

```
exit
umount /mnt/boot /mnt/srv /mnt/proc /mnt/sys /mnt/dev
umount /mnt
sync
shutdown -r now
```


After a few seconds the dropbear ssh server is coming up on your system, connect to it and unlock your system like this:

- `ssh -i .ssh/dropbear root@<yourserverip>`
- a busybox shell should come up
- unlock your lvm drive with:
- `echo -ne "<yourstrongpassphrase>" > /lib/cryptsetup/passfifo`

## Sources:
Special thanks to the people who wrote already this guides:

- http://notes.sudo.is/RemoteDiskEncryption
- https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system

## Contribution

- PRs are very welcome or open an issue if something not works for you as described

## Comments
- Tested this guide on 25.10.2017 on my own hetzner system, its working pretty good :-)

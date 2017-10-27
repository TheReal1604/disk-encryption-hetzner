## disk-encryption-hetzner

This should be a clean step-by-step guide how to setup a hetzner root server from the server auctions at hetzners "serverbörse" to get a fully encrypted software raid1 with lvm on top.

The goal of this guide is to have a server system that has encrypted drives and is remotely unlockable.

This guide *could* work at any other provider with a rescue system.

### Hardware setup
"Dedicated Root Server SB36"
- Intel Xeon E3-1246V3
- 2x HDD SATA 2,0 TB Enterprise 
- 4x RAM 8192 MB DDR3

### First steps in rescue image

- Boot to the rescue system via hetzners server management page
- install a minimal Ubuntu 16.04 LTS with hetzners "installimage" skript (https://wiki.hetzner.de/index.php/Installimage)
- I choosed the following logical volumes on my system to keep it simple:

```
lv-home (60GB) ext4
lv-log (30GB) ext4
lv-swap (10GB) swap
lv-root (all) -> means remaining space ext4
```

- after you adjusted all parameters in the install config file, press F10 to install the ubuntu minimal system
- reboot and ssh into your fresh installed ubuntu

### First steps on your fresh ubuntu installation

- connect via ssh-key you choosed before for the rescue image (attention to the .ssh/known_hosts file..)
- install busybox and dropbear
- `apt update && apt install busybox dropbear`
- Edit your `/etc/initramfs-tools/initramfs.conf` and set `BUSYBOX=y`
- Create a new ssh key for unlocking your encrypted volumes when it is rebooting
- `ssh-keygen -t rsa -b 4096 -f .ssh/dropbear`
- Create the needed folders for dropbear keys
- `mkdir -p /etc/initramfs-tools/root/.ssh/`
- `vi /etc/initramfs-tools/root/.ssh/authorized_keys`
- Paste your pub key `.ssh/dropbear.pub` in there
- reboot again to the rescue system via the hetzner webinterface

### Rescue image the second
*This steps should be done after the initial md replication*
(get the progress with `cat /proc/mdstat`)

We now rsync our installation into the new encrypted drives 

- `mkdir /oldroot/`
- `mount /dev/mapper/vg0-root /mnt/`
- `mount /dev/mapper/vg0-home /mnt/home`
- `mount /dev/mapper/vg0-log /mnt/var/log`
- `rsync -a /mnt/ /oldroot/` (this could take a while)
- `umount /mnt/home/`
- `umount /mnt/var/log/`
- `umount /mnt/`

Backup your old vg0 configuration to keep things simple and remove the old volume group:

- `vgcfgbackup vg0 -f vg0.freespace`
- `vgremove vg0`

After this, we encrypt our raid 1 now.
- `cryptsetup --cipher aes-xts-plain64 --key-size 256 --hash sha256 --iter-time 6000 luksFormat /dev/md1`
(!!!Choose a strong passphrase (something like `pwgen 64 1`)!!!)
- `cryptsetup luksOpen /dev/md1 cryptroot`
- now create the physical volume on your mapper:
- `pvcreate /dev/mapper/cryptroot`

We have now to edit your vg0 backup:
- `blkid /dev/mapper/cryptroot`
   Results in:  `/dev/mapper/cryptroot: UUID="HEZqC9-zqfG-HTFC-PK1b-Qd2I-YxVa-QJt7xQ"`
- `cp vg0.freespace /etc/lvm/backup/vg`
Now edit the `id` (UUID from above) and `device` (/dev/mapper/cryptroot) property in the file according to our installation
- `vi /etc/lvm/backup/vg0`
- Restore the vgconfig: `vgcfgrestore vg0`
- `vgchange -a y vg0`

Ok, the filesystem is missing, lets create it:

- `mkfs.ext4 /dev/vg0/root`
- `mkfs.ext4 /dev/vg0/log`
- `mkfs.ext4 /dev/vg0/home`
- `mkswap /dev/vg0/swap`

Now we mount and copy our installation back on the new lvs:

- `mount /dev/vg0/root /mnt/`
- `mkdir /mnt/home /mnt/var /mnt/var/log`
- `mount /dev/vg0/log /mnt/var/log/`
- `mount /dev/vg0/home /mnt/home`
- `rsync -a /oldroot/ /mnt/`

### Some changes to your existing linux installation
Lets mount some special filesystems for chroot usage:
- `mount /dev/md0 /mnt/boot`
- `mount --bind /dev /mnt/dev`
- `mount --bind /sys /mnt/sys`
- `mount --bind /proc /mnt/proc`
- `chroot /mnt`

To let the system know there is a new crypto device we need to edit the cryptab(/etc/crypttab):
- `vi /etc/crypttab`
- copy the following line in there: `cryptroot /dev/md1 none luks`

Regenerate the initramfs:
- `update-initramfs -u`
- `update-grub`
- `grub-install /dev/sda`
- `grub-install /dev/sdb`

To be sure the network interface is coming up:

- `vi /etc/rc.local`
- Insert the line: `/sbin/ifdown --force eth0`
- Insert the line: `/sbin/ifup --force eth0`

Time for our first reboot.. fingers crossed!

- `exit`
- `umount /mnt/boot /mnt/home /mnt/var/log /mnt/proc /mnt/sys /mnt/dev`
- `umount /mnt`
- `sync`
- `reboot`

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

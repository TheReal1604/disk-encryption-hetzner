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
- install a minimal Debian 9.4 with hetzners "installimage" skript (https://wiki.hetzner.de/index.php/Installimage)
- I chose the default raid1/lvm setup to keep it simple:

```
cat configs/simple-debian64-raid-lvm
installimage -c configs/simple-debian64-raid-lvm
```
- let the installation process complete
- reboot and ssh into your fresh installed system

### First steps on your fresh Debian installation

- connect via ssh-key you chose before for the rescue image (attention to the .ssh/known_hosts file..)
- edit `/etc/ssh/sshd_config` and set `#PasswordAuthentication yes` to `PasswordAuthentication no
- change your root password: `passwd`
- update and install dropbear-initramfs
  - `apt update && apt upgrade && apt install dropbear-initramfs`
- Create a new ssh key on your _local_ computer for unlocking your encrypted volumes when it is rebooting
  - `ssh-keygen -t ecdsa -f .ssh/id_dropbear`
- Create a new ssh config for the unlocking process
  - edit on your _local_ computer: `.ssh/config`:
  
 ```
 host myserver-unlock
   HostName <IP Adress of your Server>
   IdentityFile ~/.ssh/id_dropbear
   User root
   UserKnownHostsFile ~/.ssh/known_hosts.initramfs
 ```
 
- Back on the server create the needed folders for dropbear keys
- Create and edit `/etc/dropbear-initramfs/authorized_keys`:

```
no-port-forwarding,no-agent-forwarding,no-X11-forwarding,command="/bin/cryptroot-unlock" <paste the public key of your local computer here (.ssh/id_dropbear)>
```

- Edit `/etc/dropbear-initramfs/config` and change this line `#DROPBEAR_OPTIONS=` to this `DROPBEAR_OPTIONS='-c cryptroot-unlock'`
- run `update-initramfs -u`
- Wait for the initial md replication to finish. Get the progress with `cat /proc/mdstat`
- reboot again to the rescue system via the hetzner webinterface

### Rescue image the second
*This steps should be done after the initial md replication*
(get the progress with `cat /proc/mdstat`)

We now rsync our installation into the new encrypted drives 

- `mkdir /oldroot/`
- `mount /dev/mapper/vg0-root /mnt/`
- `rsync -av /mnt/ /oldroot/` (this can take a while)
- `umount /mnt/`

Backup your old vg0 configuration to keep things simple and remove the old volume group:

- `vgcfgbackup vg0 -f vg0.freespace`
- `vgremove vg0`

After this, we encrypt our raid 1 now.
- `cryptsetup luksFormat /dev/md1`
  - debian defaults to strong encryption, so the default values are just fine.
  - (!!!Choose a strong passphrase (something like `pwgen 64 1`)!!!)
- `cryptsetup luksOpen /dev/md1 cryptroot`
- now create the physical volume on your mapper:
- `pvcreate /dev/mapper/cryptroot`

We have now to edit your vg0 backup:

- Get the UUID with: `blkid /dev/mapper/cryptroot`
   Results in something like:  `/dev/mapper/cryptroot: UUID="HEZqC9-zqfG-HTFC-PK1b-Qd2I-YxVa-QJt7xQ" [...]`
- Copy the current vg-config: `cp vg0.freespace /etc/lvm/backup/vg0`
- Edit the configfile accordingly:
```
vg0 {
   ...
   physical_volumes {
      id = "UUID aquired with the blkid command"
      device = "/dev/mapper/cryptroot"
      ...
   }
}
```
- Restore the vgconfig using: `vgcfgrestore vg0`
- Apply changes: `vgchange -a y vg0`


Ok, the filesystem is missing, lets create it:

- `mkfs.ext4 /dev/vg0/root`
- `mkswap /dev/vg0/swap`

Now we mount and copy our installation back on the new lvs:

- `mount /dev/vg0/root /mnt/`
- `rsync -av    /oldroot/ /mnt/`

### Some changes to your existing linux installation
Lets mount some special filesystems for chroot usage:
- `mkdir -p /mnt/run/udev`
- `mount /dev/md0 /mnt/boot`
- `mount --bind /dev /mnt/dev`
- `mount --bind /sys /mnt/sys`
- `mount --bind /proc /mnt/proc`
- `mount --bind /run/udev /mnt/run/udev`
- `chroot /mnt`

To let the system know there is a new crypto device we need to edit `/etc/crypttab`:

```
cryptroot /dev/md1 none luks,initramfs`
```

NOTE: LVM may need a config change before below `grub` commands work. In this case try [these instructions](https://bbs.archlinux.org/viewtopic.php?pid=1513816#p1513816).

Regenerate the initramfs:
- `update-initramfs -u`
- `update-grub`
- `grub-install /dev/sda` or `grub-install /dev/nvme0`
- `grub-install /dev/sdb` or `grub-install /dev/nvme1`

To be sure the network interface is coming up (may be needed):

- `vi /etc/rc.local`
- Insert the line: `/sbin/ifdown --force eth0`
- Insert the line: `/sbin/ifup --force eth0`

Time for our first reboot.. fingers crossed!

- `exit`
- `umount /mnt/boot /mnt/proc /mnt/sys /mnt/dev /mnt/run/udev`
- `umount /mnt`
- `sync`
- `reboot`

After a few seconds the dropbear ssh server is coming up on your system, connect to it and get greeted by the unlock prompt immediately:

- `ssh unlock-myserver>`
- You will get disconnected if it was successful. Reconnect again with your ususal configuration.
- Thats it.

## Sources:
Special thanks to the people who wrote already this guides:

- https://cryptsetup-team.pages.debian.net/cryptsetup/README.Debian.html#remotely-unlock-encrypted-rootfs
- https://dev.to/j6s/decrypting-boot-drives-remotely-using-dropbear-2hdf
- http://notes.sudo.is/RemoteDiskEncryption
- https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system

## Contribution

- PRs are very welcome or open an issue if something not works for you as described

## Comments
- Tested this guide on 06.11.2020 on my own hetzner system

## Rescue Mode
This guide helps you in the following cases:
- Access your encrypted data from rescue mode
- Chmod into your system to fix boot problems or reset root password (or fix whatever is messed up)

### Before we start
In my case, I've restarted the server approximately one year after I’ve set up the full disk encryption. Sadly, the I couldn't reach my server afterwards. Eventually, it turned out that the problem was not complicated, it was just the server which didn't listen on the ipv6 interface. To prevent you from such things a short list to check first:
- Is the ssh port maybe default?
- Is your DNS-record also an AAAA-record and the server does not listen on ipv6?

### Boot into rescue mode and mount partitions
```
#enable dm-crypt kernel module
modprobe dm-crypt 
#list all disks (just to get an overview)
fdisk -l
#create /dev/mapper/cryptroot (md0 is the boot partition)
cryptsetup luksOpen /dev/md1 cryptroot
#scan for vg volumes
vgscan
#set volume group active
vgchange -ay 
#list partitions
ls /dev/mapper/*
#mount root and boot partitions (this depends on your setup -> see command before)
mount /dev/mapper/vg0-root /mnt
mount /dev/md0 /mnt/boot/
#chmod into your filesystem
chmod /mnt
```
### Start hacking and good luck (don't forget to unmount when you are done)

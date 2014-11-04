# Assumptions 

Fresh install of Shibby TomatoUSB

# Partition & Format USB drive

Plug in your USB drive and connect to your router via SSH (Putty) and execute the following commands:

    umount /dev/sda1
    fdisk /dev/sda

Type in the following commands to create a primary partition on your USB Flash drive

```sh
p               # list current partitions
o               # to delete all partitions
n               # new partition
p               # primary partition
1 (one)         # first partition
<press enter>   # default start block
<press enter>   # default end block #use the whole flash drive
w               # write new partition to disk
```

Format newly created partion & label disk as 'optware' (case sensitive)

```sh
umount /dev/sda1
mke2fs -j -L optware /dev/sda1

# mount partition as /opt
mount /dev/sda1 /opt

# Make sure /opt is properly mounted on a reboot.
echo "LABEL=optware /opt ext2 defaults 1 1" >> /etc/fstab
nvram setfile2nvram /etc/fstab 
nvram commit
```

# Install Optware

```sh
cd /tmp
wget http://tomatousb.org/local--files/tut:optware-installation/optware-install.sh -O - | tr -d '\r' > /tmp/optware-install.sh
chmod +x /tmp/optware-install.sh
sh /tmp/optware-install.sh
```


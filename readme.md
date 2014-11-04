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

# Install packages: CUPS, Avahi, Printer support

```sh
cd /opt
ipkg install wget-ssl

# download & unzip required files
/opt/bin/wget https://dl.dropbox.com/u/1015928/ddwrt/airprint-materials/TomatoUSB-DDWRT_Airprint-Cloudprint_support-03282013c.tar.gz --no-check-certificate
tar zxvf TomatoUSB-DDWRT_Airprint-Cloudprint_support-03282013c.tar.gz
cd TomatoUSB-DDWRT_Airprint-Cloudprint_support-03282013c/

# install base & supporting packages (this will take some time)
ipkg-opt install cups_1.5.4-2_mipsel.ipk poppler_0.12.4-1_mipsel.ipk ghostscript_8.71-3_mipsel.ipk dbus_1.2.16-2_mipsel.ipk avahi_0.6.30-2_mipsel.ipk

# install printer support packages
ipkg-opt install hplip_3.11.10-1_mipsel.ipk cups-driver-gutenprint_5.2.9-1_mipsel.ipk  gutenprint_5.2.9-1_mipsel.ipk

# install utilities
ipkg-opt install nano py26-lxml perl coreutils 
```

# Configure Avahi

Add users for avahi & dbus

```sh
echo "netdev:x:1:" >> /tmp/etc/group 
echo "avahi:x:2:" >> /tmp/etc/group 
echo "avahi:x:2:2:avahi daemon:/opt/sbin/avahi-daemon:/bin/false" >> /tmp/etc/passwd
```

*Note: enable this for subsequent reboots, also, by adding it to your startup script in Admin:Scripts:Firewall
but make sure it happens _after_ your opt/ partition mounts, but _before_ the initscripts at
/opt/etc/init.d/Snn  are executed.*

Start daemons:

```sh
/opt/etc/init.d/S20dbus start
/opt/etc/init.d/S22avahi-daemon start
/opt/etc/init.d/S88cupsd start
```

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
ipkg-opt install nano py26-lxml perl coreutils py26-cups_1.9.62-1_mipsel.ipk
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

## Start daemons:

```sh
/opt/etc/init.d/S20dbus start
/opt/etc/init.d/S22avahi-daemon start
/opt/etc/init.d/S88cupsd start
```

Confirm they are running

	ps | tail

TEST:  Load CUPS Webif via web browser. See if you can load the webif @  your_router's_ip:631

## Install Printer

B.  Configure USB printer, configure via CUPS as an AppSocket/HP JetDirect printer

Go to your_router's_ip:631, and click on the Administration tab, then choose Add Printer

1. Enter a URI for your USB printer that will direct CUPS to treat it as a port 9100 printer.
I've used both of these with success.  Note that this _does_ work with non-jet-direct printers.
URI:  socket://127.0.0.1:9100    or     URI:   file:///dev/usb/lp0

2.  Name, Description and Location as fit your needs.  ***Make sure you check the box to "Share This Printer"***
3. Choose your Make (press Continue), and then your Model (press Add Printer) (note that the Model field make take a minute to fully populate if you've chose a 'popular' Make.)
4.   Set any default policies (I chose stop-printer) 
5.  TEST:  ... by printing a Test Page (Maintenance -> Print Test Page)

*Troubleshooting: if multiple drivers are available for your printer, try another one...*


# Create Avahi Service to advertise for Desktop/Laptop printing

create a generic p910nd service file, supply it with your own values

	nano  /opt/etc/avahi/services/p9100-printer.service

#  enter this as the content for the service file: accurate information will auto-identify your printer's make and model

```xml
<service-group>
	<name replace-wildcards="yes"> my-printers-name @ my-host-name </name>
	<service protocol="any">
		<type>_pdl-datastream._tcp</type>
		<port>9100</port>
		<txt-record>product=( my-printers-make-and-model-according-to-CUPS )</txt-record>
	</service>
</service-group>
```

For my HP Laserjet 1010 attached to a router with hostname Router, it looks just like this:

```xml
<service-group>
	<name replace-wildcards="yes"> HPLaserJet-1010 @ Router:9100 </name>
	<service protocol="any">
		<type>_pdl-datastream._tcp</type>
		<port>9100</port>
		<txt-record>product=(HP LaserJet 1010)</txt-record>
	</service>
</service-group>
```

*The product= txt-record is important to have correct, so that any computer attaching via Bonjour can autoconfigure the drivers.*

reload avahi-daemon so that the new service files are recognized
	
	/opt/etc/init.d/S22avahi-daemon reload

TEST:  Choose printer via Bonjour/mDNS/Avahi-enabled  system, and print a test page for it. 

## Install AirPrint Support

add mime info to make it compatible with Apple's AirPrint requirements

	echo "image/urf application/pdf 100 pdftoraster" > /opt/share/cups/mime/airprint.convs
	echo "image/urf urf string(0,UNIRAST<00>)" > /opt/share/cups/mime/airprint.types

restart CUPS  **{absolute MUST!}**

	/opt/etc/init.d/S88cupsd restart


We use TJFontaine's python script to read CUPS's parameters of the printer and 
turn it into a service file for Airprint:

I've included a copy of it in the support tarball

```sh
cd TomatoUSB-DDWRT_Airprint-Cloudprint_support-03282013c/
python2.6  airprint-generate.py
mv *.service /opt/etc/avahi/services/
```

Execution of the airprint-generate.py script is normally nonverbose : it should -not- create any screen output.  If it does
then there is problem somewhere, most likely.

Reload avahi-daemon to make it advertise your AirPrint service.

	/opt/etc/init.d/S22avahi-daemon reload

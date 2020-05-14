# pxe-install

This project is dedicated to install Ubuntu 18.04 Server LTS via PXE.

In a sandbox approach two Virtual Machines with Ubuntu 18.04 Server were used. First an internal network was established between to two machines to test proper DHCP behavior.

A secondary network interface was created for internet access by NAT. This is only important for the PXE-Master VM.

IP configuration:

```
PXE-Master 192.168.1.1
PXE-Client 192.168.1.10-15
```


We will go through step-by-step what needs to be done on the PXE-Master.

# Step-by-step guide

## Netplan

Since we have two interfaces in out VM (one for internal VM-to-VM networking, and one NAT device) we need to configure them.

Check the interfaces with `sudo ip link` and make note of the names.

Then open the configuration file for netplan.

`sudo nano /etc/netplan/50-cloud-init.yaml`

Here you need to add a static IP for the internal networking interface since here we  will serve our DHCP server.

```
network:
    ethernets:
        enp0s3:
            dhcp4: true
        enp0s8:
            dhcp4: no
            addresses: [192.168.1.1/24]
    version: 2
```

Here we have chosen the 192.168.1.1 as IP for the DHCP Server...

## DNSMASQ

Next we need to install dnsmasq. This will be our DHCP server with PXE boot functionalities.

`sudo apt update && sudo apt install -y dnsmasq`

Afterwards we edit the configuration file. Be sure to make a backup if you have custom settings.

`sudo nano /etc/dnsmasq.conf`

If this is a fresh install just add the following lines at the bottom:

```
interface=enp0s8
bind-interfaces
domain=pxemaster.local

dhcp-range=enp0s8,192.168.1.10,192.168.1.20,255.255.255.0,1h
dhcp-option=option:router,192.168.1.1
dhcp-option=option:dns-server,192.168.1.1
dhcp-option=option:dns-server,8.8.8.8
#dhcp-host=00:50:56:39:87:7A,192.168.1.5,intnet

enable-tftp
tftp-root=/srv/tftp
dhcp-boot=pxelinux.0,pxemaster-s20,192.158.1.1
pxe-prompt="Press F8 for PXE Network boot.", 2
pxe-service=x86PC, "Install Ubuntu 18.04 Server from 192.168.1.1 via PXE",pxelinux
```

Be sure to select the correct interface! The DHCP option adjust the range and gateway settings for devices that get their IP from this server.
If you are planning to use this for a single system it would be advisable to limit the leases to certain MAC-addresses. For our sandbox VM exmaple it wont make any difference.

Now the important part is that dnsmasq will also serve a TFTP server. This will be used by PXE to serve the resources requried for network boot.
The other points can be changed to your liking.

After you saved the edit we need to create the serving folder for TFTP.

`sudo mkdir -p /srv/tftp`

Now we can start the dnsmasq service:

`sudo systemctl restart dnsmasq`

and check  the status:

`sudo systemctl status dnsmasq`

Ideally, it should be "active". Now it is a good time to check the proper function of the DHCP.

On our second machine, PXE-Client, we need to set the netplan settings of the internal network interface to `dhcp4: true`. After a reboot,
both VMs should be able to see each other in the 192.168.1.X netrange. If this does not work you have issues with the DHCP.

Hint: Check the correct network settings with your VM manager first. I used VirtualBox.

## TFTP

Now we need to populate the TFTP to be able to boot from network. For this, we download the netboot image from Ubuntu.

```
cd /tmp
wget http://archive.ubuntu.com/ubuntu/dists/bionic-updates/main/installer-amd64/current/images/netboot/netboot.tar.gz
sudo tar -xvzf netboot.tar.gz -C /srv/tftp/
sudo chown -R nobody:nogroup /srv/tftp/
```

This will provide us netbooting capabilities. The pxelinux.0 config will already allow the netboot. You can check this if you reboot the client VM with the boot order: Network boot.
Ideally, there should be a ubuntu splash screen with the netboot install option.

## Apache

At this point it would be possible to perform a network install from the minimal netboot image. However if we want to customize our install and use local resources and configurations we need to serve the OS image that we want to install.

It is possible the serve these by HTTP or NFS. NFS seems to be the more current approach but the HTTP approach was already working for me.

First, we need to install Apache Webserver:

`sudo apt install apache2`

Check the status with:

`sudo systemctl start apache2 && sudo systemctl status apache2`

Secondly, we download the ISO image for Ubuntu 18.04 Server:

`wget https://releases.ubuntu.com/18.04.4/ubuntu-18.04.4-live-server-amd64.iso`

To copy all the files we mount the ISO file:

`sudo mount -o loop ubuntu-18.04.4-live-server-amd64.iso /mnt/`

And lastly, copy the content to the HTML root:

`sudo cp -r /mnt/* /var/www/html/ubuntu-iso/`

Now to use this custom install configuration we need to provide a so-called "kickstart" file. In the simplest case it merely tells PXE where to find the correct resources:

`nano /var/www/html/ks.cfg`

And put the following content into it:

```
install
url --url http://192.168.1.1/ubuntu-iso/
```

PS: Make sure that all the files are readable in the /var/www folders. PPS: The kickstart configuration file ks.cfg can be used to fully automate your installation. More infos: https://help.ubuntu.com/community/KickstartCompatibility.

Finally, we change the PXE config to account for our newly created ks.cfg file.

## PXE Config

In the netboot folder that is being served with TFTP some adjustments are required. For this we open the default config (remember the dnsmasq.conf file).

`sudo nano /srv/tftp/pxelinux.cfg/default`

This file should already has some content. Append at the end the following:

```
label linux1804
    menu default
    menu label ^Install Ubuntu 18.04 Server from 192.168.1.1
    kernel ubuntu-installer/amd64/linux
    append ks=http://192.168.1.1/ks.cfg vga=791 initrd=ubuntu-installer/amd64/initrd.gz ramdisk_size=16432 root=/dev/rd/0 rw --
```

This is a good point to check the internet for other options. For example you can change the timeout value to 1 if you would like to automatically install the OS.
In the append part of the config we added our config file. You can either add more configurations there or provide a so-called "preseed" file to more or less achieve an unattended installation.

If you now restart your PXE-Client the system should automatically try to install Ubuntu 18.04 Server. It will not be unattended since we only provided a minimal config.
All the relevant config files can be found in this repo and in the future we will extend the kickstart config file.

This guide was made with the help of multiple website which all of which lacked some details but together provided the required infos. Please check for further input.

----------

# Resources:

https://www.tecmint.com/install-ubuntu-via-pxe-server-using-local-dvd-sources/

https://linuxhint.com/pxe_boot_ubuntu_server/

https://graspingtech.com/network-install-ubuntu-18-04/

https://help.ubuntu.com/community/PXEInstallServer

https://help.ubuntu.com/community/KickstartCompatibility

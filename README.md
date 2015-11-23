Running CoreOS on XEN VM
=====================================

An example of setting up CoreOS with cloud-config on a XEN host


##Preparing DHCP on the XEN host

DHCP is a convenient way of assigning static IP to the CoreOS vm by MAC address in DHCP and the XEN configuration.

I've used isc-dhcp-server here.

Use locally administered address ranges for your MAC address (http://serverfault.com/a/40720/292243)

Example /etc/dhcp/dhcpd.conf
``` 
subnet 10.0.0.0 netmask 255.255.255.0 {
  range 10.0.0.100 10.0.0.200;
  option domain-name-servers 8.8.8.8;
  option routers 10.0.0.254;
  option broadcast-address 10.0.0.255;
  default-lease-time 600;
  max-lease-time 7200;
}

host coreos {
  hardware ethernet 06:0:0:0:0:01;
  fixed-address 10.0.0.111;
}
```

If you’re setting up DHCP from scratch make sure you set the right interface:
Example /etc/default/isc-dhcp-server
```
INTERFACES="dummy0"
```

##Preparing the XEN VM config

XEN config file:

```
import os, re
arch = os.uname()[4]
if re.search('64', arch):
    arch_libdir = 'lib64'
else:
    arch_libdir = 'lib'

memory = 4096
shadow_memory = 8
name = "coreos"

bootloader = "/usr/local/lib/xen/bin/pygrub"
kernel = "/coreos/vmlinuz"
ramdisk = "/coreos/cpio.gz"
extra = " coreos.autologin cloud-config-url=https://gist.githubusercontent.com/dvdtoth/16a8977b3e4575031c91/raw/c5d8b0e0b4ac8af8f58f78b3ed2bda1d498aeb7d/cloud-config console=tty0"

vcpus = 4

vif = [ 'mac=06:00:00:00:00:01,bridge=dummy0,type=e1000' ]

disk        = [ 'phy:/dev/your-disk,hda,w', 'file:/root/coreos_production_iso_image.iso,hdb:cdrom,r' ]

device_model = '/usr/' + arch_libdir + '/xen-4.0/bin/qemu-dm'
# boot on floppy (a), hard disk (c) or CD-ROM (d)
# default: hard disk, cd-rom, floppy
vfb        = ['type=vnc,vnclisten=0.0.0.0,vncunused=1,vncdisplay=1,vncpasswd=yourpass']

boot="cd"
acpi = 1
apic = 1
sdl=0
stdvga=0
serial='pty'
usb = 1
usbdevice='tablet'
```

##Preparing CoreOS kernel

Grab the coreos iso and make the kernel available to the xen hypervisor.
At the time of writing this, the stable is CoreOS 766.5.0. We will use CoreOS 835.5.0 from the beta channel, you must use a CoreOS version 773.1.0+ for the kubelet to be present in the image.

```
$ wget http://beta.release.core-os.net/amd64-usr/current/coreos_production_iso_image.iso
```

Take the kernel from the image

```
$ mkdir /mnt/coreos && mount -o loop coreos_production_iso_image.iso /mnt/coreos
$ mkdir /coreos && cp /mnt/coreos/coreos/* /coreos
```

Create the cloud-config file, add any public keys you need to SSH access the vm with.
Make it available to the host. I used gist for this:
https://gist.githubusercontent.com/dvdtoth/16a8977b3e4575031c91/raw/c5d8b0e0b4ac8af8f58f78b3ed2bda1d498aeb7d/cloud-config

```
#cloud-config

    hostname: coreos

    coreos:
      etcd2:
        advertise-client-urls: "http://$public_ipv4:2379"
        listen-client-urls: "http://0.0.0.0:2379"
      units:
        - name: etcd2.service
          command: start
        - name: fleet.service
          command: start
        - name: static.network
          content: |
            [Match]
            Name=eth0

            [Network]
            Address=10.0.0.111/24
            Gateway=10.0.0.254
            DNS=8.8.8.8
    users:
      - name: core
        ssh-authorized-keys:
          - "ssh-rsa AAAAB3NzaC1yc2EAAAAD..."
          - "ssh-rsa AAAAB3NzaC1yc2EAAAAB..."

      - groups:
          - sudo
          - docker 
```

##Start the new vm

```
$ xm create /etc/xen/coreos.cfg
```
Connect via SSH or VNC to debug.


##Refs
Fork me on github: https://github.com/dvdtoth/coreos-xen/
How to make CoreOS pxe https://gist.github.com/nyarla/7319229

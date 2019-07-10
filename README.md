# buster-image

## Description

In this repository you will find all the steps and extra files used to create the buster image for the B3 platform.

Given the fact that image creation is not done every day there is no scripted/automated method to create it ; instead all the steps needed (and taken) to create the released image will be described in this README. It will be updated with each release.

The current version for the excito buster image is **1.0**

## Contents

- This README.md file, the "image create manual"
- `first-boot` which contains first-boot scripts run ... on the first boot; Currently these scripts:
  - Generate new ssh host keys
  - Remove themselves

## Image creation

### Pre-requisites

In order to create an image you need a B3 running whatever standard Linux OS you wish, ~1GiB free space and an internet connection; The only software needed on the host is the debootstrap package.

The released image are created from the [install/rescue system](https://github.com/Excito/buildroot) which provides all the necessary tools.

Everything must be run as `root`.

### Boostraping and chroot into the system

- Create a directory that will host the image (it doesn't have to be a mounted partition):
```
mkdir /mnt/target
```
- Debootstrap stretch:
```
debootstrap --arch=armel buster /mnt/target 
```
- Mount the kernel filesystems:
```
mount -t proc none /mnt/target/proc
mount -t sysfs none /mnt/target/sys
mount -o bind /dev /mnt/target/dev
mount -t devpts none /mnt/target/dev/pts
mount -t tmpfs none /mnt/target/dev/shm
```
- Create the `policy-rc.d` file which will prevent the daemons to be run inside the chroot:
```
cat > /mnt/target/usr/sbin/policy-rc.d << EOF
#!/bin/sh
exit 101
EOF
chmod 755 /mnt/target/usr/sbin/policy-rc.d
```
- Chroot into the system and setup the environment:
```
chroot /mnt/target /bin/bash
source /etc/profile
cd /root
export PS1="(chroot) $PS1"
```

### APT configuration and standard package install

- Create the `/etc/mtab` link:
```
ln -s /proc/mounts /etc/mtab
```
- Add security repository to `sources.list`:
```
cat >> /etc/apt/sources.list << EOF
deb http://security.debian.org/ buster/updates main
EOF
```
- Download and install the `excito-release-buster` package:
```
wget -q http://repo.excito.org/excito-release-buster.deb
dpkg -i excito-release-buster.deb
rm excito-release-buster.deb
```
- Update apt cache and ugprade the system:
```
apt-get update
apt-get -y dist-upgrade
```
- Install kernel and b3 utils:
```
apt-get -y install bubba3-kernel b3-utils
```
- Install locales and standard system tools:
```
apt-get -y install locales
tasksel install --new-install standard ssh-server
```
- [Optional] Install easyfind-client:
```
apt-get -y install easyfind-client
```

### [Optional] Install u-boot-tools to access u-boot configuration

Released images provide `fw_setenv` and `fw_printenv` which allows access and modification of the platform bootloader. These tools are only 'nice to have'. Beware that misuse can prevent the system from booting.

- Create the `fw_env.config` file:
```
cat > /etc/fw_env.config << EOF
# MTD definition for Bubba|3
# MTD device name       Device offset   Env. size       Flash sector size       Number of sectors
/dev/mtd1		0x000000	0x010000	0x010000
EOF
```
- Install the `u-boot-tools` and the `mtd-utils` package:
```
apt-get -y install u-boot-tools mtd-utils
```

### User and password creation

- Set the root password ('excito' without quotes on the released images):
```
passwd
```
- Create the `excito` user and set its password ('excito' without quotes on the released images):
```
useradd -m -U -s /bin/bash excito
passwd excito
```

### System configuration

- Reconfigure the exim4 mail server with the following answers:
`dpkg-reconfigure exim4-config`
  - General type of mail configuration: `local delivery only; not on a network`
  - System mail name: `b3` for the B3
  - IP-addresses to listen: `127.0.0.1 ; ::1`
  - Other destinations: *empty*
  - Keep number of DNS-queries minimal: `No`
  - Delivery method for local mail: `mbox format in /var/mail`
  - Split configuration into small files: `No`
  - Root and postmaster mail recipient: `excito`
- Configure the network:
```
cat >> /etc/network/interfaces << EOF

allow-hotplug eth0
iface eth0 inet dhcp

allow-hotplug eth1
iface eth1 inet dhcp
EOF
```
- Create `/etc/fstab`:
```
cat > /etc/fstab << EOF
/dev/sda1   /   ext3    noatime 0   1
EOF
```
- Set the hostname:
```
echo "b3" > /etc/hostname
```
### Cleanup
- Empty apt cache:
```
apt-get clean
```
- Remove previously created ssh keys (they will be recreated by the `first-boot` files):
```
rm /etc/ssh/*key*
```
- Exit the chroot:
```
exit
```
- Remove the shell history file and the previously created `policy-rc.d` file:
```
rm /mnt/target/usr/sbin/policy-rc.d /mnt/target/root/.bash_history
```
- Unmount kernel filesystem:
```
umount /mnt/target/dev/shm
umount /mnt/target/dev/pts
umount /mnt/target/dev
umount /mnt/target/sys
umount /mnt/target/proc
```

### `first-boot` files and tarball creation ###
- Download and extract the first-boot release tarball into target:
```
wget -O/mnt/target/first-boot.txz https://github.com/Excito/stretch-image/releases/download/v1.0/first-boot-1.0.txz
( cd /mnt/target; tar -xvf first-boot.txz )
rm /mnt/target/first-boot.txz
```
- Now that the image files are ready, go ahead and create the final tarball:
```
( cd /mnt/target; tar -cJvf /root/stretch-image.txz .)
```

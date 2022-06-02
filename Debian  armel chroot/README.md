# Creating Debian chroot rootfs

Guide to creating a Debian chroot rootfs image using debootrap for armel devices running old kernels (3.0.x). This is done from an x86_64 host running Debian (using qemu).

Note that Debian Jessie is the most recent release that supports a 3.0 kernel.

TODO: look into grml-debootstrap (maybe simplifies things a little?)

## Basic steps

This is not a script, run these one block at a time.

```bash
# define name of image
export IMAGE_NAME='jessie'
export IMAGE_SIZE=500 # MB

# prerequisites
sudo apt install debootstrap
sudo apt install qemu-user-static

# create and mount image
dd if=/dev/zero of=${IMAGE_NAME}.img bs=1M count=$IMAGE_SIZE status=progress 
mkfs.ext2 ${IMAGE_NAME}.img 
mkdir ${IMAGE_NAME}
sudo mount -o loop ${IMAGE_NAME}.img ${IMAGE_NAME}

# first step of rootfs
sudo debootstrap --verbose --include=less,openssh-server,locales,wget --variant=minbase --arch=armel --foreign jessie ${IMAGE_NAME}  

# enter the chroot
sudo mount -t proc proc ${IMAGE_NAME}/proc/
sudo mount --rbind /sys ${IMAGE_NAME}/sys/
sudo mount --rbind /dev ${IMAGE_NAME}/dev/

sudo cp /usr/bin/qemu-arm-static ${IMAGE_NAME}/usr/bin/
sudo chroot ${IMAGE_NAME} /usr/bin/qemu-arm-static /bin/bash --login

# do second step *within chroot*
export DEBOOTSTRAP_DIR=/debootstrap
/debootstrap/debootstrap --second-stage 

echo $(cat /etc/hostname)-chroot > /etc/hostname

echo 'LANG=en_US.UTF-8' >> /etc/profile
echo 'LANGUAGE=en_US.UTF-8' >> /etc/profile
echo 'LC_ALL=en_US.UTF-8' >> /etc/profile
locale-gen en_US.UTF-8 # not sure if needed
export LANGUAGE=en_US.UTF-8
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8

dpkg-reconfigure locales

rm /usr/bin/qemu-arm-static # not needed on target

# also need to set password with passwd
# and configure ssh at /etc/ssh/sshd_config
# e.g. enable PermitRootLogin yes
```

# Installing ffmpeg
Debian Jessie unfortunately does not include ffmpeg, so we need to install it from another repo (within the chroot).

NOTE: ffmpeg takes a long time to start on the target. Seems to be related to /dev/urandom?

```bash
echo "deb http://archive.deb-multimedia.org/ jessie-backports main" > /tmp/multimedia.list
echo "deb http://archive.deb-multimedia.org/ jessie main non-free"  >> /tmp/multimedia.list
cp /tmp/multimedia.list /etc/apt/sources.list.d/

wget -P /tmp/ http://www.deb-multimedia.org/pool/main/d/deb-multimedia-keyring/deb-multimedia-keyring_2016.8.1_all.deb
dpkg -i tmp/deb-multimedia-keyring_2016.8.1_all.deb

apt update
apt install ffmpeg
apt-get clean # may need this to recover some space if image is small

```

# Shutting down chroot

FIXME: this doesn't work properly.

```bash
# exit and cleanup
sync
exit
sudo mount --rbind /dev ${IMAGE_NAME}
sudo mount --make-rslave ${IMAGE_NAME}
sudo umount -R ${IMAGE_NAME}
sudo umount ${IMAGE_NAME}.img
```

# Make image file
```bash
gzip ${IMAGE_NAME}.img
```

# Starting chroot (on target)
```bash
export IMAGE_NAME='jessie'

sudo mount -o loop ${IMAGE_NAME}.img ${IMAGE_NAME}

sudo mount -t proc proc ${IMAGE_NAME}/proc/
sudo mount --rbind /sys ${IMAGE_NAME}/sys/
sudo mount --rbind /dev ${IMAGE_NAME}/dev/

sudo chroot ${IMAGE_NAME} /bin/bash --login
```

# Creating Debian chroot rootfs

Guide to creating a Debian chroot rootfs image using debootrap for armel devices running old kernels (3.0.x). This is done from an x86_64 host running Debian (using qemu).

Note that Debian Jessie is the most recent release that supports a 3.0 kernel.

TODO: look into grml-debootstrap (maybe simplifies things a little?)

## Steps

This is not a script, run these one block at a time.

WARNING: not full tested

```bash
# define name of image
export IMAGE_NAME='jessie'
export IMAGE_SIZE=400 # MB

# prerequisites
sudo apt install debootstrap
sudo apt install qemu-user-static

# create and mount image
dd if=/dev/zero of=${IMAGE_NAME}.img bs=1M count=$IMAGE_SIZE status=progress 
mkfs.ext2 ${IMAGE_NAME}.img 
mkdir ${IMAGE_NAME}
sudo mount -o loop ${IMAGE_NAME}.img ${IMAGE_NAME}

# first step of rootfs
sudo debootstrap  --verbose --include=less,openssh-server,wget --variant=minbase --arch=armel --foreign ${IMAGE_NAME}  

# enter the chroot
sudo mount -t proc proc ${IMAGE_NAME}/proc/
sudo mount --rbind /sys ${IMAGE_NAME}/sys/
sudo mount --rbind /dev ${IMAGE_NAME}/dev/

sudo bash -c "echo 'LANG=en_US.UTF-8' >> /${IMAGE_NAME}/etc/profile"
sudo bash -c "echo 'LANGUAGE=en_US.UTF-8' >> /${IMAGE_NAME}/etc/profile"
sudo bash -c "echo 'LC_ALL=en_US.UTF-8' >> /${IMAGE_NAME}/etc/profile"

cp /usr/bin/qemu-arm-static img/usr/bin/
sudo chroot ${IMAGE_NAME} /usr/bin/qemu-arm-static /bin/bash --login

# do second step *within chroot*
export DEBOOTSTRAP_DIR=/debootstrap
/debootstrap/debootstrap --second-stage 

#apt install locales 
#dpkg-reconfigure locales

# optional: install ffmpeg
# note: Debian Jessie does not include ffmpeg, so need to use another repo
echo "deb http://archive.deb-multimedia.org/ jessie-backports main" > /tmp/multimedia.list
echo "deb http://archive.deb-multimedia.org/ jessie main non-free"  >> /tmp/multimedia.list
cp /tmp/multimedia.list /etc/apt/sources.list.d/

wget -P /tmp/ https://www.deb-multimedia.org/pool/main/d/deb-multimedia-keyring/deb-multimedia-keyring_2016.8.1_all.deb
dpkg -i tmp/deb-multimedia-keyring_2016.8.1_all.deb

apt update
apt install ffmpeg
apt-get clean

# exit and cleanup
sync
exit
sudo mount --rbind /dev ${IMAGE_NAME}
sudo mount --make-rslave ${IMAGE_NAME}
sudo umount -R ${IMAGE_NAME}
sudo umount ${IMAGE_NAME}

```

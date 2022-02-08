 
 # Doing upgrade of Centos 7 server to Centos 8.
 
 ## 1. Migrating backup of working system to virtual guest or another server.

Preparation of test image.
Boot from Centos 7 CD into troubleshoot mode and skip to shell.
Prepare disks of guest machine. Make partitions and filesystems.

 **Sda2 - boot - ext2** 
 
 **Sda3 - root - ext4**
 
 **Sda4 - swap**

 ```
sudo mount /dev/sda3 /mnt
sudo mount /dev/sda2 /mnt/boot
 ```
Extract backup with full path.
 ```
tar -zxvf backup.tar.gz -C /mnt

mkdir /mnt/dev
mkdir /mnt/proc
mkdir /mnt/sys
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys  /mnt/sys

chroot /mnt
 ```
Installing grub:
 ```
grub-install /dev/sda
  ```
Updating configs of grub to match UUIDs
  ```
grub2-mkconfig -output /boot/grub2/grub.cfg
  ```
/etc/fstab - changing lines we need
 
We can take needed UUIDs from blkid like
  ```
echo blkid | grep /dev/sda >> /etc/fstab
  ```
Then we should rebuild initramfs.
Booting to rescue mode.
 
And find latest initrams
  ```
dracut -f /boot/initramfs-3.10.0-1160.49.1.el7.x86_64.img 3.10.0-1160.49.1.el7.x86_64 
  ```
And adding “*edd=off console=tty0*” to kernel boot params.
 
Finally we can boot into our system.  


 ## 2. Upgrade Centos 8.

**_If you have a remote system, then you should have access to IPMI or something like this to be able to boot into rescue mode just in case.._**

Do your network settings in /etc/sysconfig/network-scripts.
First we have to install epel-release.
 ```
yum install rpmconf epel-release yum-utils
 ```
Agree to install new apps and libs, then keep current configs while updating saying No.

 ```rpmconf -a ```

Find out what packages installed not from repos and recommended to delete.
 ```
package-cleanup --orphans 
package-cleanup --leaves
 ```
In my case removing **mlnx-ofa_kernel** removed **qemu-kvm** app. 

Or you can use yum to autoremove but be careful and check the packages it will try to delete.
 ```
yum autoremove
 ```
Installing dnf packet manager - it works in Centos 8 by default.
 ```
yum install dnf
 ```
Removing yum
 ```
dnf remove yum yum-metadata-parser
rm -Rf /etc/yum
 ```
From January 2022 Centos 8 stopped supporting Centos 8 and mirror.centos.org doesn’t work anymore. So we should use archive version - vault.centos.org

Do upgrade to latest packages. 
 ```
 dnf upgrade
 ```

I had a lot of errors of python on kernel-devel but all installed successfully.

Clean cache and downloaded packages.
 ```
dnf clean all
 ```
Then we install centos-release and epel-release from Centos 8. Check that link is vault (archive version of packages), not mirror (current stream packages).
 ```
dnf install http://vault.centos.org/centos/8/BaseOS/x86_64/os/Packages/{centos-linux-repos-8-3.el8.noarch.rpm,centos-linux-release-8.5-1.2111.el8.noarch.rpm,centos-gpg-keys-8-3.el8.noarch.rpm}

dnf -y upgrade https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
 ```
And because Centos 8 no more supported you will get and error if you try to upgrade with current repos. You should use vault repos from Jan 2022.
 ```
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-Linux-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-Linux-*
 ```
One more time cleaning dnf cache.
 ```
dnf clean all

 ```
 Then we have to remove old kernels.
 ```
rpm -e `rpm -q kernel`
 ```
Here we have to remove conflicting packages. Mine was kmod-kvdo.
 ```
rpm -e --nodeps kmod-kvdo 
rpm -e vdo
 ```

**In some cases you can’t remove kernels from working OS, so you should boot into recovery mode and do it manually.**

1. Download kernel, kernel-core and kernel-modules, linux-firmware.
```
wget https://vault.centos.org/centos/8/BaseOS/x86_64/os/Packages/kernel-4.18.0-348.el8.x86_64.rpm

wget https://vault.centos.org/centos/8/BaseOS/x86_64/os/Packages/kernel-modules-4.18.0-348.el8.x86_64.rpm

wget https://vault.centos.org/centos/8/BaseOS/x86_64/os/Packages/kernel-core-4.18.0-348.el8.x86_64.rpm
```
2. Boot in rescue mode.
3. Remove existing linux-firmware package and install downloaded package.
```
dnf remove kexec-tools
dnf remove ivtv-firmware
dnf remove linux-firmware

4. Then Install kernel packages by (rpm -iv kernel*)
rpm -iv linux-firmware-20210702-103.gitd79c2677.el8.noarch.rpm
rpm -iv kernel-modules-4.18.0-348.el8.x86_64.rpm

/boot/symvers -> symlink /lib/moules/4.18…/symvers.gz

rpm -iv --replacepkgs kernel-core-4.18.0-348.el8.x86_64.rpm

rpm -e --nodeps sysvinit-tools
rpm -q kernel-3.10.. and soft that depends on it
 ```
5. Reboot



**Ok, we removed old kernels and we are ready for upgrade.**

```
dnf --releasever=8 --allowerasing --setopt=deltarpm=false distro-sync
 ```

I’ve got some dependency errors, so I have to remove some packages and install one.
 ```
wget http://mirror.centos.org/centos/8-stream/AppStream/x86_64/os/Packages/mariadb-connector-c-3.1.11-2.el8_3.x86_64.rpm
rpm -i –nodeps mariadb-connector-c-3.1.11-2.el8_3.x86_64.rpm

rpm -e dhclient
dnf remove dracut-network
rpm -e –nodeps sysvinit-tools
dnf  remove python36-rpmconf
 ```
Repeat
 ```
dnf --releasever=8 --allowerasing --setopt=deltarpm=false distro-sync
 ```
Install new kernel 
 ```
dnf -y install kernel-core
 ```
Finally, install CentOS 8 minimal package.
 ```
dnf -y groupupdate "Core" "Minimal Install"
 ```
Folder exists - yum package install conflict
 ```
rm -rf /etc/yum/protected.d 
 ```
Reboot to check everything is ok.


Finalising upgrade installing removed packages.
 ```
dnf install libvirt
dnf module install virt
dnf install virt-install virt-viewer libguestfs-tools

systemctl enable libvirtd.service
systemctl start libvirtd.service
 ```
Remove keys from ssh 
 ```
rm -rf /etc/ssh/*.key
rm -rf /etc/ssh/*.pub

systemctl enable sshd
systemctl start sshd
 ```
Update ssh keys 
 ```
ssh-keygen -f /home/fire/.ssh/known_hosts -R "[192.168.1.1]:22"
 ```
Update HP repo from 7.3-7.4 to 8.4-8.5. Change values in /etc/yum.repos.d/hp.repo

spp RHEL repo from 7.2 to 8.5

mcp centos repo from 7.3 to 8.4

Update hp packages.
 ```
dnf update —allowerasing
 ```

## 3. Now we are ready to migrate to Centos 8 Stream or Rocky Linux.

Upgrade to Stream:
 ```
sudo dnf update
sudo dnf in centos-release-stream
sudo dnf swap centos-linux-repos centos-stream-repos
sudo dnf distro-sync
sudo systemctl reboot
 ```

Migrate to Rocky Linux.
 ```
curl https://raw.githubusercontent.com/rocky-linux/rocky-tools/main/migrate2rocky/migrate2rocky.sh -o migrate2rocky.sh

chmod u+x migrate2rocky.sh
 ```
And now, at long last, execute the script:
 ```
./migrate2rocky.sh -r
 ```


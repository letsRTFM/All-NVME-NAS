# Videos

- [AR900i VS BD790i - Performance and Power Usage](https://www.youtube.com/watch?v=UALDaaPusxE)
- [AR900i All NVME NAS Build - ALPINE LINUX + ZFS](TBD)

## Hardware

- Power Supply: [HDPLEX 500W GaN](https://hdplex.com/hdplex-500w-gan-aio-atx-power-supply.html)
- Motherboard: [AR900i](https://store.minisforum.com/products/minisforum-ar900i)
- CPU: [Intel Core i9-13900HX 5.2 GHz - 24 Core 32 Threads](https://browser.geekbench.com/v6/cpu/4874137) *Built into motherboard
- RAM: [Crucial 5600 96GB Kit](https://amzn.to/49RVpaD) *Affiliate Link
- Case: [DAN A4 - H20](https://amzn.to/49JtkTj) *Affiliate Link
- Storage: 6 x [Crucial P3 Plus 4TB](https://amzn.to/3Uc0iVs) *Affiliate Link
- Cooling: 
  - 1 x [Noctua NF-A12x15](https://amzn.to/3uNb0Ju) *Affiliate Link
  - 2 x [Noctua NF-P12 redux](https://amzn.to/3w0QUMA) *Affiliate Link
- Screws:
  - M2.5 [Screws for Minisforum CPU Fan Bracket](https://amzn.to/4djLXPE) *Affiliate Link
  - (Optional) [Misc Assorted PC Screws](https://amzn.to/4b4fPxQ) *Affiliate Link


## Software

- Bootloader: [ZFSBootMenu](https://docs.zfsbootmenu.org/en/v2.3.x/)
- Storage: 2 x [ZFS 6 drive raidz2](https://github.com/openzfs/zfs) (Configured on partitions instead of whole drives)
- OS: [Alpine Linux](https://docs.zfsbootmenu.org/en/v2.3.x/guides/alpine/uefi.html) (Installed on ZFS Pool) 
- Filesharing: [Samba](https://www.redhat.com/sysadmin/samba-file-sharing)

### Configuration

Based on:
[https://docs.zfsbootmenu.org/en/v2.3.x/guides/alpine/uefi.html#create-efi-boot-partition](https://docs.zfsbootmenu.org/en/v2.3.x/guides/alpine/uefi.html#create-efi-boot-partition)

Changes From Original Docs:

- don't add http repo
- Add community alpine repo
- run setup networking again after chroot
- Add nvme to mkinit features
- install linux-utils + curl earlier.
- EFI installed to EFI/BOOT/BOOTX64.EFI

#### Obtain a bootable Alpine Linux USB

- Download x86_64 Extended Alpine from [https://alpinelinux.org/downloads/](https://alpinelinux.org/downloads/)
- Copy the ISO to a usb using a tool such as [rufus](https://rufus.ie/en/)

#### Configure the Live Environment

- Boot Via the prepared Alpine Linux USB
- log in with user "root" and no password when prompted

To configure the environment... Run the following commands:
``` bash
source /etc/os-release
export ID

# Configure Networking
setup-interfaces -r   

# Set up the alpine package repositories 
cat <<EOF > /etc/apk/repositories
https://dl-cdn.alpinelinux.org/alpine/latest-stable/main/
https://dl-cdn.alpinelinux.org/alpine/latest-stable/community/
EOF
apk update  

# Install Tools needeed for configuring openzfs
apk add zfs zfs-scripts sgdisk wipefs util-linux lsblk parted
modprobe zfs 
zgenhostid -f 0x00bab10c

```


##### :warning:WARNING:warning:
##### Don't run these blindly unless you know what you're doing

To Prep the disks... Run the following commands:

``` bash
# Wipe the disks
wipefs --all /dev/nvme0n1
wipefs --all /dev/nvme1n1
wipefs --all /dev/nvme2n1
wipefs --all /dev/nvme3n1
wipefs --all /dev/nvme4n1
wipefs --all /dev/nvme5n1
```

``` bash
# Create Partition Tables
parted /dev/nvme0n1 mklabel gpt
parted /dev/nvme1n1 mklabel gpt
parted /dev/nvme2n1 mklabel gpt
parted /dev/nvme3n1 mklabel gpt
parted /dev/nvme4n1 mklabel gpt
parted /dev/nvme5n1 mklabel gpt
```

``` bash
# Create the 512MB EFI partition with 1MB offset
parted /dev/nvme0n1 mkpart primary fat32 1MB 513MB
parted /dev/nvme1n1 mkpart primary fat32 1MB 513MB
parted /dev/nvme2n1 mkpart primary fat32 1MB 513MB
parted /dev/nvme3n1 mkpart primary fat32 1MB 513MB
parted /dev/nvme4n1 mkpart primary fat32 1MB 513MB
parted /dev/nvme5n1 mkpart primary fat32 1MB 513MB
```

``` bash
# Create the first ZFS pool partition, this one is for the OS
parted /dev/nvme0n1 mkpart primary ext4 513MB 50513MB
parted /dev/nvme1n1 mkpart primary ext4 513MB 50513MB
parted /dev/nvme2n1 mkpart primary ext4 513MB 50513MB
parted /dev/nvme3n1 mkpart primary ext4 513MB 50513MB
parted /dev/nvme4n1 mkpart primary ext4 513MB 50513MB
parted /dev/nvme5n1 mkpart primary ext4 513MB 50513MB
```

``` bash
# Create the second ZFS pool partition, this one is for the DATA
parted /dev/nvme0n1 mkpart primary ext4 50513MB 100%
parted /dev/nvme1n1 mkpart primary ext4 50513MB 100%
parted /dev/nvme2n1 mkpart primary ext4 50513MB 100%
parted /dev/nvme3n1 mkpart primary ext4 50513MB 100%
parted /dev/nvme4n1 mkpart primary ext4 50513MB 100%
parted /dev/nvme5n1 mkpart primary ext4 50513MB 100%
```

``` bash
# Set the first partition as EFI
parted /dev/nvme0n1 set 1 esp on
parted /dev/nvme1n1 set 1 esp on
parted /dev/nvme2n1 set 1 esp on
parted /dev/nvme3n1 set 1 esp on
parted /dev/nvme4n1 set 1 esp on
parted /dev/nvme5n1 set 1 esp on
```

``` bash
# Update detected devices
mdev -s
```


#### Setup ZFS

Run these commands to configure the ZFS Pools
``` bash
# Create OS ZFS pool
zpool create -f -o ashift=12 \
 -O compression=zstd \
 -O acltype=posixacl \
 -O xattr=sa \
 -O relatime=on \
 -o autotrim=off \
 -m none zroot raidz2 /dev/nvme0n1p2 /dev/nvme1n1p2 /dev/nvme2n1p2 /dev/nvme3n1p2 /dev/nvme4n1p2 /dev/nvme5n1p2
```

``` bash
# Create Data ZFS pool
zpool create -f -o ashift=12 \
 -O compression=zstd \
 -O acltype=posixacl \
 -O xattr=sa \
 -O relatime=on \
 -o autotrim=off \
 -m none DATA raidz2 /dev/nvme0n1p3 /dev/nvme1n1p3 /dev/nvme2n1p3 /dev/nvme3n1p3 /dev/nvme4n1p3 /dev/nvme5n1p3
```

#### Install OS
``` bash
zfs create -o mountpoint=none zroot/ROOT
zfs create -o mountpoint=/ -o canmount=noauto zroot/ROOT/${ID}
zfs create -o mountpoint=/home zroot/home

zpool set bootfs=zroot/ROOT/${ID} zroot

zpool export zroot
zpool import -N -R /mnt zroot
zfs mount zroot/ROOT/${ID}
zfs mount zroot/home

apk --arch x86_64 -X http://dl-cdn.alpinelinux.org/alpine/latest-stable/main \
 -U --allow-untrusted --root /mnt --initdb add alpine-base

cp /etc/hostid /mnt/etc
cp /etc/resolv.conf /mnt/etc
cp /etc/apk/repositories /mnt/etc/apk
cp /etc/network/interfaces /mnt/etc/network
```

``` bash
# Chroot into the new OS
mount --rbind /dev /mnt/dev
mount --rbind /sys /mnt/sys
mount --rbind /proc /mnt/proc
chroot /mnt
# Set root password
passwd

# Enable internet
setup-interfaces -r

# Create startup targets
rc-update add hwdrivers sysinit
rc-update add networking
rc-update add hostname

# Install ZFS
apk add zfs zfs-lts lsblk
rc-update add zfs-import sysinit
rc-update add zfs-mount sysinit

echo "/etc/hostid" >> /etc/mkinitfs/features.d/zfshost.files
echo 'features="ata base keymap kms mmc scsi usb virtio zfs zfshost nvme"' > /etc/mkinitfs/mkinitfs.conf
mkinitfs -c /etc/mkinitfs/mkinitfs.conf "$(ls /lib/modules)"

# Install ZFS Boot Menu
zfs set org.zfsbootmenu:commandline="quiet" zroot/ROOT

cat << EOF >> /etc/fstab
/dev/nmve0n1p1 /boot/efi vfat defaults 0 0
EOF

mkdir -p /boot/efi
mount /boot/efi

apk add curl

mkdir -p /boot/efi/EFI/ZBM
curl -o /boot/efi/EFI/ZBM/VMLINUZ.EFI -L https://get.zfsbootmenu.org/efi
cp /boot/efi/EFI/ZBM/VMLINUZ.EFI /boot/efi/EFI/ZBM/VMLINUZ-BACKUP.EFI

apk add efibootmgr

efibootmgr -c -d "/dev/nvme0n1" -p "1" \
  -L "ZFSBootMenu (Backup)" \
  -l '\EFI\ZBM\VMLINUZ-BACKUP.EFI'

efibootmgr -c -d "/dev/nvme0n1" -p "1" \
  -L "ZFSBootMenu" \
  -l '\EFI\ZBM\VMLINUZ.EFI'

### Copy EFI to portable location
cp /boot/efi/EFI/ZBM/VMLINUZ.EFI /boot/efi/EFI/BOOT/BOOTX64.EFI
```

``` bash
# Prepare for first boot
exit	
cut -f2 -d" " /proc/mounts | grep ^/mnt | tac | while read i; do umount -l $i; done
zpool export zroot
reboot
```

#### First Boot

``` bash
rc-update add crond
rc-service start crond
apk add openssh
```

#### Set Up Cronjobs

##### verify if this already exists
- /etc/cron.d/zfs
- /usr/lib/zfs-linux/scrub

##### previous cron files do not exist we set them up manually with `crontab -e`
### Set up trim cron job
0 3 * * 6 /usr/sbin/zpool trim zroot
0 4 * * 6 /usr/sbin/zpool trim DATA

### Set up scrub cron job
0 3 * * 7 /usr/sbin/zpool scrub zroot
0 4 * * 7 /usr/sbin/zpool scrub DATA

### Mount the DATA ZFS Pool
``` bash
mkdir -p /mnt/DATA
zfs set mountpoint=/mnt/DATA DATA
```

##### Setting Up NFS Shares

``` bash
addgroup nasusers
adduser <your nfs username> nasusers
mkdir -p /mnt/DATA/Projects
chown -R root:nasusers /mnt/DATA/Projects/
```

##### Setting Up Samba Shares

``` bash
chown -R root:nasusers /mnt/DATA/Projects
chmod -R 771 /mnt/DATA/Projects
apk add samba	
smbpasswd -a <your samba username>
vi /etc/samba/smb.conf
```
Add the following to /etc/samba/smb.conf
```
[Projects]
    path = /mnt/DATA/Projects
    valid users = <your smb username>
    public = no
    writable = yes
```

```
rc-update add samba
rc-service samba start
```

run `ifconfig` to see your IP Address which other users can access your samba shares from. 

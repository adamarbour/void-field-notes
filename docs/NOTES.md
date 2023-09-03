# Wifi
1. Start bash instead of sh
```bash
bash
PS1='(live) #'
```
2. Check and make sure the devices are not blocked
```bash
rfkill
rfkill unblock wifi # Nothing was blocked but this is an example.
```
2. Get the wifi device name
```bash
ip link # wlp1s0
```
3. Connect to wifi
```bash
wpa_supplicant -B -i wlp1s0 -c <(wpa_passphrase MYSSID passphrase)
```
4. Bring the interface up
```bash
ip link set dev wlp1s0 up
```
5. Update xbps to make sure we've got the latest
```bash
xbps-install -Su xbps
```
# Partitioning and Filesystem
1. Get the device name
```bash
lsblk # nvme0n1
```
We set up the partition scheme to be like so
```
- EFI: 512M
- cryptsystem: *remaining*
```
2. Install the necessary programs
```bash
xbps-install gptfdisk btrfs-progs cryptsetup
```
3. Wipe and partition the drive
```bash
sgdisk --zap-all /dev/nvme0n1
sgdisk --clear \
--new=1:0:+512MiB --typecode=1:ef00 --change-name=1:EFI \
--new=2:0:0 --typecode=2:8300 --change-name=2:cryptsystem \
/dev/nvme0n1
```
4. Setup cryptsystem
```bash
cryptsetup luksFormat --perf-no_read_workqueue --perf-no_write_workqueue --type luks2 --cipher aes-xts-plain64 --key-size 512 --iter-time 2000 --pbkdf argon2id --hash sha3-512 /dev/disk/by-partlabel/cryptsystem
cryptsetup --allow-discards --perf-no_read_workqueue --perf-no_write_workqueue --persistent open /dev/disk/by-partlabel/cryptsystem cryptroot
```
5. Format partitions
```bash
mkfs.vfat -F32 -n EFI /dev/disk/by-partlabel/EFI
mkfs.btrfs -L ROOTFS -f /dev/mapper/cryptroot
```
6. Create subvolumes
```bash
mount /dev/mapper/cryptroot /mnt
btrfs sub create /mnt/@ && \
btrfs sub create /mnt/@home && \
btrfs sub create /mnt/@swap && \
btrfs sub create /mnt/@abs && \
btrfs sub create /mnt/@tmp && \
btrfs sub create /mnt/@srv && \
btrfs sub create /mnt/@snapshots && \
btrfs sub create /mnt/@btrfs && \
btrfs sub create /mnt/@log && \
btrfs sub create /mnt/@cache
umount /mnt
```
7. Mount the filesystem to bootstrap
```bash
OPT_DEFAULT=noatime,compress-force=zstd,commit=120,space_cache=v2,ssd,discard=async,autodefrag
EXT_OPT=nodev,nosuid,noexec

mount -o $OPT_DEFAULT,subvol=@ /dev/mapper/cryptroot /mnt
mkdir -p /mnt/{home,.swapvol,var/abs,var/tmp,srv,.snapshots,.btrfs,var/log,boot,var/cache} # Create all the required directories

mount -o $OPT_DEFAULT,subvol=@home /dev/mapper/cryptroot /mnt/home  && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@swap /dev/mapper/cryptroot /mnt/.swapvol && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@abs /dev/mapper/cryptroot /mnt/var/abs && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@tmp /dev/mapper/cryptroot /mnt/var/tmp && \
mount -o $OPT_DEFAULT,subvol=@srv /dev/mapper/cryptroot /mnt/srv && \
mount -o $OPT_DEFAULT,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots && \
mount -o $OPT_DEFAULT,subvolid=5 /dev/mapper/cryptroot /mnt/btrfs
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@log /dev/mapper/cryptroot /mnt/var/log && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvolid=@boot /dev/mapper/cryptroot /mnt/boot && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@cache /dev/mapper/cryptroot /mnt/var/cache
```
8. Mount the EFI boot volume
```bash
mount -o $EXT_OPT /dev/disk/by-partlabel/EFI /mnt/boot
```
9. Create swap file
```bash
btrfs filesystem mkswapfile --size 32g --uuid clear /mnt/.swapvol/swapfile
swapon /mnt/.swapvol/swapfile
```

# Bootstrap Install Void
1. Install pre-reqs
```bash
xbps-install xtools git
```
2. Set variables and copy keys for chroot
```bash
REPO=https://repo-default.voidlinux.org/current
ARCH=x86_64
mkdir -p /mnt/var/db/xbps/keys
cp /var/db/xbps/keys/* /mnt/var/db/xbps/keys/
```
3. Install the base system meant for my specifics (adjust for your needs) 
```bash
XBPS_ARCH=$ARCH xbps-install -S -r /mnt -R "$REPO" base-system base-devel linux-firmware-amd linux-firmware-qualcomm btrfs-progs cryptsetup refind sbctl sbsigntool gummiboot-efistub efibootmgr efitools lz4 lzip zsh zsh-autosuggestions zsh-completions nano curl wget git
```
-- SEE /usr/share/doc/efibootmgr/README.voidlinux for instructions using efibootmgr to automatically manage EFI boot entries
-- TODO Mount efivars readonly
4. Generate fstab
```bash
git clone https://github.com/glacion/genfstab
./genfstab/genfstab -L /mnt >> /mnt/etc/fstab
```
5. Change root into the target system
```bash
cp /etc/resolv.conf /mnt/etc

mount --rbind /sys /mnt/sys && mount --make-rslave /mnt/sys
mount --rbind /dev /mnt/dev && mount --make-rslave /mnt/dev
mount --rbind /proc /mnt/proc && mount --make-rslave /mnt/proc

xchroot /mnt /bin/zsh
```
6. Configure the target system
```bash
echo %HOSTNAME% > /etc/hostname

ln -sf /usr/share/zoneinfo/<timezone> /etc/localtime
hwclock -uw

nano /etc/default/libc-locales # Enable the locales I want
xbps-reconfigure -f glibc-locales

passwd # Set root password
chsh -s /bin/zsh
```
# Bootloader
1. Generate keys
```bash
sbctl create-keys
sbctl enroll-keys -im
```
3. Manually setup refind
```bash
mkdir -p /boot/EFI/refind
cp /usr/share/refind/refind_x64.efi /boot/EFI/refind/
efibootmgr --create --disk /dev/nvme0n1 --part 1 --loader /EFI/refind/refind_x64.efi --label "rEFInd Boot Manager" --unicode
mkdir /boot/EFI/refind/drivers_x64
cp /usr/share/refind/drivers_x64/btrfs_x64.efi /boot/EFI/refind/drivers_x64
cp /usr/share/refind/refind.conf-sample /boot/EFI/refind/refind.conf
cp -r /usr/share/refind/icons /boot/EFI/refind/
cp -r /usr/share/refind/fonts /boot/EFI/refind/
```
2. Configure refined
```bash
sed -i 's/#scanfor internal,external,optical,manual/scanfor manual,external/' /boot/EFI/refind/refind.conf
nano /boot/EFI/refind/refind.conf
>> 
```
4. 


2. Add custom theme
```bash
mkdir /boot/EFI/refind/themes
git clone https://github.com/dheishman/refind-dreary.git /efi/EFI/refind/themes/refind-dreary-git
mv /efi/EFI/refind/themes/refind-dreary-git/highres /efi/EFI/refind/themes/refind-dreary
rm -dR /efi/EFI/refind/themes/refind-dreary-git
```

# Configure Kernel
1. Set it to host-only mode
```bash
echo hostonly=yes >> /etc/dracut.conf
```
2. Reconfigure/regenerate the kernel
```bash
xbps-reconfigure -fa
```

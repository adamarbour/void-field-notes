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
wpa_passphrase <MYSSID> "<passphrase>" >> /etc/wpa_supplicant/wpa_supplicant.conf
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
- efi: 512M
- btrfs: *remaining*
```
2. Install the necessary programs
```bash
xbps-install gptfdisk btrfs-progs
```
3. Wipe and partition the drive
```bash
sgdisk --zap-all /dev/nvme0n1
sgdisk --clear \
--new=1:0:+512MiB --typecode=1:ef00 --change-name=1:EFI \
--new=2:0:0 --typecode=2:8300 --change-name=2:ROOTFS \
/dev/nvme0n1
```
5. Format partitions
```bash
mkfs.vfat -F32 -n EFI /dev/disk/by-partlabel/EFI
mkfs.btrfs -L BTRFS -f /dev/disk/by-partlabel/ROOTFS
```
6. Create subvolumes
```bash
mount -L BTRFS /mnt
btrfs sub create /mnt/@ && \
btrfs sub create /mnt/@home && \
btrfs sub create /mnt/@swap && \
btrfs sub create /mnt/@abs && \
btrfs sub create /mnt/@tmp && \
btrfs sub create /mnt/@srv && \
btrfs sub create /mnt/@snapshots && \
btrfs sub create /mnt/@log && \
btrfs sub create /mnt/@cache
umount /mnt
```
7. Mount the filesystem to bootstrap
```bash
OPT_DEFAULT=noatime,compress-force=zstd,commit=120,space_cache=v2,ssd,discard=async,autodefrag
EXT_OPT=nodev,nosuid,noexec

mount -o $OPT_DEFAULT,subvol=@ -L BTRFS /mnt
mkdir -p /mnt/{home,.swapvol,var/abs,var/tmp,srv,.snapshots,.btrfs,var/log,boot,var/cache} # Create all the required directories

mount -o $OPT_DEFAULT,subvol=@home -L BTRFS /mnt/home  && \
mount -o $OPT_DEFAULT,subvol=@srv -L BTRFS /mnt/srv && \
mount -o $OPT_DEFAULT,subvol=@snapshots -L BTRFS /mnt/.snapshots && \
mount -o $OPT_DEFAULT,subvolid=5 -L BTRFS /mnt/.btrfs
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@swap -L BTRFS /mnt/.swapvol && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@abs -L BTRFS /mnt/var/abs && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@tmp -L BTRFS /mnt/var/tmp && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@log -L BTRFS /mnt/var/log && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@cache -L BTRFS /mnt/var/cache
```
8. Mount the EFI boot volume
```bash
mount -o $EXT_OPT -L EFI /mnt/boot
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
XBPS_ARCH=$ARCH xbps-install -S -r /mnt -R "$REPO" base-system base-devel linux-firmware-amd linux-firmware-qualcomm btrfs-progs grub grub-x86_64-efi grub-btrfs grub-btrfs-runit iwd NetworkManager elogind sbctl sbsigntool gummiboot-efistub efibootmgr efitools efivar lz4 lzip zsh zsh-autosuggestions zsh-completions nano curl wget git void-repo-nonfree
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

sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/default/libc-locales
xbps-reconfigure -f glibc-locales

passwd # Set root password
chsh -s /bin/zsh
```

# Bootloader
1. Install bootloader
```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=VOID
```
2. Sign with sbctl to enable secure boot
```bash
sbctl create-keys
sbctl sign /boot/EFI/VOID/grubx64.efi
sbctl enroll-keys -im
```
3. 

# Configure Kernel
1. Set the defaults
```bash
nano /etc/dracut.conf.d/00-dracut-defaults.conf

## CONTENTS OF FILE ##
hostonly=yes
compress="lz4"
early_microcode=yes
show_modules=no
```
3. Reconfigure/regenerate the kernel
```bash
xbps-reconfigure -fa
```
// TODO: Show the kernel configurations + early loading

# Networking
For my purposes, I plan to use NetworkManager with an iwd backend.
1. Enable dBus
```bash
ln -s /etc/sv/dbus /etc/runit/runsvdir/default
ln -s /etc/sv/elogind /etc/runit/runsvdir/default
```
2. Setup Network Manager to use iwd as backend
```bash
mkdir -p /etc/NetworkManager/conf.d/
nano /etc/NetworkManager/iwd.conf
```
Contents of `/etc/NetworkManager/conf.d/iwd.conf`
```bash
[device]
wifi.backend=iwd
```
3. Configure iwd. Enable the built-in networking + disable IPv6 (optional)
```bash
# Contents of /etc/iwd/main.conf

[General]
EnableNetworkConfiguration=true
UseDefaultInterface=true

[Network]
EnableIPv6=false
```
5. Enable the services
```bash
ln -s /etc/sv/iwd /etc/runit/runsvdir/default
ln -s /etc/sv/NetworkManager /etc/runit/runsvdir/default
```
6. Connect to wifi...

# Firmware updates
1. Install firmware update
```bash
xbps-install fwupd fwupd-efi fwupdate
```
2. Create a tools directory if it does not exist & copy the binary + sign
```bash
mkdir -p /boot/EFI/tools
cp /usr/lib/fwupdate/EFI/void/fwupx64.efi /boot/EFI/tools
sbctl sign /boot/EFI/tools/fwupx64.efi
```
3. Setup refind option by adding it to the showtools
```bash
# /EFI/refind/refind.conf
showtools <...>, fwupdate
```
5. 

# Unlock LUKs from TPM
1. Install tpm tools
```bash
xbps-install tpm2-tools
```
2. Create a secret key for TPM and add it to Luks container
```bash
dd if=/dev/urandom of=secret.bin bs=32 count=1
cryptsetup luksAddKey /dev/disk/by-partlabel/cryptsystem secret.bin
```
3. Store the key in TPM
```bash
tpm2_createpolicy -Q --policy-pcr -l sha256:0,7 -L pcr.policy
tpm2_createprimary -C e -G rsa -c primary.ctx -Q
tpm2_create -C primary.ctx -L pcr.policy -i secret.bin -u seal.pub -r seal.priv -c seal.ctx -a "noda|adminwithpolicy|fixedparent|fixedtpm" -Q
tpm2_load -C primary.ctx -u seal.pub -r seal.priv -c seal.ctx
tpm2_evictcontrol -C o -c primary.ctx 0x81000000
```
4. Check and make sure you can unseal the key
```bash
tpm2_unseal -c 0x81000000 -p pcr:sha256:0,7
```
5. 

# Hibernate
btrfs inspect-internal map-swapfile -r /swap/swapfile
resume=UUID=4209c845-f495-4c43-8a03-5363dd433153
resume_offset=swap_file_offset

# Backup & Recovery
## Ensure we have cron working
```bash
xbps-install cronie
ln -s /etc/sv/crond /etc/runit/runsvdir/default
```
## Snapper & Schedule
```bash
xbps-install snapper

# We need to unmount snapshots because snapper will try to add it to the configuration
umount /.snapshots
rm -r /.snapshots
snapper -c root create-config /
mount -a
chmod 750 -R /.snapshots
chown :wheel /.snapshots

# Create an initial backup
snapper -c root create --description "Initial"
snapper -c root list # Check

# Modify the configuration timings
sed -i 's/^TIMELINE_MIN_AGE.*/TIMELINE_MIN_AGE="1800"/' /etc/snapper/configs/root
sed -i 's/^TIMELINE_LIMIT_HOURLY.*/TIMELINE_LIMIT_HOURLY="0"/' /etc/snapper/configs/root
sed -i 's/^TIMELINE_LIMIT_DAILY.*/TIMELINE_LIMIT_DAILY="7"/' /etc/snapper/configs/root
sed -i 's/^TIMELINE_LIMIT_WEEKLY.*/TIMELINE_LIMIT_WEEKLY="0"/' /etc/snapper/configs/root
sed -i 's/^TIMELINE_LIMIT_MONTHLY.*/TIMELINE_LIMIT_MONTHLY="0"/' /etc/snapper/configs/root
sed -i 's/^TIMELINE_LIMIT_YEARLY.*/TIMELINE_LIMIT_YEARLY="0"/' /etc/snapper/configs/root
sed -i 's/^ALLOW_GROUPS.*/ALLOW_GROUPS="wheel"/' /etc/snapper/configs/root
```
## Grub & Grub-btrfs

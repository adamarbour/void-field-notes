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
- efi: 300MiB
- btrfs: *remaining*
```
2. Install the necessary programs
```bash
xbps-install gptfdisk btrfs-progs cryptsetup
```
3. Wipe and partition the drive
```bash
sgdisk --zap-all /dev/nvme0n1
sgdisk --clear \
--new=1:0:+300MiB --typecode=1:ef00 --change-name=1:EFI \
--new=2:0:0 --typecode=2:8300 --change-name=2:ROOTFS \
/dev/nvme0n1
```
5. Format EFI partition
```bash
mkfs.vfat -F32 -n EFI /dev/disk/by-partlabel/EFI
```
6. Setup the crypt volume
```bash
cryptsetup --type luks2 --cipher aes-xts-plain --hash sha512 --iter-time 3000 --key-size 512 --pbkdf pbkdf2 --use-urandom --verify-passphrase luksFormat /dev/disk/by-partlabel/ROOTFS
cryptsetup luksOpen /dev/disk/by-partlabel/ROOTFS cryptroot
```
7. Setup the volume group and create the swap
```bash
vgcreate vg1 /dev/mapper/cryptroot
lvcreate -L 34GB -n VOID-swap vg1
lvcreate -l 100%FREE -n VOID-root vg1

mkswap -L SWAP /dev/mapper/vg1-VOID--swap
swapon /dev/mapper/vg1-VOID--swap

mkfs.btrfs -L BTRFS -n 16k /dev/mapper/vg1-VOID--root -f
```
9. Create subvolumes
```bash
mount -L BTRFS /mnt
btrfs sub create /mnt/@ && \
btrfs sub create /mnt/@root && \
btrfs sub create /mnt/@home && \
btrfs sub create /mnt/@abs && \
btrfs sub create /mnt/@tmp && \
btrfs sub create /mnt/@srv && \
btrfs sub create /mnt/@snapshots && \
btrfs sub create /mnt/@var_log && \
btrfs sub create /mnt/@var_cache
btrfs sub create /mnt/@lib_docker
umount /mnt
```
7. Mount the filesystem to bootstrap
```bash
OPT_DEFAULT=noatime,compress-force=zstd,commit=120,space_cache=v2,ssd,discard=async,autodefrag
EXT_OPT=nodev,nosuid,noexec

mount -o $OPT_DEFAULT,subvol=@ -L BTRFS /mnt
mkdir -p /mnt/{root,home,var/abs,var/tmp,srv,.snapshots,.btrfs,var/log,boot/efi,var/cache/xbps,var/lib/docker} # Create all the required directories

mount -o $OPT_DEFAULT,subvol=@root -L BTRFS /mnt/root  && \
mount -o $OPT_DEFAULT,subvol=@home -L BTRFS /mnt/home  && \
mount -o $OPT_DEFAULT,subvol=@srv -L BTRFS /mnt/srv && \
mount -o $OPT_DEFAULT,subvol=@snapshots -L BTRFS /mnt/.snapshots && \
mount -o $OPT_DEFAULT,subvolid=5 -L BTRFS /mnt/.btrfs && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@abs -L BTRFS /mnt/var/abs && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@tmp -L BTRFS /mnt/var/tmp && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@var_log -L BTRFS /mnt/var/log && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@var_cache -L BTRFS /mnt/var/cache/xbps && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@lib_docker -L BTRFS /mnt/var/lib/docker
```
8. Mount the EFI boot volume
```bash
mount -o $EXT_OPT -L EFI /mnt/boot/efi
```
9. Backup the LUKs header just in case
```bash
mkdir /mnt/var/backup/cryptsetup -p
cryptsetup luksHeaderBackup /dev/disk/by-partlabel/ROOTFS --header-backup-file /mnt/var/backup/cryptsetup/VOID.luks.bin
# Check it
cryptsetup luksDump /mnt/var/backup/cryptsetup/VOID.luks.bin
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
XBPS_ARCH=$ARCH xbps-install -S -r /mnt -R "$REPO" base-system base-devel linux-firmware linux-firmware-amd linux-firmware-network linux-firmware-qualcomm cryptsetup lvm2 btrfs-progs grub grub-x86_64-efi grub-terminus grub-btrfs grub-btrfs-runit terminus-font iwd NetworkManager avahi nss-mdns elogind sbctl sbsigntool gummiboot-efistub efibootmgr efitools efivar lz4 lzop lzip lrzip acpid cronie chrony socklog-void sudo zsh zsh-autosuggestions zsh-completions nnn htop restic snapper btrbk rclone rsync nano curl wget git ldns void-repo-nonfree fwupd fwupd-efi apparmor docker docker-compose containerd

# GRAPHICS
mesa-dri vulkan-loader mesa-vulkan-radeon mesa-vaapi mesa-vdpau xf86-video-amdgpu
# AUDIO
pipewire wireplumber-elogind alsa-pipewire libjack-pipewire bluez libspa-bluetooth easyeffects
# PRINTING
cups cups-filters hplip
# DESKTOP EXPERIENCE
gnome gnome-apps
```

-- SEE /usr/share/doc/efibootmgr/README.voidlinux for instructions using efibootmgr to automatically manage EFI boot entries
-- TODO Mount efivars readonly

4. Generate fstab
```bash
git clone https://github.com/glacion/genfstab
./genfstab/genfstab -L /mnt >> /mnt/etc/fstab

echo "tmpfs /tmp tmpfs defaults,noatime,mode=1777,size=32G 0 0" >> /mnt/etc/fstab
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
ln -sf /usr/share/zoneinfo/<timezone> /etc/localtime
hwclock -uw

passwd # Set root password
chsh -s /bin/zsh

echo %HOSTNAME% > /etc/hostname

sed -i 's/#KEYMAP="es"/KEYMAP="us"/' /etc/rc.conf
sed -i 's/#FONT="lat9w-16"/FONT="ter-132n"/' /etc/rc.conf
sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/default/libc-locales
xbps-reconfigure -f glibc-locales
```
7. Create an additional key for Luks
```bash
dd bs=512 count=80 if=/dev/urandom of=/root/crypto_keyfile.bin iflag=fullblock
cryptsetup luksAddKey /dev/disk/by-partlabel/ROOTFS /root/crypto_keyfile.bin
chmod 600 /root/crypto_keyfile.bin
```

# Configure Initram
1. Set the defaults
```bash
nano /etc/dracut.conf.d/00-dracut-defaults.conf

## CONTENTS OF FILE ##
hostonly=yes
compress="lz4"
early_microcode=yes
show_modules=no

force_drivers+=" amdgpu "
install_items+=" /root/crypto_keyfile.bin "

```
2. Regenerate initram
```bash
dracut --force --kver <version>
chmod 600 /boot/vmlinuz-*
chmod 600 /boot/initramfs-*
```

# Bootloader
1. Modify grub config
```bash
nano /etc/default/grub

## CONTENTS CHANGED
GRUB_TIMEOUT=3
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"
GRUB_CMDLINE_LINUX="rd.luks.name=10e467ce-785a-401c-b2c5-9379090653f4=cryptroot rd.luks.options=10e467ce-785a-401c-b2c5-9379090653f4=discard,password-echo=no,keyfile-timeout=10s rd.lvm.lv=vg1/vg1-VOID-root rd.lvm.lv=vg1/vg1-VOID-swap rd.luks.key=10e467ce-785a-401c-b2c5-9379090653f4=/root/crypto_keyfile.bin resume=UUID=f472c306-cd57-42ce-b044-471b9c640d8c root=UUID=494e8465-f34f-4947-bcec-09a15e2caba6"
GRUB_PRELOAD_MODULES="part_gpt cryptodisk luks2 lvm"
GRUB_ENABLE_CRYPTODISK=y
GRUB_GFXMODE=1920x1080x24
```
3. Install bootloader
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=VOID --recheck
grub-mkconfig -o /boot/grub/grub.cfg
```
2. Patch the Grub image
```bash
nano /boot/grub/grub-pre.cfg

# Contents of file. NOTE: uuid is the id without dashes of the luks device
set crypto_uuid=10e467ce785a401cb2c59379090653f4
cryptomount -u $crypto_uuid
set root=lvm/vg1-VOID-root
set prefix=($root)/boot/grub
insmod normal
normal
```
3. Create the new image
```bash
grub-mkimage -p /boot/grub -O x86_64-efi -c /boot/grub/grub-pre.cfg -o /tmp/grubx64.efi part_gpt cryptodisk luks2 lvm gcry_rijndael pbkdf2 gcry_sha256 gcry_sha512 btrfs && install -v /tmp/grubx64.efi /boot/efi/EFI/VOID/grubx64.efi
```
# Reboot to ensure we can get into the system for post install
```bash
exit
swapoff -a
umount -R /mnt
reboot now
```


---

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
```bash
nmcli device wifi connect <SSID> password <password>
```
7. Install avahi for ZeroConfig
```bash
xbps-install avahi
ln -s /etc/sv/avahi-daemon /etc/runit/runsvdir/default/
```
8. 

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



# Backup & Recovery
## Ensure we have cron working
```bash
xbps-install cronie
ln -s /etc/sv/crond /etc/runit/runsvdir/default
```
## Snapper & Schedule
```bash
xbps-install snapper grub-btrfs grub-btrfs-runit

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
```bash
# Enable the snapshot watch
ln -s /etc/sv/grub-btrfs /etc/runit/runsvdir/default/
```
# Graphics
## Drivers
```bash
xbps-install xorg-server-xwayland xf86-video-amdgpu mesa-dri mesa-vaapi mesa-vdpau vulkan-loader mesa-vulkan-radeon
```



# Quality of Life
## Power Management

## NTP

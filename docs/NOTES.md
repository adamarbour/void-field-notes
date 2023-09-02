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
--new=2:0:0 --typecode=2:8300 --change-name=3:cryptsystem \
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
btrfs sub create /mnt/@boot && \
btrfs sub create /mnt/@cache
umount /mnt
```
7. Mount the filesystem to bootstrap
```bash
OPT_DEFAULT=noatime,compress-force=zstd,commit=120,space_cache=v2,ssd,discard=async,autodefrag
EXT_OPT=nodev,nosuid,noexec
mount -o $OPT_DEFAULT,subvol=@ /dev/mapper/crypt /mnt
mkdir -p /mnt/{home,var/swap,var/abs,var/tmp,srv,.snapshots,btrfs,var/log,boot,var/cache} # Create all the required directories
mount -o $OPT_DEFAULT,subvol=@home /dev/mapper/crypt /mnt/home  && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@swap /dev/mapper/crypt /mnt/var/swap && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@abs /dev/mapper/crypt /mnt/var/abs && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@tmp /dev/mapper/crypt /mnt/var/tmp && \
mount -o $OPT_DEFAULT,subvol=@srv /dev/mapper/crypt /mnt/srv && \
mount -o $OPT_DEFAULT,subvol=@snapshots /dev/mapper/crypt /mnt/.snapshots && \
mount -o $OPT_DEFAULT,subvolid=5 /dev/mapper/crypt /mnt/btrfs
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@log /dev/mapper/crypt /mnt/var/log && \
mount -o $OPT_DEFAULT,subvolid=@boot /dev/mapper/crypt /mnt/boot && \
mount -o $OPT_DEFAULT,$EXT_OPT,subvol=@cache /dev/mapper/crypt /mnt/var/cache
```
8. 

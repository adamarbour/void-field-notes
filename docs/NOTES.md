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

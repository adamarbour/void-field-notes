# Void Linux Install Notes

## Goal 1

The purpose of this section is to describe how I installed Void on my device. I am naturally not taking the easy way out here and will be following their advanced installation guides[^1]. This is because I intend to include the following:

- EFISTUB
- btrfs + LUKS full drive encryption
- store LUKS key in the Intel TPM

## Steps

After booting into the live media, my first objective is to get the device connected to the wifi and enable `sshd` for me to remote in from another machine during the installation.

On the device:

1. WPA Supplicant is included with the live media so I'll use that to setup wifi.
   ```bash
   rfkill
   rfkill unblock wifi	# If it shows blocked
   
   # Get the wireless device name
   ip link	# wlp4s0 for me
   
   # Setup wifi using wpa_cli
   wpa_cli
   > add_network
   > set_network 0 ssid "$SSID"	# Replace with your SSID
   > set_network 0 psk "$PASSWD"	# Replace with your password
   > enable_network 0
   
   # WPA: Key negotiation completed with... SUCCESS!!
   > quit
   ```

2. Get `sshd` running so I can login remotely
   ```bash
   # Check if sshd is running
   sv status sshd	# Returns a PID. It is running.
   
   # Get the IP address
   ip a
   ```

Now I can login remotely to complete the installation.

```bash
ssh anon@0.0.0.0	# Root cannot login remotely. I don't feel like changing the config.
> voidlinux
su -	# Switch to root
> voidlinux
whoami	# Validate you're root
```

### The Install...

Now that we are remoted in, it is soooo much easier to do the rest of the installation. We will break the installation down into the following.

* Preparing the partitions
* Base installation
* Bootloader setup

#### Preparing the partitions

I've already wiped my device. We will use fdisk to setup the partitions and then create the filesystem.

1. Create the partitions
   ```bash
   # List block storage devices (mine is nvme0n1)
   lsblk
   
   # Partition the drive
   cfdisk /dev/nvme0n1
   > gpt
   > [New]
   > 1G	# Partition size for EFI (more than we need)
   > [Type]
   > "EFI System"
   > [New]
   > (default)	# Use the remainder of the space
   > [Write]
   > yes
   > [Quit]
   
   # Check to make sure the new partitions are there
   lsblk
   ```

2. Format the EFI partition
   ```bash
   mkfs.vfat -F32 -n EFI /dev/nvme0n1p1
   ```

3. Create the crypt LUKs device for us to store our root file system on
   ```bash
   cryptsetup luksFormat /dev/nvme0n1p2	# Defaults are reasonable
   > Are you sure? YES
   > Enter passphrase for /dev/nvme0n1p2: *** # Your password
   > Verify password: ***
   
   # Open the container so we can setup the rootfs
   cryptsetup open /dev/nvme0n1p2 luks
   > Enter passphrase for /dev/nvme0n1p2: *** # Password from before
   ```

4. Setup the rootfs with btrfs
   ```bash
   mkfs.btrfs -L ROOT /dev/mapper/luks
   mount /dev/mapper/luks /mnt	# Mount the device to create sub-volumes
   
   # Create the sub-volumes
   btrfs sub create /mnt/@
   btrfs sub create /mnt/@boot
   btrfs sub create /mnt/@swap
   btrfs sub create /mnt/@home
   btrfs sub create /mnt/@snapshots
   btrfs sub create /mnt/@log
   btrfs sub create /mnt/@cache
   
   # Unmount so we can re-mount the subvolumes properly
   umount /mnt
   ```

5. Mount the partitions for chroot installation
   ```bash
   OPT_DEFAULT=noatime,nodiratime,compress=zstd,space_cache=v2,ssd
   
   # Mount the root
   mount -o $OPT_DEFAULT,subvol=@ /dev/mapper/luks /mnt
   
   # Create the required directories to mount the subvolumes
   cd /mnt
   mkdir -p efi home var/cache var/log .snapshots swap boot
   
   # Mount the subvolumes
   mount -o $OPT_DEFAULT,subvol=@home /dev/mapper/luks /mnt/home
   mount -o $OPT_DEFAULT,subvol=@snapshots /dev/mapper/luks /mnt/.snapshots
   mount -o $OPT_DEFAULT,subvol=@log /dev/mapper/luks /mnt/var/log
   mount -o $OPT_DEFAULT,subvol=@cache /dev/mapper/luks /mnt/var/cache
   mount -o $OPT_DEFAULT,subvol=@swap /dev/mapper/luks /mnt/swap
   mount -o $OPT_DEFAULT,subvol=@boot /dev/mapper/luks /mnt/boot
   
   # Mount the EFI partition
   mount /dev/nvme0n1p1 /mnt/efi
   
   # Check the mount points
   lsblk
   ```

6. Setup the swap file
   ```bash
   cd /mnt/swap
   truncate -s 0 swapfile
   chattr +C swapfile
   fallocate -l 16G swapfile # Same as my system memory for hibernation
   chmod 0600 swapfile
   mkswap swapfile
   swapon swapfile
   ```

And we are ready to setup the base system.

#### Install Void Linux

There are two significant things to note about my installation that you may want to change.

- I will be using the `musl` mirror, and will set up a `glibc` chroot when needed.
- I will be using the XBPS Method[^2] to install into the chroot

Now for the install...

1. Set the required environment variables and copy the host keys into chroot
   ```bash
   REPO=https://repo-default.voidlinux.org/current/musl
   ARCH=x86_64-musl
   
   # Copy keys
   mkdir -p /mnt/var/db/xbps/keys
   cp /var/db/xbps/keys/* /mnt/var/db/xbps/keys/
   ```

2. Install the `base-system` metapackage
   ```bash
   # Base
   XBPS_ARCH=$ARCH xbps-install -Sy -r /mnt -R "$REPO" base-system base-devel
   # Need
   XBPS_ARCH=$ARCH xbps-install -Sy -r /mnt -R "$REPO" btrfs-progs cryptsetup gummiboot-efistub efibootmgr dracut-uefi sbctl efitools openssl binutils iwd lz4
   # Extra
   XBPS_ARCH=$ARCH xbps-install -Sy -r /mnt -R "$REPO" nano wget curl rsync git
   ```

3. Generate the `/etc/fstab` file.
   _NOTE: void doesn't include anything that generates the file, so it is a manual process. BUT..._

   ```bash
   # Install some stuff
   xbps-install -Sy nano wget
   
   # Grab the Arch genfstab standalone from this repository
   wget https://raw.githubusercontent.com/glacion/genfstab/master/genfstab -O /sbin/genfstab
   chmod +x /sbin/genfstab
   
   # Generate the fstab
   genfstab -U /mnt >> /mnt/etc/fstab
   ```

Now we are ready to setup the bootloader.

#### Setup bootloader

The instructions on the Void website include setting up GRUB, but for my installation, I intend to boot directly from UEFI using the `efibootmgr`. I've adapted this from an earlier setup to use here. In our next goal, we will clean this up. For now, I want to be able to boot up the device.

1. Create a folder that will hold the signed unified kernel.
   ```bash
   mkdir -p /mnt/efi/EFI/Linux
   ```

2. Let's generate EFI keys for this device and enroll them into the EFI. [^3]
   ```bash
   # We will need these tools
   xbps-install -Sy efitools sbctl binutils gummiboot-efistub lz4
   
   # A handy script for generating efi keys. We will need this for the bootloader
   wget https://raw.githubusercontent.com/jirutka/efi-mkkeys/master/efi-mkkeys -O /sbin/efi-mkkeys
   chmod +x /sbin/efi-mkkeys
   
   # Generate keys for our new system
   efi-mkkeys -s "VOID" -o /mnt/etc/efi-keys
   
   # Import the keys into sbctl to enroll them to the EFI
   sbctl import-keys --force \
   	--db-cert /mnt/etc/efi-keys/db.crt \
   	--db-key /mnt/etc/efi-keys/db.key \
   	--kek-cert /mnt/etc/efi-keys/KEK.crt \
   	--kek-key /mnt/etc/efi-keys/KEK.key \
   	--pk-cert /mnt/etc/efi-keys/PK.crt \
   	--pk-key /mnt/etc/efi-keys/PK.key
   	
   # Enroll the successfully imported keys
   sbctl enroll-keys -im
   
   # Setup Mode should be disabled and the keys are ready to go
   sbctl status
   ```

3. Create a basic `dracult.conf.d` file that we will tweak later. For now we just want to get the system booted.
   NOTE: We will be creating a brand new one in our ch-root but we will use this as the original

   ```bash
   nano /etc/dracut.conf.d/dracut-defaults.conf
   
   # Keep it for later
   cp /etc/dracut.conf.d/dracut-defaults.conf /mnt/etc/dracut.conf.d/dracut-defaults.conf.original
   
   #########################################
   #   Contents of the file below....		#
   #########################################
   hostonly=no
   hostonly_cmdline=no
   use_fstab=yes
   compress=lz4
   early_microcode=no
   show_modules=yes
   
   add_drivers+='lz4 lz4_compress'
   filesystems+='btrfs'
   add_dracutmodules+='crypt'
   
   uefi=yes
   uefi_stub=/usr/lib/gummiboot/linuxx64.efi.stub
   
   CMDLINE=(
   	rw
   	rd.luks=1
   	rd.luks.timeout=60
   	rd.luks.crypttab=0
   	rd.luks.name=$(cryptsetup luksUUID /dev/nvme0n1p2)=ROOT
   	root=LABEL=ROOT
   	rootflags=subvol=@
   )
   kernel_cmdline="${CMDLINE[*]}"
   unset CMDLINE
   ```

4. Let's sign the efi file, copy it to our target `mnt` and add it to the boot manager.
   ```bash
   # Sign the .efi file that was created
   sbctl sign /boot/EFI/Linux/linux-5.19.10_1.efi
   
   # Copy it into our target mnt
   mkdir -p /mnt/efi/EFI/Linux
   cp /boot/EFI/Linux/linux-5.19.10_1.efi /mnt/efi/EFI/Linux/linux-5.19.10_1.efi 
   
   # Add the kernel to the boot manager
   efibootmgr --create --disk /dev/nvme0n1 --label "Void Linux" --loader 'EFI\Linux\linux-5.19.10_1.efi' --unicode
   
   # Validate that it was loaded
   efibootmgr --unicode
   
   # Change the boot order. Modify for your needs.
   efibootmgr -o 0000,0001,0002,0003,...
   ```

   NOTE: Later we will modify the Kernel hooks to take care of signing and updating the boot manager.[^4]

#### The moment we've been waiting for...

1. Chroot into the installed system and set some basic settings
   ```bash
   chroot /mnt
   chown root:root /
   chmod 755 /
   
   # Set the hostname
   echo "$HOSTNAME" > /etc/hostname
   
   # Set the timezone
   ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime # Set this to your timezone
   
   # Change the swappiness to swap less
   echo vm.swappiness=10 > /usr/lib/sysctl.d/99-swappiness.conf
   
   # Set a root password
   passwd root
   ```

2. Close everything out
   ```bash
   exit	# Exit chroot
   swapoff /mnt/swap/swapfile
   umount -R /mnt
   ```

3. Now we pray...
   ```bash
   reboot
   ```

   

## References

[^1]: [Advanced Installation Guides - Void Linux Handbook](https://docs.voidlinux.org/installation/guides/index.html)
[^2]: [The XBPS Method](https://docs.voidlinux.org/installation/guides/chroot.html#the-xbps-method)
[^3]: [EFI-mkkeys](https://github.com/jirutka/efi-mkkeys/)
[^4]: [Void Kernel Hooks](https://docs.voidlinux.org/config/kernel.html?highlight=dracut#kernel-hooks)
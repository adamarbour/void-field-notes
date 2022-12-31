# Void Linux Install Notes

## Goal 1

The purpose of this section is to describe how I installed Void on my device. I am naturally not taking the easy way out here and will be following their advanced installation guides[^1]. This is because I intend to include the following:

- EFISTUB + efibootmgr
- btrfs + LUKS full drive encryption
- signed unified kernel image (UKI)
- Utilize the current LTS kernel

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
```bash
# Before we get started, set the default prompt to indicate we are in the live environment.
PS1='(live) #'
```

1. Create the partitions
   ```bash
   # List block storage devices (mine is nvme0n1)
   lsblk -o NAME,PATH
   
   # I prefer gdisk
   xbps-install -Sy gptfdisk
   
   # Partition the drive
   gdisk /dev/nvme0n1
   > o		# Create a new empty GPT
   > Y
   > n		# Add a new partition
   > (default)
   > (default)
   > +550M
   > ef00	# Set to EFI system partition
   > c		# Change a partition's name
   > EFI
   > n
   > (default)
   > (default)
   > (default)
   > (default)
   > c		# Change a partition's name
   > 2		# Change the second partition
   > ROOT
   > p		# Check the layout
   > w		# Write to disk
   > Y
   
   # Check to make sure the new partitions are there
   lsblk -o NAME,SIZE,UUID,LABEL
   ```

2. Format the EFI partition
   ```bash
   mkfs.vfat -F32 -n EFI /dev/nvme0n1p1
   ```

3. Create the crypt LUKs device for us to store our root file system on
   ```bash
   cryptsetup luksFormat --type=luks2 --label=cryptdevice /dev/nvme0n1p2
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
   ```

   This is just for my reference. You can do the same but this is not required.

   ```bash
   # Noting the output for reference
   lsblk -o NAME,LABEL,UUID
   
   ################ IGNORE ################
   > /dev/nvme0n1p1	EFI
   > /dev/nvme0n1p2	cryptdevice
   > /dev/mapper/luks	ROOT
   ```

   ```bash
   # Mount the device to create sub-volumes
   mount /dev/mapper/luks /mnt	
   
   # Create the sub-volumes
   btrfs sub create /mnt/@
   btrfs sub create /mnt/@boot
   btrfs sub create /mnt/@swap
   btrfs sub create /mnt/@home
   btrfs sub create /mnt/@snapshots
   btrfs sub create /mnt/@log
   btrfs sub create /mnt/@cache
   btrfs sub create /mnt/@tmp
   
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
   mkdir -p efi home var/cache var/log .snapshots swap boot tmp
   
   # Mount the subvolumes
   mount -o $OPT_DEFAULT,subvol=@home /dev/mapper/luks /mnt/home
   mount -o $OPT_DEFAULT,subvol=@snapshots /dev/mapper/luks /mnt/.snapshots
   mount -o $OPT_DEFAULT,subvol=@log /dev/mapper/luks /mnt/var/log
   mount -o $OPT_DEFAULT,subvol=@cache /dev/mapper/luks /mnt/var/cache
   mount -o $OPT_DEFAULT,subvol=@tmp /dev/mapper/luks /mnt/tmp
   mount -o $OPT_DEFAULT,subvol=@swap /dev/mapper/luks /mnt/swap
   mount -o $OPT_DEFAULT,subvol=@boot /dev/mapper/luks /mnt/boot
   
   # Mount the EFI partition
   mount -o defaults /dev/nvme0n1p1 /mnt/efi
   
   # Check the mount points
   lsblk -o NAME,MOUNTPOINTS
   ```

6. Setup the swap file
   ```bash
   cd /mnt/swap
   truncate -s 0 swapfile
   chattr +C swapfile
   fallocate -l 16G swapfile # Same as my system memory for hibernation
   chmod 0600 swapfile
   mkswap -L SWAP swapfile
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
   XBPS_ARCH=$ARCH xbps-install -Sy -r /mnt -R "$REPO" base-system base-devel linux-lts linux-lts-headers
   # Needed for my setup
   XBPS_ARCH=$ARCH xbps-install -Sy -r /mnt -R "$REPO" btrfs-progs cryptsetup gummiboot-efistub efibootmgr dracut dracut-uefi sbctl sbsigntool efitools openssl binutils iwd socklog-void lz4 lzip
   # Extra
   XBPS_ARCH=$ARCH xbps-install -Sy -r /mnt -R "$REPO" nano wget curl git xtools
   ```

3. Generate the `/etc/fstab` file.
   _NOTE: void doesn't include anything that generates the file, so it is a manual process. BUT..._

   ```bash
   # Install some stuff
   xbps-install -Sy nano wget
   
   # Grab the Arch genfstab standalone from this repository
   wget https://raw.githubusercontent.com/adamarbour/void-linux-genfstab/master/genfstab -O /sbin/genfstab
   chmod +x /sbin/genfstab
   
   # Generate the fstab using labels. Use -U for UUID.
   genfstab -L /mnt >> /mnt/etc/fstab
   
   # Check the result
   cat /mnt/etc/fstab
   ```

Now we are ready to setup the bootloader.

#### Chroot and finish the initial set of configurations

1. Before we dive-in we need to get a few things loaded. None of our services are running against this new installation.
   ```bash
   # So we can install packages in the chroot
   cp /etc/resolv.conf /mnt/etc
   ```

2. We also need to mount the pseudo-filesystems
   ```bash
   mount --rbind /sys /mnt/sys && mount --make-rslave /mnt/sys
   mount --rbind /dev /mnt/dev && mount --make-rslave /mnt/dev
   mount --rbind /proc /mnt/proc && mount --make-rslave /mnt/proc
   ```

3. Change to the new root file system using `xchroot`
   ```bash
   xchroot /mnt
   ```

4. Do the minimum required configurations
   ```bash
   chown root:root /
   chmod 755 /
   
   # Set the hostname
   echo "$HOSTNAME" > /etc/hostname
   
   # Set the timezone. Set this to your timezone
   ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
   
   # Change the swappiness to swap less
   echo vm.swappiness=10 > /usr/lib/sysctl.d/99-swappiness.conf
   
   # Set a root password
   passwd
   ```

5. A minor modification to the fstab file
   ```bash
   nano /etc/fstab
   
   # Replace this line...
   LABEL=EFI               /efi            vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro       0 2
   
   # With this...
   LABEL=EFI               /efi            vfat            defaults        0 0
   ```

#### Setup bootloader

The instructions on the Void website include setting up GRUB, but for my installation, I intend to boot directly from UEFI using the `efibootmgr`. I've adapted this from an earlier setup to use here. In our next goal, we will clean this up. For now, I want to be able to boot up the device.

1. Let's generate EFI keys for this device and enroll them into the EFI. [^3]

   ```bash
   # A handy script for generating efi keys. We will need this for the bootloader
   wget https://raw.githubusercontent.com/adamarbour/void-linux-efi-mkkeys/master/efi-mkkeys -O /sbin/efi-mkkeys
   chmod +x /sbin/efi-mkkeys
   
   # Generate keys for our new system
   efi-mkkeys -s "VOID" -o /etc/efi-keys
   
   # Import the keys into sbctl to enroll them to the EFI
   sbctl create-keys
   sbctl import-keys --force \
   	--db-cert /etc/efi-keys/db.crt \
   	--db-key /etc/efi-keys/db.key \
   	--kek-cert /etc/efi-keys/KEK.crt \
   	--kek-key /etc/efi-keys/KEK.key \
   	--pk-cert /etc/efi-keys/PK.crt \
   	--pk-key /etc/efi-keys/PK.key
   	
   # Enroll the successfully imported keys
   sbctl enroll-keys -im
   
   # Setup Mode should be disabled and the keys are ready to go
   sbctl status
   ```

2. Create a basic `dracult.conf.d` file that we will tweak later. For now we just want to get the system booted. [^5][^6]_

   ```bash
   # DEFAULTS
   nano /etc/dracut.conf.d/00-dracut-defaults.conf
   
   ################ CONTENTS OF FILE ################
   hostonly=yes
   hostonly_cmdline=no
   compress="lz4"
   early_microcode=no
   show_modules=yes
   
   uefi=yes
   uefi_stub=/usr/lib/gummiboot/linuxx64.efi.stub
   uefi_secureboot_cert=/etc/efi-keys/db.crt
   uefi_secureboot_key=/etc/efi-keys/db.key
   ```

   ```bash
   # MODULES - NOTE: Laying framework for future modifications
   nano /etc/dracut.conf.d/01-dracut-modules.conf
   
   ################ CONTENTS OF FILE ################
   #dracutmodules+=''		# Exclusive
   #add_dracutmodules+=''	# Inclusive
   #omit_dracutmodules+=''	# Omit
   
   #filesystems+=''	# Exclusive
   
   #force_drivers+=''	# Load early
   #drivers+=''		# Exclusive
   #add_drivers+=''	# Inclusive
   #omit_drivers+=''	# Omit
   
   #install_items+=''	# Additional binaries
   ```

   ```bash
   # KERNEL COMMAND LINE
   nano /etc/dracut.conf.d/99-dracut-cmdline.conf
   
   ################ CONTENTS OF FILE ################
   CRYPTDEVICE=$(blkid -L cryptdevice)
   CMDLINE=(
   	rw
   	rd.timeout=30
   	rd.luks=1
   	rd.luks.timeout=30
   	rd.luks.crypttab=0
   	rd.luks.uuid=$(cryptsetup luksUUID $CRYPTDEVICE)
   	root=LABEL=ROOT
   	rootflags=subvol=@
   )
   kernel_cmdline="${CMDLINE[*]}"
   unset CRYPTDEVICE
   unset CMDLINE
   ```

3. Create the `efi stub`
   ```bash
   xbps-reconfigure -fa
   
   # Create the efi kernel image using the LTS kernel
   dracut --force --kver 5.10.159_1
   ```

4. Let's copy the efi to our target `mnt` and add it to the boot manager.

   ```bash
   # Add the kernel to the boot manager
   efibootmgr --create --disk /dev/nvme0n1 --label "Void Linux (LTS)" --loader 'EFI\Linux\linux-5.10.159_1.efi' --unicode
   
   # Validate that it was loaded
   efibootmgr --unicode
   
   # Change the boot order. Modify for your needs.
   efibootmgr -o 0000,0001,0002,0003,...
   ```

   _NOTE: Later we will modify the Kernel hooks to take care of signing and updating the boot manager.[^4]_

#### The moment we've been waiting for...

1. Close everything out
   ```bash
   exit	# Exit chroot
   cd / && swapoff /mnt/swap/swapfile
   umount -R /mnt
   ```

2. Now we pray...
   ```bash
   reboot
   ```

## Tada!!

We should now be booted into our new system. You will have to enter your LUKs password before you can login. There are a few more things we need to do before we can move on.

1. I need to get the internet working using iwd (iNet wireless daemon) - my preferred network management.
   _NOTE: You could use the wpa_supplicant by enabling the service and performing the same steps at the beginning of this guide._
2. Install ufw for easy firewall management
3. Enable sshd so we can login when we need to and only allow keys
4. 0We should not log in via the root user and instead, use `sudo` to elevate privileges temporarily.
5. Setup system logging because Void does not come with a syslog daemon.
6. Enable the nonfree repository for future enhancements to our system

### Enable Logging

One of the first things we need to do is enable logging. We convnentiently added the `socklog` package during our installation. All we need to do now is just enable the services.

1. Enable the services following the _Services and Daemons - runit_ section of the handbook[^8].
   ```bash
   # Enable the socklog-unix and nanoklogd services
   ln -s /etc/sv/socklog-unix /var/service
   ln -s /etc/sv/nanoklogd /var/service
   ```

2. Validate the services are running and that there is a `socklog` folder created for logging
   ```bash
   # Check the status
   sv status socklog-unix
   sv status nanoklogd
   
   # Make sure the folder is created
   ls -l /var/log/socklog
   ```

3. We can now check logs using the `svlogtail` command

### Wifi Setup with IWD & DHCPCD

We used `wpa_supplicant` during the installation process but for my purposes, I would prefer to use iwd (iNet wireless daemon). We included this in our installation extras when we included `iwd`. However, the whole thing is lying dormant, and if you run iwctl, it tells you that it is waiting for IWD to start.

If you rtfd the IWD documentation[^7] and the Services and Daemons - runit section, you'll get a good idea of what we're going to do.

1. Create an iwd configuration file.[^9]

   ```bash
   nano /etc/iwd/main.conf
   
   ### CONTENTS OF FILE ###
   [General]
   UseDefaultInterface=true
   
   [Network]
   NameResolvingService=resolvconf
   ```

2. Create an additional resolv.conf file to include the Cloudflare dns (optional)
   ```bash
   nano /etc/resolv.conf.head
   
   ### CONTENTS OF FILE ###
   nameserver 1.1.1.2
   nameserver 1.0.0.2
   ```

3. Enable the `dbus` and `iwd` services with `runit`

   ```bash
   ln -s /etc/sv/dbus /var/service
   ln -s /etc/sv/iwd /var/service
   ln -s /etc/sv/dhcpcd /var/service
   ```

4. Check if the services are running (start them if they are not). The commands should list their separate pids.
   _NOTE: There might be a wall of text fly up on the screen. You can safely ignore this._

   ```bash
   sv status dbus
   sv status iwd
   sv status dhcpcd
   ```

5. Now we can setup our wireless internet connection.
   ```bash
   # Get your device list
   iwctl device list
   
   # Connect to the same network as before
   iwctl --passphrase=*** station <DEVICE> connect <SSID>
   ```

   This should result in the device _"link becomes ready"_. You're internet is connected!

6. Check and make sure by updating the xbps repository.
   ```bash
   xbps-install -Sy
   ```

7. If all went well, the repository data should be updated and let's go ahead and upgrade void to make sure we are up to date.
   ```bash
   # Install and upgrade the system
   xbps-install -Su
   
   # Check if any services need to be restarted
   xcheckrestart
   ```

   



## References

[^1]: [Advanced Installation Guides - Void Linux Handbook](https://docs.voidlinux.org/installation/guides/index.html)
[^2]: [The XBPS Method](https://docs.voidlinux.org/installation/guides/chroot.html#the-xbps-method)
[^3]: [EFI-mkkeys](https://github.com/jirutka/efi-mkkeys/)
[^4]: [Void Kernel Hooks](https://docs.voidlinux.org/config/kernel.html?highlight=dracut#kernel-hooks)
[^5]: [dracut.conf.5(gz) - Void Linux manpages](https://man.voidlinux.org/dracut.conf.5)
[^6]: [dracut.cmdline.7(gz) - Void Linux manpages](https://man.voidlinux.org/dracut.cmdline.7)
[^7]: [IWD - Void Linux Handbook](https://docs.voidlinux.org/config/network/iwd.html)
[^8]: [Services and Daemons - runit - Void Linux Handbook](https://docs.voidlinux.org/config/services/index.html)
[^9]: [iwd.config.5(gz) - Void Linux manpages](https://man.voidlinux.org/iwd.config.5)


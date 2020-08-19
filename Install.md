# Initial install

The first steps of the installation process are not taken care by ansible, so I'm documenting in here what I did.
The [Arch installation guide](https://wiki.archlinux.org/index.php/installation_guide)
contains useful, more detailed instructions.

Prepare the Arch live usb and boot it.

Verify we are in efi mode:

```bash
# ls /sys/firmware/efi/efivars
```

Ensure we have connectivity, then that the system clock is accurate with:

```bash
# timedatectl set-ntp true
```

Prepare the unformatted root partition ( `/dev/sdb3` ) with [parted](https://wiki.archlinux.org/index.php/Parted).

Format the partition with [LUKS](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition):

```bash
# cryptsetup -y -v luksFormat /dev/sdb3
# cryptsetup open /dev/sdb3 CryptRoot
# mkfs.ext4 /dev/mapper/CryptRoot
```

Mount the root and the boot partitions:

```bash
# mount /dev/mapper/CryptRoot /mnt
# mkdir /mnt/boot
# mount /dev/sda2 /mnt/boot
```

Check pacman mirrors, install the base system and generate fstab; finally chroot in:

```bash
# pacstrap /mnt base linux linux-firmware
# genfstab -U /mnt >> /mnt/etc/fstab
# arch-chroot /mnt
```

Set the timezone:

```bash
# ln -sf /usr/share/zoneinfo/Europe/Rome /etc/localtime
```

Run hwclock to generate `/etc/adjtime`:

```bash
# hwclock --systohc
```

Generate locales, by editing `/etc/locale.gen` and decommenting utf-8 locales for
both eng and it, then run:

```bash
# locale-gen
```

Finally set the locale in `/etc/locale.conf`:

```bash
LANG=en_US.UTF-8
```

Prepare the network by setting up `/etc/hostname` ...

```bash
flyingcik
```

... and `/etc/hosts`:

```bash
127.0.0.1	    localhost
::1		        localhost
127.0.1.1	    flyingcik.localdomain	flyingcik
```

Modify `/etc/mkinitcpio.conf` adding `keyboard`, `keymap` and `encrypt` hooks:

```bash
HOOKS=(base udev autodetect keyboard keymap modconf block encrypt filesystems keyboard fsck)
```

Create the new initramfs by running:

```bash
# mkinitcpio -P
```

Set the root password.

Install the actual [bootloader in UEFI mode](https://wiki.archlinux.org/index.php/GRUB#UEFI_systems).

```bash
# grub-install --target=x86_64-efi --efi-directory=/mnt/boot --bootloader-id=GRUB
```

Configure grub to [manage the encrypted root](https://wiki.archlinux.org/index.php/Dm-crypt/System_configuration#Boot_loader) and to allow the use of my secret key on usb by setting kernel parameters in `/etc/default/grub`:

```bash
GRUB_CMDLINE_LINUX="cryptdevice=UUID=uuid_of_the_ENCRYPTED_partition:CryptRoot cryptkey=UUID=uuid_of_the_key_partition:ext4:/key_path"
```

Install `os-prober` so Windows (sigh) can be detected correctly.

Write the grub configuration file:

```bash
# grub-mkconfig -o /boot/grub/grub.cfg
```

Save the secret key from the usb on the disk, then prepare the mounting of any other encrypted partition adding lines to `/etc/crypttab`:

```bash
CryptData   UUID=uuid_of_the_ENCRYPTED_partition    /path/to/key/on/system
CryptBackup   UUID=uuid_of_the_ENCRYPTED_partition    /path/to/key/on/system
```

Then add these volumes to `/etc/fstab`:

```bash
/dev/mapper/CryptData   /home/data  ext4    rw,relatime,data=ordered    0   2
/dev/mapper/CryptBackup   /home/backup  ext4    rw,relatime,data=ordered    0   2
```

Now exit the chroot, unmount and reboot. Once logged in, fix network, if not working:

```bash
# ip link set eno1 up
# ip addr add 192.168.1.135/24 dev eno1
# ip route add default via 192.168.1.1
# echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Install `openssh` and `dhcpcd`:

```bash
# pacman -Sy openssh dhcpcd
# systemctl start openssh
# systemctl enable openssh
```

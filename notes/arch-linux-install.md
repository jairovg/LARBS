# Install Arch Linux
## Pre-install
* [Download](https://www.archlinux.org/download/) the latest arch linux ISO.
* Verify signature

```
$ gpg --keyserver-options auto-key-retrieve --verify archlinux-version-x86_64.iso.sig
```

### Create live ISO
  * Find out the name of your USB drive with `lsblk`. Make sure that it is **not** mounted.
  * Run the following command, replacing `/dev/sd`**`x`** with your drive, e.g. `/dev/sdb`. (Do **not** append a partition number, so do **not** use something like `/dev/sdb`**`1`**)

    ```
    # dd bs=4M if=path/to/archlinux.iso of=/dev/sdx status=progress oflag=sync
    ```

* Boot the live ISO

### Set the keyboard layout
  * Use `localectl status` to view the current keyboard configurations.
  * The default [console keymap](https://wiki.archlinux.org/index.php/Console_keymap) is US.
  * To list the all the available layouts

  ```
  # ls /usr/share/kbd/keymaps/**/*.map.gz
  ```

  * Modify the layout to [spanish latam](https://en.wikipedia.org/wiki/QWERTY#Latin_America,_officially_known_as_Spanish_Latinamerican_sort).

  ```
  loadkeys la-latin1
  ```

### Verify the boot mode
To verify this, list the [efivars](https://wiki.archlinux.org/index.php/UEFI#UEFI_variables) directory

```
# ls /sys/firmware/efi/efivars
```

### Connect to the internet
#### Wireless
Verify the interface:

```
# lspci -k
```

TODO: In virtualbox it's ready setup

### Update the system clock
Use [timedatectl(1)](https://jlk.fjfi.cvut.cz/arch/manpages/man/timedatectl.1) to ensure the system clock is accurate:

```
# timedatectl set-ntp true
```

To check the service status, use `timedatectl status`.

### Partition the disks
To identify the block devices

```
# lsblk -f
```

To create an EFI System Partition, use the following commands

```
# parted /dev/nvme0n1
(parted) mklabel gpt
(parted) mkpart primary fat32 1MiB 315MiB
(parted) set 1 esp on
(parted) mkpart primary ext4 315MiB 100%
(parted) quit
```

**NOTE**: This steps does not consider a swap partition but a [swap file](https://wiki.archlinux.org/index.php/Swap#Swap_file).

### Format the partitions
Once the partitions have been created, each must be formatted with an appropriate [file system](https://wiki.archlinux.org/index.php/File_system).

```
# mkfs.vfat -F 32 /dev/nvme0n1p1
# mkfs.ext4 /dev/nvme0n1p2
```

### Mount the file systems
[Mount](https://wiki.archlinux.org/index.php/Mount) the file system on the root partition to `/mnt`.

```
# mount /dev/nvme0n1p2 /mnt
```

Mount the boot to `/mnt/boot/efi`:

```
# mkdir -p /mnt/boot/efi
# mount /dev/nvme0n1p1 /mnt/boot/efi
```

## Installation
### Select the mirrors
**NOTE:** The latest ISO don't have the `rankmirror` script, it has been moved to `pacman-contrib` package. So ignore thi part.

Packages to be installed must be downloaded from [mirror servers](https://wiki.archlinux.org/index.php/Mirrors), which are defined in `/etc/pacman.d/mirrorlist`. This file will later be copied to the new system by _pacstrap_, so it is worth getting right.

Sort the mirrors by speed uncommenting every mirror

```
# cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
# sed -i 's/^#Server/Server/' /etc/pacman.d/mirrorlist.backup
# rankmirrors -n 8 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist
```

**NOTE**: After the installation, when updating the packages using `pacman -Syyu` flags forces pacman to refresh all package lists even if they are considered to be up to date.

### Install essential packages
Use the [pacstrap](https://projects.archlinux.org/arch-install-scripts.git/tree/pacstrap.in) script to install the [base](https://www.archlinux.org/groups/x86_64/base/) and [base-devel](https://www.archlinux.org/groups/x86_64/base-devel/) group packages, Linux [kernel](https://wiki.archlinux.org/index.php/Kernel) and firmware for common hardware.

Also it should be installed packages for a or a fully functional base system. In particular, consider installing:

* userspace utilities for the management of file systems that will be used on the system,
* utilities for accessing RAID or LVM partitions,
* specific firmware for other devices not included in linux-firmware,
* software necessary for networking,
* a text editor,
* packages for accessing documentation in man and info pages: man-db, man-pages and texinfo.

```
# pacstrap /mnt base base-devel linux linux-lts linux-firmware lshw e2fsprogs dosfstools ntfs-3g vim xclip man-db man-pages texinfo dhcpcd dhclient iptables python iw networkmanager
```

#### Microcode
Processors may have [faulty behaviour](https://www.anandtech.com/show/8376/intel-disables-tsx-instructions-erratum-found-in-haswell-haswelleep-broadwelly), which the kernel can correct by updating the microcode on startup. See [Microcode](https://wiki.archlinux.org/index.php/Microcode) for details.

Depending on the processor, it will be required to add to the pacstrap script or [amd-ucode](https://www.archlinux.org/packages/?name=amd-ucode) or [intel-ucode](https://www.archlinux.org/packages/?name=intel-ucode) package.

`grub-mkconfig` will automatically detect the microcode update and configure [GRUB](https://wiki.archlinux.org/index.php/GRUB) appropriately. After installing the microcode package, regenerate the GRUB config to activate loading the microcode update by running:

```
# grub-mkconfig -o /boot/grub/grub.cfg
```

This script will be [run in the install process later](#boot-loader).

**NOTE:** The related part of the [enabling early microcode loading in custom kernels](https://wiki.archlinux.org/index.php/Microcode#Enabling_early_microcode_loading_in_custom_kernels) is a config located in the kernel code, like [linux-lts](https://git.archlinux.org/svntogit/packages.git/tree/trunk/config?h=packages/linux-lts).

## Configure the system
Generate an [fstab](https://wiki.archlinux.org/index.php/Fstab) file:

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

**NOTE:** Check first if the `/boot/efi` had not being unmounted using `lsblk`. If it had, then mounted back again and generate the `fstab` later.

## Chroot
[Change root](https://wiki.archlinux.org/index.php/Change_root) into the new system

```
# arch-chroot /mnt
```

## Time zone
Set the [time zone](https://wiki.archlinux.org/index.php/Time_zone)

```
# ln -sf /usr/share/zoneinfo/America/Bogota /etc/localtime
```

or

```
# timedatectl set-timezone America/Bogota
```

Run [hwclock(8)](https://jlk.fjfi.cvut.cz/arch/manpages/man/hwclock.8) to generate /etc/adjtime:

```
# hwclock --systohc
```

This command assumes the hardware clock is set to [UTC](https://en.wikipedia.org/wiki/UTC). See [System time#Time standard](https://wiki.archlinux.org/index.php/System_time#Time_standard) for details.

## Localization
Uncomment en_US.UTF-8 UTF-8 and other needed locales in `/etc/locale.gen`, and generate them with:

```
# { echo "en_US.UTF-8 UTF-8" ; echo "es_CO.UTF-8 UTF-8"; } >> /etc/locale.gen
# locale-gen
```

Create the [locale.conf(5)](https://jlk.fjfi.cvut.cz/arch/manpages/man/locale.conf.5) file, and set the [LANG](https://wiki.archlinux.org/index.php/Variable) variable accordingly:

```
/etc/locale.conf
LANG=en_US.UTF-8
LC_ADDRESS=es_CO.UTF-8
LC_IDENTIFICATION=es_CO.UTF-8
LC_MEASUREMENT=es_CO.UTF-8
LC_MONETARY=es_CO.UTF-8
LC_NAME=es_CO.UTF-8
LC_NUMERIC=es_CO.UTF-8
LC_PAPER=es_CO.UTF-8
LC_TELEPHONE=es_CO.UTF-8
LC_TIME=es_CO.UTF-8
```

If you set the [keyboard layout](https://wiki.archlinux.org/index.php/Installation_guide#Set_the_keyboard_layout), make the changes persistent in [vconsole.conf(5)](https://jlk.fjfi.cvut.cz/arch/manpages/man/vconsole.conf.5):

```
/etc/vconsole.conf
KEYMAP=la-latin1
FONT=
FONT_MAP=
```

## Network configuration
Create the [hostname](https://wiki.archlinux.org/index.php/Hostname) file:

```
/etc/hostname
<hostname>
```

Add matching entries to [hosts(5)](https://jlk.fjfi.cvut.cz/arch/manpages/man/hosts.5):

```
/etc/hosts
127.0.0.1	localhost
::1		    localhost
127.0.1.1	<hostname>.localdomain	<hostname>
```
If the system has a permanent IP address, it should be used instead of `127.0.1.1`.

Complete the [network configuration](https://wiki.archlinux.org/index.php/Network_configuration) for the newly installed environment.

Enable NetworkManager service:

```
# systemctl enable NetworkManager.service
```

## Initramfs
Creating a new _initramfs_ is usually not required, because [mkinitcpio](https://wiki.archlinux.org/index.php/Mkinitcpio) was run on installation of the [kernel](https://wiki.archlinux.org/index.php/Kernel) package with _pacstrap_.

## Root password
Set the root [password](https://wiki.archlinux.org/index.php/Password):

```
# passwd
```

## Additional user
Create a standard user account. Replace `<username>` with your preferred username.

```
# useradd -m -g users -G wheel,storage,power -s /bin/bash <username>
```

Set a password for this user.

```
passwd <username>
```

Allow member of wheel group to use sudo.

```
EDITOR=vim visudo
```

Find this line.

```
# %wheel ALL=(ALL) ALL
```

Remove the # sign and save this file.

Allows the wheel users to run this commands without password: Add to the end

```
%wheel ALL=(ALL) NOPASSWD: /usr/bin/shutdown,/usr/bin/reboot,/usr/bin/systemctl suspend,/usr/bin/wifi-menu,/usr/bin/mount,/usr/bin/umount,/usr/bin/pacman -Syyuw --noconfirm"
```

## Boot loader
Intall [GRUB](https://wiki.archlinux.org/index.php/GRUB):

```
# pacman -S grub efibootmgr
```

Run the [grub-install(8)](https://jlk.fjfi.cvut.cz/arch/manpages/man/grub-install.8) command:

```
# grub-install /dev/nvme0n1 --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
```

Use the _grub-mkconfig_ tool to generate `/boot/grub/grub.cfg`:

```
# grub-mkconfig -o /boot/grub/grub.cfg
```

## Reboot
Exit the chroot environment:

```
# exit
```

Unmount all the partitions:

```
# umount -R /mnt
```

**NOTE:** If there is any _busy_ partition, then identify it using [fuser(1)](https://jlk.fjfi.cvut.cz/arch/manpages/man/fuser.1).

Reboot:

```
# reboot
```

**NOTE:** any partitions still mounted will be automatically unmounted by _systemd_. Remember to remove the installation media and then login into the new system with the root account.

# Post-installation
## Swapfile
First create and intialize the file to hold the swap. For example, to create a 4GB swapfile, you could use the command:

```
sudo fallocate -l 4G /swapfile
sudo mkswap /swapfile
```

Set the appropriate permissions on the file. It should be readable and writable only by root. This can be done with the command:

```
sudo chmod u=rw,go= /swapfile
```

Next we need to enable the swapfile with the swapon command. Following our example above this could be done with:

``
sudo swapon /swapfile
``

In order to ensure that the swap is enabled at boot we can add an entry to /etc/fstab. You can add the line to ftab manually or using the command:

```
sudo bash -c "echo /swapfile none swap defaults 0 0 >> /etc/fstab"
```

## Pacman wrapper - yay
yay is in the AUR repository, so it needs a manual install:

```
$ cd /tmp
$ curl -s0 https://aur.archlinux.org/cgit/aur.git/snapshot/yay.tar.gz | sudo -u <username> tar -zxv
$ cd yay
$ sudo -u <username> makepkg --noconfirm -si
```

## Graphical environment
### i3wm
Install X.org (X):

```
# yay -S xorg-server xorg-xinit xorg-xprop xorg-xwininfo
```

Install i3-gaps and related:

```
# yay -S i3-gaps i3status i3blocks
# yay -S sxhkd rxvt-unicode urxvt-perls urxvt-resize-font-git compton rofi dunst calcurse xdotool htop-vim-git the_silver_searcher tmux bc
```

Install browser manager and related

```
# yay -S ranger w3m mediainfo poppler atool highlight odt2txt
```

Install network manager and fonts

```
# yay -S network-manager-applet
# yay -S ttf-linux-libertine ttf-inconsolata ttf-symbola ttf-joypixels ttf-font-awesome powerline-fonts-git
```

Make the X Server start `i3` when it starts:

```
~/.xinitrc
exec i3
```

Setup proper kayboard layout for X11

```
# localectl --no-convert set-x11-keymap latam pc105
```

Install `lightdm` display manager:

```
# yay -S lightdm lightdm-webkit2-greeter lightdm-webkit-theme-litarvan
# systemctl enable lightdm.service
```
```
/etc/lightdm/lightdm.conf
[Seat:*]
greeter-session=lightdm-webkit2-greeter
```

```
/etc/lightdm/lightdm-webkit.conf
webkit-theme=litarvan
```

### Audio
Install audio controllers

```
yay -S pulseaudio pulseaudio-alsa pulsemixer
```

Install music related packages

```
yay -S mpd mpc ncmpcpp socat
```

Install: castero

### RSS
Install newsboat and related

```
# yay -S newsboat urlscan w3m mpv sxiv youtube-dl feh
```

### Cronjobs
Install `cronie` and start it

```
# yay -S cronie
# systemctl enable --now cronie
```

### Dotfiles
```
git clone --bare https://github.com/<username>/dotfiles.git $HOME/.cfg
alias config='/usr/bin/git --git-dir=$HOME/.cfg/ --work-tree=$HOME'
config checkout
config config --local status.showUntrackedFiles no
```

# Configuration to check
* Block devices `lsblk -f`

  ```
  NAME        FSTYPE MOUNTPOINT
  nvme0n1
  ├─nvme0n1p1 vfat   /boot/efi
  └─nvme0n1p2 ext4   /
  ```
* Partitions table `parted /dev/nvme0n1 print`

  ```
  Model: Unknown (unknown)
  Disk /dev/nvme0n1: 256GB
  Sector size (logical/physical): 512B/512B
  Partition Table: gpt
  Disk Flags:

  Number  Start   End    Size   File system  Name  Flags
  1      2097kB  317MB  315MB  fat32              boot, esp
  2      317MB   256GB  256GB  ext4
  ```

* `/etc/fstab`

  ```
  # /etc/fstab: static file system information.
  #
  # Use 'blkid' to print the universally unique identifier for a device; this may
  # be used with UUID= as a more robust way to name devices that works even if
  # disks are added and removed. See fstab(5).
  #
  # <file system>                           <mount point>  <type>  <options>        <dump>  <pass>
  UUID=5973-8D97                            /boot/efi      vfat    defaults,noatime 0 2
  UUID=4997007a-d66b-498e-bec8-f5c122662a4a /              ext4    defaults,noatime 0 1
  /swapfile none swap defaults 0 0
  ```

  * `/etc/locale.conf`
  ```
  LANG=en_US.UTF-8
  ```

  * `/etc/vconsole.conf`
  ```
  KEYMAP=la-latin1
  FONT=
  FONT_MAP=
  ```

  * `/etc/default/grub`
  ```
  GRUB_DEFAULT=saved
  GRUB_TIMEOUT=5
  GRUB_TIMEOUT_STYLE=menu
  GRUB_DISTRIBUTOR='Manjaro'
  GRUB_CMDLINE_LINUX_DEFAULT="quiet"
  GRUB_CMDLINE_LINUX=""

  # If you want to enable the save default function, uncomment the following
  # line, and set GRUB_DEFAULT to saved.
  GRUB_SAVEDEFAULT=true

  # Preload both GPT and MBR modules so that they are not missed
  GRUB_PRELOAD_MODULES="part_gpt part_msdos"

  # Uncomment to enable booting from LUKS encrypted devices
  #GRUB_ENABLE_CRYPTODISK=y

  # Uncomment to use basic console
  GRUB_TERMINAL_INPUT=console

  # Uncomment to disable graphical terminal
  #GRUB_TERMINAL_OUTPUT=console

  # The resolution used on graphical terminal
  # note that you can use only modes which your graphic card supports via VBE
  # you can see them in real GRUB with the command 'videoinfo'
  GRUB_GFXMODE=auto

  # Uncomment to allow the kernel use the same resolution used by grub
  GRUB_GFXPAYLOAD_LINUX=keep

  # Uncomment if you want GRUB to pass to the Linux kernel the old parameter
  # format "root=/dev/xxx" instead of "root=/dev/disk/by-uuid/xxx"
  #GRUB_DISABLE_LINUX_UUID=true

  # Uncomment to disable generation of recovery mode menu entries
  GRUB_DISABLE_RECOVERY=true

  # Uncomment and set to the desired menu colors.  Used by normal and wallpaper
  # modes only.  Entries specified as foreground/background.
  GRUB_COLOR_NORMAL="light-gray/black"
  GRUB_COLOR_HIGHLIGHT="green/black"

  # Uncomment one of them for the gfx desired, a image background or a gfxtheme
  #GRUB_BACKGROUND="/usr/share/grub/background.png"
  GRUB_THEME="/usr/share/grub/themes/manjaro/theme.txt"

  # Uncomment to get a beep at GRUB start
  #GRUB_INIT_TUNE="480 440 1"
  ```

# Virtualbox
Creating an image using `EFI` will boot to UEFI shell after power off. In order to fix it you will need to do find the boot loader first

```
Shell>bcfg boot dump -v
Shell>FS0:
FS0:>ls
FS0:>cd EFI
FS0:>cd GRUB
FS0:>ls
FS0:>exit
```

Add a new entry based on the previous found GRUB file and after boot (using exit command) selects the new entry.

```
Shell>bcfg boot add 03 FS0:\EFI\GRUB\grubx64.efi "GRUB"
Shell>bcfg boot mv 3 0
Shell>exit
```

Installing the guest additions
```
# pacman -S virtualbox
```

Load the guest addiitions
```
# modprobe -a vboxguest vboxsf vboxvideo
# touch /etc/modules-load.d/virtualbox.conf
```

`/etc/default/grub`
```
vboxguest
vboxsf
vboxvideo
```

# References
* Anrch linux installation guide: https://wiki.archlinux.org/index.php/Installation_guide
* Post installation general guide: https://wiki.archlinux.org/index.php/General_recommendations
* How to Install Arch Linux [Step by Step Guide]: https://itsfoss.com/install-arch-linux
* Luke Smith's .Xdefaults: https://github.com/LukeSmithxyz/voidrice/blob/496aea4504c70fe165faf90db0bf1d95efa00caf/.Xdefaults
* Swapfile: https://wiki.manjaro.org/index.php?title=Add_a_/swapfile
* Virtualbox EFI issue: https://bbs.archlinux.org/viewtopic.php?id=158003
* Simple X Hotkey deamon: https://youtu.be/2ClckQzJTlk
* sxhkdrc sample: https://github.com/svenstaro/dotfiles/blob/master/sxhkd/.config/sxhkd/sxhkdrc
* Wallpaper: https://www.tokkoro.com/picsup/1422752-alice-in-wonderland-2010.jpg
* Rofi config: https://www.youtube.com/watch?v=YMiqNJaqvrk
* Rofi config: https://www.reddit.com/r/unixporn/comments/3ow56l/i3_clean_and_minimal_i3_desktop_inspired_by_os_x/
* Rofi menu to ask for password: https://github.com/davatorium/rofi/issues/584#issuecomment-384555551
* Gruvbox colors config: https://github.com/bardisty/gruvbox-rofi/blob/master/gruvbox-dark.rasi
* Fonts setup: https://jichu4n.com/posts/how-to-set-default-fonts-and-font-aliases-on-linux/
* Rofi fontawesome: https://github.com/wstam88/rofi-fontawesome
* Unix / Linux - Regular Expressions with SED: https://www.tutorialspoint.com/unix/unix-regular-expressions.htm
* Vim cheat sheet: https://vim.rtorr.com/
* Vim cheat sheet: https://vimsheet.com/
* Use vim help: https://vim.fandom.com/wiki/Learn_to_use_help
* Git log alias: https://stackoverflow.com/a/34467298

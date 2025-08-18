
# Arch Linux on encrypted ZFS Root Installation Guide

Early stage guide of capturing steps for a custom Arch Linux installation process 
to enable ZFS, with native encryption, as the root file system.

The application ZFSBootMenu is used to manage the boot process.

Steps will be documented at some point in the future.

## Create Live Installation ISO

Alpine Extended

### Download

### Image

## Boot Live ISO

Login as root. No password.

### Setup Network Devices

```
$ setup-interfaces -r
```

```
$ rc-service networking restart
```

```
$ ping archlinux.org
```

### Add Support Packages 

```
$ setup-apkrepos
$ apk update
```

```
$ apk add sgdisk parted curl wget util-linux vim nano zfs efibootmgr zstd tar
```

### Partition Disks

```
export DISK="/dev/nvme0n1"
export EFI_PART="p1"
export ZFS_PART="p2"
```

```
wipefs --all --force "$DISK"
```

```
$ sgdisk -n 1:0:+1G -t 1:ef00 -c 1:"EFI: System Partition" $DISK
$ sgdisk -n 2:0:0 -t 2:bf00 -c 2:"ZFS Pool Member" $DISK
```

```
$ partx -u $DISK
$ mdev -s
```

### Erase EFI Boot Entries

```
efibootmgr

# All boot entries will be deleted. Ensure this is ok!

efibootmgr \
  | grep -E "^Boot[0-9]{4}" \
  | awk '{print $1}' \
  | sed 's/Boot//g; s/\*//g' \
  | while read -r ENTRY; do efibootmgr -b $ENTRY -B; done
```

### Create EFI Boot Filesystems

```
$ mkfs.vfat -F32 "$DISK$EFI_PART"
```

### Create ZFS Encryption Key

```
$ mkdir -p /etc/zfs
$ echo "my-encryption-key" > /etc/zfs/zroot.key
$ chmod 000 /etc/zfs/zroot.key
```

### Create ZFS Pool

```
$ modprobe zfs
```


```
$ zpool create \
  -f \
  -o ashift=12 \
  -o autotrim=on \
  -O acltype=posixacl \
  -O xattr=sa \
  -O relatime=on \
  -O compression=lz4 \
  -O encryption=aes-256-gcm \
  -O keylocation=file:///etc/zfs/zroot.key \
  -O keyformat=passphrase \
  -m none \
  zroot \
  $DISK$ZFS_PART
```

Ensure it's correct.

```
$ zpool status
```

### Create ZFS Data Sets

```
$ zfs create -o mountpoint=none zroot/ROOT
$ zfs create -o mountpoint=/ -o canmount=noauto zroot/ROOT/arch
```

```
$ zfs create -o mountpoint=/home zroot/home
```

```
$ zpool set bootfs=zroot/ROOT/arch zroot
```

```
$ zfs umount -a
$ zpool export -a
```

### Mount ZFS datasets for Arch bootstrap

```
$ zpool import -N -R /mnt zroot
$ zfs load-key -a

$ zfs mount zroot/ROOT/arch
$ zfs mount zroot/home

$ df -h | grep "/mnt"
```

### Mount EFI boot partitions

```
$ mkdir -p /mnt/boot/efi
$ mount $DISK$EFI_PART /mnt/boot/efi
```

### Download and extract Arch Bootstrap Image

```
$ curl -L https://geo.mirror.pkgbuild.com/iso/latest/archlinux-bootstrap-x86_64.tar.zst \
  | zstd -d \
  | tar -x -C /mnt --strip-components=1
```

### Prepare Chroot 

```
$ cp /etc/resolv.conf /mnt/etc/resolv.conf
```

```
$ mkdir -p /mnt/etc/zfs && cp /etc/zfs/zroot.key /mnt/etc/zfs/zroot.key
```

```
# Select a mirror from the mirrorlist.
$ vi /mnt/etc/pacman.d/mirrorlist

```

```
$ mount --rbind /proc /mnt/proc
$ mount --rbind /sys /mnt/sys
$ mount --rbind /dev /mnt/dev
```

### Enter Arch Bootstrap Chroot

```
$ chroot /mnt
```

```
$ ln -sf /proc/self/mounts /etc/mtab
```

```
$ pacman-key --init
$ pacman-key --populate archlinux
```

```
$ echo -e "\n[archzfs]\nServer = https://archzfs.com/\$repo/\$arch\nSigLevel = Optional TrustAll" >> /etc/pacman.conf
```

### Install Base Arch Packages

```
$ pacman -Sy \
  base \
  base-devel \
  linux \
  linux-headers \
  intel-ucode \
  linux-firmware \
  sof-firmware \
  fwupd \
  sudo \
  wget \
  curl \
  git \
  zfs-dkms \
  zfs-utils \
  efibootmgr \
  fakeroot \
  debugedit
```

```
$ pacman -Sy \
  vim \
  nano
```

```
$ pacman -Sy \
  networkmanager \
  iwd \
  dhclient 
```

### Configure Base Arch Chroot

```
$ ln -sf /usr/share/zoneinfo/US/Pacific /etc/localtime
$ hwclock --systohc
```

```
$ echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
$ echo "LANG=en_US.UTF-8" > /etc/locale.conf
$ locale-gen
```

```
$ echo "localhost" > /etc/hostname
```

```
$ echo "$DISK$EFI_PART /boot/efi vfat defaults,noatime 0 2" >> /etc/fstab
$ echo "tmpfs /tmp tmpfs rw,nosuid,nodev,noexec,relatime,size=4G 0 0" >> /etc/fstab
```

```
passwd
```

### Prepare ZFS Installation

```
$ zgenhostid $(hostid)
```

```
$ systemctl enable \
  zfs-import-cache \
  zfs-mount \
  zfs.target \
  iwd \
  NetworkManager \
  fwupd
```



```
$ echo 'FILES+=(/etc/zfs/zroot.key)' >> /etc/mkinitcpio.conf
$ sed -i 's|filesystems|zfs filesystems|' /etc/mkinitcpio.conf

$ zfs set org.zfsbootmenu:commandline="rw quiet" zroot/ROOT
$ zfs set org.zfsbootmenu:keysource="" zroot
```

### Download, Build, Install ZFSBootMenu

```
$ useradd -m -G wheel builduser
$ echo "builduser ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```

```
$ su - builduser
```

```
$ git clone https://aur.archlinux.org/perl-boolean.git
$ cd perl-boolean && makepkg -si --noconfirm && cd ~
```

```
$ git clone https://aur.archlinux.org/perl-sort-versions.git
$ cd perl-sort-versions && makepkg -si --noconfirm && cd ~
```

```
$ git clone https://aur.archlinux.org/perl-yaml-pp.git
$ cd perl-yaml-pp && makepkg -si --noconfirm && cd ~
```

```
$ git clone https://aur.archlinux.org/zfsbootmenu.git
$ cd zfsbootmenu && makepkg -si --noconfirm && cd ~
```

```
$ exit
$ userdel -r builduser
$ sed -i '/builduser/d' /etc/sudoers
```

### Configure ZFSBootMenu

```
# See https://docs.zfsbootmenu.org/en/v3.0.x/general/mkinitcpio.html
$ vim /etc/zfsbootmenu/config.yaml
```

```
$ sed -i 's|filesystems|zfs filesystems|' /etc/zfsbootmenu/mkinitcpio.conf
```

### Build Ramdisk

```
$ generate-zbm
```

### Configure EFI Entries

```
$ efibootmgr -c -d "$DISK" -p 1 \
  -L "ZFSBootMenu (NVME1)" \
  -l '\EFI\ZBM\VMLINUZ-LINUX.EFI'
```

### Exit chroot, Unmount ZFS datasets, Reboot

## Post Install (Optional)

```
export $TODAY=$(date +%F)
```

```
$ zfs snapshot -r zroot/ROOT/arch@initial-install-$TODAY
```

```
# https://wiki.archlinux.org/title/NetworkManager

$ nmcli device wifi list
$ nmcli device wifi --ask connect <SSID> 
$ nmcli connection show
```

```
$ pacman -Sy \
  hwinfo \
  man-db \
  less \
  tmux \
  unzip
```

```
$ git clone https://aur.archlinux.org/paru.git
$ cd paru && makepkg -si && cd ~
```

### More Packages (optional)

```
$ pacman -Sy \
  syslog-ng

$ systemctl enable syslog-ng@default.service
$ systemctl start syslog-ng@default.service
```

```
$ pacman -Sy \
  openssh \
  ufw \
  rsync

$ systemctl enable ssh 
$ systemctl start ssh 

$ ufw enable 
$ ufw allow SSH

$ systemctl start ufw 
$ systemctl enable ufw
```

```
$ pacman -Sy \
  neovim 
```

```
$ pacman -Sy \
  jdk-openjdk \
  clojure
```

```
$ pacman -Sy \
  greetd \
  greetd-tuigreet

$ sed -i 's|^command = .*$|command = "tuigreet -c bash"|' /etc/greetd/config.toml

$ systemctl enable greetd
```

```
$ pacman -Sy \
  docker \
  docker-compose

$ systemctl enable docker
$ systemctl start docker
```

```
$ pacman -Sy \
  nvidia-open-dkms \
  nvidia-container-toolkit 
```

```
$ pacman -Sy \
  pipewire \
  pipewire-audio \
  pipewire-jack \
  pipewire-pulse \
  pipewire-session-manager
```

```
$ pacman -Sy \
  bluez \
  bluez-utils \ 
  bluez-tools

$ systemctl enable bluetooth 
$ systemctl start bluetooth

$ bluetoothctl
```

```
$ pacman -Sy \
  cups

$ systemctl enable cups 
$ systemctl start cups
```

```
$ pacman -Sy \
  seatd

$ systemctl enable seatd
$ systemctl start seatd
```

```
$ pacman -Sy \
  wayland \
  wl-clipboard \
  wlr-randr \
  wlroots \
  clipman
```

```
$ pacman -Sy \
  ttf-hack-nerd \
  ttf-font-awesome
```

```
$ pacman -Sy \
  waybar \
  wofi \
  foot \
  kitty
```

```
$ pacman -Sy \
  sway \
  swaylock \
  swayidle \
  swaybg \
  autotiling 
```

```
$ pacman -Sy \
  hyprland \
  hyprcursor \
  hyprgraphics \
  hypridle \
  hyprland-protocols \
  hyprlang \
  hyprlock \
  hyprpaper \
  hyprpolkitagent \
  hyprshot \
  hyprutils \
  hyrpwayland-scanner \
  xdg-desktop-portal-hyprland
```

## Useful tooling

### Snapshot creation before Pacman Install / Upgrade 

/etc/pacman.d/hooks/90-zfs-pre-snapshot.hook:

```
[Trigger]
Operation = Install
Operation = Upgrade
Operation = Remove
Type = Package
Target = *

[Action]
Description = Create recursive ZFS snapshot of zroot/ROOT/arch
When = PreTransaction
Exec = /usr/local/bin/zfs-pre-pacman-snapshot.sh
```

/usr/local/bin/zfs-pre-pacman-snapshot.sh

```
#!/bin/sh

DATASET="zroot/ROOT/arch"
TIMESTAMP=$(date +%F-%T)
SNAPSHOT_NAME="$DATASET@pre-pacman-$TIMESTAMP"

logger -t "zfs-pre-pacman-snapshot" "Creating ZFS snapshot of dataset ${DATASET} at ${SNAPSHOT_NAME}."

zfs snapshot -r "$SNAPSHOT_NAME"
```

/etc/pacman.d/hooks/00-zfs-dkms-guard.hook

```
[Trigger]
Operation = Install
Operation = Upgrade
Type = Path
# target the LTS kernel
#Target = usr/lib/modules/*-lts/vmlinuz
# ...or the mainline one
# Target = !usr/lib/modules/*-rt*arch*/vmlinuz
# Target = usr/lib/modules/*-arch*/vmlinuz
# ...or all of them
Target = usr/lib/modules/*/vmlinuz

[Action]
Description = Avoid ZFS-incompatible kernels
When = PreTransaction
Exec = /usr/local/bin/zfs_dkms_guard.sh
AbortOnFail
NeedsTargets
```

/usr/local/bin/zfs_dkms_guard.sh 

```
#!/usr/bin/env bash

set -eo pipefail
shopt -s inherit_errexit

vercomp() {
    readarray -td. one <<<"$1"
    readarray -td. two <<<"$2"
    for i in 0 1; do
        if [[ "${one[$i]}" -lt "${two[$i]}" ]]; then
            printf -- '-1'
            return
        fi
        if [[ "${one[$i]}" -gt "${two[$i]}" ]]; then
            printf '1'
            return
        fi
    done
    printf '0'
}

# requires ZFS >= 0.8.0
zfs_meta_file="$(grep '/META\b' < <(pacman -Qql zfs-dkms))"
zfs_linux_min="$(awk '/Linux-Minimum/ { print $2 }' "$zfs_meta_file")"
zfs_linux_max="$(awk '/Linux-Maximum/ { print $2 }' "$zfs_meta_file")"

while read -r target_path; do
    kernel_version="$(sed -E 's|.*lib/modules/([[:digit:]]+\.[[:digit:]]+).*|\1|' <<<"$target_path")"
    if [[ "$(vercomp "$kernel_version" "$zfs_linux_min")" -lt 0 ]]; then
        printf 'Kernel version %s is below ZFS minimum compatible version %s!\n' "$kernel_version" "$zfs_linux_min" >&2
        exit 1
    fi
    if [[ "$(vercomp "$kernel_version" "$zfs_linux_max")" -gt 0 ]]; then
        printf 'Kernel version %s is above ZFS maximum compatible version %s!\n' "$kernel_version" "$zfs_linux_max" >&2
        exit 1
    fi
done
```

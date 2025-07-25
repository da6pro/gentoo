# Simple Gentoo installation guide

Below is the short guide to be prepared to install Gentoo on Thinkapd T480.
Considerations:
- encrypted root filesystem
- bassed on LVM (vg0)
- profile desktop on systemd
- newest kernel
- working sound
- working graphics and XOrg
- working firewalld
- working network through wifi wlp3s0 interface; dhcp (static ip from router)
- ccache used as feature to compile the things
- installed and configured Xorg without selected X Window Manager

# Boot from livecd minimal

Create pendrive with Gentoo Live image minimal and boot it.

# Configure network

```
wpa_passphrase WIFISSID password >/etc/wpa_supplicant/wpa_supplicant-wlp3s0.conf
wpa_supplicant -c /etc/wpa_supplicant/wpa_supplicant-wlp3s0.conf -i wlp3s0 -f /var/log/wpa_supplicant.log -B
dhcpcd wlp3s0
ip a
```

# Partitioning

Layout:

```
gentoo ~ # fdisk -l
Disk /dev/nvme0n1: 238.47 GiB, 256060514304 bytes, 500118192 sectors
Disk model: *
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: *

Device             Start       End   Sectors  Size Type
/dev/nvme0n1p1      2048   2099199   2097152    1G Linux filesystem	<- /boot/efi (vfat)
/dev/nvme0n1p2   2099200   4196351   2097152    1G Linux filesystem	<- /boot     (vfat)
/dev/nvme0n1p3   4196352 486541311 482344960  230G Linux filesystem	<- /	     (ext4) (vg0/root, vg0/var, vg0/home)
/dev/nvme0n1p4 486541312 500117503  13576192  6.5G Linux swap           <- swap
```

# Create and mount filesystems

```
mkfs.vfat /dev/nvme0n1p1
mkfs.vfat -F32 /dev/nvme0n1p2
```

configure encrypted partition for rootfs

```
cryptsetup luksFormat -c aes-xts-plain64 -s 512 /dev/nvme0n1p3
cryptsetup luksOpen /dev/nvme0n1p3 lvm
```

create LVM layout on encrypted disk (named lvm)

```
pvcreate /dev/mapper/lvm
vgcreate vg0 /dev/mapper/lvm
lvcreate -L 30G -n swap vg0
lvcreate -L 50G -n root vg0
lvcreate -L 20G -n var vg0
lvcreate -L 100G -n home vg0
mkfs.ext4 /dev/vg0/root
mkfs.ext4 /dev/vg0/var
mkfs.ext4 /dev/vg0/home
mkswap /dev/vg0/swap
swapon /dev/vg0/swap
mount /dev/vg0/root /mnt/gentoo/
mkdir -p /mnt/gentoo/home
mount /dev/vg0/home /mnt/gentoo/home
mkdir -p /mnt/gentoo/var
mount /dev/vg0/var /mnt/gentoo/var
```

mount virtual filesystems

```
mount -t proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
mount -t tmpfs -o nosuid,nodev,noexec shm /dev/shm
chmod 1777 /dev/shm
```

# Copy resolver

```
cp -f /etc/resolv.conf /mnt/gentoo/etc/
```

# Get Stage3

```
cd /mnt/gentoo
wget https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/20211003T170529Z/stage3-amd64-systemd-20211003T170529Z.tar.xz
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
rm -f *.tar.xz
```

# Install ccache (before make.conf)

```
emerge -avq ccache
```

`/mnt/gentoo/etc/portage/make.conf`

```
# Available x86 instruction sets; https://wiki.gentoo.org/wiki/CPU_FLAGS_*
CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sse sse2 sse3 sse4_1 sse4_2 ssse3"

# Allow spawning 8 parallel make jobs; Target 8 logical CPU threads; 8 + 1 would make the CPU choke https://wiki.gentoo.org/wiki/MAKEOPTS
MAKEOPTS="-j8"

# Install only firmware for the current CPU
MICROCODE_SIGNATURES="-S"

# Accept all licences
ACCEPT_LICENSE="*"

# Restrict Emerge to build just a single package (--jobs 1 * -j8 => 8); https://wiki.gentoo.org/wiki/EMERGE_DEFAULT_OPTS
EMERGE_DEFAULT_OPTS="--jobs 1 --quiet-build y -avq"

# Save messages after emerging to /var/log/portage/elog
PORTAGE_ELOG_SYSTEM="echo save"

# Grub will use UEFI 54 instead of legacy BIOS
GRUB_PLATFORM="efi-64"

# Minor security tuning
FEATURES="userfetch binpkg-request-signature ccache"

# CCACHE
CCACHE_DIR="/var/tmp/ccache"

LINGUAS="en pl"
L10N="en en-US"

# Intel GMA Gen 9 - Coffee Lake; https://wiki.gentoo.org/wiki/Intel
VIDEO_CARDS="intel i915"

# Input devices
INPUT_DEVICES="libinput"

# Filter LLVM's targets to x86 only
LLVM_TARGETS="X86"

# This sets the language of build output to English.
# Please keep this setting intact when reporting bugs.
LC_MESSAGES=C.utf8

# USE falgs
USE="X gtk gnome systemd -kde pgo sqlite -ipv6 cryptsetup lvm xvfb"
```

# Chroot

```
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) $PS1"
```

# mount /boot 

```
mount /dev/nvme0n1p2 /boot
```

# Sync Portage

```
emerge --sync
```


# Profile

```
eselect profile list
eselect profile set 4 # -> default/linux/amd64/23.0/desktop/systemd (stable)
```

# Timezone

```
echo Europe/Warsaw > /etc/timezone
emerge --config sys-libs/timezone-data
```

# emerge vim (needed to edit config files)

```
emerge vim
```

# Locale

`/etc/locale.gen`


```
C.UTF8 UTF-8
en_US ISO-8859-1
en_US.UTF-8 UTF-8
pl_PL ISO-8859-2
pl_PL.UTF-8 UTF-8
```

```
locale-gen
eselect locale list
eselect locale set 8 # mine is pl_PL.utf8; choose own
env-update && source /etc/profile && export PS1="(chroot) $PS1"
```

# minimal software

```
emerge gentoolkit sqlite htop eix cryptsetup
```

# Sync Portage with eix

```
eix-sync
mkdir -p /etc/portage/package.{accept_keywords,license,mask,unmask,use}
```

# Rebuild the World

```
time emerge -qavu --deep --with-bdeps=y --newuse  --keep-going --backtrack=30 @world
```

# Install sources

```
emerge sys-kernel/gentoo-sources
```

# Intel microcode firmware

```
echo "sys-firmware/intel-microcode intel-ucode" >>/etc/portage/package.license/intel-microcode
echo "sys-kernel/linux-firmware linux-fw-redistributable" >> /etc/portage/package.license/linux-firmware
emerge linux-firmware intel-microcode xf86-video-intel
```

# Select kernel

```
eselect kernel list
eselect kernel set 1
```

# configure Grub2

[/etc/default/grub]

```
GRUB_CMDLINE_LINUX="dolvm crypt_root=/dev/nvme0n1p3 init=/usr/lib/systemd/systemd"
```

# Configure kernel

```
zcat /proc/config.gz >/root/.kernel
genkernel --luks --lvm  --kernel-config=/root/.kernel all
grub2-mkconfig -o /boot/grub/grub.cfg
```

# Bootloader

```
mkdir -p /etc/portage/package.use
echo "sys-boot/grub device-mapper" > /etc/portage/package.use/grub
emerge -avq grub:2
grub-install --target=x86_64-efi --efi-directory=/boot
```

# Rescue

## Boot Gentoo from livecd source
## Disable Network Manager

```
/etc/init.d/NetworManager stop
```

Configure WiFi

Check WiFi iterface

```
ip a
```

`wlp3s0` - wifi inet

```
cryptsetup luksOpen /dev/nvme0n1p3 lvm
vgchange -a y vg0
swapon /dev/vg0/swap
mount /dev/vg0/root /mnt/gentoo
mount /dev/vg0/var /mnt/gentoo/var
mount /dev/vg0/home /mnt/gentoo/home
mount /dev/nvme0n1p2 /mnt/gentoo/boot
mount /dev/nvme0n1p1 /mnt/gentoo/boot/efi
cp -Lrf /etc/resolv.conf /mnt/gentoo/etc/
mount -t proc proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) $PS1"
```

## recompile kernel

```
genkernel --lvm --luks --gpg --busybox all
```


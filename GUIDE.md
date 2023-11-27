#https://wiki.gentoo.org/wiki/Full_Disk_Encryption_From_Scratch_Simplified
#https://wiki.gentoo.org/wiki/Kernel/Gentoo_Kernel_Configuration_Guide
#https://wiki.gentoo.org/wiki/AMDGPU
#https://wiki.gentoo.org/wiki/AMD_microcode
#https://wiki.gentoo.org/wiki/Dm-crypt
#https://aj.immo/2021/11/gentoo-with-efistub-encrypted-btrfs/
#https://www.reddit.com/r/archlinux/comments/11aqxnk/install_with_luks_btrfs_efistub_sdencrypt/?rdt=62503
#https://wiki.gentoo.org/wiki/EFI_stub
#https://old.reddit.com/r/Gentoo/comments/urvse5/make_luks_efistub_prompt_without_initramfs/
#https://forums.gentoo.org/viewtopic-t-1155217.html?sid=264442c72d00e39d2952bfcf75b2b630

fdisk /dev/nvme0n1
# partition 1 => 1G, EFI System
# partition 2 => 32G, Linux Swap
# partition 3 => rest, Linux filesystem

mkfs.vfat -F 32 /dev/nvme0n1p1
fatlabel /dev/nvme0n1p1 BOOT
#mkswap -L SWAP /dev/nvme0n1p2
cryptsetup luksFormat /dev/nvme0n1p3

cryptsetup luksOpen /dev/nvme0n1p3 cryptroot

mkfs.btrfs -L ROOT /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt/gentoo
cd /mnt/gentoo
btrfs subvolume create @
btrfs subvolume create @snapshots
btrfs subvolume create @etc
cd
umount /mnt/gentoo
mount -o noatime,compress=zstd,space_cache=v2,discard=async,subvol=@ /dev/mapper/cryptroot /mnt/gentoo
mkdir -p /mnt/gentoo/{efi/EFI,home,etc/keys,.snapshots}
mount -o noatime,compress=zstd,space_cache=v2,discard=async,subvol=@snapshots /dev/mapper/cryptroot /mnt/gentoo/.snapshots
mount /dev/nvme0n1p1 /mnt/gentoo/efi

# Download stage 3 tar ball to /mnt/gentoo
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner

# if using arch based distro
fstabgen -U /mnt/gentoo >> /mnt/gentoo/etc/fstab

mkdir -p /mnt/gentoo/etc/keys
dd if=/dev/random of=/mnt/gentoo/etc/keys/swap.key bs=8388607 count=1

cryptsetup luksFormat /dev/nvme0n1p2 /mnt/gentoo/etc/keys/swap.key

nano /mnt/gentoo/etc/portage/make.conf
---
# These settings were set by the catalyst build script that automatically
# built this stage.
# Please consult /usr/share/portage/config/make.conf.example for a more
# detailed example.
COMMON_FLAGS="-O2 -march=znver3 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"
MAKEOPTS="-j14 -l14"
FEATURES="buildpkg candy collision-protect ipc-sandbox network-sandbox parallel-fetch parallel-install"
ACCEPT_KEYWORDS="amd64"
VIDEO_CARDS="amdgpu radeonsi"

USE="elogind -systemd -cups -gnome wayland"

# This sets the language of build output to English.
# Please keep this setting intact when reporting bugs.
LC_MESSAGES=C.utf8

GENTOO_MIRRORS="https://gentoo.osuosl.org/ https://mirror.leaseweb.com/gentoo/ rsync://mirror.leaseweb.com/gentoo/ rsync://rsync.gtlib.gatech.edu/gentoo https://mirrors.mit.edu/gentoo-distfiles/ rsync://mirrors.mit.edu/gentoo-distfiles/ https://mirrors.rit.edu/gentoo/ rsync://mirrors.rit.edu/gentoo/ https://mirror.rackspace.com/gentoo/ rsync://mirror.rackspace.com/gentoo/ https://mirror.clarkson.edu/gentoo/ rsync://mirror.clarkson.edu/gentoo/ https://mirror.servaxnet.com/gentoo/"
---

mkdir --parents /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/

mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run

chroot /mnt/gentoo /bin/bash
source /etc/profile && export PS1="(chroot) ${PS1}"
emerge-webrsync
emerge --sync --quiet
emerge --ask --quiet --update --deep --newuse @world
echo -e "app-arch/unrar unRAR\nsys-kernel/linux-firmware @BINARY-REDISTRIBUTABLE" >> /etc/portage/package.license
emerge -aq neovim
ln -s /usr/bin/nvim /usr/bin/vi
ln -s /usr/bin/nvim /usr/bin/vim


echo "America/Chicago" > /etc/timezone
emerge --config sys-libs/timezone-data
vi /etc/locale.gen
---
en_US.UTF-8 UTF-8
---

locale-gen
eselect locale list
eselect locale set 4
env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
#rm -rf /bin/cpio
emerge -aq sys-kernel/linux-firmware sys-kernel/gentoo-sources sys-apps/pciutils sys-kernel/dracut sys-fs/btrfs-progs app-arch/gzip app-portage/cpuid2cpuflags dev-vcs/git sys-fs/cryptsetup sys-apps/busybox sys-boot/efibootmgr
eselect kernel list
eselect kernel set 1
cd /usr/src/linux
make defconfig && make localyesconfig && make menuconfig

vi /etc/dracut.conf
---
hostonly="yes"
dracutmodules+=" bash kernel-modules busybox udev-rules usrmount base fs-lib shutdown "
add_dracutmodules+=" btrfs crypt dm rootfs-block "
---

mkstub

echo "desktop" > /etc/hostname

vi /etc/hosts
---
# IPv4 and IPv6 localhost aliases
127.0.0.1       localhost
::1             localhost
127.0.0.1       desktop.localhost       desktop
---

emerge -aq app-admin/sysklogd sys-process/fcron sys-apps/mlocate app-shells/bash-completion net-misc/chrony sys-fs/dosfstools sys-fs/e2fsprogs #net-misc/dhcpcd
emerge --config sys-process/fcron
rc-update add sysklogd default
rc-update add chronyd default
rc-update add dmcrypt boot

#!/bin/bash

set -e

# Basit bir disk seçimi UI'si
select_disk() {
  echo "Mevcut diskler:"
  lsblk -d -n -o NAME,SIZE | nl
  read -p "Kurulum için disk numarasını seçin: " disk_number
  DISK=$(lsblk -d -n -o NAME | sed -n "${disk_number}p")
  if [ -z "$DISK" ]; then
    echo "Geçersiz seçim! Tekrar deneyin."
    exit 1
  fi
  DISK="/dev/$DISK"
  echo "Seçilen disk: $DISK"
}

# Bölümleme işlemi
partition_disk() {
  echo "Diski bölümle: $DISK"
  parted -s "$DISK" mklabel gpt
  parted -s "$DISK" mkpart ESP fat32 1MiB 512MiB
  parted -s "$DISK" set 1 boot on
  parted -s "$DISK" mkpart primary ext4 512MiB 100%
  mkfs.fat -F32 "${DISK}1"
  mkfs.btrfs "${DISK}2"
  mount "${DISK}2" /mnt
  mkdir -p /mnt/boot
  mount "${DISK}1" /mnt/boot
}

# Temel sistem kurulumu
install_base() {
  pacstrap /mnt base linux linux-firmware btrfs-progs
  genfstab -U /mnt >> /mnt/etc/fstab
}

# Bootloader kurulumu (GRUB veya systemd-boot)
install_bootloader() {
  arch-chroot /mnt bash <<EOF
  echo "Bootloader seçimi:"
  echo "1) GRUB"
  echo "2) systemd-boot"
  read -p "Seçiminiz (1/2): " boot_choice
  if [ "\$boot_choice" -eq 1 ]; then
    pacman -S --noconfirm grub efibootmgr
    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
    grub-mkconfig -o /boot/grub/grub.cfg
  elif [ "\$boot_choice" -eq 2 ]; then
    bootctl install
    echo "default arch.conf" > /boot/loader/loader.conf
    echo "title Arch Linux" > /boot/loader/entries/arch.conf
    echo "linux /vmlinuz-linux" >> /boot/loader/entries/arch.conf
    echo "initrd /initramfs-linux.img" >> /boot/loader/entries/arch.conf
    echo "options root=PARTUUID=$(blkid -s PARTUUID -o value ${DISK}2) rw" >> /boot/loader/entries/arch.conf
  else
    echo "Geçersiz seçim!"
    exit 1
  fi
EOF
}

# Masaüstü ortamları kurulum seçeneği
install_desktops() {
  arch-chroot /mnt bash <<EOF
  echo "Masaüstü ortamları:"
  echo "1) GNOME"
  echo "2) KDE Plasma"
  echo "3) XFCE"
  echo "4) Cinnamon"
  echo "5) Mate"
  echo "6) LXQt"
  echo "7) i3"
  echo "8) Deepin"
  echo "9) Tümünü yükle"
  read -p "Seçiminiz (1-9): " de_choice
  case "\$de_choice" in
    1) pacman -S --noconfirm gnome ;;
    2) pacman -S --noconfirm plasma ;;
    3) pacman -S --noconfirm xfce4 xfce4-goodies ;;
    4) pacman -S --noconfirm cinnamon ;;
    5) pacman -S --noconfirm mate mate-extra ;;
    6) pacman -S --noconfirm lxqt ;;
    7) pacman -S --noconfirm i3 ;;
    8) pacman -S --noconfirm deepin deepin-extra ;;
    9) pacman -S --noconfirm gnome plasma xfce4 xfce4-goodies cinnamon mate mate-extra lxqt i3 deepin deepin-extra ;;
    *) echo "Geçersiz seçim!" ;;
  esac
EOF
}

# Giriş yöneticisi seçimi
install_display_manager() {
  arch-chroot /mnt bash <<EOF
  echo "Giriş yöneticisi:"
  echo "1) GDM (GNOME için)"
  echo "2) SDDM (KDE için)"
  echo "3) LightDM"
  read -p "Seçiminiz (1/2/3): " dm_choice
  case "\$dm_choice" in
    1) pacman -S --noconfirm gdm; systemctl enable gdm ;;
    2) pacman -S --noconfirm sddm; systemctl enable sddm ;;
    3) pacman -S --noconfirm lightdm lightdm-gtk-greeter; systemctl enable lightdm ;;
    *) echo "Geçersiz seçim!" ;;
  esac
EOF
}

# Btrfs snapshot desteği
setup_snapshots() {
  arch-chroot /mnt bash <<EOF
  pacman -S --noconfirm snapper grub-btrfs
  snapper -c root create-config /
  systemctl enable snapper-timeline.timer
  systemctl enable snapper-cleanup.timer
  systemctl enable grub-btrfs.path
EOF
}

# AUR yardımcı programı yay kurulumu
install_yay() {
  arch-chroot /mnt bash <<EOF
  pacman -S --noconfirm git base-devel
  git clone https://aur.archlinux.org/yay.git /tmp/yay
  cd /tmp/yay
  makepkg -si --noconfirm
EOF
}

# Ana script akışı
main() {
  echo "Arch Installer başlıyor..."
  select_disk
  partition_disk
  install_base
  install_bootloader
  install_desktops
  install_display_manager
  setup_snapshots
  install_yay
  echo "Kurulum tamamlandı! Sistemi yeniden başlatabilirsiniz."
}

main

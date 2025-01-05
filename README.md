# Arch Linux Setup for AMD 13-inch Laptop

## Steps

### 1. Setup Keyboard Layout
```bash
loadkeys us
```

### 2. Verify Boot Mode
```bash
cat /sys/firmware/efi/fw_platform_size
```

### 3. Setup Internet Connection
```bash
iwctl
device list
station <device> scan
station <device> get-networks
station <device> connect <network>
station <device> show
exit
```

### 4. Check Connection
```bash
dhcpcd
ping -c 3 archlinux.org
```

### 5. Set Timezone
```bash
timedatectl set-timezone America/New_York
```

### 6. Check Sector Size
```bash
lsblk
sudo fdisk -l <drive>
```

### 7. Partition Disks
1. **Create Partitions:**
    ```
    Command: g
    Command: n
    Partition number:
    First sector:
    Last sector: 1046529
    Command: t
    Partition type or alias: 1 (set EFI type)
    Command: n
    Partition number:
    First sector:
    Last sector: 208664577
    Command: n
    Partition number:
    First sector:
    Last sector:
    Command: p (check dimensions)
    Command: w
    ```

### 8. Format Partitions
```bash
mkfs.fat -F32 -n EFI /dev/vda1
```

### 9. Encrypt and Format Partitions
- **Root Partition:**
    ```bash
    cryptsetup luksFormat -h sha256 /dev/nvme0n1p2
    cryptsetup luksHeaderBackup /dev/nvme0n1p2 --header-backup-file /root/system-header-backup.img
    cryptsetup open /dev/nvme0n1p2 system
    mkfs.ext4 -L system /dev/mapper/system
    ```
- **Home Partition:**
    ```bash
    cryptsetup luksFormat -h sha256 /dev/nvme0n1p3
    cryptsetup luksHeaderBackup /dev/nvme0n1p3 --header-backup-file /root/home-header-backup.img
    cryptsetup open /dev/nvme0n1p3 home
    mkfs.ext4 -L home /dev/mapper/home
    ```

### 10. Mount Partitions
```bash
mount LABEL=system /mnt
mkdir /mnt/boot
mkdir /mnt/home
mount LABEL=EFI /mnt/boot
mount LABEL=home /mnt/home
```

### 11. Update Mirror List
```bash
reflector -c us > /etc/pacman.d/mirrorlist
```

### 12. Install Base System
```bash
pacstrap /mnt base linux linux-firmware
```

### 13. Generate FSTAB
```bash
genfstab -L /mnt >> /mnt/etc/fstab
```

### 14. Copy Encrypted Partition Headers
```bash
cp /root/system-header-backup.img /mnt/root
cp /root/home-header-backup.img /mnt/root
```

### 15. Enter Chroot Environment
```bash
arch-chroot /mnt
```

### 16. Configure System
- **Set Timezone:**
    ```bash
    ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
    hwclock --systohc
    ```
- **Set Locale:**
    ```bash
    echo "en_US.UTF-8 UTF-8" | tee /etc/locale.gen
    locale-gen
    echo "LANG=en_US.UTF-8" >> /etc/locale.conf
    echo "KEYMAP=us" >> /etc/vconsole.conf
    ```
- **Set Hostname:**
    ```bash
    echo "alex-arch" >> /etc/hostname
    ```
- **Set Hosts:**
    ```bash
    echo "127.0.0.1 localhost" >> /etc/hosts
    echo "127.0.1.1 XXXX-linux.local XXXX-linux" >> /etc/hosts
    ```

### 17. Setup Networking
```bash
pacman -S dhcpcd wpa_supplicant networkmanager
systemctl enable NetworkManager dhcpcd
```

### 18. Install Utilities
- **Midnight Commander:**
    ```bash
    pacman -S mc
    echo "EDITOR=/usr/bin/mcedit" >> /etc/profile.d/editor.sh
    ```
- **Vim:**
    ```bash
    pacman -Sy
    pacman -S vim
    ```

### 19. Configure Hooks
```bash
vim /etc/mkinitcpio.conf
HOOKS=(systemd autodetect modconf kms keyboard sd-vconsole block sd-encrypt filesystems resume fsck)
```

### 20. Install AMD Driver
```bash
pacman -S amd-ucode
```

### 21. Create Initramfs
```bash
mkinitcpio -p linux
```

### 22. Install System Boot
```bash
bootctl install
```

### 23. Configure Boot Loader
- **Loader Configuration:**
    ```bash
    vim /boot/loader/loader.conf
    # Insert:
    default arch*.conf
timeout 5
editor yes
console-mode auto
    ```
- **Default Entry:**
    ```bash
    vim /boot/loader/entries/arch.conf
    # Insert:
    title Arch Linux
    linux /vmlinuz-linux
    initrd /amd-ucode.img
    initrd /initramfs-linux.img
    options rd.luks.name=UUID_of_system_partition=system rd.luks.name=UUID_of_home_partition=home root=/dev/mapper/system amdgpu.si_display=0 acpi_osi="!Windows 2000" rw splash
    ```
- **Fallback Entry:**
    ```bash
    vim /boot/loader/entries/arch-fallback.conf
    # Same as default
    ```

### 24. Set Root Password
```bash
passwd
```

### 25. Exit Chroot
```bash
exit
```

### 26. Reboot
```bash
reboot
```


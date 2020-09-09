# Harchened

This project covers installing and hardening Arch Linux (UEFI systems).

First some basic configurations will be made and then the hardening will be implemented following a checklist to get a fully functional and safe workstation.

All security measures are included in a [checklist][check], This checklist is inspired by the [hardening checklist of the University of Texas at Austin][utexas].

This checklist includes [CIS Benchmarks][cis] and some additional settings that I consider interesting and are not covered by the CIS.

## Index

1. [**Installation**][1]
2. [**Network Security**][2]
3. [**Kernel Hardening**][3]
4. [**OS Hardening**][4]
5. [**Environment (optional)**][5]

## 1. Installation
### 1.1 Keyboard and internet connection
```
loadkeys es
wifi-menu
```
### 1.2 Create partitions (LVM on LUKS)

First you have to partition the disk, in this case an LVM partition encrypted with LUKS will be created with the advantage of being able to decrypt all the partitions by entering the password only once every startup.
```
fdisk /dev/sda
    g ; n ; 1 ; ENTER ; +500M ; t ; 1
    n ; 2 ; ENTER ; +500M
    n ; 3 ; ENTER ; +130G ; t ; 3 ; 30 ; w

mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2

cryptsetup luksFormat /dev/sda3 --cipher aes-xts-plain64
cryptsetup open --type luks /dev/sda3 cryptDisk

pvcreate --dataalignment 1m /dev/mapper/cryptDisk
vgcreate LVM /dev/mapper/cryptDisk

lvcreate -L 100G LVM -n root
lvcreate -L 10G LVM -n home
lvcreate -L 4G LVM -n tmp
lvcreate -L 4G LVM -n var
lvcreate -L 4G LVM -n var.tmp
lvcreate -L 4G LVM -n var.log
lvcreate -L 4G LVM -n var.log.audit

mkfs.ext4 /dev/LVM/root
mkfs.ext4 /dev/LVM/home
mkfs.ext4 /dev/LVM/tmp
mkfs.ext4 /dev/LVM/var
mkfs.ext4 /dev/LVM/var.tmp
mkfs.ext4 /dev/LVM/var.log
mkfs.ext4 /dev/LVM/var.log.audit

mount /dev/LVM/root /mnt
mkdir /mnt/boot
mount /dev/sda2 /mnt/boot
mkdir /mnt/home
mount -o nodev /dev/LVM/home /mnt/home
mkdir /mnt/tmp
mount -o nodev,nosuid,noexec /dev/LVM/tmp /mnt/tmp
mkdir /mnt/var
mount /dev/LVM/var /mnt/var
mkdir /mnt/var/tmp
mount -o nodev,nosuid,noexec /dev/LVM/var.tmp /mnt/var/tmp
mkdir /mnt/var/log
mount /dev/LVM/var.log /mnt/var/log
mkdir /mnt/var/log/audit
mount /dev/LVM/var.log.audit /mnt/var/log/audit

pacstrap /mnt base reflector
genfstab -U /mnt >> /mnt/etc/fstab
echo -e "tmpfs\t/dev/shm\ttmpfs\tdefaults,nodev,nosuid,noexec\t0 0" >> /etc/fstab
echo -e "proc\t/proc\tproc\thidepid=2\t0 0" >> /etc/fstab

arch-chroot /mnt

```
### 1.3 Install basic packages
The fastest mirrors are configured before installing all packages.
```
reflector --latest 200 --sort rate --save /etc/pacman.d/mirrorlist
pacman -S linux linux-lts linux-headers linux-lts-headers linux-firmware \
    lvm2 grub efibootmgr dosfstools os-prober mtools \
    base-devel intel-ucode vim networkmanager sudo git nftables
```
### 1.4 Initial ramdisks & grub
```
vim /etc/mkinitcpio.conf
    HOOKS=(base udev autodetect modconf block encrypt lvm2 filesystems keyboard fsck)
mkinitcpio -p linux
mkinitcpio -p linux-lts

vim /etc/default/grub
    GRUB_ENABLE_CRYPTODISK=y
    GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 cryptdevice=/dev/sda3:LVM:allow-discards quiet"
mkdir /boot/EFI
mount /dev/sda1 /boot/EFI
grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck
grub-mkconfig -o /boot/grub/grub.cfg
```
### 1.5 Basic configs
```
echo KEYMAP=es > /etc/vconsole.conf
systemctl enable NetworkManager
echo Harchened > /etc/hostname
ln -s /usr/share/zoneinfo/Europe/Madrid /etc/localtime
hwclock --systohc --utc
```
### 1.6 Create user & disable root account
```
passwd
useradd -m -G wheel,storage,power x
passwd x
visudo
    %wheel ALL=(ALL) ALL
su x
sudo passwd -l root

```
### 1.7 Install yay & powerpill
```
cd /opt
sudo git clone https://aur.archlinux.org/yay-git.git
sudo chown -R x:x yay-git
cd yay-git
makepkg -si
yay -S powerpill
exit
exit
reboot

```
## 2. Network Security
### 2.1 Connect wifi & disable IPv6
```
nmcli d wifi list
nmcli d wifi connect <my_wifi> password <password>
nmcli con mod <my_wifi> ipv6.method "disabled"
```
### 2.2 nftables
```
curl https://gitlab.com/sapellaniz/harchened/-/raw/master/nftables.conf > nftables.conf
sudo systemctl enable nftables
```
### 2.3 Network parameters
```
curl https://gitlab.com/sapellaniz/harchened/-/raw/master/networking_secure_values.conf > networking_secure_values.conf
sudo mv networking_secure_values.conf /etc/sysctl.d
```
## 3. Kernel Hardening
### 3.1 Lynis
```
sudo powerpill -S lynis
sudo lynis audit system > kernel_secure_values.conf
vim kernel_secure_values.conf
    # Delete everything except the lines of the "Kernel Hardening" section with the word DIFFERENT
    # Convert the rest of the lines into the sysctl configuration format.
    # Set each parameter to the exp value that's currently within the pairs of parentheses.
sudo mv kernel_secure_values.conf /etc/sysctl.d/
reboot
```
## 4. OS Hardening
### 4.1 umask
```
sudo vim /etc/profile
    umask 027
```
### 4.2 Remove orphans
```
sudo powerpill -Rns $(powerpill -Qtdq)
```
### 4.3 Chmod last command
```
sudo chmod o-rx $(which last)
```
## 5. Environment (optional)
### 5.1 x-server & window manager & hotkey-daemon & terminator
```
sudo powerpill -S xorg bspwm sxhkd terminator xorg-xinit
mkdir ~/.config/bspwm ~/.config/sxhkd
cp /usr/share/doc/bspwn/examples/bspwmrc ~/.config/bspwm
cp /usr/share/doc/bspwn/examples/sxkhdrc ~/.config/sxhkd
chmod u+x ~/.config/bspwm/bspwmrc
vim ~/.config/sxhkd
    # Change the supr + return hotkey for terminator
echo "sxhkd & exec bspwm" > ~/.xinitrc
vim ~/.bashrc
    # setxkbmap es
    # if systemctl -q is-active graphical.target && [[ ! $DISPLAY && $XDG_VTNR -eq 1 ]]; then
    #   exec startx
    # fi
```
### 5.2 firefox & wget
```
sudo powerpill -S firefox wget
```
### 5.3 wallpaper & pointer
```
sudo powerpill -S xwallpaper
wget ...wallpaper.png -O ~/.config/wall.png
echo "xwallpaper --focus ~/.config/wall.png" >> ~/.config/bspwm/bspwmrc
echo "xsetroot -cursor_name left_ptr" >> ~/.config/bspwm/bspwmrc
```
### 5.4 rofi
```
sudo powerpill -S rofi
vim ~/.config/sxhkd/sxhkdrc
    # supr + d
    #   rofi -show run
```
### 5.5 polybar
```
yay -S polybar-git
mkdir ~/.config/polybar
wget ...config -O ~/.config/polybar/config
vim ~/.config/polybar/launch.sh
    # killall -q polybar
    # while pgrep -u $UID -x polybar >/dev/null; do sleep 1; done
    # polybar mybar
chmod +x ~/.config/polybar/launch.sh
echo "~/.config/polybar/launch.sh &" >> /.config/bspwm/bspwmrc
```

[1]:https://gitlab.com/sapellaniz/harchened#1-installation
[2]:https://gitlab.com/sapellaniz/harchened#2-network-security
[3]:https://gitlab.com/sapellaniz/harchened#3-kernel-hardening
[4]:https://gitlab.com/sapellaniz/harchened#4-os-hardening
[5]:https://gitlab.com/sapellaniz/harchened#5-environment-optional
[check]:https://gitlab.com/sapellaniz/harchened/-/blob/master/checklist.csv
[cis]:https://gitlab.com/sapellaniz/harchened/-/blob/master/CIS_CentOS_Linux_7_Benchmark_v2.2.0.pdf
[utexas]:https://security.utexas.edu/os-hardening-checklist/linux-7

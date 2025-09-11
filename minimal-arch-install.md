## Minimal Arch Install

### 1 Live ISO

```
iwctl --passphrase "<pass>" station wlan0 connect <WiFi> 
passwd                          # set root pwd 
sudo systemctl start sshd       # optional remote install 
timedatectl set-ntp true
```

---

### 2 Disk layout (NVMe, GPT)

```
lsblk sgdisk --zap-all /dev/nvme0n1 
sgdisk -n1:0:+512MiB -t1:ef00 -c1:"EFI"      /dev/nvme0n1 
sgdisk -n2:0:0        -t2:8300 -c2:"cryptroot" /dev/nvme0n1 
partprobe /dev/nvme0n1 
lsblk                    # nvme0n1p1 / p2 now exist
````

---

### 3 Encrypt & format

```
cryptsetup luksFormat /dev/nvme0n1p2 
cryptsetup open /dev/nvme0n1p2 cryptroot  

mkfs.fat -F32  /dev/nvme0n1p1 
mkfs.ext4      /dev/mapper/cryptroot
```

```
mount /dev/mapper/cryptroot /mnt 
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

---

### 4 Bootstrap packages

```
pacstrap -K /mnt \   
base linux linux-firmware amd-ucode \
vim sudo networkmanager bluez bluez-utils \
grub efibootmgr cryptsetup
```

```
genfstab -U /mnt >> /mnt/etc/fstab 
arch-chroot /mnt
```

---

### 5 Core system config

```
ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime 
hwclock --systohc

sed -i 's/#en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen 
locale-gen 
echo "LANG=en_US.UTF-8" > /etc/locale.conf  

echo "<hostname>" > /etc/hostname 
printf "127.0.1.1\t<hostname>.localdomain\t<hostname>\n" >> /etc/hosts
```

```
vim /etc/mkinitcpio.conf        # HOOKS=(... encrypt filesystems fsck) 
mkinitcpio -P
```

---

### 6 GRUB (EFI + LUKS)

```
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

UUID=$(blkid -s UUID -o value /dev/nvme0n1p2) sed -i "s|^GRUB_CMDLINE_LINUX=.*|GRUB_CMDLINE_LINUX=\"cryptdevice=UUID=$UUID:cryptroot 

root=/dev/mapper/cryptroot\"|" /etc/default/grub  
grub-mkconfig -o /boot/grub/grub.cfg
```

---

### 7 Drivers (Intel CPU | AMD | NVIDIA | Thinkpad | Asus G-Series)

Intel:
```
pacman -S mesa vulkan-intel
```

AMD:
```
pacman -S mesa vulkan-radeon xf86-video-amdgpu #xf86 driver only if x11
```

Nvidia:
* Must enable `[multilib]` for Nvidia drivers
```
sed -Ei '/^\[multilib\]/,+1 s/^#//' /etc/pacman.conf
pacman -Sy

pacman -S nvidia nvidia-utils lib32-nvidia-utils nvidia-prime
echo "options nvidia-drm modeset=1" > /etc/modprobe.d/nvidia.conf
```

ASUS
```
pacman-key --recv-keys 8F654886F17D497FEFE3DB448B15A6B0E9A3FA35
pacman-key --lsign-key 8F654886F17D497FEFE3DB448B15A6B0E9A3FA35
printf "[g14]\nServer = https://arch.asus-linux.org\n" >> /etc/pacman.conf

pacman -Sy
pacman -S supergfxctl switcheroo-control asusctl power-profiles-daemon
systemctl enable supergfxd switcheroo-control power-profiles-daemon
```

Thinkpad:
```
pacman -S tlp tlp-rdw fwupd            # power + firmware
systemctl enable tlp
```

Network / Bluetooth:
```
systemctl enable NetworkManager systemctl enable bluetooth


```

---

### 8 Swap & user
```
fallocate -l 16G /swapfile 
chmod 600 /swapfile 
mkswap /swapfile 

echo '/swapfile none swap defaults 0 0' >> /etc/fstab  
```

```
useradd -m -G wheel <user> 
passwd <user> 

EDITOR=vim visudo          # uncomment %wheel ALL=(ALL:ALL) ALL
```

---

### 9 Finish

```
exit 
umount -R /mnt 
reboot
```

---

### Alternative GPU/firmware scenarios

| Scenario                                       | Commands                                                                                                                                           |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Intel iGPU only (e.g., ThinkPad)**           | `pacman -S mesa vulkan-intel`  <br>`i915.enable_psr=0` (grub line) if flicker                                                                      |
| **AMD dGPU + AMD iGPU (ThinkPad E-series)**    | Already covered by `mesa` & `amdgpu` (kernel). Add:  <br>`pacman -S xf86-video-amdgpu` if X11, and `sway`/`wlroots` works out-of-box               |
| **Intel CPU + NVIDIA dGPU (Optimus ThinkPad)** | Enable multilib, then:  <br>`pacman -S nvidia nvidia-prime lib32-nvidia-utils`  <br>Use **`prime-run <app>`** or `env __NV_PRIME_RENDER_OFFLOAD=1` |
| **ThinkPads**                                  | Install `tlp tlp-rdw` and enable `tlp.service` for power saving.<br>firmware updates via `fwupd`                                                   |


## After main install

### 1 Packages

```

sudo pacman -S xdg-user-dirs base-devel git
xdg-user-dirs-update
```

### Network Configuration

```
sudo vim /etc/iwd/main.conf
[General]
EnableNetworkConfiguration=true

[Network]
DHCP=yes

sudo vim /etc/system/logind.conf
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore

sudo systemctl enable --now systemd-resolved.service #Handle DNS
```          

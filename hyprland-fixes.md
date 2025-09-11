## Nvidia for Hyprland fixes

---
### 1 Packages and permissions
```
sudo pacman -S seatd                        # installs seatd + libseat
sudo systemctl enable --now seatd.service   # daemon socket-activates
sudo usermod -aG seat $USER                 # log out / back in once

sudo pacman -S nvidia-dkms nvidia-utils lib32-nvidia-utils egl-wayland
^ ONLY NEED DKMS IF YOURE DUALBOOTING
```
### 2 Pass over tasks to GPU
```
render on amdgpu first, nvidia second
env = WLR_DRM_DEVICES,/dev/dri/card1:/dev/dri/card0

nvidia-specific vars
env = LIBVA_DRIVER_NAME,nvidia
env = __GLX_VENDOR_LIBRARY_NAME,nvidia
```
### 3 Find GPU name 
```
for c in /sys/class/drm/card*/device/driver; do
card=$(basename "$(dirname "$(dirname "$c")")")
driver=$(basename "$(readlink "$c")")
printf "%s → %s\n" "$card" "$driver" done card0 → nvidia card1 → amdgp

sudo chflags nouchg /etc/hosts && sudo chflags noschg /etc/hosts
```



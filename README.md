# What is this?
- a patch to enable kwin_wayland to be run with the drm backend, while capturing input from a dummy wayland window

# What for?
- Dual-GPU VFIO setups, where dynamic rebind & unbind is not feasible and / or causes performance issues
- Developing kwin_wayland using a dual gpu (can be IGPU and dedicated too) setup, without the hassle of switching ttys or sshing
- Maximizing gaming performance by running only the game on the compositor

# What is changed?
- Open /dev/dri/cardX directly not over the session (may require root)
- run two backends simultaneously to allow wayland input capture but draw screen to drm
- remove pointer acceleration from given input
- always lock pointer in empty wayland window on keypress to allow for easier entry (alt-tab out of it)
- optimized? renderloop to recomposite every 2ms, to allow up to 500hz of output (kwin only composits based on frame showing duration, which causes 240hz not to be always reached)

# Known issues
- This is a hacky patch, it works but shutdown is not clean due to C++ and pointer shenanigans. Do not worry, the OS is the ultimate garbage collector in this case.
- Screen recording obviously doesn't work, use gpu-screen-recorder and a fifo to stream to OBS or something, ffmpeg with kmsgrab does not work yet, since one color scheme is not implemented currently
- Use kscreen-doctor to change display settings, settings before undbind should be used though, do not forget WAYLAND_DISPLAY=wayland-1

# Usage
## Building
- get kwin from https://invent.kde.org/plasma/kwin
- look at your current package version from your favorite package manager
- `git checkout` the appropiate tag (it won't build if you don't use the correct version due to dependency issues)
- apply patch with `git apply kwin_nested.patch`
- `mkdir build && cd build`
- `cmake ..`
- `make -j16`

## Running
- unbind current kwin_wayland from gpu: `sudo udevadm trigger /dev/dri/card1 -c remove`
- example command line for my use case: `KWIN_DRM_DEVICES=/dev/dri/card1 taskset 0x00FF00FF dbus-run-session gpuloader.sh ./bin/kwin_wayland --xwayland` <br>
Explanation: <br>
  KWIN_DRM_DEVICES=/dev/dri/card1 -> set this to the gpu you want to use as drm device <br>
  taskset -> CPU core assignement (i have a 7950X3D CPU to assign it to the X3D cores) <br>
  gpuloader.sh -> this is from the https://github.com/Bensikrac/VFIO-Nvidia-dynamic-unbind repo, it sets the envvars:
  `__GLX_VENDOR_LIBRARY_NAME=nvidia __EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/10_nvidia.json VK_DRIVER_FILES=/usr/share/vulkan/icd.d/nvidia_icd.json` <br>
  replace envvars above with appropiate ones, further info is in the mentioned VFIO-Nvidia-dynamic-unbind repo
  dbus-run-session -> Prevents breakage of global shortcuts etc, as mentioned in some development guide for kwin
- Starting games: `DISPLAY=:1 WAYLAND_DISPLAY=wayland-1 taskset 0x00FF00FF gpuloader.sh $command` (same graphics envvar thing and taskset as above)

# Further Reading
https://github.com/Bensikrac/VFIO-Nvidia-dynamic-unbind
   

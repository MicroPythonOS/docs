# OS Development on Linux

Most users can simply use a pre-built binary from the [releases page](https://github.com/MicroPythonOS/MicroPythonOS/releases) and install it [manually](installing-on-esp32.md) or using the [web installer](https://install.MicroPythonOS.com).

But if you want to modify things under the hood, you're in the right place!

## Get the prerequisites

Make sure you have everything installed to compile code:

```
sudo apt update
sudo apt-get install -y build-essential libffi-dev pkg-config cmake ninja-build gnome-desktop-testing libasound2-dev libpulse-dev libaudio-dev libjack-dev libsndio-dev libx11-dev libxext-dev libxrandr-dev libxcursor-dev libxfixes-dev libxi-dev libxss-dev libxkbcommon-dev libdrm-dev libgbm-dev libgl1-mesa-dev libgles2-mesa-dev libegl1-mesa-dev libdbus-1-dev libibus-1.0-dev libudev-dev fcitx-libs-dev libpipewire-0.3-dev libwayland-dev libdecor-0-dev libv4l-dev librlottie-dev
```


{!os-development/compiling.md!}

{!os-development/running-on-desktop.md!}

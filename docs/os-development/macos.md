# OS Development on MacOS

Most users can simply use a pre-built binary from the [releases page](https://github.com/MicroPythonOS/MicroPythonOS/releases) and install it [manually](installing-on-esp32.md) or using the [web installer](https://install.MicroPythonOS.com).

But if you want to modify things under the hood, you're in the right place!

## Get the prerequisites

Make sure you have everything installed to compile code:

```
xcode-select --install
brew install pkg-config libffi ninja make SDL2
```

{!os-development/compiling.md!}

{!os-development/running-on-desktop.md!}

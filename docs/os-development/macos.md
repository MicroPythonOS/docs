# OS Development on MacOS

Most users can just use a pre-built binary from the [releases page](https://github.com/MicroPythonOS/MicroPythonOS/releases) and install it manually or using the [web installer](https://install.MicroPythonOS.com).

But if for some reason that one doesn't work, or you really want to modify things under the hood, you're in the right place here!

## Get the prerequisites

Clone the repositories:

```
git clone --recurse-submodules https://github.com/MicroPythonOS/MicroPythonOS.git
```

That will take a while, because it recursively clones MicroPython, LVGL, ESP-IDF and all their dependencies.

While that's going on, make sure you have everything installed to compile code:

```
xcode-select --install || true # already installed on github
brew install pkg-config libffi ninja make SDL2
```

{!os-development/compiling.md!}

{!os-development/running-on-desktop.md!}

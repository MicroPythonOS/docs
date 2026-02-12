# Building for Windows

As the main dependency ([lvgl_micropython](https://github.com/lvgl-micropython/lvgl_micropython), which bundles LVGL and MicroPython) doesn't support Windows, MicroPythonOS also doesn't officially support it.

But [users report](https://github.com/MicroPythonOS/MicroPythonOS/issues/31) that the Linux desktop version runs just fine under WSL2 on Windows 11.
So you can use that to create and test apps on desktop, which is very convenient.

Compiling MicroPythonOS from source is only necessary for deep, low-level OS development like modifying the underlying MicroPython, LVGL or C bindings, so you might not need that.
If you do, and really want to use Windows, then you might be able to get it to build using WSL2 on Windows 11 after trying the [Linux instructions](linux.md).

Even without self-compiling everything, you can still participate in the bulk of the fun: creating cool apps!

To do so, run it on desktop, or install a pre-built firmware on a [supported device](../getting-started/supported-hardware.md) and then over to [Creating Apps](../apps/creating-apps.md).

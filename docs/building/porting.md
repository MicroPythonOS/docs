# Porting to a new device

Porting MicroPythonOS to a new device can be fairly easy, if it's a common device that can already run MicroPython.

Ideally, the hardware drivers (for your display) are also already available in the [lvgl_micropython](https://github.com/lvgl-micropython/lvgl_micropython) project, which MicroPythonOS heavily relies upon.

Of course, porting is always quite a technical undertaking, so if you want to make your life easy, just purchase one of the already supported devices - they're fairly cheap.

## What to write

By design, the only device-specific code for MicroPythonOS is found in the ```internal_filesystem/boot*.py``` files.

## Steps to port to a new device

1. Compile [lvgl_micropython](https://github.com/lvgl-micropython/lvgl_micropython) for the new device.

	The goal is to have it boot and show a MicroPython REPL shell on the serial line.

	Take a look at our [build_lvgl_micropython.sh](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/scripts/build_lvgl_micropython.sh) script. A "dev" build (without any "frozen" filesystem) is preferred as this will still change a lot.

	Also go over the [official lvgl_micropython documentation](https://github.com/lvgl-micropython/lvgl_micropython/blob/main/README.md) for porting instructions. If you're in luck, your device is already listed in the esp32 BOARD list. Otherwise use a generic one like `BOARD=ESP32_GENERIC` with `BOARD_VARIANT=SPIRAM` or `BOARD=ESP32_GENERIC_S3` with `BOARD_VARIANT=SPIRAM_OCT` if it has an SPIRAM.

2. Figure out how to initialize the display for the new device

    Use the MicroPython REPL shell on the serial port to type or paste (CTRL-E) MicroPython code.
    
    Check out how it's done for the [Waveshare 2 inch Touch Screen](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/internal_filesystem/boot.py) and for the [Fri3d Camp 2024 Badge](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/internal_filesystem/boot_fri3d-2024.py). You essentially need to set the correct pins to which the display is connected (like `LCD_SCLK`, `LCD_MOSI`, `LCD_MOSI` etc.) and also set the resolution of the display (`TFT_HOR_RES`, `TFT_VER_RE`S).
    
    After a failed attempt, reset the device to make sure the hardware is in a known initial state again.
    
    If the display stays black, and your pins are correct, you may need to change between reset_state=0 and reset_state=1.
    
    If the colors are off, play around with `color_space=lv.COLOR_FORMAT.RGB565`, `color_byte_order=st7789.BYTE_ORDER_BGR` and `rgb565_byte_swap=True`.

    If you're successful, the display will still be black because LVGL isn't drawing anything to it yet.
    So use this code (based on main.py and helloworld.py) at the end to display something:
    <pre>
    ```
    import lvgl as lv
    import task_handler
    task_handler.TaskHandler(duration=5)
    label = lv.label(lv.screen_active())
    label.set_text('Hello World!')
    label.center()
    ```
    </pre>

3. Put the initialization code in a custom boot_...py file for your device

4. Copy the custom boot_...py file and the generic MicroPythonOS files to your device (see [install.sh](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/scripts/install.sh)

    After reset, your custom boot_...py file should initialize the display and then MicroPythonOS should start, run the launcher, which shows the icons etc.

5. Add more hardware support

    If your device has a touch screen, check out how it's initialized for the [Waveshare 2 inch Touch Screen](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/internal_filesystem/boot.py).
    If it has buttons for input, check out the KeyPad code for the [Fri3d Camp 2024 Badge](https://github.com/MicroPythonOS/MicroPythonOS/blob/main/internal_filesystem/boot_fri3d-2024.py).
    
    Now you should be able to control the device, connect to WiFi and install more apps from the AppStore.
    
    This would be a good time to create a pull-request to merge your boot_...py file into the main codebase so the support becomes official!

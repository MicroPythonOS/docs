By default, the unit tests will run automatically when the [GitHub workflows](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/.github/workflows) are triggered.

But when you're working on a unit test, or investigating a failure, you probably want to run it locally for quick iteration.

The tests are stored in [tests/](https://github.com/MicroPythonOS/MicroPythonOS/tree/main/tests)

Simple (non-graphical) test filenames start with `test_` and ends with `.py`.

Graphical tests start with `test_graphical_` and also end with `.py`.

# On Desktop

The following assumes you have MicroPythonOS running on desktop, meaning this works:

```
./scripts/run_desktop.sh
```

Then you can run a specific unit test by providing its file as an argument:

```
./tests/unittest.sh tests/test_package_manager.py
```

To run all unit tests, do:

```
./tests/unittest.sh
```

This takes some time, during which you'll see the MicroPythonOS window pop up often.

There's also a syntax checker:

```
./tests/syntax.sh
```

# On Device

The unit tests can also run on a physical device, like an ESP32 that's connected with a USB cable, which is more representative and can help to test features that are not available on desktop.

This takes quite some time, because it restarts the device before each test, to make sure there's no leftover state from a previous test lingering.

Note that, since these tests are currently not automatically checked, some of these might have accidentally been broken without it being noticed.
If you see something you can fix, feel free to do so, but you're not expected to clean up someone else's mess if you didn't break it yourself.

The following assumes you have the MicroPythonOS REPL shell show up when you run mpremote.py, meaning this works:

```
./lvgl_micropython/lib/micropython/tools/mpremote/mpremote.py ls 
```

Then you can run a specific unit test by adding the --ondevice option and providing its file as an argument:

```
./tests/unittest.sh --ondevice tests/test_package_manager.py
```

To run all unit tests, do:

```
./tests/unittest.sh --ondevice # this takes a long time
```



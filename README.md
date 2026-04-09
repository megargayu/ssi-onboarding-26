# SSI Onboarding '26

If you haven't already, please read [`ONBOARDING.md`](https://github.com/stanford-ssi/samwise-flight-software/blob/main/docs/ONBOARDING.md) in the original `samwise-flight-software` repository.

## Files

1. `.bazelversion` - Tells bazel what version to put us on. Right now, this is set to `8.5.1`.
2. `MODULE.bazel` - The build configuration for bazel.
   - This creates a `module` for `ssi-onboarding-26`.
   - This adds the dependencies of `platforms`, `pico-sdk`, and configures `pico-sdk` to target `rp2350` (our pico version) and use specific toolchains for Mac/Linux/WSL2.
3. `.bazelrc` - Configurations for different _types_ of builds for `bazel`. This is more important for `samwise-flight-software`, as we have different places we want to build (more on this later) - but for this, we just have `pico`.
   - Sets version of C and C++, as well as some common configs
   - Creates a `pico` build profile that properly links to `pico-sdk` and adds debugging symbols (can be useful for debugging with a RasPi Debug Probe).
4. `BUILD.bazel` - The actual entry point of bazel, which defines what actually get compile.
   - Creates a binary, `ssi-onboarding-26`, which compiles `src/main.c`, and depends on a lot of `pico-sdk` targets that are required for working with `pico-sdk`.
   - In `samwise-flight-software`, `deps` essentially contains every piece of code we write, using dependency tree analysis to stitch together the entire codebase.
5. `bzl/bin_aspect.bzl` - Don't worry too much about this, it essentially uses bazel's `aspect` to link to `pico-sdk` properly. This is frankly overkill and we probably don't need it.
   - Note that while this code _looks_ like `Python`, it is actually something called [`Starlark`](https://github.com/bazelbuild/starlark) which is a dialect of `Python`. It's essentially a `Makefile` but in `Python`.
6. `bzl/BUILD.bazel` - Allows `bzl` to be its own module so that bazel can find `bzl/bin_aspect.bzl`.
7. `platforms/BUILD.bazel` - Platform definitions that are referenced in `MODULE.bazel`.

Don't worry if you don't understand this all - all of this code is already written for SAMWISE, so you may never need to interact with these components! It's still good to have an idea of what each thing does.

## Code

Put this in `src/main.c`:

```c
#include "pico/stdlib.h"

int main() {
    stdio_usb_init();

    gpio_init(PICO_DEFAULT_LED_PIN);
    gpio_set_dir(PICO_DEFAULT_LED_PIN, GPIO_OUT);
    while (1) {
        gpio_put(PICO_DEFAULT_LED_PIN, 0);
        sleep_ms(250);
        gpio_put(PICO_DEFAULT_LED_PIN, 1);
        sleep_ms(1000);
    }
}
```

## Building

Run:

```bash
bazel build :ssi-onboarding-26 --config=pico
```

Which should generate a bunch of folders, including most importantly `bazel-bin/ssi-onboarding-26.uf2`, which is the actual binary! This is essentially the same command for `SAMWISE`, except we use `:samwise` (a different target name) and various configurations, as explained briefly above (`.bazelrc`).

## Uploading

1. Unplug your PICO
2. Hold the BOOT button on your PICO
3. While holding, plug it in. A window should pop up with a directory that corresponds to the PICO, or it should generally be available in Finder/Windows Explorer/etc.
4. Copy `bazel-bin/ssi-onboarding-26.uf2` into the drive that shows up.
5. The folder should close almost immediately, and after the pico reboots, the light should start to blink!

If all of this works, success!

## Uploading (Cool version)

If you are not all about that drag-and-drop life:

1. Install [`picotool`](https://github.com/raspberrypi/pico-sdk-tools/releases) from releases for your computer.
   - Linux: I recommend putting this in `/usr/local/bin/picotool` and then adding `/usr/local/bin` to `PATH` if it's not there already.
     - To check PATH, run `echo $PATH | grep /usr/local/bin`. If the output has a highlighted version of that string, then you don't need to do anything.
     - To add to PATH, edit `~/.bashrc` or `~/.zshrc` if you are using `zsh`, and add the line:
       ```bash
       export PATH="/usr/local/bin:$PATH"
       ```
       Then restart your shell.
   - Mac: `brew install picotool` should work.
   - Windows: Should be similar to Linux, but I've never tried it before. (Good luck!)
2. Unplug your PICO
3. Hold the BOOT button on your PICO
4. While holding, plug it in. A window should pop up with a directory that corresponds to the PICO, or it should generally be available in Finder/Windows Explorer/etc.
5. In the repository, run:

   ```bash
   picotool load bazel-bin/ssi-onboarding-26.uf2 -f
   ```

   You may have to click the "RESET" button after doing this - but the LED should start blinking!

   Note that you may have to put `sudo` at the beginning if it doesn't work.

   Additionally, if you are on `WSL`, you must use `usbipd` (see instructions below)

Note this is cool but also useful. You no longer need to reattach and attach the stick now. If you repeat step 5 again and again, it will automatically force the pico to reboot and then flash the new code. Try it out yourself - edit `main.c` and try again! (Make sure to build first!).

## WSL2 Integration

If you are on WSL2, you also have to do an additional step:

Your pico is only accessible from Windows by default. In order for WSL to see it, you have to tell Windows to link it into WSL.

Every time you connect the Pico and either want to `tio` or `picotool`, therefore:

1. Unplug your PICO
2. Hold the BOOT button on your PICO
3. While holding, plug it in. A window should pop up with a directory that corresponds to the PICO, or it should generally be available in Finder/Windows Explorer/etc.
4. From powershell, run `usbipd list`. You should see a bunch of stuff, including:

```
BUSID  VID:PID    DEVICE                                                        STATE
2-3    2e8a:000f  USB Mass Storage Device, RP2350 Boot                          Not shared
```

Each `BUSID` is a port on your computer. Therefore, it usually is the same if you plug it into the same port, but can change around otherwise.

5. If `STATE == Not shared`, then run `usbipd bind --busid {your-bus-id}`. For example, I would use `usbipd bind --busid 2-3` for myself. Usually, you only need to do this once, unless you change ports, update, etc.

6. Now that state is shared, run:

   ```powershell
   usbipd attach --wsl --busid {your-bus-id} --auto-attach
   ```

   It should output like:

   ```
   usbipd: info: Using WSL distribution 'Ubuntu' to attach; the device will be available in all WSL 2 distributions.
   usbipd: info: Loading vhci_hcd module.
   usbipd: info: Detected networking mode 'nat'.
   usbipd: info: Using IP address 172.17.192.1 to reach the host.
   usbipd: info: Starting endless attach loop; press Ctrl+C to quit.
   WSL Monitoring host 172.17.192.1 for BUSID: 2-3
   WSL 2026-04-08 17:39:21 Device 2-3 is available. Attempting to attach...
   WSL 2026-04-08 17:39:21 Attach command for device 2-3 succeeded.
   ```

   And also make the signature Windows detach noise (indicating it is no longer attached to Windows).

   This will automatically repeatedly attach your pico to WSL, until you quit the program (i.e. CTRL+C). So if you reboot or reflash any software, it will keep trying to attach your pico to WSL.

7. Now, `picotool info` (or `sudo picotool info`) should say something like:
   ```
   Program Information
   binary start:  0x10000000
   binary end:    0x10003cd4
   target chip:   RP2350
   image type:    ARM Secure
   ```
   And running step 5 in "Uploading (Cool version)" should now produce a blinking LED! Note that it will make the attach/detach noise a lot of times as the `--auto-attach` in the Powershell window will force it to attach to WSL as soon as possible. This may be annoying; you may have to deal with it :(.

## Reading from PICO

What if we want to read logs in real time?

1. Install `tio`:
   - Mac: `brew install tio`
   - Linux: `sudo snap install tio --classic`
     - If you don't have `snap`, install it, for example, on Ubuntu:
       ```bash
       sudo apt update
       sudo apt install snapd
       ```
     - Note you may have to add snap's bin to path - so in `.bazelrc` or `.zshrc`:
       ```bash
       export PATH="/snap/bin:$PATH"
       ```
   - Windows: No idea :)
2. Now, let's edit `main.c` to actually log a counter:

   ```c
   #include "pico/stdlib.h"
   #include "pico/printf.h"

   int main() {
       stdio_usb_init();
       int counter = 0;

       gpio_init(PICO_DEFAULT_LED_PIN);
       gpio_set_dir(PICO_DEFAULT_LED_PIN, GPIO_OUT);
       while (1) {
           gpio_put(PICO_DEFAULT_LED_PIN, 0);
           sleep_ms(250);
           gpio_put(PICO_DEFAULT_LED_PIN, 1);
           sleep_ms(1000);

           printf("Hello, world! Our counter is: %d\n", counter);
           counter++;
       }
   }
   ```

3. Now, before we flash, let's setup `tio` by running (in either a regular terminal or on WSL, a separate *Linux* terminal while still having the `usbipd attach` command running):
    ```bash
    tio /dev/ttyACM0
    ``` 
    (You may need to do `sudo`).

4. Now, either use method (a) or the cool version to build and flash the new code onto the pico.
5. You should see something like this!:
    ```
    [17:56:05.115] Waiting for tty device..
    [17:56:09.122] Connected to /dev/ttyACM0
    Hello, world! Our counter is: 2
    Hello, world! Our counter is: 3
    Hello, world! Our counter is: 4
    Hello, world! Our counter is: 5
    Hello, world! Our counter is: 6
    Hello, world! Our counter is: 7
    Hello, world! Our counter is: 8
    Hello, world! Our counter is: 9
    Hello, world! Our counter is: 10
    ```

    Note that sometimes the first few logs are not available as `tio` connects after the code already started running on the pico.



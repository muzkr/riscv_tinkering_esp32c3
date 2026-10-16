
## Prologue

I originally started working on the RP2350, but then I learned about the ESP32-C3 from somewhere, and one of its features caught my great interest: the USB Serial/JTAG Controller. Basically, this is a hardware-hardwired CDC + JTAG composite device that uses only 2 pins - instead of 6 (JTAG 4 + UART 2) - to provide JTAG debugging and serial printing - a valuable feature for development. A very clever design.

ESP32 chips are very popular. The documentation quality is quite high (especially for a Chinese chip manufacturer). I briefly browsed the English datasheet and TRM, and while a few expressions are less clear, I have the advantage of being able to read the Chinese documentation side by side. IDF is the SDK for ESP32 (libraries + tools), and I've seen positive reviews from developers. I browsed the documentation and consider it to be high quality.

> https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/get-started/index.html

IDF provides a GUI installer (called EIM) that supports mainstream operating systems. The installation process was generally smooth and the guidance was satisfactory, except for a few places lacking clear instructions - what I personally care about most is the custom installation location. I had to reinstall once to get the desired result: a custom IDF installation involves 4 directories:

- `.espressif` directory. This is what EIM calls the installation location, which is actually the local repository for IDF
- `tools` and `dist` directories, which EIM does not clearly explain. **These must be customized**, otherwise they will be installed under `~/.espressif` (even if a custom `.espressif` directory was specified above). These are the tool installation directory and the compressed package download directory, respectively
- `~/.espressif`: There's not much content here, mainly EIM installation records and some scripts

Basically, the entire IDF installation is a Python venv. Activate it by sourcing the script, in my particular case:

> ~/.espressif/tools/activate_idf_v6.1.sh

Alternatively, after running EIM, it will list installed IDF versions and provide activation of the venv (which will open a terminal emulator).

Currently, I feel I'll mainly use tools/openocd-esp32, which supports the ESP32-C3's unique USB serial/JTAG interface. For the toolchain, I'll use the RISC-V32 GCC provided by xpack. I might use a few other IDF tools, like `idf.py` and `espefuse`, etc. I'm new to ESP32/IDF, so I'll explore gradually.

## The Chip

There are many cheap development boards, so-called core boards or mini boards, which are minimal systems. Prices are around $2. I bought a higher quality one at the upper end of the price range. It's the Fx4 model with 4 MB (!) built-in flash.

- CPU: rv32imc, up to 160 MHz
- 400 KB SRAM (?!), plus 8 KB SRAM in the RTC domain
- QFN32 package (very small!), 16 GPIO pins (very few!)

The default clock source is XTAL / 2. The board has a 40 MHz crystal (for now, I can keep the default for a while without needing special clock configuration).

Boot modes include flash boot mode and download mode, depending on efuse configuration and the state of several strapping pins. Under default configuration, pulling gpio9 high will enter flash boot mode.

On the board, gpio9 is pulled up (thus defaulting to flash boot), and can be pulled down to GND via a push button (to enter download mode).

In download mode, programs can be flashed via USB serial/JTAG or UART0.

Flash start address is 0x42000000.

The TRM's description of the boot ROM is extremely incomplete, basically lacking. There are 2 types of flash boot mode:

- Normal: Load the program from flash to SRAM and execute
- Direct: The first 2 words of flash are 0xaedb041d

For both boot methods, what is the format/layout of the program binary? What size? Does it need some kind of partition? How to specify the entry function (initial PC) and initial SP? ...

The "IDF User Guide > API Guides > App Startup Flow" explains normal boot. It has partitions and boot stage2. But my main plan is for direct mode. Now I'm starting to doubt the quality (completeness) of ESP32 documentation...

To be honest, the RP2350 datasheet is somewhat stiff and difficult to read, but the information completeness is not a problem. Should I go back to RP2350?


## Direct Boot

OK.. there's an official direct boot example:

> https://github.com/espressif/esp32c3-direct-boot-example

I have cloned it into this repository, at the root. According to the [README](./esp32c3-direct-boot-example/README.md): 

- Direct boot is supported from silicon rev 3: I need to check that of mine
- Simply prefix the binary with two 0xaedb041d magic words. The boot ROM sets up the MMU to map flash to virtual addresses 0x42000000 (for code execution) and 0x3C000000 (for read-only data access), then jumps to address 0x42000008 (i.e., the instruction at offset 8 in flash, immediately after the magic numbers) - that is the entry point of the program

To check the chip revision number, run the following command:

```
$ esptool chip-id
...
Connected to ESP32-C3 on /dev/ttyACM0:
Chip type:          ESP32-C3 (QFN32) (revision v0.4)
Features:           Wi-Fi, BT 5 (LE), Single Core, 160MHz, Embedded Flash 4MB (XMC)
Crystal frequency:  ...
```

According to the document "Compatibility Advisory for Chip Revision Numbering Scheme", rev 3 and rev 4 correspond to v0.3 and v0.4 in the current revision naming scheme, respectively. So I can confirm that my chip supports direct boot. 


## Address Map

SRAM address mapping:

```
           DBUS               IBUS
                +-----------+ 0x4037_c000
                | SRAM0     |
                | (16 KB)   |
    0x3fc8_0000 +-----------+ 0x4038_0000
                | SRAM1     |
                | (384 KB)  |
                |           |
    0x3fce_0000 +-----------+ 0x403e_0000
```

SRAM0 is used as flash cache.

Flash address mapping is realized by the MMU. The DBUS and IBUS each have 8 MB of address space. The flash physical addresses mapped by DBUS and IBUS don't need to be aligned or completely overlapping, so the maximum accessible physical flash size is 16 MB.

In direct boot mode, the boot ROM configures the MMU so that DBUS and IBUS map to the same physical address range, as follows:

```
           DBUS               IBUS
    0x3c00_0000 +-----------+ 0x4200_0000
                | Flash     |
                | (8 MB Max)|
                |           |
```

I think the biggest missing piece in the DS/TRM documentation is about the boot ROM and flash access:

- Boot ROM: boot behavior/logic, common functions, etc.
- MMU
- SPI0/1

Flash, either on-chip or off-chip, is accessed via SPI0/1.

I believe the boot ROM contains common functions for flash erase/programming; otherwise, how would IAP work? I guess the IDF SDK should/most likely contains related APIs. I really hate and resist studying IDF, or even reverse engineering the boot ROM.

## The Board

Details of the development board.

As mentioned earlier, the board has a 40 MHz crystal.

The CHIP_EN pin is pulled up, and can be pulled down to ground via a push button to reset.

GPIO8 controls an LED, active low. It's pulled up, so the LED is off by default.

GPIO 2/8/9 are strapping pins that determine the boot mode. All are pulled up; GPIO9 is connected to ground via a push button.

- Default (GPIO9 pulled up): flash boot
- Push button pressed (GPIO9 grounded): GPIO2/8 are pulled up, so download mode

There is one USB port connected to USB serial/JTAG.

GPIO pins are brought out to headers on both sides of the board, including CHIP_EN and GPIO 2/8/9 mentioned above. Power-related pins are also brought out to headers.

The board features a PCB antenna; external antennas are not supported.

# CrossPoint Reader for the Seeed reTerminal E1001

This firmware turns the [Seeed reTerminal E1001](https://wiki.seeedstudio.com/getting_started_with_reterminal_e1001/) into an e-book reader.

It is a port of [CrossPoint Reader](https://github.com/crosspoint-reader/crosspoint-reader), an open-source e-reader firmware. For the full feature list, read the [upstream README](https://github.com/crosspoint-reader/crosspoint-reader#readme).

## What works

- Reads EPUB books from the microSD card.
- Uses the full 7.5 inch screen in landscape (800 x 480).
- Turns pages with the keys on the top edge.
- Shows the battery level.

## What does not work yet

- Grayscale text. Text shows in black and white only.
- Online updates. To update, flash a new file over USB or use the SD card update in Settings.
- Sleep and wake. These are not fully tested.

## What you need

- A reTerminal E1001. Board version 1.2 is tested. Version 1.0 is not tested.
- A microSD card, formatted FAT32 or exFAT.
- A USB-C data cable.
- A computer with Python 3.

## Warning

Flashing this firmware erases the Seeed firmware. To go back, flash the Seeed firmware again with the tools on the [Seeed wiki](https://wiki.seeedstudio.com/getting_started_with_reterminal_e1001/).

To keep a copy of your current firmware, back it up before you flash. The backup takes about 25 minutes:

```
esptool -c esp32s3 -p PORT -b 230400 read-flash 0x0 0x2000000 backup.bin
```

## Flash the firmware

1. Install esptool: `pip install esptool`
2. Download `crosspoint-reterminal-e1001-full.bin` from [Releases](../../releases).
3. Set the power switch on the device to ON.
4. Connect the device to your computer with the USB cable.
5. Find the port name:
   - macOS: `ls /dev/cu.usbserial-*`
   - Linux: `ls /dev/ttyUSB*`
   - Windows: open Device Manager and look under "Ports" for a COM port.
6. Run this command. Replace `PORT` with your port name:

   ```
   esptool -c esp32s3 -p PORT -b 230400 write-flash 0x0 crosspoint-reterminal-e1001-full.bin
   ```

7. Wait for "Hash of data verified". The device restarts and shows the home screen.

Keep the speed (`-b`) at 230400 or lower. Faster speeds fail on this device.

## Add books

1. Copy `.epub` files to the microSD card.
2. Put the card in the device.
3. Restart the device.

## Use the keys

Hold the device in landscape with the keys on top.

| Key | Press | Hold 1 second | Hold 3 seconds |
|---|---|---|---|
| Left | Previous page or item | | |
| Middle | Next page or item | | |
| Right | Select | Back | Sleep |

## Build from source

You need [pioarduino](https://github.com/pioarduino/platformio-core) 6.1.19. Newer PlatformIO versions fail to build this project.

```
pip install pioarduino==6.1.19
git clone --recursive -b reterminal-e1001 https://github.com/StreamlinedStartup/crosspoint-reader.git
cd crosspoint-reader
pio run -e reterminal_e1001 -t upload --upload-port PORT
```

Hardware notes for developers are in [docs/reterminal-e1001.md](docs/reterminal-e1001.md).

## License

MIT, the same as CrossPoint Reader. See [LICENSE](LICENSE).

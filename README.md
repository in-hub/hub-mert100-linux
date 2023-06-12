# HUB-MERT100 / HUB-RT100 on Linux

## Driver for MCP2221 chip

Both device types are based on an MCP2221 chip. This chip is an USB to I²C protocol converter and allows communication with the internal real-time clock (RTC) chip (both device types) and the FRAM chip (HUB-MERT100 only). While the corresponding `hid-mcp2221` driver is shipped with Linux kernel 6.0 and newer, it's recommended to build the version supplied by in.hub since it contains additional bug fixes required for proper and performant communication with the internal chips.

### Prepare kernel driver

1. Update kernel source tree:
   - Linux kernel 6.0 and newer: replace the existing file `drivers/hid/hid-mcp2221.c` in your Linux kernel source tree with [the in.hub MCP2221 driver](https://download.inhub.de/mert100/hid-mcp2221.c).
   - Linux kernel older than 6.0: apply the [MCP2221 driver integration patch](https://download.inhub.de/mert100/add-hid-mcp2221-driver.patch) to your Linux kernel source tree.
2. Make sure the kernel config options `CONFIG_HID_MCP2221`, `CONFIG_HIDRAW` and `CONFIG_USB_HID` are set.
3. Rebuild your kernel and deploy it to your device.

### Test kernel driver

1. Boot your device with the updated Linux kernel
2. Attach the HUB-MERT100/HUB-RT100 stick to a USB port of your device.
3. Run `lsmod | grep mcp2221` which should indicate that the kernel module `hid_mcp2221` is loaded
4. Verify that at least once device instance symlink exists at `/sys/bus/hid/drivers/mcp2221` by issueing `cd /sys/bus/hid/drivers/mcp2221/*/i2c-*/` and checking that the file `new_device` exists.

## Real-time clock

Both devices types are equipped with an M41T82 RTC chip. The corresponding Linux kernel driver is called `rtc-m41t80` and needs to be replaced with a customized version in order to be compatible with the MCP2221 I²C chip.

### Prepare kernel driver

1. Apply the [M41T80 driver patch](https://download.inhub.de/mert100/rtc-m41t80-use-byte-transfers-instead-of-block-reads.patch)
2. Make sure the kernel config option `CONFIG_RTC_DRV_M41T80` is enabled.
3. Rebuild your kernel and deploy it to your device.

### Instantiate kernel driver

Run

    echo m41t82 0x68 > /sys/bus/hid/drivers/mcp2221/*/i2c-*/new_device

to instantiate a new RTC instance at address 0x68 of the MCP2221 I²C bus.

Now check for a new device `/dev/rtcX`. The exact name depends on the number of RTCs that already exist. If `/dev/rtc0` already existed before, the HUB-(ME)RT100-RTC appears as `/dev/rtc1`. The command

    dmesg | grep rtc-m41t80

can be used to search for RTC related kernel log messages. They should indicate which RTC device was instantiated, e.g:

    rtc-m41t80 8-0068: registered as rtc1

Now the RTC can be used as usual with the `hwclock` tool, e.g. `hwclock -w -f /dev/rtc1` to write the current system time to the RTC or `hwclock -r -f /dev/rtc1` to read it again and use it as system time.

## FRAM memory

HUB-MERT100 devices additionally are equipped with a non-volatile I²C FRAM memory which is accessible via the MCP2221 chip. Since the FRAM chip follows the common EEPROM I²C protocol, the `at24` EEPROM driver can be used. Make sure your kernel has the `at24` driver included (kernel config option `CONFIG_EEPROM_AT24`).

### Setting I/O limit

It is neccessary to limit the size of I/O transfers to 60 bytes since the internal buffer of the MCP2221 chip is not large enough to hold more data. For doing so,

a) add the `at24.io_limit=60` parameter to your kernel command line

OR

b) pass `io_limit=60` as additional parameter when calling `modprobe at24`

OR

c) create a file `/etc/modprobe.d/at24.conf` with a line `options at24 io_limit=60`

### Instantiate EEPROM device file

Now the EEPROM driver can be instantiated with a simple command:

    echo 24c1024 0x50 > /sys/bus/hid/drivers/mcp2221/*/i2c-*/new_device

The kernel should print the following log messages:

    at24 3-0050: 131072 byte 24c1024 EEPROM, writable, 1 bytes/write
    i2c i2c-3: new_device: Instantiated device 24c1024 at 0x50

The EEPROM device file representing the contents of the FRAM should have a size of 131072 (128 kB):

    ls -l /sys/bus/hid/drivers/mcp2221/*/i2c-*/*-0050/eeprom
    -rw------- 1 root root 131072 /sys/bus/hid/drivers/mcp2221/0003:04D8:00DD.0001/i2c-3/3-0050/eeprom

The EEPROM device file can be read from and written to like a regular file.

**IMPORTANT:** Due to limitations of the MCP2221 driver, currently only transfers (read/write transactions) of up to 60 bytes are possible. Always check for -EIO and -ETIMEDOUT when reading to or writing from the memory device file and repeat the operation if necessary.

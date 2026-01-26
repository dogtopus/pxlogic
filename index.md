# PXLogic protocol documentation

## lsusb

See [lsusb.txt](./lsusb.txt)

## Channels

- Interface 0
    - Channel 1: Control register access channel
    - Channel 2: Data acquisition channel
    - Channel 3: Device configuration data channel
- Interface 1
    - Channel 4: Unknown registers (used for debugging?)
    - Channel 5: Unknown
    - Channel 6: Unknown
    - Channel 7: Unknown

## Firmware files

There are 3 firmware files: the MCU firmware (`SCI_LOGIC.bin`), the FPGA stage 1 firmware (`hspi_ddr_RST.bin`) and the FPGA stage 2 firmware (`hspi_ddr.bin`).

To initialize the device, the following steps should be performed:

1. Check `MCU_FW_VERSION` with the last known firmware version. If mismatch, send the MCU firmware to the device.
2. Use `MCU_RESET` register to reset the MCU if firmware update has been performed.
3. Send the FPGA stage 1 firmware.
4. Send the FPGA stage 2 firmware.

## Control register

Control registers are 32-bit registers that contain device identification, device states, and can be used to control device function.

There are 2 banks of registers, starting at address `0x0000` and `0x2000` respectively. The bank at `0x0000` seems to be sampler configuration registers, while the bank at `0x2000` seems to be general device configuration registers.

These registers are accessed through Interface 0, Channel 1 via a simple packet-based protocol. Each request and response packet consists the following (in little-endian):

| Offset | Type | Name | Description |
| - | - | - | - |
| 0 | u32 | sync_dir | First 16 bit is always `0xfefe`. Last bit indicates direction (0: write, 1: read) |
| 4 | u32 | len | Always `0x08`. |
| 8 | u32 | reg_addr | Register address. |
| 12 | u32 | reg_data | New value to be written / Current value in register. |

If the device successfully fulfilled a write request, the `reg_data` in the response packet shall be set to `0xfefefefe` to indicate success.

### Register map

| Offset | Name | R/W | Description |
| - | - | - | - |
| 0x0000 | MODE | W | Sampler mode config. |
| 0x0010 | CHANNEL_MASK | W | Channel mask configuration. |
| 0x0014 | CLK_CONF | W | Sampler clock config. |
| 0x0018 | CLK_DIV | R/W | Sampler clock divider. |
| 0x001c | SAMPLE_FRAME_SIZE | W | Sampler frame size (guess) (`BUFSIZE` in PXView). |
| 0x0020 | - | W | Unknown. Should write 0 to it after setting `TRIG_*` |
| 0x0024 | TRIG_ZERO | W | Low level trigger enable. |
| 0x0028 | TRIG_ONE | W | High level trigger enable. |
| 0x002c | TRIG_RISE | W | Rising edge trigger enable. |
| 0x0030 | TRIG_FALL | W | Falling edge trigger enable. |
| 0x2008 | XFER_FRAME_SIZE | W | Maximum trasfer frame size (guess) (`BUFSIZE` in PXView). |
| 0x200c | FWRAM_READ_START | W | Start address of the FWRAM read request. Must be page-aligned (4KiB). |
| 0x2010 | FWRAM_READ_END | W | End address of the FWRAM read request. Must be page-aligned (4KiB). |
| 0x2014 | FWRAM_READ_PAGE | W | FWRAM page to read from. |
| 0x2018 | FWRAM_WRITE_START | W | Start address of the FWRAM write request. Must be page-aligned (4KiB). |
| 0x201c | FWRAM_WRITE_END | W | End address of the FWRAM write request. Must be page-aligned (4KiB). |
| 0x2020 | FWRAM_WRITE_PAGE | W | FWRAM page to write to. |
| 0x2024 | NUM_SAMPLES_LO | W | Lower 32-bit of number of samples to take (in bytes) (`limit_samples2Byte` in PXView) |
| 0x2028 | NUM_SAMPLES_HI | W | Upper 32-bit of number of samples to take (in bytes) (`limit_samples2Byte` in PXView) |
| 0x202c | BLOCK_START | W | Should be set to 0 (`set_block_start` in PXView). |
| 0x2030 | MCU_RESET | W | Writing 0 to it resets the USB interface controller. |
| 0x2034 | MCU_FW_VERSION | R | The USB interface controller firmware version. |
| 0x204c | ENABLED_NUM_CH | W | Total number of channels enabled (`ch_num` in PXView). |
| 0x2050 | TRIGGER_POS_SET | W | TODO: Unknown (`trigger_pos_set` in PXView). |
| 0x2054 | TRIGGER_POS_REAL | R | Unused. TODO: Unknown (`trigger_pos_real` in PXView). |
| 0x2058 | DEV_VARIANT | R | Device variant (`logic_mode` in PXView). |

### CLK_CONF

| Bitpos (E:I) | Name | Description |
| - | - | - |
| 3:0 | - | Unknown. Should be set to 0. |
| 6:3 | CLK_CONF_SELECT | Sampler clock select (`gpio_mode` in PXView). |
| 32:6 | - | Unknown. Should be set to 0. |

#### CLK_CONF_SELECT

Select a base sampler clock. The `F_SAMPLER` is determined by this value.

| Value | Name | Notes |
| - | - | - |
| 0 | CLK_1GHZ | 1 GHz sampler clock. |
| 1 | CLK_500MHZ | 500 MHz sampler clock. |
| 2 | CLK_250MHZ | 250 MHz sampler clock. |
| 3 | CLK_125MHZ | 125 MHz sampler clock. |
| 4 | CLK_800MHZ | 800 MHz sampler clock. |
| 5 | CLK_400MHZ | 400 MHz sampler clock. |
| 6 | CLK_200MHZ | 200 MHz sampler clock. |
| 7 | CLK_100MHZ | 100 MHz sampler clock. |

### CLK_DIV

The clock divider value (minus 1) of the sampler.

Actual clock frequency is determined by `F_SAMPLER / (CLK_DIV + 1)`. For example, `CLK_DIV` of 1 with `F_SAMPLER` of 100MHz results in a final sampler clock rate of 50MHz.

### FWRAM_READ_PAGE / FWRAM_WRITE_PAGE

| Value | Name | Notes |
| - | - | - |
| 0 | MCU_PROG_FLASH | First 48KiB is bootloader (32KiB is currently used), then the main program starts at the 48KiB mark. |
| 1 | FPGA_FLASH | FPGA Flash (unused). |
| 2 | FPGA_RAM | FPGA RAM (unused). |
| 4 | FPGA_CFGRAM | Where the bit file goes. Length varies. |

### DEV_VARIANT

Known values are as follows:

| Value | Name | Device Name | Notes |
| - | - | - | - |
| 0 | VARIANT_32CH | PX Logic 32 | 32 ch, 1Gsps |
| 1 | VARIANT_16CH_1000 | PX Logic 16 Pro | 16 ch, 1Gsps |
| 2 | VARIANT_16CH_500 | PX Logic 16 Plus | 16 ch, 500Msps |
| 3 | VARIANT_16CH_250 | PX Logic 16 Base | 16 ch, 250Msps |

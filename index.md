# PXLogic info and protocol documentation

PXLogic is a series of ultra-low-cost logic analyzers that has theoretical performance comparable to the DreamSourceLab DSLogic U3 series, but available at a much lower price point due to the use of lower cost components.

Depending on the hardware configuration, it supports up to 32 channels, external trigger input / output, 1 PWM output channel of up to 1MHz, and sample rate up to 1Gsps (with 2 channels enabled simultaneously in streaming mode, and 8 channels enabled simultaneously in buffered mode).

## Hardware

- **FPGA**: Pangomicro Logos family (PGL22G-6CMBG324)
- **USB interface controller**: WCH CH569W
- **RAM**: Micron MT41K256M16TW-107:P DDR3 256Mx16bit
- **PMIC**: Everanalog EA3059
- **Input voltage divider**: 1:2 (standard voltage divider made with two 100kOhm resistors)
- **Input ESD protection**: 9x STMicroelectronics USBLC6-4SC6 TVS diode array
- **USB-C mux**: Via Labs VL162 (purpose unknown)
- **USB HighSpeed hub controller**: CoreChips SL2.1s (purpose unknown)

## lsusb

See [lsusb.txt](./lsusb.txt)

## Endpoints

- Interface 0
    - Endpoint 1: Control register access endpoint
    - Endpoint 2: Sample data FIFO endpoint
    - Endpoint 3: Device configuration / firmware RAM data FIFO endpoint
- Interface 1
    - Endpoint 4: Unknown registers (used for debugging?)
    - Endpoint 5: Unknown
    - Endpoint 6: Unknown
    - Endpoint 7: Unknown

## Firmware files

There are 3 firmware files: the MCU firmware (`SCI_LOGIC.bin`), the FPGA stage 1 firmware (`hspi_ddr_RST.bin`) and the FPGA stage 2 firmware (`hspi_ddr.bin`).

To initialize the device, the following steps should be performed:

1. Check `MCU_FW_VERSION` with the last known firmware version. If mismatch, send the MCU firmware to the device.
2. Use `MCU_RESET` register to reset the MCU if firmware update has been performed.
3. Send the FPGA stage 1 firmware.
4. Send the FPGA stage 2 firmware.

## Control register

Control registers are 32-bit registers that contain certain device identification information, device states, and can be used to control device function.

There are 2 banks of registers, starting at address `0x0000` and `0x2000` respectively. The bank at `0x0000` seems to be sampler configuration registers, while the bank at `0x2000` seems to be general device configuration registers.

These registers are accessed through Interface 0, Endpoint 1 via a simple packet-based protocol. Each request and response packet consists the following (in little-endian):

| Offset | Type | Name | Description |
| - | - | - | - |
| 0 | u32 | sync_dir | Higher 16 bit is always `0xfefe`. Last bit indicates direction (0: write, 1: read) |
| 4 | u32 | len | Always `0x08`. |
| 8 | u32 | reg_addr | Register address. |
| 12 | u32 | reg_data | New value to be written / Current value in register. |

If the device successfully fulfilled a write request, the `reg_data` in the response packet shall be set to `0xfefefefe` to indicate success.

### Register map

| Offset | Name | R/W | Description |
| - | - | - | - |
| 0x0000 | MODE | R/W | Sampler mode config. |
| 0x0004 | PWM_VREF_CMP_PERIOD | R/W | Coarse VREF generator period comparator input. (`pwm_max` in PXView). |
| 0x0008 | PWM_VREF_CMP_DUTY | R/W | Coarse VREF generator duty cycle comparator input. |
| 0x0010 | CHANNEL_EN | W | Channel enable. |
| 0x0014 | CLK_CONF | R/W | Sampler clock config. |
| 0x0018 | CLK_DIV | R/W | Sampler clock divider. |
| 0x001c | SAMPLE_FRAME_SIZE | W | Sampler frame size (guess) (`BUFSIZE` in PXView). |
| 0x0020 | STOP | W | Stop sampler. |
| 0x0024 | TRIG_LOW | W | Low level trigger bitmask. |
| 0x0028 | TRIG_HIGH | W | High level trigger bitmask. |
| 0x002c | TRIG_RISING | W | Rising edge trigger bitmask. |
| 0x0030 | TRIG_FALLING | W | Falling edge trigger bitmask. |
| 0x003c | TRIG_EXT_MODE | W | External trigger mode. |
| 0x0040 | PWM0_CONF | W | PWM0 config. |
| 0x0044 | PWM0_CMP_PERIOD | W | PWM0 period comparator input. |
| 0x0048 | PWM0_CMP_DUTY | W | PWM0 duty cycle comparator input. |
| 0x004c | PWM1_CONF | W | PWM1 config. |
| 0x0050 | PWM1_CMP_PERIOD | W | PWM1 period comparator input. |
| 0x0054 | PWM1_CMP_DUTY | W | PWM1 duty cycle comparator input. |
| 0x0058 | TRIG_OUT_EN | W | Trigger out enable. |
| 0x2008 | XFER_FRAME_SIZE | W | Maximum transfer frame size (guess) (`BUFSIZE` in PXView). |
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
| 0x2050 | TRIGGER_POS | W | Desired trigger position (capture ratio in samples). |
| 0x2054 | TRIGGER_POS_REAL | R | Actual trigger position (unused). |
| 0x2058 | DEV_VARIANT | R | Device variant (`logic_mode` in PXView). |

### 0x0000 - MODE

This register holds some basic configuration options for the sampler.

| Bitpos (E:I) | Name | Description |
| - | - | - |
| 0 | - | Unknown. Should be set to 1 during initialization and 0 before setting the channel triggers. |
| 1 | STREAMING | Turn on streaming mode. |
| 2 | - | Unknown. Should be set to 1 during initialization and 0 before setting the channel triggers. |
| 3 | FILTER_EN | Enable one sample filter. |
| 4 | - | Unknown. Should toggle this after programming the VREF generator. |
| 32:5 | - | Unknown. Should be set to 0. |


### 0x0004 - PWM_VREF_CMP_PERIOD

This value should have the same meaning as `PWM*_CMP_PERIOD`, however this is unverified.

The driving clock frequency should logically be the same as the other PWM channels, however PXView seems to suggest that the clock frequency for this PWM block is 120MHz instead of the usual 125MHz. This might be a typo but no further research was done on this.

The period is likely to be 10kHz, but due to the above mentioned issue, actual period may differ.

During trigger level selection, PXView writes a fixed value `12000` to it.

### 0x0008 - PWM_VREF_CMP_DUTY

This value should have the same meaning as `PWM*_CMP_DUTY`, however this is unverified.

During trigger level selection, PXView writes the value `Vth * (100.0 / 200.0) / 3.334 * PWM_VREF_CMP_PERIOD` to it. The `100.0 / 200.0` part reflects the frontend design (i.e. it divides the input voltage by 2 using two 100kOhm resistors).

### 0x0010 - CHANNEL_EN

Pretty straightforward: just a bitfield that has the LSB representing channel 0, and the MSB representing channel 31. Note that on variants with less than 32 channels one should probably keep the unsupported channels at 0.

### 0x0014 - CLK_CONF

| Bitpos (E:I) | Name | Description |
| - | - | - |
| 3:0 | SELECT | Sampler clock select (`gpio_mode` in PXView). |
| 3 | EDGE | Sampler clock edge (0: rising edge, 1: falling edge / invert clock) (`clock_edge` in PXView). |
| 32:4 | - | Unknown. Should be set to 0. |

#### 0x0014\[3:0\] - CLK_CONF_SELECT

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

#### 0x0014\[4\] - CLK_CONF_EDGE

Set whether or not to invert the clock of the sampler / set whether the sampler runs on rising edge or falling edge.

### 0x0018 - CLK_DIV

The clock divider value (minus 1) of the sampler.

Actual clock frequency is determined by `F_SAMPLER / (CLK_DIV + 1)`. For example, `CLK_DIV` of 1 with `F_SAMPLER` of 100MHz results in a final sampler clock rate of 50MHz.

In PXView, this is only used alongside with `F_SAMPLER` of 100MHz (i.e. `CLK_CONF_SELECT = CLK_100MHZ`).

### 0x0018 - STOP

Seems to be controlling the sampler activity.

Should write `0xffffffff` to it before configuring the sampler and `0x0` to it before actually start sampling.

### 0x001c - SAMPLE_FRAME_SIZE

Seems to be the frame size in bytes used by the sampler. Normally this should match `XFER_FRAME_SIZE`.

Value must both be page-aligned to 4KiB, and aligned to number of enabled input channels.

PXView determines this value through the following process:

- Determine the maximum size the buffer can be of (MAX).
  - For USB Super Speed, this is 4MiB.
  - For USB HighSpeed and below, this is 4.8Mbit (10ms of data at 480Mbps).
- Determine the typical size of the buffer (TYP).
  - This is 10ms of data at the sample rate, times the number of channels.
- If TYP goes above MAX, use MAX, rounded down to align with both the number of channels and page boundary (`(MAX / num_enabled_channels / 4096) * 4096 * num_enabled_channels` in integer math).
- Otherwise, use TYP.

### 0x0024..=0x0030 - TRIG_\*

Control whether to trigger on low/high/rising edge/falling edge. They are simple bitfields like `CHANNEL_EN`.

Two things to note:

1. To trigger on both edges, set both `TRIG_RISING[n]` and `TRIG_FALLING[n]`.
2. `TRIG_LOW[n]` and `TRIG_HIGH[n]` should probably not be 1 at the same time.

### 0x003c - TRIG_EXT_MODE

This control the Trigger In channel behavior.

| Value | Name | Notes |
| - | - | - |
| 0 | TRIG_NONE | No trigger. |
| 1 | TRIG_RISING | Trigger on rising edge. |
| 2 | TRIG_HIGH | Trigger on high level. |
| 3 | TRIG_FALLING | Trigger on falling edge. |
| 4 | TRIG_LOW | Trigger on low level. |
| 5 | TRIG_EDGE | Trigger on any edge. |

### 0x0040 - PWM0_CONF

| Bitpos (E:I) | Name | Description |
| - | - | - |
| 1:0 | PWM_EN | 1: Enabled, 0: Disabled. |
| 32:2 | - | Unknown. Probably unused. Should be set to 0. |

### 0x0044 - PWM0_CMP_PERIOD

If the period controller counter counts above this value, the counter will be reset, starting a new period. This value should be kept above 0.

The relationship between this and the output period frequency is `F_OUT = F_PWM / (PWM0_CMP_PERIOD + 1)`, where `F_PWM` is normally 125MHz.

### 0x0048 - PWM0_CMP_DUTY

If the duty cycle controller counter counts above this value, the counter will be reset, causing the output to become low. This value should be kept less than `PWM0_CMP_PERIOD`.

The relationship between this and the output duty cycle is `DUTY_OUT = (PWM0_CMP_DUTY + 1) / (PWM0_CMP_PERIOD + 1)` where `DUTY_OUT` is a real number between 0 and 1.

### 0x004c..=0x0054 - PWM1_\*

The layout is the same as the `PWM0_*` registers. Currently this is unfinished and using it may cause undesired effect. PXView always writes 0 to `PWM1_CONF` to disable it when starting a new capture session.

### 0x2008 - XFER_FRAME_SIZE

Seems to be the frame size in bytes used by the USB controller. Normally this should match `SAMPLE_FRAME_SIZE`.

Value must both be page-aligned to 4KiB, and aligned to number of enabled input channels.

### 0x200c - FWRAM_READ_PAGE

| Value | Name | Notes |
| - | - | - |
| 0 | MCU_PROG_FLASH | First 48KiB is bootloader (24KiB actually on chip, PXLogic allocates 32KiB block for the upload), then the main program starts at the 48KiB mark. |
| 1 | FPGA_FLASH | FPGA Flash (unused). |
| 2 | FPGA_RAM | FPGA RAM (unused). |
| 4 | FPGA_CFGRAM | Where the bit file goes. Length varies. |

### 0x2020 - FWRAM_WRITE_PAGE

See `0x200c - FWRAM_READ_PAGE`.

### 0x2058 - DEV_VARIANT

Known values are as follows:

| Value | Name | Device Name | Notes |
| - | - | - | - |
| 0 | VARIANT_32 | PX Logic 32 | 32 ch, 1Gsps |
| 1 | VARIANT_16_PRO | PX Logic 16 Pro | 16 ch, 1Gsps |
| 2 | VARIANT_16_PLUS | PX Logic 16 Plus | 16 ch, 500Msps |
| 3 | VARIANT_16_BASE | PX Logic 16 Base | 16 ch, 250Msps |

## Driver design notes

### Device probing

First, search for USB devices that match the following udev rules:

```udev
# PX Logic 32 running older firmware (WCH 5237)
SUBSYSTEM=="usb", ATTR{idVendor}=="1a86", ATTR{idProduct}=="5237", ATTR{manufacturer}=="PX"
# PX Logic with recent firmware (libusb generic VID:PID)
SUBSYSTEM=="usb", ATTR{idVendor}=="16c0", ATTR{idProduct}=="05dc", ATTR{manufacturer}=="PX"
```

After finding the target devices, read the `DEV_VARIANT` register to determine the exact type.

### PXView predefined sample rate config

| Rate | CLK_CONF_SELECT | CLK_DIV |
| - | - | - |
| 2kHz | CLK_100MHZ | 49999 |
| 5kHz | CLK_100MHZ | 19999 |
| 10kHz | CLK_100MHZ | 9999 |
| 20kHz | CLK_100MHZ | 4999 |
| 40kHz | CLK_100MHZ | 2499 |
| 50kHz | CLK_100MHZ | 1999 |
| 100kHz | CLK_100MHZ | 999 |
| 200kHz | CLK_100MHZ | 499 |
| 400kHz | CLK_100MHZ | 249 |
| 500kHz | CLK_100MHZ | 199 |
| 1MHz | CLK_100MHZ | 99 |
| 2MHz | CLK_100MHZ | 49 |
| 4MHz | CLK_100MHZ | 24 |
| 5MHz | CLK_100MHZ | 19 |
| 10MHz | CLK_100MHZ | 9 |
| 20MHz | CLK_100MHZ | 4 |
| 25MHz | CLK_100MHZ | 3 |
| 50MHz | CLK_100MHZ | 1 |
| 100MHz | CLK_100MHZ | 0 |
| 125MHz | CLK_125MHZ | 0 |
| 200MHz | CLK_200MHZ | 0 |
| 250MHz | CLK_250MHZ | 0 |
| 400MHz | CLK_400MHZ | 0 |
| 500MHz | CLK_500MHZ | 0 |
| 800MHz | CLK_800MHZ | 0 |
| 1GHz | CLK_1GHZ | 0 |

### PXView device configuration process

Configuration upload is done in `pxlogic.c:start_transfers()`.

The entire process is as follows:

- Configure PWM output channels
  - Setting `PWM*_CONF` to 0 before making changes to `PWM*_CMP_PERIOD` and `PWM*_CMP_DUTY`.
  - These two registers should be set according to the rules defined in the register doc.
  - PWM1 is disabled, thus `PWM1_CONF_PWM_EN` should always be set to 0.
- Clear the `BLOCK_START` register.
- Clear the STALL condition on EP2 and EP4.
- Configure the VREF PWM channel (`PWM_VREF_CMP_*`)
- Clear the `CHANNEL_EN` register.
- Toggle bits in the `MODE` register.
  - In a single write request, set bit 0 and bit 2, set `STREAMING` bit if device will be working in streaming mode, and clear all the other bits.
  - In two subsequent write requests, keep all the bits the same as above, and toggle the bit 4 on and off.
  - TODO: The exact behavior of this is unknown. Could be a reset mechanism of some sort. More testing needed.
- Write `0xffffffff` to the `STOP` register.
- Compute and set `SAMPLE_FRAME_SIZE` and `XFER_FRAME_SIZE`.
- Compute and set `NUM_SAMPLES_{HI,LO}`.
- Compute and set `CLK_CONF` and `CLK_DIV` based on *PXView predefined sample rate config*.
- Configure `TRIG_EXT_MODE` and turn on `TRIG_EXT_EN` if requested by the user.
- Configure sampler clock (`CLK_CONF` and `CLK_DIV`).
- Set the `ENABLED_NUM_CH` register.
- Set the `TRIGGER_POS` register based on capture ratio.
- Clear the `BLOCK_START` register again.
- Set the `CHANNEL_EN` register to reflect the channel status.
- Configure internal triggers (`TRIG_{LOW,HIGH,RISING,FALLING}`).
- Clear the `STOP` register.

After configuration, the device will start sending capture data through IN EP2.

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
| 0x0000 | MODE | W | Sampler mode config. |
| 0x0004 | PWM_VREF_CMP_PERIOD | W | Coarse VREF generator period comparator input. (`pwm_max` in PXView). |
| 0x0008 | PWM_VREF_CMP_DUTY | W | Coarse VREF generator duty cycle comparator input. |
| 0x0010 | CHANNEL_MASK | W | Channel mask configuration. |
| 0x0014 | CLK_CONF | W | Sampler clock config. |
| 0x0018 | CLK_DIV | R/W | Sampler clock divider. |
| 0x001c | SAMPLE_FRAME_SIZE | W | Sampler frame size (guess) (`BUFSIZE` in PXView). |
| 0x0020 | - | W | Unknown. Should write `0xffffffff` to it before configuring the sample count and 0 to it after setting `TRIG_*` |
| 0x0024 | TRIG_ZERO | W | Low level trigger bitmask. |
| 0x0028 | TRIG_ONE | W | High level trigger bitmask. |
| 0x002c | TRIG_RISE | W | Rising edge trigger bitmask. |
| 0x0030 | TRIG_FALL | W | Falling edge trigger bitmask. |
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
| 0x2050 | TRIGGER_POS_SET | W | TODO: Unknown (`trigger_pos_set` in PXView). |
| 0x2054 | TRIGGER_POS_REAL | R | Unused. TODO: Unknown (`trigger_pos_real` in PXView). |
| 0x2058 | DEV_VARIANT | R | Device variant (`logic_mode` in PXView). |

### 0x0004 - PWM_VREF_CMP_PERIOD

This value should have the same meaning as `PWM*_CMP_PERIOD`, however this is unverified.

The driving clock frequency should logically be the same as the other PWM channels, however PXView seems to suggest that the clock frequency for this PWM block is 120MHz instead of the usual 125MHz. This might be a typo but no further research was done on this.

The period is likely to be 10kHz, but due to the above mentioned issue, actual period may differ.

During trigger level selection, PXView writes a fixed value `12000` to it.

### 0x0008 - PWM_VREF_CMP_DUTY

This value should have the same meaning as `PWM*_CMP_DUTY`, however this is unverified.

During trigger level selection, PXView writes the value `Vth * (100.0 / 200.0) / 3.334 * PWM_VREF_CMP_PERIOD` to it. The `100.0 / 200.0` part reflects the frontend design (i.e. it divides the input voltage by 2 using two 100kOhm resistors).

### 0x0014 - CLK_CONF

| Bitpos (E:I) | Name | Description |
| - | - | - |
| 3:0 | - | Unknown. Should be set to 0. |
| 6:3 | CLK_CONF_SELECT | Sampler clock select (`gpio_mode` in PXView). |
| 32:6 | - | Unknown. Should be set to 0. |

#### 0x0014\[6:3\] - CLK_CONF_SELECT

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

### 0x0018 - CLK_DIV

The clock divider value (minus 1) of the sampler.

Actual clock frequency is determined by `F_SAMPLER / (CLK_DIV + 1)`. For example, `CLK_DIV` of 1 with `F_SAMPLER` of 100MHz results in a final sampler clock rate of 50MHz.

In PXView, this is only used alongside with `F_SAMPLER` of 100MHz (i.e. `CLK_CONF_SELECT = CLK_100MHZ`).

### 0x001c - SAMPLE_FRAME_SIZE

Seems to be the frame size used by the sampler. Normally this should match `XFER_FRAME_SIZE`.

### 0x003c - TRIG_EXT_MODE

This control the Trigger In channel behavior.

| Value | Name | Notes |
| - | - | - |
| 0 | TRIGGER_NONE | No trigger. |
| 1 | TRIGGER_RISING | Trigger on rising edge. |
| 2 | TRIGGER_HIGH | Trigger on high level. |
| 3 | TRIGGER_FALLING | Trigger on falling edge. |
| 4 | TRIGGER_LOW | Trigger on low level. |
| 5 | TRIGGER_EDGE | Trigger on any edge. |

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

Seems to be the frame size used by the USB controller. Normally this should match `SAMPLE_FRAME_SIZE`.

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
| 0 | VARIANT_32CH | PX Logic 32 | 32 ch, 1Gsps |
| 1 | VARIANT_16CH_1000 | PX Logic 16 Pro | 16 ch, 1Gsps |
| 2 | VARIANT_16CH_500 | PX Logic 16 Plus | 16 ch, 500Msps |
| 3 | VARIANT_16CH_250 | PX Logic 16 Base | 16 ch, 250Msps |

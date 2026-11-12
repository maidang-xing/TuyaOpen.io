---
title: AC792N_Develop_Board Peripheral Mapping
description: "AC792N / WL83 fixed board pins, GPIO numbering and TKL-to-SDK peripheral mappings."
keywords:
  - ac792n-develop-board
  - ac792n
  - ac7926a
  - wl83
  - jieli
  - peripheral mapping
  - pin mapping
  - tuyaopen hardware
  - tkl
  - mipi-dsi
  - camera
---

# AC792 / WL83 software peripheral mapping

How the AC792N_Develop_Board (AC792 / WL83) maps its pins to TKL: first the pins
the board fixes at build time, then each peripheral. The numbers are what the TKL
checks accept; whether a package bonds a pad out is a schematic question.

**Verified on hardware (2026-11-11):** SPI, PWM, IIC (real SHT30 at 0x44) and the
USER0 flash partition, with the shared `examples/peripherals/*`.

## Board pins fixed at build time

Written into the vendor SDK from `boards/JIELI/AC792N_Develop_Board/Kconfig` and
`audio_config.h`; a project does not configure them again.

| Board Pin | Function | Notes |
| --- | --- | --- |
| PD1 / PE11 | Log UART0 TX / RX | 115200 8N1; `TCFG_UART0_TX_IO = TCFG_DEBUG_PORT = IO_PORTD_01`, baud rate from `CONFIG_JIELI_UART_LOG_BAUDRATE` |
| HUSB DPB / DMB | USB device | Mass storage (download/upgrade) + optional CDC-ACM; `usb_role="host"` → `TCFG_HUSB_DEVICE` |
| PB0 / PB1 | MIPI-DSI reset / backlight | ST7701S 480x800; the data lanes are fixed by the MIPI-DSI controller |
| PC7 / PC6 / PC9 | MIC1 ADC0 P / N, bias MICBIASA | On-board differential electret input |
| PC11 / PC12 / PC10 | MIC2 ADC1 P / N, bias MICBIASB | The channel actually captured (`JIELI_AUDIO_MIC_CHANNEL_MAP = AUDIO_ADC_MIC_1`); ADC0 stays closed |
| PE15 | Audio PA mute | LTK5313 EN through an inverting transistor; `PA_MUTE_LEVEL=0` means mute is low and playback drives it high |
| PD0 | Button K1, ADC0 channel 0 | ADKEY ladder, pressed range 0-96 |

`JIELI_AUDIO_ADC_ALL_CHANNEL_OPEN=1` only lets the vendor ADC init every
configured channel; the TKL capture path still maps MIC2 only. Opening both
on-board microphones trips `audio_adc_mic_open ch err` and resets the board.

`tkl_log_output` has two sinks, selected by `CONFIG_ENABLE_JIELI_USB_CDC_LOG`:

- `n` (default) - vendor debug UART (`vprintf`), PD1 at 115200.
- `y` - USB CDC-ACM (`cdc_write_data`, tries USB0 and USB1) on HUSB, shown as a
  COM port on Windows. The port appears about two seconds in, so the early boot
  log is lost.

## GPIO

- One flat numbering shared with TKL: `16 x port index + bit`.
- WL83 ports are not contiguous; the ranges below are what the TKL check accepts.
- Input, push-pull output and 10 kOhm pull-up/pull-down. No open-drain, no
  both-edge or level-triggered interrupt; GPIO interrupts use per-pin EXTI.

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PA0-PA15 | GPIO | TUYA_GPIO_NUM_0 - TUYA_GPIO_NUM_15 | / |
| PB0-PB15 | GPIO | TUYA_GPIO_NUM_16 - TUYA_GPIO_NUM_31 | / |
| PC0-PC13 | GPIO | TUYA_GPIO_NUM_32 - TUYA_GPIO_NUM_45 | / |
| PD0-PD15 | GPIO | TUYA_GPIO_NUM_48 - TUYA_GPIO_NUM_63 | / |
| PE0-PE15 | GPIO | TUYA_GPIO_NUM_64 - TUYA_GPIO_NUM_79 | / |
| PF0-PF5 | GPIO | TUYA_GPIO_NUM_80 - TUYA_GPIO_NUM_85 | / |
| FUSB DPA / DMA | GPIO | TUYA_GPIO_NUM_96 / 97 | / |
| HUSB DPB / DMB | GPIO | TUYA_GPIO_NUM_112 / 113 | / |

PC14 / PC15 and PF6-PF15 do not exist on WL83.

## UART

- TKL UART0 maps to SDK `uart0`; TKL UART1 maps to SDK `uart2`, not SDK UART1.
- 8N1, non-zero baud rate, no flow control.
- WL83 is a per-pin crossbar, so one `tkl_io_pinmux_config()` call per pin, with
  flat pin numbers.
- The log console pins are a board fact in the vendor `board.c`: change the board
  Kconfig for those, not the pinmux API.

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PD1 / PE11 (default) | UART0 TX / RX, log console | TUYA_UART0_TX / RX | TUYA_UART_NUM_0 |
| PB4 / PB5 (default) | UART1 TX / RX, SDK `uart2` (J26 header) | TUYA_UART1_TX / RX | TUYA_UART_NUM_1 |

```c
tkl_io_pinmux_config(49, TUYA_UART0_TX);   /* TX -> PD1  */
tkl_io_pinmux_config(75, TUYA_UART0_RX);   /* RX -> PE11 */
tkl_io_pinmux_config(50, TUYA_UART1_TX);   /* TKL UART1 -> SDK uart2 */
tkl_io_pinmux_config(51, TUYA_UART1_RX);
```

## IIC

- Hardware IIC0 / IIC1 support host mode and 7-bit addresses at 100 kHz, 400 kHz
  and 1 MHz; 3.4 MHz is not supported.
- IIC2 is software (bit-banged) with fixed timing (`sw_iic_delay=80`); the speed
  requested through TKL has no effect, so use low-speed devices only.
- All three ports are registered by the adapter and do not depend on the vendor
  `USE_IIC_TEST_DEMO`.
- Board defaults live in `JIELI_BOARD_IICn_*_PIN`; the platform stages them into
  `tuya_board_periph_config.h`, so no C edit is needed.

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PA4 (default) | IIC0 SCL, hardware | TUYA_IIC0_SCL | TUYA_I2C_NUM_0 |
| PA5 (default) | IIC0 SDA, hardware | TUYA_IIC0_SDA | TUYA_I2C_NUM_0 |
| PA6 (default) | IIC1 SCL, hardware | TUYA_IIC1_SCL | TUYA_I2C_NUM_1 |
| PA7 (default) | IIC1 SDA, hardware | TUYA_IIC1_SDA | TUYA_I2C_NUM_1 |
| PA2 (default) | IIC2 SCL, software | TUYA_IIC2_SCL | TUYA_I2C_NUM_2 |
| PA3 (default) | IIC2 SDA, software | TUYA_IIC2_SDA | TUYA_I2C_NUM_2 |

The hardware ports sit on the crossbar, so a run-time request overrides the board
default (the adapter re-applies it after `dev_open()`):

```c
tkl_io_pinmux_config(4, TUYA_IIC0_SCL);   /* SCL -> PA4 */
tkl_io_pinmux_config(5, TUYA_IIC0_SDA);   /* SDA -> PA5 */
```

The software IIC2 has no crossbar route and can only be moved through the board
config.

## ADC

`ch` is the sequential index returned by the TKL pin-to-function lookup, not an
arbitrary SDK channel number. ADC0, 10 bit, external scan sampling only; the SDK
samples at most 8 channels per request.

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PD0 | ADC0 ch0 (`ADC_IO_CH_PD00`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PD1 | ADC0 ch1 (`ADC_IO_CH_PD01`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PD6 | ADC0 ch2 (`ADC_IO_CH_PD06`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PD7 | ADC0 ch3 (`ADC_IO_CH_PD07`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PD8 | ADC0 ch4 (`ADC_IO_CH_PD08`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PD9 | ADC0 ch5 (`ADC_IO_CH_PD09`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PD10 | ADC0 ch6 (`ADC_IO_CH_PD10`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PE4 | ADC0 ch7 (`ADC_IO_CH_PE04`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PE5 | ADC0 ch8 (`ADC_IO_CH_PE05`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PE6 | ADC0 ch9 (`ADC_IO_CH_PE06`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PE7 | ADC0 ch10 (`ADC_IO_CH_PE07`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PE8 | ADC0 ch11 (`ADC_IO_CH_PE08`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PE9 | ADC0 ch12 (`ADC_IO_CH_PE09`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PC13 | ADC0 ch13 (`ADC_IO_CH_PC13`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| FUSB DPA | ADC0 ch14 (`ADC_IO_CH_DP`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| HUSB DPB | ADC0 ch15 (`ADC_IO_CH_DM`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |

**Changing the pins**: configure the pad as GPADC, then read it with the
sequential index `tkl_io_pin_to_func()` returns.

```c
tkl_io_pinmux_config(48, TUYA_ADC0);   /* PD0 -> ADC channel 0 */
```

## SPI

- Three controllers (SPI0 / SPI1 / SPI2), all registered by the adapter and
  independent: `tkl_spi_init(port, ...)` picks one, and the stock
  `peripherals/spi` example selects it with `CONFIG_EXAMPLE_SPI_PORT`.
- Master only, 8-bit, mode 0. The rate is set at init through
  `IOCTL_SPI_SET_BAUDRATE`.
- The vendor path clocks MSB first; `TUYA_SPI_ORDER_LSB2MSB` is done in software.
- CS is caller-managed: the SPI controller has no chip-select register, so there
  is no hardware CS at all. The driver never asserts one around `dev_write` /
  `dev_read`; its `IOCTL_SPI_SET_CS_STATUS` drives a GPIO for you but the TKL
  layer does not use it, and the TKL API has no CS argument.
- The pins are a true crossbar: peripheral inputs go through per-signal
  selector registers (`FI_SPIx_*`) and outputs through the per-pin function mux,
  so any pin that exists on the package can carry CLK / DO / DI. Limits: a
  signal has exactly one route at a time, only CLK / DO / DI are muxed (no
  d2/d3, so single-lane only), and a pin the SDK cannot give to that function
  fails `gpio_enable_function()` - the adapter reports `OPRT_NOT_SUPPORTED`.
- PA2-PA7 belong to the IIC adapter, PB4/PB5 are the UART1 pads and PB8-PB13
  carry the MCPWM channels, so the defaults below avoid them.

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PC3 (default) | SPI0 CLK | TUYA_SPI0_CLK | TUYA_SPI_NUM_0 |
| PC4 (default) | SPI0 DO / MOSI | TUYA_SPI0_MOSI | TUYA_SPI_NUM_0 |
| PC5 (default) | SPI0 DI / MISO | TUYA_SPI0_MISO | TUYA_SPI_NUM_0 |
| PC0 (default) | SPI1 CLK | TUYA_SPI1_CLK | TUYA_SPI_NUM_1 |
| PC1 (default) | SPI1 DO / MOSI | TUYA_SPI1_MOSI | TUYA_SPI_NUM_1 |
| PC2 (default) | SPI1 DI / MISO | TUYA_SPI1_MISO | TUYA_SPI_NUM_1 |
| PD2 (default) | SPI2 CLK | TUYA_SPI2_CLK | TUYA_SPI_NUM_2 |
| PD3 (default) | SPI2 DO / MOSI | TUYA_SPI2_MOSI | TUYA_SPI_NUM_2 |
| PD4 (default) | SPI2 DI / MISO | TUYA_SPI2_MISO | TUYA_SPI_NUM_2 |

WL83 has a per-pin crossbar, so a run-time request works if it comes before the
`tkl_spi_init()` of that port; the call only records the request and the vendor
driver muxes during `dev_open()`. Moving pins on an open bus needs
`tkl_spi_deinit()` first.

```c
tkl_io_pinmux_config(10, TUYA_SPI0_CLK);    /* PA10 as SPI0 CLK */
tkl_io_pinmux_config(11, TUYA_SPI0_MOSI);
tkl_io_pinmux_config(12, TUYA_SPI0_MISO);
tkl_spi_init(TUYA_SPI_NUM_0, &cfg);
```

**Changing the pins permanently**: edit `JIELI_BOARD_SPIn_CLK_PIN` / `_DO_PIN` /
`_DI_PIN` (one block per controller) in `boards/JIELI/AC792N_Develop_Board/Kconfig`;
the platform stages them into `tuya_board_periph_config.h` and the adapter builds
its platform data from them.

## PWM

- WL83 uses MCPWM. The adapter registers `pwm1` with four channels, each on a
  complementary H/L pin pair remapped through the crossbar.
- PWM4-PWM11 and PWM capture are not supported.
- `duty` is taken against `cycle`; `cycle == 0` is treated as 10000.

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PB8 / PB9 | PWM0, MCPWM CH0 H / L | TUYA_PWM0 | TUYA_PWM_NUM_0 |
| PA8 / PA10 | PWM1, MCPWM CH1 H / L | TUYA_PWM1 | TUYA_PWM_NUM_1 |
| PB10 / PB11 | PWM2, MCPWM CH2 H / L | TUYA_PWM2 | TUYA_PWM_NUM_2 |
| PB12 / PB13 | PWM3, MCPWM CH3 H / L | TUYA_PWM3 | TUYA_PWM_NUM_3 |

A TKL channel is a complementary pair while `TUYA_PWMn` names one pin, so the
semantics do not line up and run-time `tkl_io_pinmux_config()` is not supported
for PWM. Change `JIELI_BOARD_PWMn_H_PIN` / `_L_PIN` in the board Kconfig instead:
the platform stages them into `tuya_board_periph_config.h` and they initialise
`s_pwm1_pd.mcpwm_port[]` (first four entries H, last four L).

## TIMER

| TKL Timer | Behaviour |
| --- | --- |
| TIMER0 - TIMER5 | Software logical numbers sharing the SDK microsecond timer service, not independent hardware timers |
| Concurrency | One active timer at a time; a second concurrent start returns `OPRT_RESOURCE_NOT_READY` |
| Callback | Runs in the SDK timer task (`call_in_irq=0`), not a hardware ISR |

## Flash

The 8 MiB flash is laid out by the vendor flasher (`isd_config.ini`). The
TuyaOpen area stays at `TUYAOPEN_ADR=0x300000` (`TUYAOPEN_LEN=0x0D1000`,
`OPT=1`), so a download or OTA never erases it.

`CONFIG_JIELI_BOARD_DUAL_BANK` is off by default, so a download produces the
single-bank map. It stays off because the A/B layout halves the application
area to 1.42 MiB and the chat bot demo alone is 2.46 MiB of code.

Single-bank map (default):

| Region | Range | Size | Use |
| --- | --- | --- | --- |
| Boot image | 0x000000-0x07F000 | 508 KiB | vendor `uboot.boot` and boot arguments (`FLASH_BIN_SIZE`) |
| Application | 0x07F000-0x300000 | 2.5 MiB | code and resources (`LAVE_SIZE` 0x279000 inside the area) |
| TuyaOpen | 0x300000-0x3D1000 | 836 KiB | the five partitions below |
| BTIF | 0x3D1000-0x3D2000 | 4 KiB | vendor BT interface |
| USER | 0x3D2000-0x3D3000 | 4 KiB | vendor customer reserve |
| Free | 0x3D3000-0x800000 | 4.18 MiB | unused |

Turning `CONFIG_JIELI_BOARD_DUAL_BANK` on stages `CONFIG_DOUBLE_BANK_ENABLE` and
`CONFIG_SUPPORT_DOUBLE_UBOOT` into the vendor `app_config.h`; the flasher then
emits `DB_UPDATE_DATA=YES`, `UBOOT_AREA_SIZE=64K,64K` and lays out the A/B map
below, which is what this board was measured with:

| Region | Range | Size | Use |
| --- | --- | --- | --- |
| Boot image | 0x000000-0x020000 | 128 KiB | two 64 KiB `uboot.boot` copies |
| App bank 0 | 0x020000-0x18C000 | 1.42 MiB | running image, code at 0x020020 |
| App bank 1 | 0x18C000-0x2F8000 | 1.42 MiB | OTA target, code at 0x18C020 |
| VM | 0x2F8000-0x300000 | 32 KiB | vendor VM, auto-placed below TuyaOpen |
| TuyaOpen | 0x300000-0x3D1000 | 836 KiB | unchanged |

Each bank opens with a 32-byte header (image CRC, code length) followed by the
code, which is why `get_current_boot_info()` reports a base address 0x20 past
the bank start; `get_backup_area_info()` on the running image returns
0x18C020 / 0x16BFE0. Code plus resources must fit one bank.

`tkl_flash` exposes five partitions inside the TuyaOpen area; erase and write
outside them return `OPRT_INVALID_PARM`:

| Partition | Range | Size | Use |
| --- | --- | --- | --- |
| KV_KEY | 0x300000-0x301000 | 4 KiB | TuyaOpen KV index |
| KV_DATA | 0x301000-0x311000 | 64 KiB | TuyaOpen KV data |
| UF | 0x311000-0x391000 | 512 KiB | user file area |
| RCD | 0x391000-0x3C1000 | 192 KiB | not used by the tree |
| USER0 | 0x3C1000-0x3D1000 | 64 KiB | general purpose, block 4096 |

Upgrades:

- **`tkl_ota` drives the vendor dual-bank engine** once
  `CONFIG_JIELI_BOARD_DUAL_BANK` is on. `tkl_ota_start_notify()`
  lifts the flash write protection, calls `dual_bank_passive_update_init()` and
  `dual_bank_update_allow_check()`; `tkl_ota_data_process()` feeds
  `dual_bank_update_write()` in at most 4 KiB chunks (the size the engine
  reports); `tkl_ota_end_notify()` runs `dual_bank_update_verify()` and, when
  the read-back CRC matches, `dual_bank_update_burn_boot_info()` and reboots
  into the new bank. `tkl_ota_get_ability()` reports `TUYA_OTA_FULL` with a
  0xF0000 ceiling.
- **The engine needs its `dw_update` task.**
  `dual_bank_passive_update_init()` creates that task by name, so
  `platform/JIELI/tuyaos/entry/jieli_app_entry.c` declares it under
  `CONFIG_DOUBLE_BANK_ENABLE`; without the row OTA start fails with -112.
- **The payload is a vendor OTA package.** It starts with an `update_data`
  wrapper head and a directory of per-file heads (`app_area`, `uboot_core`, ...)
  whose CRC fields the vendor tool computes at build time. A byte copy of the
  running bank is rejected at the head directory, so bench checks cover the
  engine handshake (init, `allow_check`, head parsing) and the write path,
  while the complete bank switch needs an image from the vendor OTA flow.
- The switch is a build-time choice: turning it on changes the flash map, so a
  board already in the field should be re-flashed once with the new layout
  rather than updated into it.

The free area after USER is vendor headroom - nothing allocates it today. It
would be the natural home for a larger UF if that is ever needed, but enlarging
the TuyaOpen area means changing `TUYAOPEN_ADR`/`TUYAOPEN_LEN` and migrating
whatever the KV and UF partitions already hold.

## Other peripherals

| Peripheral | Status |
| --- | --- |
| USB device (HUSB) | Mass storage + optional CDC-ACM; `tos.py flash` downloads over this port |
| MIPI-DSI | ST7701S 480x800, TDD / TDL adapted |
| DVP / MIPI CSI2 camera | Adapted in platform PR #23, not merged yet |
| Watchdog / RTC / touch / QSPI | Not implemented in TKL |

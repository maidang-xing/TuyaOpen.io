---
title: AC79_DevKitBoard Peripheral Mapping
description: "AC79 / WL82 fixed board pins, GPIO numbering and TKL-to-SDK peripheral mappings."
keywords:
  - ac79-devkitboard
  - ac791
  - wl82
  - jieli
  - peripheral mapping
  - pin mapping
  - tuyaopen hardware
  - tkl
---

# AC79 / WL82 software peripheral mapping

How the AC79_DevKitBoard (AC791 / WL82) maps its pins to TKL: first the pins the
board fixes at build time, then each peripheral. The numbers are what the TKL
checks accept; whether a package bonds a pad out is a schematic question.

**Verified on hardware (2026-11-11):** SPI0/SPI1/SPI2, PWM, IIC (real SHT30 at
0x44) and the USER0 flash partition, with the shared `examples/peripherals/*`.

## Board pins fixed at build time

Written into the vendor SDK from `boards/JIELI/AC79_DevKitBoard/Kconfig` and
`audio_config.h`; a project does not configure them again.

| Board Pin | Function | Notes |
| --- | --- | --- |
| PB3 | Log UART1 TX | 115200 8N1, RX unconfigured. `CONFIG_JIELI_UART_LOG_PORT=1`; the platform inserts the UART1 platform data into the staged `board.c` |
| PA5 / PA6 | Service UART0 TX / RX | TAL CLI, `JIELI_BOARD_HAS_SERVICE_UART=y` |
| FUSB DPA / DMA | USB device | Mass storage + optional CDC-ACM; `usb_role="device"` → `TCFG_FUSB_DEVICE` |
| PC9 | Panel DC (ILI9488 320x480, MCU8080) | CS, data and clock belong to the vendor `emi` device; reset defaults to PB0 |
| PB2 | Audio PA mute | `PA_MUTE_LEVEL=0`, release delay 200 ms |
| PB1 | Button K1, ADC0 channel 3 | ADKEY ladder, pressed range 824-1007 |
| — | Microphone `LADC_CH_MIC1_P_N` / DAC channel 0x05 | Vendor defaults, pins not overridden |

`tkl_log_output` has two sinks, selected by `CONFIG_ENABLE_JIELI_USB_CDC_LOG`:

- `n` (default) - vendor debug UART (`vprintf`), PB3 at 115200.
- `y` - USB CDC-ACM (`cdc_write_data`) on FUSB. The COM port appears about two
  seconds in, so the early boot log is lost.

## GPIO

- One flat numbering shared with TKL: `16 x port index + bit`.
- Input, push-pull output and 10 kOhm pull-up/pull-down. No open-drain, no
  both-edge or level-triggered interrupt; at most two GPIOs can be remapped to
  external interrupts.

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PA0-PA15 | GPIO | TUYA_GPIO_NUM_0 - TUYA_GPIO_NUM_15 | / |
| PB0-PB15 | GPIO | TUYA_GPIO_NUM_16 - TUYA_GPIO_NUM_31 | / |
| PC0-PC15 | GPIO | TUYA_GPIO_NUM_32 - TUYA_GPIO_NUM_47 | / |
| PD0-PD15 | GPIO | TUYA_GPIO_NUM_48 - TUYA_GPIO_NUM_63 | / |
| PE0-PE15 | GPIO | TUYA_GPIO_NUM_64 - TUYA_GPIO_NUM_79 | / |
| PF0-PF15 | GPIO | TUYA_GPIO_NUM_80 - TUYA_GPIO_NUM_95 | / |
| PG0-PG15 | GPIO | TUYA_GPIO_NUM_96 - TUYA_GPIO_NUM_111 | / |
| PH0-PH15 | GPIO | TUYA_GPIO_NUM_112 - TUYA_GPIO_NUM_127 | / |

## UART

- TKL UART0 maps to SDK `uart0`; TKL UART1 maps to SDK `uart2`.
- 8N1, non-zero baud rate, no flow control.
- WL82 has no UART IRQ-event ioctl, so the adapter polls the unread-byte count
  every 10 ms and notifies TAL from a task. The `uart2` ISR is registered on
  `IRQ_UART2_IDX`; receive has not been checked on hardware yet.
- UART1 is opt-in (`JIELI_BOARD_UART1_ENABLE`, default `n`): opening SDK `uart2`
  stops the vendor debug console, and PH2 is also the PWM0 output. A board can
  have a second UART or logs, not both.

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PA5 / PA6 (default) | UART0 TX / RX, group 0 | TUYA_UART0_TX / RX | TUYA_UART_NUM_0 |
| PB1 / PB2 | UART0 TX / RX, group 1 | TUYA_UART0_TX / RX | TUYA_UART_NUM_0 |
| PA3 / PA4 | UART0 TX / RX, group 2 | TUYA_UART0_TX / RX | TUYA_UART_NUM_0 |
| PH0 / PH1 | UART0 TX / RX, group 3 | TUYA_UART0_TX / RX | TUYA_UART_NUM_0 |
| PH2 / PH3 (default) | UART1 TX / RX, SDK `uart2` | TUYA_UART1_TX / RX | TUYA_UART_NUM_1 |
| PB3 | Log console TX (vendor, not TKL) | / | / |

**Changing the pins**: WL82 cannot move one pin at a time - TX and RX are fixed
pairs and both calls are needed. A single pin, or a pair from different groups,
returns `OPRT_NOT_SUPPORTED`. UART1 has a single group, so
`JIELI_BOARD_UART1_TX_PIN` / `_RX_PIN` must stay PH2/PH3; the build rejects any
other pair.

```c
tkl_io_pinmux_config(17, TUYA_UART0_TX);   /* group 1 TX -> PB1 */
tkl_io_pinmux_config(18, TUYA_UART0_RX);   /* group 1 RX -> PB2 */
```

## IIC

- IIC0 is software (bit-banged, fixed `sw_iic_delay=50`); the speed requested
  through TKL has no effect, so use low-speed devices only.
- IIC1 is the hardware host, 400 kHz only.
- Board defaults live in `JIELI_BOARD_IIC0_*_PIN` / `JIELI_BOARD_IIC1_*_PIN`;
  the platform stages them into `tuya_board_periph_config.h`.
- The software port has no pinmux route and can only be moved through the board
  config. The hardware port switches by group at run time.

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PA0 (default) | IIC0 SCL, software | TUYA_IIC0_SCL | TUYA_I2C_NUM_0 |
| PA1 (default) | IIC0 SDA, software | TUYA_IIC0_SDA | TUYA_I2C_NUM_0 |
| PA7 (default) | IIC1 SCL, hardware, group 1 | TUYA_IIC1_SCL | TUYA_I2C_NUM_1 |
| PA8 (default) | IIC1 SDA, hardware, group 1 | TUYA_IIC1_SDA | TUYA_I2C_NUM_1 |
| PC1 / PC2 | IIC1 SCL / SDA, group 2 | TUYA_IIC1_SCL / SDA | TUYA_I2C_NUM_1 |
| PH0 / PH1 | IIC1 SCL / SDA, group 3 | TUYA_IIC1_SCL / SDA | TUYA_I2C_NUM_1 |
| USB DPA / DMA | IIC1 SCL / SDA, group 0 | TUYA_IIC1_SCL / SDA | TUYA_I2C_NUM_1 |

The USB group turns the USB0 pads into plain IO, so it cannot be used together
with USB0.

```c
tkl_io_pinmux_config(33, TUYA_IIC1_SCL);   /* group 2 SCL -> PC1 */
tkl_io_pinmux_config(34, TUYA_IIC1_SDA);   /* group 2 SDA -> PC2 */
```

## ADC

`ch` is the sequential index returned by the TKL pin-to-function lookup, not an
arbitrary SDK channel number. ADC0, 10 bit, external scan sampling only; the SDK
samples at most 10 channels per request.

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PA7 | ADC0 ch0 (`AD_CH_PA07`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PA8 | ADC0 ch1 (`AD_CH_PA08`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PA10 | ADC0 ch2 (`AD_CH_PA10`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PB1 | ADC0 ch3 (`AD_CH_PB01`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PB6 | ADC0 ch4 (`AD_CH_PB06`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PB7 | ADC0 ch5 (`AD_CH_PB07`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PC0 | ADC0 ch6 (`AD_CH_PC00`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PC1 | ADC0 ch7 (`AD_CH_PC01`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PC9 | ADC0 ch8 (`AD_CH_PC09`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PC10 | ADC0 ch9 (`AD_CH_PC10`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PH0 | ADC0 ch10 (`AD_CH_PH00`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PH3 | ADC0 ch11 (`AD_CH_PH03`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| USB DMA | ADC0 ch12 (`AD_CH_DM`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| USB DPA | ADC0 ch13 (`AD_CH_DP`) | TUYA_ADC0 | TUYA_ADC_NUM_0 |

**Changing the pins**: configure the pad as GPADC, then read it with the
sequential index `tkl_io_pin_to_func()` returns.

```c
tkl_io_pinmux_config(17, TUYA_ADC0);   /* PB1 -> AD_CH_PB01 (TKL channel 3) */
```

## SPI

- Three controllers, all registered by the adapter and independent:
  `tkl_spi_init(port, ...)` with port 0, 1 or 2 picks one, and the stock
  `peripherals/spi` example selects it with `CONFIG_EXAMPLE_SPI_PORT`.
- Master only, 8-bit, mode 0. Rate is fixed at 1 MHz: WL82 has no baud-rate
  ioctl, so any other rate is rejected rather than silently wrong.
- The vendor path clocks MSB first; `TUYA_SPI_ORDER_LSB2MSB` is done in software.
- CS is caller-managed: the SPI controllers have no chip-select register, and the
  vendor driver does not assert one around `dev_write` / `dev_read` - the
  vendor's own SPI master example and NOR driver toggle it as a plain GPIO.
  `tkl_spi_send` / `recv` / `transfer` keep that contract, and the TKL API has no
  CS argument.
- Pins are a fixed group table in `asm/spi.h`, chosen by the platform data's
  `port` letter, not a per-pin mux: `tkl_io_pinmux_config(pin, TUYA_SPI*)`
  returns `OPRT_NOT_SUPPORTED`.

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PD4 / PD5 / PD1 | SPI0 CLK / DO / DI, group A (SDK table only, the adapter selector rejects it) | TUYA_SPI0_CLK / MOSI / MISO | TUYA_SPI_NUM_0 |
| PC3 / PC4 / PC0 (default) | SPI0 CLK / DO / DI, group B | TUYA_SPI0_CLK / MOSI / MISO | TUYA_SPI_NUM_0 |
| PB6 / PB7 / PB5 | SPI1 CLK / DO / DI, group A | TUYA_SPI1_CLK / MOSI / MISO | TUYA_SPI_NUM_1 |
| PC9 / PC10 / PC8 (default) | SPI1 CLK / DO / DI, group B | TUYA_SPI1_CLK / MOSI / MISO | TUYA_SPI_NUM_1 |
| PH0 / PH1 / PH2 (default) | SPI2 CLK / DO / DI, group A | TUYA_SPI2_CLK / MOSI / MISO | TUYA_SPI_NUM_2 |
| USB DPA / DMA / PG15 | SPI2 CLK / DO / DI, group B (USB pads, not selectable) | TUYA_SPI2_CLK / MOSI / MISO | TUYA_SPI_NUM_2 |
| PA3 / PA4 / PA2 | SPI2 CLK / DO / DI, group C | TUYA_SPI2_CLK / MOSI / MISO | TUYA_SPI_NUM_2 |

The group is selected by the platform data. `jieli_wl82_spi_pinmux_select()`
can switch it before the controller is opened:

```c
SPI2_PLATFORM_DATA_BEGIN(s_spi2_pd)
    .clk    = 1000000,
    .mode   = SPI_STD_MODE,
    .port   = 'A',           /* A = PH0/PH1/PH2, C = PA3/PA4/PA2 */
    .attr   = SPI_SCLK_L_UPH_SMPH | SPI_BIDIR_MODE,
SPI2_PLATFORM_DATA_END()
```

`JIELI_BOARD_SPI1_*_PIN` in the board Kconfig applies to WL83 only.

## PWM

- WL82 has no MCPWM: PWM0 is the TIMER2 output on one pin, with no
  complementary pair. PWM1-PWM11 and PWM capture are not supported.
- `duty` is taken against `cycle`; `cycle == 0` is treated as 10000.
- `tkl_pwm_multichannel_start()` is not available.

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PH2 (default) | PWM0, `PWM_TIMER2_OPCH2` | TUYA_PWM0 | TUYA_PWM_NUM_0 |

The pin comes from `JIELI_BOARD_PWM0_PIN` in the board Kconfig. Any IO can carry
the TIMER2 output, but WL82 pinmux has no PWM route, so
`tkl_io_pinmux_config()` cannot move it at run time.

## TIMER

| TKL Timer | Behaviour |
| --- | --- |
| TIMER0 - TIMER5 | Software logical numbers sharing the SDK microsecond timer service, not independent hardware timers |
| Concurrency | One active timer at a time; a second concurrent start returns `OPRT_RESOURCE_NOT_READY` |
| Callback | Runs in the SDK timer task (`call_in_irq=0`), not a hardware ISR |

## Flash

The 8 MiB flash is laid out by the vendor flasher (`isd_config.ini`); the
TuyaOpen area is declared there as `TUYAOPEN_ADR=0x300000`,
`TUYAOPEN_LEN=0x0D1000`, `OPT=1`, so a download or OTA never erases it.

`CONFIG_JIELI_BOARD_DUAL_BANK` is off by default on this board, so the
single-bank map below is what a download produces today. Turning it on stages
`CONFIG_DOUBLE_BANK_ENABLE` and `CONFIG_SUPPORT_DOUBLE_UBOOT` into the vendor
`app_config.h`; the flasher then emits `DB_UPDATE_DATA=YES`,
`UBOOT_AREA_SIZE=64K,64K`, lays out two 64 KiB uboot copies, two application
banks and a 32 KiB VM just below 0x300000, and TuyaOpen does not move. The AC792N
page carries the measured numbers for that layout; on the AC79 read them from the
flasher's `FLASH INFO` output the first time it is enabled.

Single-bank map (default):

| Region | Range | Size | Use |
| --- | --- | --- | --- |
| Boot image | 0x000000-0x043000 | 268 KiB | vendor `uboot.boot` and boot arguments (`FLASH_BIN_SIZE`) |
| Application | 0x043000-0x300000 | 2.74 MiB | code and resources (`LAVE_SIZE` 0x2B5000 inside the area) |
| TuyaOpen | 0x300000-0x3D1000 | 836 KiB | the five partitions below |
| BTIF | 0x3D1000-0x3D2000 | 4 KiB | vendor BT interface |
| USER | 0x3D2000-0x3D3000 | 4 KiB | vendor customer reserve |
| Free | 0x3D3000-0x800000 | 4.18 MiB | unused |

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
  `CONFIG_JIELI_BOARD_DUAL_BANK` is on (off by default here, see above).
  `tkl_ota_start_notify()` lifts the flash write protection, calls
  `dual_bank_passive_update_init()` and `dual_bank_update_allow_check()`;
  `tkl_ota_data_process()` feeds `dual_bank_update_write()` in at most 4 KiB
  chunks; `tkl_ota_end_notify()` runs `dual_bank_update_verify()` and, when the
  read-back CRC matches, `dual_bank_update_burn_boot_info()` and reboots into
  the new bank. `tkl_ota_get_ability()` reports `TUYA_OTA_FULL` with a 0xF0000
  ceiling.
- **The engine needs its `dw_update` task.** It is declared in
  `platform/JIELI/tuyaos/entry/jieli_app_entry.c` under
  `CONFIG_DOUBLE_BANK_ENABLE`; without the row OTA start fails with -112.
- **The payload is a vendor OTA package,** starting with an `update_data`
  wrapper head plus per-file heads whose CRC fields the vendor tool computes.
  A byte copy of the running bank is rejected at the head directory, so the
  complete bank switch needs an image from the vendor OTA flow.

The free area after USER is vendor headroom - nothing allocates it today. It
would be the natural home for an OTA staging partition or a larger UF if either
is added later, but enlarging the TuyaOpen area means changing
`TUYAOPEN_ADR`/`TUYAOPEN_LEN` and migrating whatever the KV and UF partitions
already hold.

## Other peripherals

| Peripheral | Status |
| --- | --- |
| USB device (FUSB) | Mass storage + optional CDC-ACM; `tos.py flash` downloads over this port |
| MCU8080 + ILI9488 | 320x480, TDD / TDL adapted |
| Watchdog / RTC / touch / QSPI | Not implemented in TKL |

---
id: ac79-devkitboard-peripheral-mapping
title: AC79_DevKitBoard 外设映射说明
description: "AC791 / WL82 开发板的固定管脚、GPIO 编号，以及 TKL 到 SDK 的外设映射。"
keywords:
  - AC79_DevKitBoard
  - AC791
  - WL82
  - JieLi
  - 杰理
  - 外设映射
  - 管脚映射
  - TKL
  - TuyaOpen 硬件
---

本文说明 AC79_DevKitBoard（AC791 / WL82）的管脚分配：先给出板子上已经固定使用的管脚，
再逐个外设给出 TKL 与 SDK 的映射。表中的编号是 TKL 校验能接受的值，不证明芯片封装已
引出（bond-out），也不证明其他开发板已布线。

**2026-11-11 实机验证：** SPI0/SPI1/SPI2、PWM、IIC（含 0x44 的 SHT30 实物）与 USER0
flash 分区均已用 `examples/peripherals/*` 在本板跑通。

## 板级固定管脚

由 `boards/JIELI/AC79_DevKitBoard/Kconfig` 与 `audio_config.h` 写入厂商 SDK，工程侧
无需再次配置。

| Board Pin | 功能 | 说明 |
| --- | --- | --- |
| PB3 | 日志 UART1 TX | 115200 8N1，RX 未配置。`CONFIG_JIELI_UART_LOG_PORT=1`；平台把 UART1 平台数据插入 staged `board.c` |
| PA5 / PA6 | 服务 UART0 TX / RX | TAL CLI，`JIELI_BOARD_HAS_SERVICE_UART=y` |
| FUSB DPA / DMA | USB device | 大容量存储 + 可选 CDC-ACM；`usb_role="device"` → `TCFG_FUSB_DEVICE` |
| PC9 | 屏幕 DC（ILI9488 320x480，MCU8080） | CS、数据与时钟属于厂商 `emi` 设备；复位默认 PB0 |
| PB2 | 功放静音 | `PA_MUTE_LEVEL=0`，释放延时 200 ms |
| PB1 | 按键 K1，ADC0 通道 3 | ADKEY 分压，按下范围 824-1007 |
| — | 麦克风 `LADC_CH_MIC1_P_N` / DAC 通道 0x05 | 厂商默认值，板级未覆盖管脚 |

`tkl_log_output` 有两个输出端，由 `CONFIG_ENABLE_JIELI_USB_CDC_LOG` 选择：

- `n`（默认）- 厂商调试串口（`vprintf`），PB3，115200。
- `y` - FUSB 上的 USB CDC-ACM（`cdc_write_data`）。COM 口约 2 秒后才出现，早期启动
  日志会丢失。

## GPIO

- 与 TKL 共用一套扁平编号：`16 x 端口序号 + 位号`。
- WL82 引出端口 A-H，TKL 校验接受 0-127。
- 支持输入、推挽输出、10 kOhm 上下拉；不支持开漏、双边沿与电平触发中断；最多两个
  GPIO 可映射到外部中断。

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

- TKL UART0 对应 SDK `uart0`；TKL UART1 对应 SDK `uart2`。
- 仅支持 8N1、非零波特率、无流控。
- WL82 没有 UART 中断事件 ioctl，只有发送可用，TAL / CLI 的接收路径不可用；需要
  双向控制台请使用 792（WL83）。
- UART1 默认关闭（`JIELI_BOARD_UART1_ENABLE`，默认 `n`）：打开 SDK `uart2` 会停掉
  厂商调试串口，且 PH2 同时是 PWM0 输出，一块板只能二选一。

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PA5 / PA6（默认） | UART0 TX / RX，组 0 | TUYA_UART0_TX / RX | TUYA_UART_NUM_0 |
| PB1 / PB2 | UART0 TX / RX，组 1 | TUYA_UART0_TX / RX | TUYA_UART_NUM_0 |
| PA3 / PA4 | UART0 TX / RX，组 2 | TUYA_UART0_TX / RX | TUYA_UART_NUM_0 |
| PH0 / PH1 | UART0 TX / RX，组 3 | TUYA_UART0_TX / RX | TUYA_UART_NUM_0 |
| PH2 / PH3（默认） | UART1 TX / RX，SDK `uart2` | TUYA_UART1_TX / RX | TUYA_UART_NUM_1 |
| PB3 | 日志串口 TX（厂商，非 TKL） | / | / |

**修改管脚**：WL82 不能单独移动一个管脚，TX / RX 是固定成对的，必须两次调用一起
下发。只传一个管脚或传入不同组的管脚都会返回 `OPRT_NOT_SUPPORTED`。

```c
tkl_io_pinmux_config(17, TUYA_UART0_TX);   /* 组 1 TX -> PB1 */
tkl_io_pinmux_config(18, TUYA_UART0_RX);   /* 组 1 RX -> PB2 */
```

## IIC

- IIC0 为软件 IIC（GPIO 模拟，时序固定 `sw_iic_delay=50`），TKL 下发的速率不生效，
  只能接低速器件。
- IIC1 为硬件主机，仅 400 kHz。
- 板级默认值在 `JIELI_BOARD_IIC0_*_PIN` / `JIELI_BOARD_IIC1_*_PIN`，平台会写入
  `tuya_board_periph_config.h`。
- 软件端口没有 pinmux 通路，只能改板级配置；硬件端口可在运行时按组切换。

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PA0（默认） | IIC0 SCL，软件 | TUYA_IIC0_SCL | TUYA_I2C_NUM_0 |
| PA1（默认） | IIC0 SDA，软件 | TUYA_IIC0_SDA | TUYA_I2C_NUM_0 |
| PA7（默认） | IIC1 SCL，硬件，组 1 | TUYA_IIC1_SCL | TUYA_I2C_NUM_1 |
| PA8（默认） | IIC1 SDA，硬件，组 1 | TUYA_IIC1_SDA | TUYA_I2C_NUM_1 |
| PC1 / PC2 | IIC1 SCL / SDA，组 2 | TUYA_IIC1_SCL / SDA | TUYA_I2C_NUM_1 |
| PH0 / PH1 | IIC1 SCL / SDA，组 3 | TUYA_IIC1_SCL / SDA | TUYA_I2C_NUM_1 |
| USB DPA / DMA | IIC1 SCL / SDA，组 0 | TUYA_IIC1_SCL / SDA | TUYA_I2C_NUM_1 |

USB 组会把 USB0 的 pad 变成普通 IO，不能与 USB0 同时使用。

```c
tkl_io_pinmux_config(33, TUYA_IIC1_SCL);   /* 组 2 SCL -> PC1 */
tkl_io_pinmux_config(34, TUYA_IIC1_SDA);   /* 组 2 SDA -> PC2 */
```

## ADC

`ch` 是 TKL 管脚功能查表返回的顺序号，不是任意 SDK 通道号。仅支持 ADC0、10 位、外部
扫描采样；SDK 单次最多采样 10 个通道。

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PA7 | ADC0 ch0（`AD_CH_PA07`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PA8 | ADC0 ch1（`AD_CH_PA08`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PA10 | ADC0 ch2（`AD_CH_PA10`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PB1 | ADC0 ch3（`AD_CH_PB01`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PB6 | ADC0 ch4（`AD_CH_PB06`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PB7 | ADC0 ch5（`AD_CH_PB07`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PC0 | ADC0 ch6（`AD_CH_PC00`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PC1 | ADC0 ch7（`AD_CH_PC01`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PC9 | ADC0 ch8（`AD_CH_PC09`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PC10 | ADC0 ch9（`AD_CH_PC10`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PH0 | ADC0 ch10（`AD_CH_PH00`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PH3 | ADC0 ch11（`AD_CH_PH03`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| USB DMA | ADC0 ch12（`AD_CH_DM`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| USB DPA | ADC0 ch13（`AD_CH_DP`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |

**修改管脚**：先把 pad 配置为 GPADC，再用 `tkl_io_pin_to_func()` 返回的顺序号读取。

```c
tkl_io_pinmux_config(17, TUYA_ADC0);   /* PB1 -> AD_CH_PB01（TKL 通道 3） */
```

## SPI

- 三个控制器全部由 adapter 注册且互相独立：`tkl_spi_init(port, ...)` 的 port 取
  0 / 1 / 2 选择其中之一，示例 `peripherals/spi` 用 `CONFIG_EXAMPLE_SPI_PORT` 选择。
- 仅主机、8 位、模式 0。速率固定 1 MHz：WL82 没有波特率 ioctl，其它速率会被直接
  拒绝，而不是悄悄跑错。
- 厂商通路高位在前，`TUYA_SPI_ORDER_LSB2MSB` 由软件完成。
- CS 由调用方管理：SPI 控制器没有片选寄存器，驱动也不会在 `dev_write` / `dev_read`
  前后自动拉 CS，厂商自己的 SPI 主机示例和 NOR 驱动都用普通 GPIO 自己拉；
  `tkl_spi_send` / `recv` / `transfer` 沿用同样的约定，TKL 接口里也没有 CS 参数。
- 管脚是 `asm/spi.h` 里的固定组表，由平台数据的 `port` 字母选择，不是逐脚 mux：
  `tkl_io_pinmux_config(pin, TUYA_SPI*)` 返回 `OPRT_NOT_SUPPORTED`。

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PD4 / PD5 / PD1 | SPI0 CLK / DO / DI，组 A（仅 SDK 表项，adapter 的选择接口不接受） | TUYA_SPI0_CLK / MOSI / MISO | TUYA_SPI_NUM_0 |
| PC3 / PC4 / PC0（默认） | SPI0 CLK / DO / DI，组 B | TUYA_SPI0_CLK / MOSI / MISO | TUYA_SPI_NUM_0 |
| PB6 / PB7 / PB5 | SPI1 CLK / DO / DI，组 A | TUYA_SPI1_CLK / MOSI / MISO | TUYA_SPI_NUM_1 |
| PC9 / PC10 / PC8（默认） | SPI1 CLK / DO / DI，组 B | TUYA_SPI1_CLK / MOSI / MISO | TUYA_SPI_NUM_1 |
| PH0 / PH1 / PH2（默认） | SPI2 CLK / DO / DI，组 A | TUYA_SPI2_CLK / MOSI / MISO | TUYA_SPI_NUM_2 |
| USB DPA / DMA / PG15 | SPI2 CLK / DO / DI，组 B（USB 脚，不可选） | TUYA_SPI2_CLK / MOSI / MISO | TUYA_SPI_NUM_2 |
| PA3 / PA4 / PA2 | SPI2 CLK / DO / DI，组 C | TUYA_SPI2_CLK / MOSI / MISO | TUYA_SPI_NUM_2 |

组由平台数据选择；`jieli_wl82_spi_pinmux_select()` 可在控制器打开前切换：

```c
SPI2_PLATFORM_DATA_BEGIN(s_spi2_pd)
    .clk    = 1000000,
    .mode   = SPI_STD_MODE,
    .port   = 'A',           /* A = PH0/PH1/PH2，C = PA3/PA4/PA2 */
    .attr   = SPI_SCLK_L_UPH_SMPH | SPI_BIDIR_MODE,
SPI2_PLATFORM_DATA_END()
```

板级 Kconfig 的 `JIELI_BOARD_SPI1_*_PIN` 只对 WL83 生效。

## PWM

- WL82 没有 MCPWM：PWM0 是 TIMER2 的单脚输出，没有互补对。PWM1-PWM11 与 PWM 捕获
  均不支持。
- `duty` 相对 `cycle` 计算，`cycle == 0` 按 10000 处理。
- 不支持 `tkl_pwm_multichannel_start()`。

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PH2（默认） | PWM0，`PWM_TIMER2_OPCH2` | TUYA_PWM0 | TUYA_PWM_NUM_0 |

管脚来自板级 Kconfig 的 `JIELI_BOARD_PWM0_PIN`。任何 IO 都能承载 TIMER2 输出，但
WL82 的 pinmux 没有 PWM 通路，`tkl_io_pinmux_config()` 无法在运行时移动它。

## TIMER

| TKL 定时器 | 行为 |
| --- | --- |
| TIMER0 - TIMER5 | 软件逻辑编号，共用 SDK 微秒定时器服务，不是独立的硬件定时器 |
| 并发 | 同时只有一个活动定时器，第二次启动返回 `OPRT_RESOURCE_NOT_READY` |
| 回调 | 运行在 SDK 定时器任务中（`call_in_irq=0`），不是硬件中断 |

## Flash

整片 8 MiB flash 由厂商烧录工具按 `isd_config.ini` 划分；其中 TuyaOpen 区声明为
`TUYAOPEN_ADR=0x300000`、`TUYAOPEN_LEN=0x0D1000`、`OPT=1`，下载或 OTA 都不会擦除它。

板级 Kconfig 里 `CONFIG_JIELI_BOARD_DUAL_BANK` 默认打开，会往暂存的厂商
`app_config.h` 写入 `CONFIG_DOUBLE_BANK_ENABLE` 和 `CONFIG_SUPPORT_DOUBLE_UBOOT`。
烧录工具随之输出 `DB_UPDATE_DATA=YES`、`UBOOT_AREA_SIZE=64K,64K`，把厂商应用区切成
两个 bank，并把 32 KiB 的 VM 自动放到 0x300000 之前；TuyaOpen 区不动。AC792N 上实测
的结果是两份 64 KiB uboot（0x000000-0x020000）、bank 0 0x020000-0x18C000、
bank 1 0x18C000-0x2F8000、VM 0x2F8000-0x300000；AC79 的边界来自同一套规则，第一次
双备份下载时以烧录工具打印的 `FLASH INFO` 为准。

关掉板级选项时是单备份布局：

| 区域 | 地址范围 | 大小 | 用途 |
| --- | --- | --- | --- |
| Boot 镜像 | 0x000000-0x043000 | 268 KiB | 厂商 `uboot.boot` 与启动参数（`FLASH_BIN_SIZE`） |
| 应用区 | 0x043000-0x300000 | 2.74 MiB | 代码与资源（区内 `LAVE_SIZE` 为 0x2B5000） |
| TuyaOpen | 0x300000-0x3D1000 | 836 KiB | 下面五个分区 |
| BTIF | 0x3D1000-0x3D2000 | 4 KiB | 厂商 BT 接口区 |
| USER | 0x3D2000-0x3D3000 | 4 KiB | 厂商为客户预留区 |
| 空闲 | 0x3D3000-0x800000 | 4.18 MiB | 未使用 |

`tkl_flash` 在 TuyaOpen 区内暴露五个分区，越界擦写返回 `OPRT_INVALID_PARM`：

| 分区 | 地址范围 | 大小 | 用途 |
| --- | --- | --- | --- |
| KV_KEY | 0x300000-0x301000 | 4 KiB | TuyaOpen KV 索引 |
| KV_DATA | 0x301000-0x311000 | 64 KiB | TuyaOpen KV 数据 |
| UF | 0x311000-0x391000 | 512 KiB | 用户文件区 |
| RCD | 0x391000-0x3C1000 | 192 KiB | 仓库未使用 |
| USER0 | 0x3C1000-0x3D1000 | 64 KiB | 通用分区，块大小 4096 |

升级相关：

- **`tkl_ota` 走的是厂商双备份引擎**：`tkl_ota_start_notify()` 解除 flash 写保护，
  调 `dual_bank_passive_update_init()` 和 `dual_bank_update_allow_check()`；
  `tkl_ota_data_process()` 按不超过 4 KiB 的块调 `dual_bank_update_write()`；
  `tkl_ota_end_notify()` 调 `dual_bank_update_verify()`，回读 CRC 一致后调
  `dual_bank_update_burn_boot_info()` 并重启进新 bank。`tkl_ota_get_ability()`
  上报 `TUYA_OTA_FULL`，上限 0xF0000。
- **引擎依赖 `dw_update` 任务**：它在
  `platform/JIELI/tuyaos/entry/jieli_app_entry.c` 的 `CONFIG_DOUBLE_BANK_ENABLE`
  分支里声明；缺了它 OTA 启动会返回 -112。
- **升级载荷是厂商 OTA 包**：以 `update_data` 包裹头和一组文件头开头，CRC 字段由
  厂商工具在打包时计算。直接把运行 bank 的字节拷过去会在解析文件头时被拒，完整切换
  需要厂商 OTA 流程产出的镜像。

USER 之后的空闲区是厂商预留的余量，当前没有任何分区占用；以后若要扩大 UF，这里是最
自然的位置，但扩大 TuyaOpen 区意味着改 `TUYAOPEN_ADR` / `TUYAOPEN_LEN`，并且要迁移
KV、UF 里已有的数据。

## 其他外设

| 外设 | 状态 |
| --- | --- |
| USB device（FUSB） | 大容量存储 + 可选 CDC-ACM；`tos.py flash` 通过该口下载 |
| MCU8080 + ILI9488 | 320x480，TDD / TDL 已适配 |
| 看门狗 / RTC / 触摸 / QSPI | TKL 未实现 |

---
id: ac792n-develop-board-peripheral-mapping
title: AC792N_Develop_Board 外设映射说明
description: "AC792 / WL83 开发板的固定管脚、GPIO 编号，以及 TKL 到 SDK 的外设映射。"
keywords:
  - AC792N_Develop_Board
  - AC792N
  - AC7926A
  - WL83
  - JieLi
  - 杰理
  - 外设映射
  - 管脚映射
  - TKL
  - TuyaOpen 硬件
  - MIPI-DSI
  - 摄像头
---

本文说明 AC792N_Develop_Board（AC792 / WL83）的管脚分配：先给出板子上已经固定使用的
管脚，再逐个外设给出 TKL 与 SDK 的映射。表中的编号是 TKL 校验能接受的值，不证明芯片
封装已引出（bond-out），也不证明其他开发板已布线。

**2026-11-11 实机验证：** SPI、PWM、IIC（含 0x44 的 SHT30 实物）与 USER0 flash 分区
均已用 `examples/peripherals/*` 在本板跑通。

## 板级固定管脚

由 `boards/JIELI/AC792N_Develop_Board/Kconfig` 与 `audio_config.h` 写入厂商 SDK，工程
侧无需再次配置。

| Board Pin | 功能 | 说明 |
| --- | --- | --- |
| PD1 / PE11 | 日志 UART0 TX / RX | 115200 8N1；`TCFG_UART0_TX_IO = TCFG_DEBUG_PORT = IO_PORTD_01`，波特率来自 `CONFIG_JIELI_UART_LOG_BAUDRATE` |
| HUSB DPB / DMB | USB device | 大容量存储（下载/升级）+ 可选 CDC-ACM；`usb_role="host"` → `TCFG_HUSB_DEVICE` |
| PB0 / PB1 | MIPI-DSI 复位 / 背光 | ST7701S 480x800；数据 lane 由 MIPI-DSI 控制器固定 |
| PC7 / PC6 / PC9 | MIC1 ADC0 P / N，偏置 MICBIASA | 板载差分驻极体输入 |
| PC11 / PC12 / PC10 | MIC2 ADC1 P / N，偏置 MICBIASB | 实际采集的通道（`JIELI_AUDIO_MIC_CHANNEL_MAP = AUDIO_ADC_MIC_1`），ADC0 保持关闭 |
| PE15 | 功放静音 | LTK5313 EN 经反相三极管；`PA_MUTE_LEVEL=0` 表示静音电平为低、播放时拉高 |
| PD0 | 按键 K1，ADC0 通道 0 | ADKEY 分压，按下范围 0-96 |

`JIELI_AUDIO_ADC_ALL_CHANNEL_OPEN=1` 只让厂商 ADC 初始化所有已配置通道，TKL 采集路径
仍只映射 MIC2。同时打开板载两路麦克风会触发 `audio_adc_mic_open ch err` 并复位。

`tkl_log_output` 有两个输出端，由 `CONFIG_ENABLE_JIELI_USB_CDC_LOG` 选择：

- `n`（默认）- 厂商调试串口（`vprintf`），PD1，115200。
- `y` - HUSB 上的 USB CDC-ACM（`cdc_write_data`，先试 USB0 再试 USB1），在 Windows 上
  显示为 COM 口。COM 口约 2 秒后才出现，早期启动日志会丢失。

## GPIO

- 与 TKL 共用一套扁平编号：`16 x 端口序号 + 位号`。
- WL83 端口不连续，下表的范围就是 TKL 校验接受的值。
- 支持输入、推挽输出、10 kOhm 上下拉；不支持开漏、双边沿与电平触发中断；GPIO 中断
  使用逐脚 EXTI。

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

WL83 不存在 PC14 / PC15 与 PF6-PF15。

## UART

- TKL UART0 对应 SDK `uart0`；TKL UART1 对应 SDK `uart2`，不是 SDK UART1。
- 仅支持 8N1、非零波特率、无流控。
- WL83 是逐脚 crossbar，每个管脚一次 `tkl_io_pinmux_config()`，使用扁平编号。
- 日志串口管脚是厂商 `board.c` 里的板级事实，改板级 Kconfig，不要用 pinmux 接口。

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PD1 / PE11（默认） | UART0 TX / RX，日志串口 | TUYA_UART0_TX / RX | TUYA_UART_NUM_0 |
| PB4 / PB5（默认） | UART1 TX / RX，SDK `uart2`（J26 排针） | TUYA_UART1_TX / RX | TUYA_UART_NUM_1 |

```c
tkl_io_pinmux_config(49, TUYA_UART0_TX);   /* TX -> PD1  */
tkl_io_pinmux_config(75, TUYA_UART0_RX);   /* RX -> PE11 */
tkl_io_pinmux_config(50, TUYA_UART1_TX);   /* TKL UART1 -> SDK uart2 */
tkl_io_pinmux_config(51, TUYA_UART1_RX);
```

## IIC

- 硬件 IIC0 / IIC1 支持主机模式和 7 位地址，可选 100 kHz、400 kHz、1 MHz；不支持
  3.4 MHz。
- IIC2 为软件 IIC（GPIO 模拟），时序固定（`sw_iic_delay=80`），TKL 下发的速率不生效，
  只能接低速器件。
- 三路都由 adapter 注册，不依赖厂商 `USE_IIC_TEST_DEMO`。
- 板级默认值在 `JIELI_BOARD_IICn_*_PIN`，平台写入 `tuya_board_periph_config.h`，无需
  改 C 代码。

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PA4（默认） | IIC0 SCL，硬件 | TUYA_IIC0_SCL | TUYA_I2C_NUM_0 |
| PA5（默认） | IIC0 SDA，硬件 | TUYA_IIC0_SDA | TUYA_I2C_NUM_0 |
| PA6（默认） | IIC1 SCL，硬件 | TUYA_IIC1_SCL | TUYA_I2C_NUM_1 |
| PA7（默认） | IIC1 SDA，硬件 | TUYA_IIC1_SDA | TUYA_I2C_NUM_1 |
| PA2（默认） | IIC2 SCL，软件 | TUYA_IIC2_SCL | TUYA_I2C_NUM_2 |
| PA3（默认） | IIC2 SDA，软件 | TUYA_IIC2_SDA | TUYA_I2C_NUM_2 |

硬件端口在 crossbar 上，运行时请求会覆盖板级默认值（adapter 在 `dev_open()` 之后重新
下发）：

```c
tkl_io_pinmux_config(4, TUYA_IIC0_SCL);   /* SCL -> PA4 */
tkl_io_pinmux_config(5, TUYA_IIC0_SDA);   /* SDA -> PA5 */
```

软件 IIC2 没有 crossbar 通路，只能改板级配置。

## ADC

`ch` 是 TKL 管脚功能查表返回的顺序号，不是任意 SDK 通道号。仅支持 ADC0、10 位、外部
扫描采样；SDK 单次最多采样 8 个通道。

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PD0 | ADC0 ch0（`ADC_IO_CH_PD00`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PD1 | ADC0 ch1（`ADC_IO_CH_PD01`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PD6 | ADC0 ch2（`ADC_IO_CH_PD06`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PD7 | ADC0 ch3（`ADC_IO_CH_PD07`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PD8 | ADC0 ch4（`ADC_IO_CH_PD08`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PD9 | ADC0 ch5（`ADC_IO_CH_PD09`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PD10 | ADC0 ch6（`ADC_IO_CH_PD10`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PE4 | ADC0 ch7（`ADC_IO_CH_PE04`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PE5 | ADC0 ch8（`ADC_IO_CH_PE05`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PE6 | ADC0 ch9（`ADC_IO_CH_PE06`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PE7 | ADC0 ch10（`ADC_IO_CH_PE07`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PE8 | ADC0 ch11（`ADC_IO_CH_PE08`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PE9 | ADC0 ch12（`ADC_IO_CH_PE09`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| PC13 | ADC0 ch13（`ADC_IO_CH_PC13`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| FUSB DPA | ADC0 ch14（`ADC_IO_CH_DP`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |
| HUSB DPB | ADC0 ch15（`ADC_IO_CH_DM`） | TUYA_ADC0 | TUYA_ADC_NUM_0 |

**修改管脚**：先把 pad 配置为 GPADC，再用 `tkl_io_pin_to_func()` 返回的顺序号读取。

```c
tkl_io_pinmux_config(48, TUYA_ADC0);   /* PD0 -> ADC 通道 0 */
```

## SPI

- 三路控制器（SPI0 / SPI1 / SPI2）全部由 adapter 注册且互相独立：
  `tkl_spi_init(port, ...)` 选择其中之一，示例 `peripherals/spi` 用
  `CONFIG_EXAMPLE_SPI_PORT` 选择。
- 仅主机、8 位、模式 0；速率在初始化时通过 `IOCTL_SPI_SET_BAUDRATE` 设置。
- 厂商通路高位在前，`TUYA_SPI_ORDER_LSB2MSB` 由软件完成。
- CS 由调用方管理：SPI 控制器没有片选寄存器，不存在硬件 CS。驱动也不会在
  `dev_write` / `dev_read` 前后自动拉 CS；它提供的 `IOCTL_SPI_SET_CS_STATUS` 可以代劳
  驱动一个 GPIO，但 TKL 层没有使用，TKL 接口本身也没有 CS 参数。
- 管脚是真正的 crossbar：外设输入走逐信号的 `FI_SPIx_*` 选择寄存器，输出走逐脚功能
  mux，因此封装上存在的任何管脚都能承载 CLK / DO / DI。限制：一个信号同一时间只有一条
  路由；只 mux CLK / DO / DI（没有 d2/d3，只支持单线）；SDK 若不接受该脚的功能，
  `gpio_enable_function()` 会失败，adapter 返回 `OPRT_NOT_SUPPORTED`。
- PA2-PA7 属于 IIC adapter，PB4/PB5 是 UART1 管脚，PB8-PB13 承载 MCPWM，因此下面的
  默认值都避开了这些脚。

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PC3（默认） | SPI0 CLK | TUYA_SPI0_CLK | TUYA_SPI_NUM_0 |
| PC4（默认） | SPI0 DO / MOSI | TUYA_SPI0_MOSI | TUYA_SPI_NUM_0 |
| PC5（默认） | SPI0 DI / MISO | TUYA_SPI0_MISO | TUYA_SPI_NUM_0 |
| PC0（默认） | SPI1 CLK | TUYA_SPI1_CLK | TUYA_SPI_NUM_1 |
| PC1（默认） | SPI1 DO / MOSI | TUYA_SPI1_MOSI | TUYA_SPI_NUM_1 |
| PC2（默认） | SPI1 DI / MISO | TUYA_SPI1_MISO | TUYA_SPI_NUM_1 |
| PD2（默认） | SPI2 CLK | TUYA_SPI2_CLK | TUYA_SPI_NUM_2 |
| PD3（默认） | SPI2 DO / MOSI | TUYA_SPI2_MOSI | TUYA_SPI_NUM_2 |
| PD4（默认） | SPI2 DI / MISO | TUYA_SPI2_MISO | TUYA_SPI_NUM_2 |

WL83 是逐脚 crossbar，只要在该端口第一次 `tkl_spi_init()` 之前调用就能生效：该接口只
记录请求，厂商驱动在 `dev_open()` 时才真正切换管脚。总线已打开后再改管脚需要先
`tkl_spi_deinit()`。

```c
tkl_io_pinmux_config(10, TUYA_SPI0_CLK);    /* PA10 作为 SPI0 CLK */
tkl_io_pinmux_config(11, TUYA_SPI0_MOSI);
tkl_io_pinmux_config(12, TUYA_SPI0_MISO);
tkl_spi_init(TUYA_SPI_NUM_0, &cfg);
```

**长期修改管脚**：改 `boards/JIELI/AC792N_Develop_Board/Kconfig` 里的
`JIELI_BOARD_SPIn_CLK_PIN` / `_DO_PIN` / `_DI_PIN`（每路一组），平台写入
`tuya_board_periph_config.h`，adapter 据此生成平台数据。

## PWM

- WL83 使用 MCPWM。adapter 注册 `pwm1` 的四个通道，每个通道是一对经 crossbar 重映射
  的互补 H / L 管脚。
- 不支持 PWM4-PWM11 与 PWM 捕获。
- `duty` 相对 `cycle` 计算，`cycle == 0` 按 10000 处理。

| Board Pin | Function | Software Pin | Software Port |
| --- | --- | --- | --- |
| PB8 / PB9 | PWM0，MCPWM CH0 H / L | TUYA_PWM0 | TUYA_PWM_NUM_0 |
| PA8 / PA10 | PWM1，MCPWM CH1 H / L | TUYA_PWM1 | TUYA_PWM_NUM_1 |
| PB10 / PB11 | PWM2，MCPWM CH2 H / L | TUYA_PWM2 | TUYA_PWM_NUM_2 |
| PB12 / PB13 | PWM3，MCPWM CH3 H / L | TUYA_PWM3 | TUYA_PWM_NUM_3 |

TKL 一个通道是互补对，而 `TUYA_PWMn` 只能表示一个管脚，语义对不上，因此 PWM 不支持
运行时 `tkl_io_pinmux_config()`。请改板级 Kconfig 的 `JIELI_BOARD_PWMn_H_PIN` /
`_L_PIN`：平台写入 `tuya_board_periph_config.h`，用于初始化
`s_pwm1_pd.mcpwm_port[]`（前四项为 H，后四项为 L）。

## TIMER

| TKL 定时器 | 行为 |
| --- | --- |
| TIMER0 - TIMER5 | 软件逻辑编号，共用 SDK 微秒定时器服务，不是独立的硬件定时器 |
| 并发 | 同时只有一个活动定时器，第二次启动返回 `OPRT_RESOURCE_NOT_READY` |
| 回调 | 运行在 SDK 定时器任务中（`call_in_irq=0`），不是硬件中断 |

## Flash

整片 8 MiB flash 由厂商烧录工具按 `isd_config.ini` 划分；其中 TuyaOpen 区声明为
`TUYAOPEN_ADR=0x300000`、`TUYAOPEN_LEN=0x0D1000`、`OPT=1`，下载或 OTA 都不会擦除它。

板级 Kconfig 里 `CONFIG_JIELI_BOARD_DUAL_BANK` 默认关闭，所以现在下载得到的是下面的
单备份布局。保持关闭的原因是 A/B 布局会把应用区减半到 1.42 MiB，而 chat bot demo 仅
代码就有 2.46 MiB。

单备份布局（默认）：

| 区域 | 地址范围 | 大小 | 用途 |
| --- | --- | --- | --- |
| Boot 镜像 | 0x000000-0x07F000 | 508 KiB | 厂商 `uboot.boot` 与启动参数（`FLASH_BIN_SIZE`） |
| 应用区 | 0x07F000-0x300000 | 2.5 MiB | 代码与资源（区内 `LAVE_SIZE` 为 0x279000） |
| TuyaOpen | 0x300000-0x3D1000 | 836 KiB | 下面五个分区 |
| BTIF | 0x3D1000-0x3D2000 | 4 KiB | 厂商 BT 接口区 |
| USER | 0x3D2000-0x3D3000 | 4 KiB | 厂商为客户预留区 |
| 空闲 | 0x3D3000-0x800000 | 4.18 MiB | 未使用 |

打开 `CONFIG_JIELI_BOARD_DUAL_BANK` 会往暂存的厂商 `app_config.h` 写入
`CONFIG_DOUBLE_BANK_ENABLE` 和 `CONFIG_SUPPORT_DOUBLE_UBOOT`，烧录工具随之输出
`DB_UPDATE_DATA=YES`、`UBOOT_AREA_SIZE=64K,64K`，按下面这块板实测的 A/B 布局排：

| 区域 | 地址范围 | 大小 | 用途 |
| --- | --- | --- | --- |
| Boot 镜像 | 0x000000-0x020000 | 128 KiB | 两份 64 KiB 的 `uboot.boot` |
| App bank 0 | 0x020000-0x18C000 | 1.42 MiB | 当前运行镜像，代码在 0x020020 |
| App bank 1 | 0x18C000-0x2F8000 | 1.42 MiB | OTA 目标区，代码在 0x18C020 |
| VM | 0x2F8000-0x300000 | 32 KiB | 厂商 VM，自动放在 TuyaOpen 前面 |
| TuyaOpen | 0x300000-0x3D1000 | 836 KiB | 不变 |

每个 bank 开头有 32 字节头（镜像 CRC、代码长度），所以 `get_current_boot_info()`
报的代码基址比 bank 起始地址大 0x20；在运行镜像上调用 `get_backup_area_info()`
返回 0x18C020 / 0x16BFE0。代码加资源必须装进一个 bank。

`tkl_flash` 在 TuyaOpen 区内暴露五个分区，越界擦写返回 `OPRT_INVALID_PARM`：

| 分区 | 地址范围 | 大小 | 用途 |
| --- | --- | --- | --- |
| KV_KEY | 0x300000-0x301000 | 4 KiB | TuyaOpen KV 索引 |
| KV_DATA | 0x301000-0x311000 | 64 KiB | TuyaOpen KV 数据 |
| UF | 0x311000-0x391000 | 512 KiB | 用户文件区 |
| RCD | 0x391000-0x3C1000 | 192 KiB | 仓库未使用 |
| USER0 | 0x3C1000-0x3D1000 | 64 KiB | 通用分区，块大小 4096 |

升级相关：

- **`tkl_ota` 走的是厂商双备份引擎**（需要先打开上面的
  `CONFIG_JIELI_BOARD_DUAL_BANK`）：`tkl_ota_start_notify()` 解除 flash 写保护，
  调 `dual_bank_passive_update_init()` 和 `dual_bank_update_allow_check()`；
  `tkl_ota_data_process()` 按引擎上报的缓冲大小（最大 4 KiB）分块调
  `dual_bank_update_write()`；`tkl_ota_end_notify()` 调 `dual_bank_update_verify()`，
  回读 CRC 一致后调 `dual_bank_update_burn_boot_info()` 并重启进新 bank。
  `tkl_ota_get_ability()` 上报 `TUYA_OTA_FULL`，上限 0xF0000。
- **引擎依赖 `dw_update` 任务**：`dual_bank_passive_update_init()` 按名字创建它，
  因此 `platform/JIELI/tuyaos/entry/jieli_app_entry.c` 在 `CONFIG_DOUBLE_BANK_ENABLE`
  下声明了这一行；缺了它 OTA 启动会返回 -112。
- **升级载荷是厂商 OTA 包**：以 `update_data` 包裹头和一组文件头（`app_area`、
  `uboot_core` 等）开头，CRC 字段由厂商工具在打包时计算。直接把运行 bank 的字节
  拷过去会在解析文件头时被拒，所以台架验证覆盖的是引擎握手（init、`allow_check`、
  文件头解析）和写入路径，完整切换需要厂商 OTA 流程产出的镜像。

USER 之后的空闲区是厂商预留的余量，当前没有任何分区占用；以后若要扩大 UF，这里是最
自然的位置，但扩大 TuyaOpen 区意味着改 `TUYAOPEN_ADR` / `TUYAOPEN_LEN`，并且要迁移
KV、UF 里已有的数据。

## 其他外设

| 外设 | 状态 |
| --- | --- |
| USB device（HUSB） | 大容量存储 + 可选 CDC-ACM；`tos.py flash` 通过该口下载 |
| MIPI-DSI | ST7701S 480x800，TDD / TDL 已适配 |
| DVP / MIPI CSI2 摄像头 | 已在 platform PR #23 适配，尚未合入 |
| 看门狗 / RTC / 触摸 / QSPI | TKL 未实现 |

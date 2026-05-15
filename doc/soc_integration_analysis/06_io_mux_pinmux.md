# 06 - IO 复用与引脚管理

## 1. Pinmux 架构

OpenTitan 的 **Pin Multiplexer (pinmux)** 是一个自动生成的模块，负责管理芯片外部引脚与内部模块IO之间的映射。

### 1.1 整体结构

```
        ┌──────────────── SoC 内部 ────────────────┐
        │                                          │
        │  ┌────────┐  ┌────────┐  ┌────────┐    │
        │  │ UART0  │  │  I2C0  │  │  GPIO  │    │
        │  │ tx/rx  │  │ sda/scl│  │ [31:0] │    │
        │  └───┬────┘  └───┬────┘  └───┬────┘    │
        │      │            │            │         │
        │  ┌───▼────────────▼────────────▼─────┐   │
        │  │           PINMUX                   │   │
        │  │  ┌─────────────────────────────┐   │   │
        │  │  │ MIO (Multiplexed IO)        │   │   │
        │  │  │ - 输入选择矩阵              │   │   │
        │  │  │ - 输出选择矩阵              │   │   │
        │  │  │ - 睡眠模式控制              │   │   │
        │  │  │ - 唤醒检测器                │   │   │
        │  │  ├─────────────────────────────┤   │   │
        │  │  │ DIO (Dedicated IO)          │   │   │
        │  │  │ - 直连引脚 (不可复用)        │   │   │
        │  │  │ - 如 USB_DP/DN, SPI_CLK     │   │   │
        │  │  ├─────────────────────────────┤   │   │
        │  │  │ JTAG TAP 隔离/选择          │   │   │
        │  │  └─────────────────────────────┘   │   │
        │  └───────────────┬────────────────────┘   │
        │                  │                         │
        └──────────────────┼─────────────────────────┘
                           │
        ┌──────────────────▼─────────────────────┐
        │             PADRING                     │
        │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐     │
        │  │PAD 0│ │PAD 1│ │PAD 2│ │PAD N│     │
        │  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘     │
        └─────┼────────┼──────┼────────┼─────────┘
              │        │      │        │
           ═══╧════════╧══════╧════════╧═══  芯片引脚
```

### 1.2 MIO vs DIO

| 类型 | 说明 | 示例 |
|------|------|------|
| **MIO** (Multiplexed IO) | 可复用引脚，运行时可编程选择功能 | GPIO, UART, I2C |
| **DIO** (Dedicated IO) | 固定功能引脚，不可复用 | USB_DP/DN, SPI_D0-D3, JTAG |

## 2. IO 引脚声明

### 2.1 在 IP 中声明 IO

```hjson
// uart.hjson
available_input_list: [
  {name: "rx", desc: "Serial receive bit"}
]
available_output_list: [
  {name: "tx", desc: "Serial transmit bit"}
]
```

```hjson
// spi_host.hjson - 带宽度的DIO
available_inout_list: [
  {name: "sd", width: "4", desc: "SPI data bus"},
]
available_output_list: [
  {name: "sck", desc: "SPI clock"},
  {name: "csb", desc: "SPI chip select (active low)"},
]
```

### 2.2 在顶层配置引脚映射

```hjson
// top_earlgrey.hjson (pinmux section)
pinmux: {
  // MIO引脚列表 (可复用)
  ios: [
    // MIO Peripheral Input
    {name: "uart0_rx",  type: "input",  width: 1, connection: "muxed"},
    {name: "uart1_rx",  type: "input",  width: 1, connection: "muxed"},
    {name: "i2c0_sda",  type: "inout",  width: 1, connection: "muxed"},
    {name: "gpio",      type: "inout",  width: 32, connection: "muxed"},
    // ...

    // DIO引脚 (固定连接)
    {name: "spi_host0_sd", type: "inout", width: 4, connection: "direct"},
    {name: "spi_host0_sck", type: "output", width: 1, connection: "direct"},
    {name: "usb_dp",     type: "inout",  width: 1, connection: "direct"},
    {name: "usb_dn",     type: "inout",  width: 1, connection: "direct"},
  ]

  // 物理PAD列表
  pads: [
    {name: "IOA0",  type: "BidirStd",  bank: "IOA"},
    {name: "IOA1",  type: "BidirStd",  bank: "IOA"},
    // ...
    {name: "IOB0",  type: "BidirStd",  bank: "IOB"},
    // ...
    {name: "IOC0",  type: "BidirOd",   bank: "IOC"},  // Open-Drain
    // ...
    {name: "IOR0",  type: "BidirStd",  bank: "IOR"},
    // ...
  ]
}
```

## 3. Pinmux 自动生成

### 3.1 生成流程

```
top_earlgrey.hjson (pinmux section + 所有IP的IO声明)
        │
        ▼
topgen/merge.py: amend_pinmux_io()
  - 收集所有IP的available_input/output/inout_list
  - 根据connection类型分为MIO/DIO
  - 计算MIO pad数量
        │
        ▼
ipgen: pinmux模板实例化
  - 生成MIO输入选择矩阵寄存器
  - 生成MIO输出选择矩阵寄存器
  - 生成DIO直连控制寄存器
  - 生成睡眠模式控制寄存器
  - 生成唤醒检测器
        │
        ▼
hw/top_earlgrey/ip_autogen/pinmux/ (输出)
```

### 3.2 生成的寄存器

| 寄存器组 | 功能 | 数量 |
|----------|------|------|
| `MIO_PERIPH_INSEL_*` | MIO输入选择(哪个pad连到哪个外设输入) | N_MIO_IN |
| `MIO_OUTSEL_*` | MIO输出选择(哪个外设输出连到哪个pad) | N_MIO_PAD |
| `MIO_PAD_ATTR_*` | MIO pad属性(上拉/下拉/驱动强度) | N_MIO_PAD |
| `DIO_PAD_ATTR_*` | DIO pad属性 | N_DIO_PAD |
| `MIO_PAD_SLEEP_*` | 睡眠模式pad状态 | N_MIO_PAD |
| `WKUP_DETECTOR_*` | 唤醒模式检测器配置 | N_WKUP_DET |

### 3.3 输入选择矩阵

```
MIO_PERIPH_INSEL[i] 寄存器值:
  0 = 常量0
  1 = 常量1
  2 = MIO PAD 0
  3 = MIO PAD 1
  ...
  N+1 = MIO PAD N-1
```

### 3.4 输出选择矩阵

```
MIO_OUTSEL[j] 寄存器值:
  0 = 常量0 (引脚驱动低)
  1 = 常量1 (引脚驱动高)
  2 = 高阻态
  3 = 外设输出 0 (如uart0_tx)
  4 = 外设输出 1 (如uart1_tx)
  ...
```

## 4. PAD 属性控制

### 4.1 支持的 PAD 类型

```hjson
// padring定义
pads: [
  {name: "IOA0", type: "BidirStd"},     // 标准双向
  {name: "IOA1", type: "BidirOd"},      // 开漏双向
  {name: "IOA2", type: "InputStd"},     // 仅输入
  {name: "IOA3", type: "AnalogIn0"},    // 模拟输入
]
```

### 4.2 可编程 PAD 属性

每个 PAD 通过寄存器可配置：
- **上拉/下拉** (pull_en, pull_select)
- **驱动强度** (drive_strength)
- **施密特触发** (schmitt_en)
- **开漏模式** (od_en)
- **输入反相** (invert)
- **虚拟开漏** (virtual_od_en)

## 5. 唤醒检测器

### 5.1 唤醒源声明

```hjson
// pinmux.hjson
wakeup_list: [
  {name: "pin_wkup_req", desc: "pin wake request"},
  {name: "usb_wkup_req", desc: "usb wake request"},
]
```

### 5.2 唤醒检测模式

```
WKUP_DETECTOR_CFG[i]:
  mode:
    0 = Posedge  (上升沿)
    1 = Negedge  (下降沿)
    2 = Edge     (任意沿)
    3 = TimedHigh (持续高电平)
    4 = TimedLow  (持续低电平)
  filter: 是否启用毛刺滤波
  miodio: 选择MIO还是DIO作为检测源
```

## 6. JTAG TAP 复用

Pinmux 负责 JTAG 引脚的复用控制：

```
Life Cycle 状态 → JTAG TAP 选择:
  - TEST/RMA: 连接 LC TAP
  - DEV/PROD: 连接 RV_DM TAP (如果使能)
  - 其他: 断开
```

确保在不同生命周期状态下，调试接口的访问受控。

## 7. Padring 生成

### 7.1 Padring 配置

```
hw/top_earlgrey/
├── padring.core           # FuseSoC core文件
├── physical_pads.core     # 物理PAD库引用
└── rtl/
    └── padring.sv         # Padring顶层(含PAD实例化)
```

### 7.2 PAD 实例化模式

```systemverilog
// 自动生成的padring.sv (简化)
module padring (
  // MIO PADs
  inout wire [N_MIO_PADS-1:0] mio_pad,
  // DIO PADs
  inout wire [N_DIO_PADS-1:0] dio_pad,
  // ...
);

  // MIO PAD实例化
  prim_pad_wrapper #(.PadType(BidirStd)) u_mio_pad_0 (
    .inout_io(mio_pad[0]),
    .in_o(mio_in[0]),
    .out_i(mio_out[0]),
    .oe_i(mio_oe[0]),
    .attr_i(mio_attr[0])
  );
  // ...
endmodule
```

## 8. 与其他项目的对比

| 特性 | OpenTitan Pinmux | 典型SoC Pinmux |
|------|-----------------|----------------|
| 配置方式 | 运行时寄存器可编程 | 通常固定/仅启动时配置 |
| MIO数量 | ~47 MIO + ~16 DIO | 视芯片而定 |
| 唤醒支持 | 内置唤醒检测器 | 通常单独模块 |
| 安全特性 | 基于LC的TAP隔离 | 较少 |
| 睡眠模式 | 每pin独立睡眠状态 | 通常仅输出保持 |

## 9. 集成到自定义项目的要点

1. **确定MIO/DIO划分**：高速/模拟接口设为DIO，通用接口设为MIO
2. **PAD库适配**：根据目标工艺调整PAD类型
3. **唤醒策略**：根据低功耗需求配置唤醒检测器
4. **安全隔离**：设计基于生命周期的IO访问控制
5. **引脚规划**：预留足够MIO支持功能复用灵活性

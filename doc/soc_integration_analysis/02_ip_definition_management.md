# 02 - IP 定义与管理

## 1. IP 定义格式 (Hjson)

每个 IP 通过一个 `.hjson` 文件完整描述其接口、参数和寄存器。

### 1.1 文件位置

```
hw/ip/{ip_name}/data/{ip_name}.hjson      # 标准IP定义
hw/ip_templates/{ip_name}/data/*.hjson     # 模板化IP
hw/top_*/ip_autogen/{ip_name}/data/*.hjson # 自动生成的IP
```

### 1.2 Hjson 核心字段

以 UART 为例（`hw/ip/uart/data/uart.hjson`）：

```hjson
{
  name:       "uart"
  human_name: "UART"
  cip_id:     "30"      // 唯一IP标识符

  // 时钟复位声明
  clocking: [
    {clock: "clk_i", reset: "rst_ni", primary: true}
  ]

  // 总线接口
  bus_interfaces: [
    {protocol: "tlul", direction: "device"}
  ]

  // 中断列表
  interrupt_list: [
    {name: "tx_watermark",  type: "status"},
    {name: "rx_watermark",  type: "status"},
    {name: "tx_empty",      type: "status"},
    {name: "rx_overflow",   type: "event"},
    {name: "rx_frame_err",  type: "event"},
    {name: "rx_break_err",  type: "event"},
    {name: "rx_timeout",    type: "event"},
    {name: "rx_parity_err", type: "event"},
    {name: "tx_done",       type: "event"},
  ]

  // 告警列表
  alert_list: [
    {name: "fatal_fault", desc: "..."}
  ]

  // 外部IO引脚
  available_input_list: [
    {name: "rx", desc: "Serial receive bit"}
  ]
  available_output_list: [
    {name: "tx", desc: "Serial transmit bit"}
  ]

  // 可配置参数
  param_list: [
    {name: "RxFifoDepth", default: "64", type: "int"},
    {name: "TxFifoDepth", default: "32", type: "int"},
  ]

  // 寄存器定义
  regwidth: "32"
  registers: [...]
}
```

### 1.3 关键字段说明

| 字段 | 用途 | 被谁消费 |
|------|------|----------|
| `name` | 模块类型标识 | topgen 匹配实例到IP类型 |
| `clocking` | 时钟/复位端口声明 | topgen 连接时钟树 |
| `bus_interfaces` | 总线协议/方向 | tlgen 生成crossbar连接 |
| `interrupt_list` | 中断源声明 | topgen 路由到PLIC |
| `alert_list` | 告警源声明 | topgen 路由到Alert Handler |
| `available_input/output_list` | IO引脚 | topgen 分配给pinmux |
| `param_list` | 可配置参数 | ipgen/topgen 参数化例化 |
| `inter_signal_list` | 模块间信号 | topgen 生成跨模块连接 |
| `registers` | 寄存器定义 | reggen 生成RTL/RAL/头文件 |

## 2. 模块间信号 (Inter-Module Signals)

```hjson
inter_signal_list: [
  // 单向信号 (uni)
  {struct: "lc_tx", type: "uni", act: "rcv", name: "lc_escalate_en",
   package: "lc_ctrl_pkg", width: "1"},

  // 请求-响应 (req_rsp)
  {struct: "otp_keymgr_key", type: "uni", act: "rcv", name: "otp_key",
   package: "otp_ctrl_pkg"},
]
```

**信号类型**：
- `uni`：单向信号 (req → rcv)
- `req_rsp`：请求-响应握手 (req ↔ rsp)
- `io`：简单IO连接

**连接模式**：
- `1-to-1`：点对点
- `1-to-N`：一对多分发
- `broadcast`：广播

## 3. IP 模板化系统 (ipgen)

### 3.1 模板目录结构

```
hw/ip_templates/{ip_name}/
├── data/
│   └── {ip_name}.hjson.tpl    # Mako模板化的hjson
├── rtl/
│   └── {ip_name}.sv.tpl       # Mako模板化的RTL
└── {ip_name}.core.tpl          # FuseSoC core文件模板
```

### 3.2 参数传递

在 `top_earlgrey.hjson` 中通过 `param_decl` 传递参数：

```hjson
{
  name: "gpio",
  type: "gpio",
  template_type: "gpio",
  attr: "ipgen",
  param_decl: {
    GpioAsHwStrapsEn: "0",
    GpioAsyncOn: "1"
  }
}
```

### 3.3 ipconfig 文件

每个自动生成的IP有对应的 `.ipconfig.hjson`：

```hjson
// top_earlgrey_clkmgr.ipconfig.hjson
{
  instance_name: "clkmgr_aon",
  param_values: {
    topname: "earlgrey",
    NumGroups: "7",
    // ...
  }
}
```

## 4. IP 实例化规则

### 4.1 在 top_earlgrey.hjson 中声明实例

```hjson
module: [
  {
    name: "uart0",                    // 实例名(唯一)
    type: "uart",                     // IP类型(对应hw/ip/uart)
    clock_srcs: {clk_i: "io_div4"},   // 时钟端口映射
    clock_group: "peri",              // 时钟组(决定门控策略)
    reset_connections: {rst_ni: "lc_io_div4"},  // 复位端口映射
    base_addr: {hart: "0x40000000"}, // 基地址
  },
]
```

### 4.2 多实例支持

同一IP类型可实例化多次：
```hjson
{name: "uart0", type: "uart", base_addr: {hart: "0x40000000"}},
{name: "uart1", type: "uart", base_addr: {hart: "0x40010000"}},
{name: "uart2", type: "uart", base_addr: {hart: "0x40020000"}},
{name: "uart3", type: "uart", base_addr: {hart: "0x40030000"}},
```

### 4.3 独立复位域

不同实例可分配独立复位域：
```hjson
{name: "i2c0", reset_connections: {rst_ni: "i2c0"}},
{name: "i2c1", reset_connections: {rst_ni: "i2c1"}},
{name: "i2c2", reset_connections: {rst_ni: "i2c2"}},
```

## 5. IP 版本管理

```hjson
revisions: [
  {
    version: "2.0.0",
    life_stage: "L1",
    design_stage: "D3",
    verification_stage: "V2S",
    dif_stage: "S2",
  }
]
```

## 6. 安全对抗措施声明

```hjson
countermeasures: [
  {name: "BUS.INTEGRITY",        desc: "End-to-end bus integrity scheme."},
  {name: "INTERSIG.MUBI",        desc: "Multibit encoded inter-module signals."},
  {name: "RX.CONSISTCHK",        desc: "Consistency checks on RX data."},
]
```

## 7. IP 查找机制

topgen 通过以下路径查找 IP：
1. `hw/ip/{type}/data/{type}.hjson` — 标准IP
2. `hw/ip_templates/{template_type}/` — 模板IP
3. `hw/top_{topname}/ip/{name}/` — 顶层特有IP
4. `hw/top_{topname}/ip_autogen/{name}/` — 自动生成IP

## 8. 与 IP-XACT 的对比

| 维度 | OpenTitan Hjson | IP-XACT (IEEE 1685) |
|------|----------------|---------------------|
| 格式 | Hjson (人类友好JSON) | XML |
| 标准化 | 项目私有 | IEEE工业标准 |
| EDA兼容 | 需自定义工具 | 商用EDA原生支持 |
| 表达力 | 高(自定义扩展灵活) | 高(但XML冗长) |
| 学习曲线 | 中(需学习OpenTitan约定) | 高(XML Schema复杂) |
| 寄存器描述 | 内置(reggen) | 通过Spirit扩展 |
| 互连描述 | 内置(xbar定义) | 通过Design/Generator |
| 验证生成 | 内置(gen_dv) | 需额外工具 |
| 版本控制友好 | 是(纯文本简洁) | 差(XML diff困难) |

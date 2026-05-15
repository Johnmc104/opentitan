# 03 - 总线互联与地址空间

## 1. 总线协议：TileLink-UL

OpenTitan 使用 **TileLink Uncached Lightweight (TL-UL)** 作为片上总线协议：

- 基于 SiFive TileLink 规范
- 仅支持读/写操作（无缓存一致性）
- 带端到端完整性校验 (ECC)
- 适合安全关键嵌入式系统

### 1.1 TL-UL 信号结构

```systemverilog
// 请求通道 (Host → Device)
typedef struct packed {
  logic                         a_valid;
  tl_a_op_e                     a_opcode;   // Get/PutFullData/PutPartialData
  logic [top_pkg::TL_SZW-1:0]  a_size;
  logic [top_pkg::TL_AIW-1:0]  a_source;   // 请求ID
  logic [top_pkg::TL_AW-1:0]   a_address;
  logic [top_pkg::TL_DBW-1:0]  a_mask;
  logic [top_pkg::TL_DW-1:0]   a_data;
  tl_a_user_t                   a_user;     // 完整性/安全标签
  logic                         d_ready;
} tl_h2d_t;

// 响应通道 (Device → Host)
typedef struct packed {
  logic                         d_valid;
  tl_d_op_e                     d_opcode;   // AccessAck/AccessAckData
  logic [top_pkg::TL_SZW-1:0]  d_size;
  logic [top_pkg::TL_AIW-1:0]  d_source;
  logic [top_pkg::TL_DW-1:0]   d_data;
  logic                         d_error;
  tl_d_user_t                   d_user;
  logic                         a_ready;
} tl_d2h_t;
```

## 2. Crossbar 架构

OpenTitan 采用**两级 Crossbar**：

```
                    ┌─────────────────────────────────────────────┐
                    │           xbar_main (主总线矩阵)             │
                    │                                             │
   rv_core_ibex ───►│   ROM_CTRL    SRAM     FLASH    PERI_XBAR  │
   (corei/cored)   │   RV_DM       AES      HMAC    KMAC        │
                    │   OTBN        KEYMGR   CSRNG   ...         │
   rv_dm ──────────►│                                             │
                    └──────────────────────┬──────────────────────┘
                                           │
                    ┌──────────────────────▼──────────────────────┐
                    │           xbar_peri (外设总线矩阵)            │
                    │                                             │
                    │   UART0-3    I2C0-2    GPIO     SPI_DEVICE  │
                    │   RV_TIMER   PWM       PINMUX  AON_TIMER   │
                    │   SENSOR_CTRL  AST     USBDEV  ...         │
                    └─────────────────────────────────────────────┘
```

## 3. Crossbar 配置文件格式

### 3.1 xbar_main.hjson 结构

```hjson
{
  name: "main",
  type: "xbar",

  // 时钟/复位
  clock_primary: "clk_main_i",
  other_clock_list: ["clk_fixed_i", "clk_spi_host0_i", "clk_spi_host1_i", "clk_usb_i"],
  reset_primary: "rst_main_ni",
  other_reset_list: ["rst_fixed_ni", "rst_spi_host0_ni", "rst_spi_host1_ni", "rst_usb_ni"],

  // 节点定义
  nodes: [
    // Host 节点
    {name: "rv_core_ibex.corei", type: "host", addr_space: "hart",
     clock: "clk_main_i", reset: "rst_main_ni", pipeline: false},
    {name: "rv_core_ibex.cored", type: "host", addr_space: "hart",
     clock: "clk_main_i", reset: "rst_main_ni", pipeline: false},
    {name: "rv_dm.sba", type: "host", addr_space: "hart",
     clock: "clk_main_i", reset: "rst_main_ni", pipeline_byp: false},

    // Device 节点
    {name: "rom_ctrl.rom", type: "device",
     clock: "clk_main_i", reset: "rst_main_ni",
     req_fifo_pass: true, rsp_fifo_pass: false},
    {name: "sram_ctrl_main.ram", type: "device",
     clock: "clk_main_i", reset: "rst_main_ni",
     req_fifo_pass: true, rsp_fifo_pass: true},
    // ...
  ],

  // 连接关系 (邻接表)
  connections: {
    rv_core_ibex.corei: [
      "rom_ctrl.rom",
      "rv_dm.mem",
      "sram_ctrl_main.ram",
      "flash_ctrl.mem"
    ],
    rv_core_ibex.cored: [
      "rom_ctrl.rom", "rom_ctrl.regs",
      "rv_dm.mem", "rv_dm.regs",
      "sram_ctrl_main.ram", "sram_ctrl_main.regs",
      "flash_ctrl.core", "flash_ctrl.prim", "flash_ctrl.mem",
      "uart0", "uart1", "uart2", "uart3",
      "gpio", "spi_device", "spi_host0", "spi_host1",
      // ...
    ],
  }
}
```

### 3.2 节点属性

| 属性 | 说明 |
|------|------|
| `name` | 节点名(对应模块实例.端口) |
| `type` | host/device |
| `addr_space` | 所属地址空间 |
| `clock`/`reset` | 所在时钟/复位域 |
| `pipeline` | 是否插入流水线寄存器 |
| `req_fifo_pass` | 请求FIFO是否为透传(组合逻辑) |
| `rsp_fifo_pass` | 响应FIFO是否为透传 |
| `pipeline_byp` | 流水线旁路控制 |

## 4. Crossbar 生成工具 (tlgen)

### 4.1 工具架构

```
util/tlgen/
├── elaborate.py    # Crossbar展开：计算路由、仲裁逻辑
├── xbar.py         # Xbar数据结构
├── validate.py     # 验证连接规则
├── generate.py     # RTL生成
├── generate_tb.py  # 测试平台生成
├── item.py         # 节点/连接数据模型
└── lib.py          # 辅助工具
```

### 4.2 生成的RTL组件

tlgen 生成以下 SystemVerilog 组件：

```
xbar_{name}.sv                    # 顶层crossbar模块
├── tl_socket_m1.sv              # M:1 复用器 (多host→1device)
├── tl_socket_1n.sv              # 1:N 解复用器 (1host→多device)
├── tlul_fifo_sync.sv            # 同步FIFO (跨时钟域)
├── tlul_fifo_async.sv           # 异步FIFO
└── tlul_err.sv                  # 错误响应生成
```

### 4.3 地址解码

自动生成地址解码逻辑：
```systemverilog
// Auto-generated address decode
always_comb begin
  case (tl_h2d.a_address[31:28])
    4'h0: dev_sel = ROM_CTRL;      // 0x0000_0000
    4'h1: dev_sel = SRAM_CTRL;     // 0x1000_0000
    4'h2: dev_sel = FLASH_CTRL;    // 0x2000_0000
    4'h4: dev_sel = PERI_XBAR;     // 0x4000_0000
    // ...
  endcase
end
```

## 5. 地址空间管理

### 5.1 地址空间定义

```hjson
addr_spaces: [
  {
    name: "hart",
    desc: "The main address space, shared between the CPU and DM",
    subspaces: [
      {
        name: "mmio",
        desc: "MMIO region",
        nodes: ["uart0", "uart1", "gpio", "spi_device", ...]
      }
    ]
  }
]
```

### 5.2 基地址分配

每个模块实例指定基地址：
```hjson
{name: "uart0",  base_addr: {hart: "0x40000000"}},
{name: "uart1",  base_addr: {hart: "0x40010000"}},
{name: "gpio",   base_addr: {hart: "0x40040000"}},
```

### 5.3 地址空间布局 (EarlGrey)

```
0x0000_0000 ─ 0x0000_7FFF  ROM (32KB)
0x1000_0000 ─ 0x1001_FFFF  SRAM (128KB)
0x2000_0000 ─ 0x200F_FFFF  Flash (1MB)
0x4000_0000 ─ 0x4FFF_FFFF  MMIO 外设区
  0x4000_0000  UART0
  0x4001_0000  UART1
  0x4002_0000  UART2
  0x4003_0000  UART3
  0x4004_0000  GPIO
  0x4005_0000  SPI_DEVICE
  ...
0x4100_0000 ─ 0x41FF_FFFF  加密加速器区
  0x4110_0000  AES
  0x4112_0000  HMAC
  0x4112_0000  KMAC
  ...
```

## 6. 总线桥 (Peri Xbar)

主Crossbar与外设Crossbar之间通过 **TL-UL Socket** 桥接：

```hjson
// xbar_main.hjson 中的 peri 节点
{name: "peri", type: "device",
 clock: "clk_fixed_i", reset: "rst_fixed_ni",
 xbar: true,  // 标记为另一个crossbar
 addr_range: [{base_addr: "0x40000000", size_byte: "0x10000000"}]
}
```

`xbar: true` 标记表示该节点是另一个crossbar而非终端设备。

## 7. 跨时钟域处理

Crossbar 节点可位于不同时钟域：

```hjson
// 主时钟域设备
{name: "sram_ctrl_main.ram", clock: "clk_main_i", reset: "rst_main_ni"},

// USB时钟域设备
{name: "usbdev", clock: "clk_usb_i", reset: "rst_usb_ni"},

// SPI时钟域设备
{name: "spi_host0", clock: "clk_spi_host0_i", reset: "rst_spi_host0_ni"},
```

tlgen 自动在不同时钟域之间插入异步FIFO。

## 8. 安全特性

### 8.1 总线完整性

TL-UL 带端到端完整性校验：
- 请求通道 ECC (cmd_intg, data_intg)
- 响应通道 ECC
- 检测传输错误和注入攻击

### 8.2 访问控制 (RACL)

```hjson
// 基于角色的访问控制策略
racl_mappings: {
  uart0: {
    regs: {policy: "default_rw"}
  },
  keymgr: {
    regs: {policy: "secure_only"}
  }
}
```

## 9. 验证要点

tlgen 验证包括：
- 所有 host 可达所有声明的 device
- 地址范围无重叠
- 时钟域标注完整
- 流水线/FIFO 配置合理

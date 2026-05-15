# 07 - 自定义项目实现方案

## 1. 方案概述

基于 OpenTitan 自动化 SoC 集成经验，为自定义项目提出两种可行方案：

| 方案 | 适用场景 | 核心技术 |
|------|----------|----------|
| **方案A：IP-XACT + 自定义生成器** | 需要与商用EDA互操作 | IEEE 1685 + Python/TCL生成 |
| **方案B：OpenTitan式Hjson** | 纯自研/开源生态 | Hjson + Python代码生成 |
| **方案C：混合方案** | 兼顾标准化与灵活性 | IP-XACT存储 + 自定义生成引擎 |

**推荐：方案C（混合方案）** — 以 IP-XACT 作为 IP 描述标准，配合自定义 Python 生成器处理 SoC 集成逻辑。

## 2. 方案C 详细架构

### 2.1 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     IP 描述层 (IP-XACT)                          │
├─────────────────────────────────────────────────────────────────┤
│  vendor/library/ip_name/version/                                 │
│  ├── ip_name.xml            # IP-XACT Component (寄存器/端口)   │
│  ├── ip_name_bus.xml        # Bus Interface定义                 │
│  └── ip_name_params.xml     # 可配置参数                         │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                    SoC 集成描述层 (YAML/Hjson)                    │
├─────────────────────────────────────────────────────────────────┤
│  soc_config/                                                     │
│  ├── soc_top.yaml           # SoC拓扑定义(实例化/连接)          │
│  ├── clock_tree.yaml        # 时钟网络定义                       │
│  ├── reset_tree.yaml        # 复位网络定义                       │
│  ├── address_map.yaml       # 地址空间分配                       │
│  ├── interrupt_map.yaml     # 中断路由配置                       │
│  ├── pinmux_config.yaml     # IO复用配置                         │
│  └── bus_topology.yaml      # 总线矩阵拓扑                       │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                    生成引擎层 (Python)                            │
├─────────────────────────────────────────────────────────────────┤
│  generators/                                                     │
│  ├── ip_parser.py           # IP-XACT解析(使用ipxact库)         │
│  ├── soc_elaborator.py      # SoC配置展开与验证                  │
│  ├── bus_generator.py       # 总线矩阵RTL生成                   │
│  ├── clk_rst_generator.py   # 时钟复位网络生成                   │
│  ├── intr_generator.py      # 中断控制器配置生成                 │
│  ├── pinmux_generator.py    # Pinmux RTL生成                    │
│  ├── top_generator.py       # 顶层RTL组装                       │
│  ├── sw_generator.py        # 软件头文件生成                     │
│  └── doc_generator.py       # 文档生成                          │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                    输出制品层                                      │
├─────────────────────────────────────────────────────────────────┤
│  output/                                                         │
│  ├── rtl/                   # 生成的SystemVerilog                │
│  ├── sw/                    # C/Rust头文件、驱动框架             │
│  ├── dv/                    # UVM验证环境                        │
│  └── doc/                   # 自动文档                           │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 IP-XACT 与生成器的分工

| 职责 | IP-XACT (标准化) | 自定义生成器 (灵活性) |
|------|------------------|---------------------|
| IP端口定义 | ✓ | |
| 寄存器描述 | ✓ | 生成RTL/头文件 |
| 总线接口 | ✓ (Bus Abstraction) | 生成Crossbar |
| 参数化 | ✓ (Parameters) | 模板实例化 |
| SoC拓扑 | | ✓ (YAML配置) |
| 时钟/复位 | 端口声明 | 网络生成 |
| 中断路由 | 端口声明 | PLIC配置/连接 |
| 地址分配 | | ✓ (自动/手动) |
| Pinmux | IO声明 | MUX矩阵生成 |

## 3. IP 描述规范设计

### 3.1 IP-XACT Component 模板

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ipxact:component xmlns:ipxact="http://www.accellera.org/XMLSchema/IPXACT/1685-2014"
                  xmlns:myext="http://myproject.com/extensions">
  <ipxact:vendor>mycompany</ipxact:vendor>
  <ipxact:library>ip_lib</ipxact:library>
  <ipxact:name>uart</ipxact:name>
  <ipxact:version>1.0.0</ipxact:version>

  <!-- 总线接口 -->
  <ipxact:busInterfaces>
    <ipxact:busInterface>
      <ipxact:name>apb_slave</ipxact:name>
      <ipxact:busType vendor="amba" library="bus" name="APB4" version="1.0"/>
      <ipxact:slave>
        <ipxact:memoryMapRef memoryMapRef="uart_regs"/>
      </ipxact:slave>
    </ipxact:busInterface>
  </ipxact:busInterfaces>

  <!-- 端口 -->
  <ipxact:model>
    <ipxact:ports>
      <ipxact:port><ipxact:name>clk_i</ipxact:name>
        <ipxact:wire><ipxact:direction>in</ipxact:direction></ipxact:wire>
      </ipxact:port>
      <ipxact:port><ipxact:name>rst_ni</ipxact:name>
        <ipxact:wire><ipxact:direction>in</ipxact:direction></ipxact:wire>
      </ipxact:port>
      <ipxact:port><ipxact:name>intr_tx_watermark_o</ipxact:name>
        <ipxact:wire><ipxact:direction>out</ipxact:direction></ipxact:wire>
      </ipxact:port>
    </ipxact:ports>
  </ipxact:model>

  <!-- 寄存器 -->
  <ipxact:memoryMaps>
    <ipxact:memoryMap>
      <ipxact:name>uart_regs</ipxact:name>
      <ipxact:addressBlock>
        <ipxact:name>regs</ipxact:name>
        <ipxact:baseAddress>0x0</ipxact:baseAddress>
        <ipxact:range>0x40</ipxact:range>
        <ipxact:width>32</ipxact:width>
        <!-- 寄存器定义... -->
      </ipxact:addressBlock>
    </ipxact:memoryMap>
  </ipxact:memoryMaps>

  <!-- 扩展：中断/时钟/IO声明 -->
  <ipxact:vendorExtensions>
    <myext:interrupts>
      <myext:interrupt name="tx_watermark" type="status"/>
      <myext:interrupt name="rx_overflow" type="event"/>
    </myext:interrupts>
    <myext:clocking>
      <myext:clock name="clk_i" is_primary="true"/>
      <myext:reset name="rst_ni" associated_clock="clk_i"/>
    </myext:clocking>
    <myext:io_pins>
      <myext:input name="rx" muxable="true"/>
      <myext:output name="tx" muxable="true"/>
    </myext:io_pins>
  </ipxact:vendorExtensions>
</ipxact:component>
```

### 3.2 简化的 YAML IP 描述（备选）

对于不需要 IP-XACT 互操作的场景，可用更简洁的 YAML：

```yaml
# ip/uart/uart.yaml
name: uart
version: 1.0.0

clocking:
  - {port: clk_i, reset: rst_ni, primary: true}

bus_interfaces:
  - {protocol: apb4, role: slave, address_block: regs}

interrupts:
  - {name: tx_watermark, type: status}
  - {name: rx_overflow, type: event}
  - {name: rx_frame_err, type: event}

io_pins:
  inputs:
    - {name: rx, muxable: true}
  outputs:
    - {name: tx, muxable: true}

parameters:
  - {name: RxFifoDepth, type: int, default: 64}
  - {name: TxFifoDepth, type: int, default: 32}

registers:
  base_width: 32
  blocks:
    - name: regs
      size: 0x40
      registers: [...]  # 详细寄存器定义
```

## 4. SoC 集成配置设计

### 4.1 SoC 顶层配置 (soc_top.yaml)

```yaml
# soc_config/soc_top.yaml
name: my_soc
datawidth: 32

power_domains:
  - {name: aon, always_on: true}
  - {name: pd0, default: true}

modules:
  # CPU核
  - name: cpu0
    type: riscv_core
    domain: pd0
    clock: main
    reset: sys

  # 外设实例
  - name: uart0
    type: uart
    domain: pd0
    clock: peri
    reset: peri
    base_addr: 0x40000000
    params: {RxFifoDepth: 128}

  - name: uart1
    type: uart
    domain: pd0
    clock: peri
    reset: peri
    base_addr: 0x40010000

  - name: i2c0
    type: i2c
    domain: pd0
    clock: peri
    reset: i2c0  # 独立复位域
    base_addr: 0x40020000

  # 存储
  - name: sram0
    type: sram_ctrl
    domain: pd0
    clock: main
    reset: sys
    base_addr: 0x10000000
    params: {SramDepth: 32768}
```

### 4.2 时钟树配置 (clock_tree.yaml)

```yaml
# soc_config/clock_tree.yaml
sources:
  - {name: main, freq_hz: 200000000, always_on: false}
  - {name: peri, freq_hz: 100000000, always_on: false}
  - {name: aon,  freq_hz: 32768,     always_on: true}

derived:
  - {name: peri_div2, source: peri, divider: 2}
  - {name: peri_div4, source: peri, divider: 4}

groups:
  - name: infra
    gating: none        # 不可门控
    clocks: [main]

  - name: peri
    gating: software    # 软件可门控
    clocks: [peri, peri_div2, peri_div4]

  - name: security
    gating: hint        # hint门控
    clocks: [main]
```

### 4.3 总线拓扑配置 (bus_topology.yaml)

```yaml
# soc_config/bus_topology.yaml
protocol: axi4_lite  # 或 apb4, ahb_lite, tilelink_ul

crossbars:
  - name: main_xbar
    clock: main
    reset: sys
    hosts:
      - {name: cpu0.imem, priority: 0}
      - {name: cpu0.dmem, priority: 0}
      - {name: dma0, priority: 1}
    devices:
      - {name: sram0, addr_range: [0x10000000, 0x10008000]}
      - {name: flash0, addr_range: [0x20000000, 0x20100000]}
      - {name: peri_bridge, addr_range: [0x40000000, 0x50000000], is_bridge: true}

  - name: peri_xbar
    clock: peri
    reset: peri
    hosts:
      - {name: peri_bridge}
    devices:
      - {name: uart0, addr_range: [0x40000000, 0x40001000]}
      - {name: uart1, addr_range: [0x40010000, 0x40011000]}
      - {name: i2c0,  addr_range: [0x40020000, 0x40021000]}
      - {name: gpio0, addr_range: [0x40030000, 0x40031000]}
```

### 4.4 中断映射 (interrupt_map.yaml)

```yaml
# soc_config/interrupt_map.yaml
controller:
  type: plic  # 或 nvic, gic
  target: cpu0
  max_priority: 7

# 自动收集所有模块中断，也可手动覆盖编号
auto_collect: true
manual_overrides:
  # 强制指定某些中断的编号
  uart0.tx_watermark: 1
  uart0.rx_overflow: 2
```

### 4.5 Pinmux 配置 (pinmux_config.yaml)

```yaml
# soc_config/pinmux_config.yaml
mio_pads: 48   # 可复用引脚总数
dio_pads: 12   # 固定功能引脚数

mio_functions:
  - {name: uart0_tx, type: output}
  - {name: uart0_rx, type: input}
  - {name: i2c0_sda, type: inout}
  - {name: i2c0_scl, type: inout}
  - {name: gpio, type: inout, width: 32}

dio_connections:
  - {pad: USB_DP, module: usb0, pin: dp}
  - {pad: USB_DN, module: usb0, pin: dn}
  - {pad: SPI_CLK, module: spi0, pin: sck}

pad_types:
  - {name: BidirStd, features: [pull_up, pull_down, drive_strength]}
  - {name: BidirOd, features: [pull_up, open_drain]}
  - {name: InputOnly, features: [schmitt, pull_up, pull_down]}
```

## 5. 生成器工具链设计

### 5.1 工具链架构

```python
# generators/main.py - 主入口
class SocGenerator:
    def __init__(self, config_dir: str, ip_lib_dir: str, output_dir: str):
        self.config = self.load_configs(config_dir)
        self.ip_lib = self.load_ip_library(ip_lib_dir)

    def generate(self):
        # Stage 1: 解析与验证
        self.parse_ips()          # 解析IP-XACT或YAML
        self.validate_config()    # 验证配置一致性

        # Stage 2: 展开
        self.elaborate_clocks()   # 展开时钟树
        self.elaborate_resets()   # 展开复位树
        self.elaborate_bus()      # 展开总线拓扑
        self.elaborate_interrupts()  # 收集中断
        self.elaborate_pinmux()   # 展开IO复用

        # Stage 3: 生成
        self.gen_clock_module()   # 时钟管理器RTL
        self.gen_reset_module()   # 复位管理器RTL
        self.gen_crossbar()       # Crossbar RTL
        self.gen_interrupt_ctrl() # 中断控制器配置
        self.gen_pinmux()         # Pinmux RTL
        self.gen_top_module()     # 顶层互连RTL
        self.gen_sw_headers()     # 软件头文件
        self.gen_documentation()  # 文档
```

### 5.2 关键技术选型

| 组件 | 推荐技术 | 备选 |
|------|----------|------|
| IP描述格式 | IP-XACT 2014 | YAML + JSON Schema |
| SoC配置格式 | YAML | Hjson, TOML |
| 生成引擎 | Python 3.10+ | |
| 模板引擎 | Mako / Jinja2 | |
| IP-XACT解析 | `ipyxact` 库 | `lxml` + 自定义解析 |
| 验证框架 | Pydantic | Cerberus |
| 总线协议 | AXI4-Lite / APB4 | AHB-Lite, TileLink |
| 中断控制器 | RISC-V PLIC | ARM NVIC, Custom |
| 寄存器生成 | 自研 reggen | SystemRDL, PeakRDL |

### 5.3 IP-XACT Python 解析示例

```python
from ipyxact.ipyxact import Component

def parse_ip_xact(filepath: str) -> dict:
    """解析IP-XACT文件，提取集成所需信息"""
    comp = Component()
    comp.load(filepath)

    ip_info = {
        'name': comp.name,
        'version': comp.version,
        'bus_interfaces': [],
        'interrupts': [],
        'clocks': [],
        'io_pins': [],
        'registers': [],
    }

    # 提取总线接口
    for bi in comp.busInterfaces.busInterface:
        ip_info['bus_interfaces'].append({
            'name': bi.name,
            'bus_type': bi.busType.name,
            'role': 'slave' if bi.slave else 'master',
        })

    # 提取寄存器
    for mm in comp.memoryMaps.memoryMap:
        for ab in mm.addressBlock:
            for reg in ab.register:
                ip_info['registers'].append({
                    'name': reg.name,
                    'offset': reg.addressOffset,
                    'size': reg.size,
                })

    # 从vendorExtensions提取自定义信息
    # (中断、时钟、IO等)

    return ip_info
```

## 6. 实施路线图

### Phase 1：基础框架 (4-6周)

- [ ] 定义 IP 描述规范 (IP-XACT 扩展 或 YAML Schema)
- [ ] 实现 IP 解析器
- [ ] 实现 SoC 配置解析器与验证器
- [ ] 实现基础模板引擎集成

### Phase 2：总线与地址 (4-6周)

- [ ] 实现总线协议适配层 (AXI4-Lite/APB4)
- [ ] 实现 Crossbar RTL 生成器
- [ ] 实现地址解码器生成
- [ ] 实现跨时钟域桥接

### Phase 3：时钟复位 (3-4周)

- [ ] 实现时钟树展开
- [ ] 实现时钟管理器生成
- [ ] 实现复位树展开
- [ ] 实现复位管理器生成

### Phase 4：中断与IO (3-4周)

- [ ] 实现中断收集与PLIC配置
- [ ] 实现 Pinmux 生成器
- [ ] 实现 Padring 生成

### Phase 5：顶层集成 (2-3周)

- [ ] 实现顶层模块组装
- [ ] 实现软件头文件生成
- [ ] 实现文档自动生成

### Phase 6：验证与打磨 (3-4周)

- [ ] 完整流程验证 (一个示例SoC)
- [ ] CI/CD 集成
- [ ] 错误报告优化
- [ ] 用户文档编写

## 7. 关键决策点

### 7.1 总线协议选择

| 协议 | 复杂度 | 性能 | 生态 | 建议场景 |
|------|--------|------|------|----------|
| APB4 | 低 | 低 | 广泛 | 低速外设 |
| AHB-Lite | 中 | 中 | 广泛 | 单主MCU |
| AXI4-Lite | 中 | 中高 | 最广 | 通用SoC |
| AXI4 | 高 | 高 | 最广 | 高性能SoC |
| TileLink-UL | 中 | 中 | RISC-V | RISC-V生态 |

**建议**：主总线用 AXI4-Lite，外设总线用 APB4，通过 AXI-to-APB 桥连接。

### 7.2 IP描述格式选择

| 场景 | 推荐 | 理由 |
|------|------|------|
| 需要EDA工具集成 | IP-XACT | 工业标准，Synopsys/Cadence/Mentor支持 |
| 纯开源/自研 | YAML | 简洁、版本控制友好 |
| 混合环境 | IP-XACT + YAML桥接 | 两者优点兼顾 |

### 7.3 寄存器描述工具

| 工具 | 优点 | 缺点 |
|------|------|------|
| SystemRDL | 工业标准，商用工具支持 | 需要许可 |
| PeakRDL (开源) | 兼容SystemRDL，Python生态 | 社区小 |
| 自研reggen | 完全可控 | 开发成本 |
| IP-XACT内置 | 标准化 | XML冗长 |

**建议**：使用 PeakRDL 或自研轻量 reggen。

## 8. 与 OpenTitan 方案的差异

| 维度 | OpenTitan | 推荐方案 |
|------|-----------|----------|
| IP格式 | Hjson (私有) | IP-XACT (标准) + YAML (集成) |
| 总线 | TileLink-UL | AXI4-Lite + APB4 |
| 时钟管理 | 自动生成clkmgr | 自动生成 + 可选外部IP |
| 中断 | PLIC | PLIC (RISC-V) 或 NVIC (ARM) |
| 模板 | Mako | Jinja2 (社区更大) |
| 构建 | Bazel | Make/CMake + Python |
| EDA集成 | 弱 | IP-XACT原生支持 |

## 9. 风险与应对

| 风险 | 影响 | 应对措施 |
|------|------|----------|
| IP-XACT复杂度高 | 开发周期延长 | 先用YAML原型，后补IP-XACT导出 |
| Crossbar生成质量 | 时序/面积 | 参考开源AXI Interconnect IP |
| 工具链维护 | 长期成本 | 良好的测试覆盖 + 文档 |
| 人员学习曲线 | 初期效率低 | 提供示例SoC + 教程 |

## 10. 推荐开源参考资源

| 资源 | 说明 |
|------|------|
| [OpenTitan](https://github.com/lowRISC/opentitan) | 本分析对象 |
| [PeakRDL](https://github.com/SystemRDL/PeakRDL) | SystemRDL编译器 |
| [AXI Interconnect (pulp-platform)](https://github.com/pulp-platform/axi) | AXI互连IP |
| [LiteX](https://github.com/enjoy-digital/litex) | Python SoC生成框架 |
| [Chipyard](https://github.com/ucb-bar/chipyard) | RISC-V SoC生成框架 |
| [FuseSoC](https://github.com/olofk/fusesoc) | 包管理与构建 |
| [ipyxact](https://github.com/olofk/ipyxact) | IP-XACT Python解析 |

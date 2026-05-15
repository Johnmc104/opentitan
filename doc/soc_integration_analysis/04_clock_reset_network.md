# 04 - 时钟复位网络

## 1. 时钟系统架构

OpenTitan 的时钟管理采用集中式架构，通过 **Clock Manager (clkmgr)** 统一管控。

### 1.1 时钟源层次

```
                    AST (Analog Sensor Top)
                    ├── main_clk  (100 MHz)
                    ├── io_clk    (96 MHz)
                    ├── usb_clk   (48 MHz)
                    └── aon_clk   (200 KHz, Always-On)
                            │
                    ┌───────▼───────┐
                    │   Clock Mgr   │
                    │  (clkmgr_aon) │
                    ├───────────────┤
                    │ 分频:         │
                    │  io_div2 48MHz│
                    │  io_div4 24MHz│
                    ├───────────────┤
                    │ 门控组:        │
                    │  powerup      │
                    │  secure       │
                    │  peri         │
                    │  timers       │
                    │  infra        │
                    └───────┬───────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
         uart0/1/2/3   aes/hmac/kmac   rv_core_ibex
         (io_div4)     (main)          (main)
```

### 1.2 时钟源定义

在 `top_earlgrey.hjson` 中：

```hjson
clocks: {
  hier_paths: {
    top: "clkmgr_aon_clocks.",  // 顶层时钟结构体路径
    ext: "",                     // 外部时钟端口
    lpg: "clkmgr_aon_cg_en.",   // 时钟门控使能路径
  },

  srcs: [
    {name: "main", aon: "no",  freq: "100000000"},
    {name: "io",   aon: "no",  freq: "96000000"},
    {name: "usb",  aon: "no",  freq: "48000000"},
    {name: "aon",  aon: "yes", freq: "200000", ref: true}
  ],

  derived_srcs: [
    {name: "io_div2", aon: "no", div: 2, src: "io", freq: "48000000"},
    {name: "io_div4", aon: "no", div: 4, src: "io", freq: "24000000"}
  ],
}
```

### 1.3 时钟组 (Clock Groups)

```hjson
groups: [
  // 不可门控 - AST自管理
  {name: "ast",     src: "ext", sw_cg: "no"},

  // 不可门控 - 上电必需
  {name: "powerup", src: "top", sw_cg: "no"},

  // 软件可门控 - 外设时钟
  {name: "peri",    src: "top", sw_cg: "yes", unique: "no"},

  // hint门控 - 安全/加密模块
  {name: "secure",  src: "top", sw_cg: "hint"},

  // 不可门控 - 基础设施
  {name: "infra",   src: "top", sw_cg: "no"},

  // 不可门控 - 定时器
  {name: "timers",  src: "top", sw_cg: "no"},
]
```

**门控策略**：
| sw_cg | 含义 | 适用场景 |
|-------|------|----------|
| `no` | 不可门控 | 上电必需、基础设施 |
| `yes` | 软件直接门控 | 通用外设 |
| `hint` | 软件提供建议，硬件决定 | 安全模块(有空闲检测) |

### 1.4 模块时钟分配

```hjson
// top_earlgrey.hjson module section
{
  name: "uart0",
  clock_srcs: {clk_i: "io_div4"},   // 端口clk_i连接io_div4
  clock_group: "peri",               // 属于peri组(可门控)
}

{
  name: "aes",
  clock_srcs: {
    clk_i: "main",                   // 主时钟
    clk_edn_i: "main"               // EDN接口时钟
  },
  clock_group: "trans",              // 安全事务组
}
```

## 2. clkmgr 自动生成

### 2.1 生成流程

```
top_earlgrey.hjson (clocks section)
        │
        ▼
ipgen/clkmgr_gen.py (提取参数)
        │
        ▼
hw/ip_templates/clkmgr/ (Mako模板)
        │
        ▼
hw/top_earlgrey/ip_autogen/clkmgr/ (生成结果)
```

### 2.2 生成的寄存器

clkmgr 自动生成以下可编程寄存器：
- `CLK_ENABLES`: 各可门控时钟的使能位
- `CLK_HINTS`: 各hint时钟的建议位
- `CLK_HINTS_STATUS`: 实际时钟状态
- `MEASURE_CTRL_*`: 时钟频率监测控制
- `RECOV_ERR_CODE`: 可恢复错误状态

## 3. 复位系统架构

### 3.1 复位域层次

```
             POR (Power-On Reset)
               │
    ┌──────────┼──────────┐
    │          │          │
   POR_AON   POR_IO    POR_USB
    │          │          │
    ▼          ▼          ▼
  ┌────────────────────────────┐
  │      Reset Manager         │
  │       (rstmgr_aon)        │
  ├────────────────────────────┤
  │ 系统复位:                   │
  │   rst_sys_*               │
  │ LC复位:                    │
  │   rst_lc_*                │
  │ 外设独立复位:               │
  │   rst_i2c0, rst_i2c1...   │
  │   rst_spi_device...       │
  └────────────────────────────┘
```

### 3.2 复位节点定义

```hjson
resets: {
  hier_paths: {
    top: "rstmgr_aon_resets.",  // 顶层复位结构体路径
    ext: "",                     // 外部复位端口
  },

  nodes: [
    // 外部不可控复位
    {name: "por_aon", gen: false, type: "ext", parent: ""},

    // 顶层生成复位
    {name: "lc_src", gen: true, type: "top",
     parent: "por_aon", clock: "aon"},
    {name: "sys_src", gen: true, type: "top",
     parent: "lc_src", clock: "aon"},

    // 内部(IP生成)复位
    {name: "por", gen: true, type: "int", parent: "por_aon", clock: "main"},
    {name: "por_io", gen: true, type: "int", parent: "por_aon", clock: "io"},
    {name: "por_io_div2", gen: true, type: "int", parent: "por_aon", clock: "io_div2"},
    {name: "por_io_div4", gen: true, type: "int", parent: "por_aon", clock: "io_div4"},
    {name: "por_usb", gen: true, type: "int", parent: "por_aon", clock: "usb"},
    {name: "lc", gen: true, type: "int", parent: "lc_src", clock: "main"},
    {name: "lc_io_div4", gen: true, type: "int", parent: "lc_src", clock: "io_div4"},
    {name: "sys", gen: true, type: "int", parent: "sys_src", clock: "main"},
    {name: "sys_io_div4", gen: true, type: "int", parent: "sys_src", clock: "io_div4"},

    // 外设独立复位 (软件可控)
    {name: "spi_device", gen: true, type: "int",
     parent: "sys_src", clock: "io_div4", sw: true},
    {name: "i2c0", gen: true, type: "int",
     parent: "sys_src", clock: "io_div4", sw: true},
    {name: "i2c1", gen: true, type: "int",
     parent: "sys_src", clock: "io_div4", sw: true},
    // ...
  ]
}
```

### 3.3 复位节点属性

| 属性 | 说明 |
|------|------|
| `gen` | true=生成的, false=外部输入 |
| `type` | ext=外部, top=顶层, int=内部 |
| `parent` | 父复位(形成树状层次) |
| `clock` | 同步释放所用时钟域 |
| `sw` | true=软件可控复位 |

### 3.4 复位请求源

```hjson
reset_requests: {
  // 内部复位请求
  int: [
    {name: "MainPwr", module: "pwrmgr_aon",
     desc: "Power manager main power domain reset request"},
    {name: "Esc", module: "alert_handler",
     desc: "Escalation reset request from alert handler"},
  ],
  // 调试复位请求
  debug: [
    {name: "Ndm", module: "rv_dm",
     desc: "Non-debug module reset request from debug module"},
  ]
}
```

### 3.5 模块复位连接

```hjson
{
  name: "aes",
  reset_connections: {
    rst_ni:     "sys",          // 主复位(系统级)
    rst_edn_ni: "sys"           // EDN接口复位
  },
}

{
  name: "spi_device",
  reset_connections: {
    rst_ni: "spi_device"        // 独立复位域(可单独复位)
  },
}
```

## 4. 电源域管理

### 4.1 电源域定义

```hjson
power: {
  domains: ["Aon", "0"],  // AON域 + 主电源域
  default: "0"
}
```

### 4.2 AON (Always-On) 模块

带 `_aon` 后缀的模块属于AON域：
```hjson
{name: "pwrmgr_aon",     ...},
{name: "rstmgr_aon",     ...},
{name: "clkmgr_aon",     ...},
{name: "pinmux_aon",     ...},
{name: "aon_timer_aon",  ...},
{name: "sram_ctrl_ret_aon", ...},
```

## 5. rstmgr 自动生成

### 5.1 生成内容

rstmgr 根据 `resets` 配置自动生成：
- 复位树(多级复位展开)
- 每个复位域的同步释放逻辑
- 软件复位控制寄存器
- 复位原因记录寄存器
- crash dump (复位时CPU状态保存)
- 一致性检查 (glitch检测)

### 5.2 生成的RTL模块

```
rstmgr_aon/
├── rtl/
│   ├── rstmgr.sv              # 顶层复位管理器
│   ├── rstmgr_ctrl.sv         # 复位控制逻辑
│   ├── rstmgr_crash_info.sv   # Crash信息记录
│   └── rstmgr_cnsty_chk.sv   # 一致性检查
└── data/
    └── rstmgr.hjson           # 自动生成的寄存器描述
```

## 6. 时钟-复位对应关系

每个复位信号必须与一个时钟域关联，确保同步释放：

```
时钟域         复位信号
─────────     ──────────
main       →  rst_por, rst_lc, rst_sys
io_div4    →  rst_por_io_div4, rst_lc_io_div4, rst_sys_io_div4,
              rst_spi_device, rst_i2c0, rst_i2c1, rst_i2c2, ...
usb        →  rst_por_usb, rst_usb
io_div2    →  rst_por_io_div2
```

## 7. 安全特性

### 7.1 时钟安全
- 时钟频率监测 (超频/欠频检测)
- 时钟门控状态可读回
- Jitter控制

### 7.2 复位安全
- 复位一致性检查 (anti-glitch)
- 复位原因不可清除(直到软件确认)
- 分层复位(LC复位不影响AON)
- 复位展宽(确保最短复位脉宽)

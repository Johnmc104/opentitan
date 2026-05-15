# 05 - 中断与告警系统

## 1. 中断系统架构

OpenTitan 使用 RISC-V 标准 **Platform-Level Interrupt Controller (PLIC)** 管理中断。

### 1.1 中断收集流程

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  UART0   │  │   I2C0   │  │   GPIO   │
│ 9 irqs   │  │ 1 irq    │  │ 32 irqs  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │
     ▼              ▼              ▼
┌─────────────────────────────────────────────┐
│          RV_PLIC (186 sources)              │
│  ┌──────────────────────────────────────┐   │
│  │ 优先级寄存器 (per source)             │   │
│  │ 使能寄存器 (per target × source)     │   │
│  │ 阈值寄存器 (per target)              │   │
│  │ Claim/Complete 接口                   │   │
│  └──────────────────────────────────────┘   │
└──────────────────────┬──────────────────────┘
                       │ irq_o
                       ▼
              ┌────────────────┐
              │ rv_core_ibex   │
              │ (MIP.MEIP)     │
              └────────────────┘
```

### 1.2 中断源声明

在每个 IP 的 hjson 中声明中断：

```hjson
// uart.hjson
interrupt_list: [
  {name: "tx_watermark",  type: "status", desc: "..."},
  {name: "rx_watermark",  type: "status", desc: "..."},
  {name: "tx_empty",      type: "status", desc: "..."},
  {name: "rx_overflow",   type: "event",  desc: "..."},
  {name: "rx_frame_err",  type: "event",  desc: "..."},
  {name: "rx_break_err",  type: "event",  desc: "..."},
  {name: "rx_timeout",    type: "event",  desc: "..."},
  {name: "rx_parity_err", type: "event",  desc: "..."},
  {name: "tx_done",       type: "event",  desc: "..."},
]
```

**中断类型**：
- `status`：电平触发，持续有效直到条件清除
- `event`：边沿触发，脉冲式

### 1.3 中断编号自动分配

topgen 按以下规则自动分配中断编号：
1. 按 `module` 数组中的顺序
2. 每个模块的中断按 `interrupt_list` 顺序
3. 编号从 1 开始（0 保留为无中断）

生成结果保存在 `top_earlgrey.gen.hjson`。

### 1.4 自动生成的中断映射

```c
// 自动生成的 C 头文件 (sw层)
typedef enum top_earlgrey_plic_irq {
  kTopEarlgreyPlicIrqIdNone = 0,
  kTopEarlgreyPlicIrqIdUart0TxWatermark = 1,
  kTopEarlgreyPlicIrqIdUart0RxWatermark = 2,
  // ... 共186个中断源
  kTopEarlgreyPlicIrqIdLast = 186,
} top_earlgrey_plic_irq_t;
```

## 2. RV_PLIC 自动生成

### 2.1 参数自动确定

```hjson
// rv_plic.hjson (自动生成)
param_list: [
  {name: "NumSrc",    default: "186", local: "true"},  // 自动计算
  {name: "NumTarget", default: "1",   local: "true"},  // CPU数量
  {name: "PRIO",      default: "3"},                   // 优先级位宽
]
```

`NumSrc` 由 topgen 根据所有模块的 `interrupt_list` 总和自动计算。

### 2.2 生成文件

```
hw/top_earlgrey/ip_autogen/rv_plic/
├── rtl/
│   ├── rv_plic.sv           # PLIC顶层
│   ├── rv_plic_gateway.sv   # 中断网关(边沿检测)
│   └── rv_plic_target.sv    # 目标仲裁
├── data/
│   └── rv_plic.hjson        # 寄存器定义
└── dv/                      # 验证环境
```

## 3. 默认 PLIC 路由

```hjson
// top_earlgrey.hjson
default_plic: "rv_plic"
```

所有未明确指定 PLIC 目标的模块中断，默认路由到 `rv_plic`。

## 4. 告警系统 (Alert Handler)

### 4.1 告警架构

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│   AES    │  │   HMAC   │  │  KEYMGR  │
│ alert[0] │  │ alert[0] │  │ alert[0] │
│ alert[1] │  │          │  │ alert[1] │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │
     ▼              ▼              ▼
┌─────────────────────────────────────────────┐
│        Alert Handler (~65 alerts)           │
│  ┌──────────────────────────────────────┐   │
│  │ 差分信号对 (ping-pong protocol)      │   │
│  │ 分类 (Class A/B/C/D)                │   │
│  │ 升级策略 (阶段: NMI→复位→擦除)       │   │
│  │ 超时计数器                            │   │
│  └──────────────────────────────────────┘   │
└──────────────────────┬──────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
    NMI中断         系统复位       OTP擦除
  (CPU响应)     (rstmgr_aon)   (最终手段)
```

### 4.2 告警声明

```hjson
// aes.hjson
alert_list: [
  {name: "recov_ctrl_update_err",  desc: "..."},
  {name: "fatal_fault",            desc: "..."},
]
```

### 4.3 告警通信协议

告警使用**差分信号对**通信，具有固有的故障检测能力：
```systemverilog
typedef struct packed {
  logic alert_p;  // 正相
  logic alert_n;  // 反相
} alert_tx_t;

typedef struct packed {
  logic ping_p;   // 测试ping正相
  logic ping_n;   // 测试ping反相
  logic ack_p;    // 确认正相
  logic ack_n;    // 确认反相
} alert_rx_t;
```

### 4.4 升级策略

Alert Handler 支持可编程升级：

```
Phase 0: 中断 CPU (NMI)
   │ 超时
   ▼
Phase 1: 生成复位请求
   │ 超时
   ▼
Phase 2: 更强复位/安全擦除
   │ 超时
   ▼
Phase 3: 终极措施
```

### 4.5 默认 Alert Handler

```hjson
// top_earlgrey.hjson
default_alert_handler: "alert_handler"
```

所有模块的告警默认路由到此 Handler。

## 5. 跨模块信号 (Inter-Module Signals)

### 5.1 信号类型

除中断和告警外，模块间还有其他信号需要路由：

```hjson
// 在IP的inter_signal_list中声明
inter_signal_list: [
  // LC状态广播
  {struct: "lc_tx", type: "uni", act: "rcv",
   name: "lc_escalate_en", package: "lc_ctrl_pkg"},

  // OTP密钥传输
  {struct: "otp_keymgr_key", type: "uni", act: "rcv",
   name: "otp_key", package: "otp_ctrl_pkg"},

  // EDN (Entropy Distribution)
  {struct: "edn", type: "req_rsp", act: "req",
   name: "edn", package: "edn_pkg"},
]
```

### 5.2 连接模式

| 模式 | 说明 | 示例 |
|------|------|------|
| 1-to-1 | 点对点 | keymgr ← otp_ctrl (密钥) |
| 1-to-N | 一对多 | lc_ctrl → 多个模块 (LC状态) |
| broadcast | 广播 | alert_handler → 所有模块 (升级) |

### 5.3 顶层连接定义

```hjson
// top_earlgrey.hjson (inter_module section)
inter_module: {
  connect: {
    // 点对点连接
    "otp_ctrl.otp_keymgr_key": ["keymgr.otp_key"],

    // 广播连接
    "lc_ctrl.lc_dft_en": [
      "otp_ctrl.lc_dft_en",
      "pinmux_aon.lc_dft_en",
      "sram_ctrl_main.lc_dft_en"
    ],

    // EDN分发
    "edn0.edn_cmd_req": [
      "aes.edn",
      "otbn.edn_rnd",
      "kmac.entropy"
    ],
  }
}
```

## 6. 中断/告警数量汇总 (EarlGrey)

| 类别 | 数量 |
|------|------|
| PLIC 中断源 | 186 |
| 告警通道 | ~65 |
| 告警分类 | 4 (A/B/C/D) |
| 唤醒源 | ~6 |
| NMI 源 | 升级产生 |

## 7. topgen 中断/告警处理函数

```python
# util/topgen/merge.py 关键函数
amend_interrupt()       # 收集所有模块中断，分配编号
amend_alert()          # 收集所有模块告警，分配编号
commit_interrupt_modules()  # 最终确认中断映射
commit_alert_modules()     # 最终确认告警映射
commit_alert_connections() # 生成告警信号连接
create_alert_lpgs()        # 创建告警低功耗组
```

## 8. 软件接口生成

topgen 同时生成软件侧接口：
- C 枚举定义（中断ID、告警ID）
- 中断处理函数原型
- PLIC 寄存器偏移量
- DIF (Device Interface Functions) 框架

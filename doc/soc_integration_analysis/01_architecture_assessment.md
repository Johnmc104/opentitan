# 01 - 总体架构评估

## 1. 设计哲学

OpenTitan 的 SoC 集成采用**声明式配置 + 代码生成**的方法论：

- **关注点分离**：IP 不知道 SoC 细节，SoC 配置文件负责集成
- **渐进式细化**：信息从 IP 定义 → SoC 配置 → 生成 RTL 逐步细化
- **可复用性**：同一 IP 可在不同 SoC 变体中复用
- **自动化优先**：最小化手动互连，最大化一致性

## 2. 核心配置文件体系

```
hw/top_earlgrey/
├── data/
│   ├── top_earlgrey.hjson          # ★ 顶层SoC集成规格(核心文件)
│   ├── xbar_main.hjson             # 主crossbar定义
│   ├── xbar_peri.hjson             # 外设crossbar定义
│   └── autogen/
│       └── top_earlgrey.gen.hjson  # 完全展开后的生成配置
├── ip_autogen/                     # 自动生成的参数化IP
│   ├── clkmgr/                     # 时钟管理器
│   ├── rstmgr/                     # 复位管理器
│   ├── pinmux/                     # 引脚复用器
│   ├── rv_plic/                    # RISC-V 中断控制器
│   ├── pwrmgr/                     # 电源管理器
│   └── alert_handler/              # 告警处理器
├── templates/                       # Mako模板
│   ├── chiplevel.sv.tpl            # 芯片顶层模板
│   └── toplevel.sv.tpl             # SoC顶层模板
└── rtl/autogen/                    # 生成的RTL输出
```

## 3. 生成工具链

### 3.1 核心工具

| 工具 | 入口脚本 | 功能 |
|------|----------|------|
| **topgen** | `util/topgen.py` | 顶层模块生成器，编排整个生成流程 |
| **tlgen** | `util/tlgen.py` | TileLink Crossbar 生成器 |
| **ipgen** | `util/ipgen/` | IP模板实例化器 |
| **reggen** | `util/reggen/` | 寄存器模型生成器 |
| **raclgen** | `util/raclgen/` | 访问控制策略生成器 |

### 3.2 topgen 子模块结构

```
util/topgen/
├── lib.py            # 核心工具函数(文件查找、加载、写入)
├── merge.py          # 配置合并与展开
├── validate.py       # 综合验证规则
├── clocks.py         # 时钟域提取与管理
├── resets.py         # 复位信号生成与验证
├── top.py            # 顶层模块数据表示
├── intermodule.py    # 模块间信号定义
├── gen_dv.py         # 测试平台生成
├── gen_top_docs.py   # 文档生成
├── c.py / rust.py    # SW头文件生成
└── templates/        # Mako模板集
```

## 4. 端到端生成流程

```
┌─────────────────────────────────────────────────────────────────┐
│ 输入层                                                           │
├─────────────────────────────────────────────────────────────────┤
│  hw/ip/*/data/*.hjson        各IP的接口/寄存器定义               │
│  hw/top_*/data/top_*.hjson   SoC拓扑/集成规格                   │
│  hw/top_*/data/xbar_*.hjson  Crossbar拓扑                       │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Stage 1: 加载验证    │
                    │  - 加载所有IP定义     │
                    │  - 加载xbar定义       │
                    │  - 逐项验证          │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Stage 2: 展开合并    │
                    │  - 展开通用外设       │
                    │  - 拓扑排序ipgen IP   │
                    │  - 生成clkmgr/rstmgr │
                    │  - 生成pinmux/rv_plic │
                    │  - 生成xbar          │
                    │  - 验证互连          │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Stage 3: 生成完整配置│
                    │  输出: *.gen.hjson    │
                    │  (所有地址/中断已解析) │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Stage 4: 输出制品    │
                    │  - 顶层RTL           │
                    │  - Crossbar RTL      │
                    │  - 寄存器模型/RAL     │
                    │  - C/Rust头文件       │
                    │  - 验证环境          │
                    │  - 文档              │
                    └─────────────────────┘
```

## 5. 命令行调用

```bash
# 典型生成命令
util/topgen.py -t hw/top_earlgrey/data/top_earlgrey.hjson \
               -o hw/top_earlgrey/

# 仅生成crossbar
util/tlgen.py -t hw/top_earlgrey/data/xbar_main.hjson \
              -o hw/top_earlgrey/ip/xbar_main/
```

## 6. 模块属性类型

top_earlgrey.hjson 中模块有多种类型：

| attr | 说明 |
|------|------|
| `normal` (默认) | 普通IP，直接例化 |
| `templated` | 参数化模板IP，需通过topgen处理 |
| `ipgen` | 新版模板IP，使用ipgen流程 |
| `reggen_top` | 非模板但需要reggen，且例化在顶层 |
| `reggen_only` | 需要reggen但不在顶层例化 |

## 7. 数据格式选择：Hjson

OpenTitan 选择 Hjson（Human JSON）作为配置格式，原因：
- 支持注释（JSON不支持）
- 宽松语法（无需严格引号/逗号）
- 人类友好的编辑体验
- Python/JavaScript 均有解析库
- 可表达复杂嵌套结构

## 8. 验证贯穿全流程

每个生成阶段都内置验证：
1. **IP验证**：寄存器映射正确性、中断数量、端口存在性
2. **Xbar验证**：连接规则、时钟域一致性、地址空间冲突
3. **顶层验证**：所有中断已路由、时钟复位覆盖、地址无碰撞

## 9. 优势总结

| 特性 | 描述 |
|------|------|
| 单一真相源 | 一个hjson文件定义SoC全部拓扑 |
| 零手动互连 | 所有信号连接自动生成 |
| 一致性保证 | 工具验证地址冲突/信号遗漏 |
| 多目标输出 | 从同一配置生成RTL/SW/验证/文档 |
| 可扩展 | 新增IP只需添加hjson并注册到顶层 |

## 10. 局限性

| 限制 | 影响 |
|------|------|
| 仅支持TileLink | 如需AXI/AHB需额外适配 |
| Hjson非工业标准 | 无法与EDA工具直接互操作 |
| 学习曲线 | 自定义topgen模板需深入理解工具链 |
| 紧耦合 | 工具链与OpenTitan仓库结构耦合 |

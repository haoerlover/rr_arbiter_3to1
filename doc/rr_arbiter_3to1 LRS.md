# 3选1轮询仲裁器 逻辑规格说明书（LRS）

**模块名称**：rr_arbiter_3to1
**文档类型**：LRS（Logic Requirement Specification）
**版本**：v1.0
**适用阶段**：3 路 valid/ready 输入流合并为 1 路 valid/ready 输出流的 RTL 设计、验证与集成

**对齐记录：**

| 日期 | 参与人员 | 对齐内容 | 确认结果 |
|------|----------|----------|----------|
| 2026-08-18 | 用户 / Codex | 接口采用 3 路输入数据流合并 1 路输出数据流，端口包含 `in_valid_i[2:0]`、`in_ready_o[2:0]`、`in_data_i[2:0][DATA_WIDTH-1:0]`、`out_valid_o`、`out_ready_i`、`out_data_o`、`grant_o[2:0]` | 已确认 |
| 2026-08-18 | 用户 / Codex | 轮询指针仅在 `out_valid_o && out_ready_i` 成功握手后前进，复位后优先级从 input0 开始 | 已确认 |

# 1. 引言

## 1.1 文档目的

本 LRS 定义 `rr_arbiter_3to1` 的对外接口、仲裁行为、时序、性能、异常处理和维测要求。RTL 实现、testbench、自检用例和集成文件以本文档为功能契约。

## 1.2 适用范围

`rr_arbiter_3to1` 适用于单时钟域内 3 路上游 producer 共享 1 路下游 consumer 的数据路径。模块完成请求选择、输入 ready 反压、输出 valid/data 保持、轮询公平性维护和握手计数可观测。

## 1.3 不覆盖范围

以下能力不属于 v1.0 RTL 强制实现项：

| 项目 | 范围结论 | 原因 |
|------|----------|------|
| CSR / APB 配置口 | 不实现 | 仲裁路数固定为 3，数据位宽通过静态参数 `DATA_WIDTH` 配置 |
| 中断 | 不实现 | 模块无软件任务完成事件和错误中断输出 |
| 安全空间访问控制 | 不实现 | 模块无地址、命令和安全属性输入 |
| SRAM / RAM / 深 FIFO | 不实现 | 模块仅使用少量寄存器保存输出拍和轮询指针 |
| 跨时钟域 | 不实现 | 所有输入、输出和状态均归属 `clk_i` |

## 1.4 术语表

| 术语 | 定义 |
|------|------|
| valid/ready 握手 | 当 `valid=1` 且 `ready=1` 时，一个数据拍在当前 `clk_i` 上升沿完成传输 |
| 上游 producer | 驱动 `in_valid_i[x]` 和 `in_data_i[x]` 的外部模块 |
| 下游 consumer | 驱动 `out_ready_i` 并接收 `out_valid_o/out_data_o` 的外部模块 |
| 轮询指针 | 下一次仲裁扫描的起始输入编号，取值为 0、1、2 |
| grant | 与当前输出拍绑定的 one-hot 输入来源标识 |

# 2. 模块概述

## 2.1 功能概述

`RRA3.LRS.SUMM.01`：模块在系统数据路径中位于 3 路上游数据源和 1 路下游处理单元之间，对同一时钟域内的输入请求进行 round-robin 仲裁，并将被选中的数据拍输出到下游。

`RRA3.LRS.SUMM.02`：模块应用场景包括多通道采集数据汇聚、多个协议适配器共享单个处理流水线、多个 DMA 子通道共享单个写入口、多个计算前级共享单个后级队列。

`RRA3.LRS.SUMM.03`：模块不改变数据内容，不生成包头，不拆分或合并多个数据拍；每次输出握手对应一个输入数据拍。

## 2.2 系统位置与数据流

| 顺序 | 参与对象 | 信号 | 行为 |
|------|----------|------|------|
| 1 | input0/input1/input2 | `in_valid_i[x]`, `in_data_i[x]` | 上游在有数据时拉高 valid 并保持数据稳定，直到对应 `in_ready_o[x]` 握手 |
| 2 | `rr_arbiter_3to1` | `in_ready_o[2:0]` | 模块按轮询指针从 3 路输入中选择 1 路，并仅对被选择路返回 ready |
| 3 | `rr_arbiter_3to1` | `out_valid_o`, `out_data_o`, `grant_o` | 模块向下游输出被选择数据拍及来源标识 |
| 4 | 下游 consumer | `out_ready_i` | 下游接受输出拍时拉高 ready；未接受时模块保持输出 valid/data/grant |
| 5 | `rr_arbiter_3to1` | 内部轮询指针 | 输出握手成功后，下一次扫描从刚服务输入的后一号输入开始 |

## 2.3 设计目标

`RRA3.LRS.SUMM.04`：在 `out_ready_i=1` 且至少一路输入持续有效时，模块启动后达到每周期 1 个数据拍的输出吞吐率。

`RRA3.LRS.SUMM.05`：在所有输入持续有效且下游持续 ready 的场景下，输出来源顺序为 input0、input1、input2 循环。

# 3. 功能描述

## 3.1 仲裁选择

`RRA3.LRS.FUNC.01`：复位释放后，轮询指针值为 0。首次可服务输入从 input0 开始扫描，扫描顺序为 0、1、2。

`RRA3.LRS.FUNC.02`：每次仲裁时，模块从当前轮询指针指向的输入开始，按编号递增并在 2 后回绕到 0，选择第一路 `in_valid_i[x]=1` 的输入。

`RRA3.LRS.FUNC.03`：当多路输入同时有效时，本轮被选择路由轮询指针决定；未被选择的有效输入保持等待。

`RRA3.LRS.FUNC.04`：当所有输入均无效时，模块不得产生新的输出拍，`out_valid_o` 为 0，`grant_o` 为 0。

## 3.2 valid/ready 握手

`RRA3.LRS.FUNC.05`：模块只对当前可接受且被选择的输入拉高对应 `in_ready_o[x]`，其余 `in_ready_o` 位为 0。

`RRA3.LRS.FUNC.06`：当输出拍已存在且 `out_ready_i=0` 时，模块保持 `out_valid_o=1`、`out_data_o` 和 `grant_o` 不变，并且不接受新的输入数据拍。

`RRA3.LRS.FUNC.07`：当 `out_valid_o && out_ready_i` 在 `clk_i` 上升沿成立时，当前输出拍完成传输，轮询指针更新为当前 grant 输入编号的后一号输入。

`RRA3.LRS.FUNC.08`：模块采用 1 拍输出寄存器。输入握手成功后的下一拍起，输出侧可以观察到对应 `out_valid_o/out_data_o/grant_o`。

## 3.3 数据与来源标识

`RRA3.LRS.FUNC.09`：`out_data_o` 等于被 grant 输入路的 `in_data_i[x]`，宽度为 `DATA_WIDTH` bit。

`RRA3.LRS.FUNC.10`：`grant_o` 为 one-hot 编码，仅在 `out_valid_o=1` 时有效。`grant_o=3'b001` 表示 input0，`grant_o=3'b010` 表示 input1，`grant_o=3'b100` 表示 input2。

## 3.4 正常轮询时序

```wavedrom
{ "config": { "hscale": 2 },
  "signal": [
    { "name": "clk_i",             "wave": "p........." },
    { "name": "rst_n_i",           "wave": "0.1......." },
    { "name": "in_valid_i[2:0]",   "wave": "x.=.=.=..x", "data": ["001", "011", "110"] },
    { "name": "out_ready_i",       "wave": "1........." },
    { "name": "in_ready_o[2:0]",   "wave": "x..===...x", "data": ["001", "010", "100"] },
    { "name": "out_valid_o",       "wave": "0..1.....0" },
    { "name": "grant_o[2:0]",      "wave": "x..===...x", "data": ["001", "010", "100"] },
    { "name": "out_data_o",        "wave": "x..===...x", "data": ["D0", "D1", "D2"] },
    { "name": "arb_ptr",           "wave": "=..===...=", "data": ["P0", "P1", "P2", "P0", "P1"] }
  ]
}
```

关键节点：

- cycle 2：复位释放，轮询指针处于 P0。
- cycle 3：input0 被接受；因输出寄存器存在，`out_valid_o` 从下一拍起有效。
- cycle 4-6：输出侧连续握手，grant 顺序为 input0、input1、input2。
- cycle 6 后：最后一次成功输出来自 input2，下一次扫描起点回到 input0。

## 3.5 下游反压时序

```wavedrom
{ "config": { "hscale": 2 },
  "signal": [
    { "name": "clk_i",             "wave": "p........." },
    { "name": "rst_n_i",           "wave": "0.1......." },
    { "name": "in_valid_i[2:0]",   "wave": "x.=......x", "data": ["001"] },
    { "name": "out_ready_i",       "wave": "1..0..1..." },
    { "name": "in_ready_o[2:0]",   "wave": "x..=..0...", "data": ["001"] },
    { "name": "out_valid_o",       "wave": "0..1....0." },
    { "name": "grant_o[2:0]",      "wave": "x..=....x.", "data": ["001"] },
    { "name": "out_data_o",        "wave": "x..=....x.", "data": ["D0"] },
    { "name": "arb_ptr",           "wave": "=........=", "data": ["P0", "P1"] }
  ]
}
```

关键节点：

- cycle 3：模块接受 input0 数据拍并写入输出寄存器。
- cycle 4-6：下游反压，模块保持输出数据与 grant，不接受新的输入拍。
- cycle 7：`out_ready_i=1` 后输出握手完成，轮询指针前进到 P1。

# 4. 接口描述

## 4.1 参数

| 参数 | 默认值 | 合法范围 | 说明 |
|------|--------|----------|------|
| `DATA_WIDTH` | 32 | 1 到 1024 | 每路输入数据和输出数据宽度 |
| `NUM_INPUTS` | 3 | 固定为 3 | v1.0 RTL 固定三路仲裁，不对外开放改配 |

## 4.2 时钟复位接口

| 信号名 | 方向 | 位宽 | 复位值 | 说明 |
|--------|------|------|--------|------|
| `clk_i` | input | 1 | 不适用 | 单一工作时钟，所有输入输出和内部状态均在上升沿采样或更新 |
| `rst_n_i` | input | 1 | 0 | 低有效异步复位，同步释放到 `clk_i` 域 |

## 4.3 输入数据接口

| 信号名 | 方向 | 位宽 | 复位值 | 说明 |
|--------|------|------|--------|------|
| `in_valid_i[2:0]` | input | 3 | 0 | 每路输入数据有效指示 |
| `in_ready_o[2:0]` | output | 3 | 0 | 每路输入接受指示；同一周期最多 1 bit 为 1 |
| `in_data_i[2:0][DATA_WIDTH-1:0]` | input | `3*DATA_WIDTH` | 不适用 | 每路输入数据；上游须在 valid 拉高且 ready 未返回期间保持对应数据稳定 |

## 4.4 输出数据接口

| 信号名 | 方向 | 位宽 | 复位值 | 说明 |
|--------|------|------|--------|------|
| `out_valid_o` | output | 1 | 0 | 输出数据有效指示 |
| `out_ready_i` | input | 1 | 1 | 下游输出接受指示 |
| `out_data_o[DATA_WIDTH-1:0]` | output | `DATA_WIDTH` | 0 | 输出数据 |
| `grant_o[2:0]` | output | 3 | 0 | 当前输出拍来源 one-hot 标识，仅在 `out_valid_o=1` 时有效 |

## 4.5 接口时序与约束

`RRA3.LRS.INTF.01`：所有输入 valid/data、输出 ready 均由 `clk_i` 上升沿采样。

`RRA3.LRS.INTF.02`：上游在 `in_valid_i[x]=1 && in_ready_o[x]=0` 期间须保持 `in_data_i[x]` 稳定；违反该约束会导致该路输出数据不等于上游原始意图。

`RRA3.LRS.INTF.03`：下游在 `out_valid_o=1 && out_ready_i=0` 期间观察到的 `out_data_o/grant_o` 保持稳定。

`RRA3.LRS.INTF.04`：`in_ready_o` 同周期最多 1 bit 为 1，不存在同时接受两路输入的行为。

`RRA3.LRS.INTF.05`：输入侧握手和输出侧握手归属同一 `clk_i` 时钟域，不存在 CDC 路径。

## 4.6 CSR 适用性说明

`RRA3.LRS.INTF.06`：模块自身不提供 APB、AHB、AXI-Lite 或 CSR 端口。所有配置均由参数静态确定，运行期无软件可写寄存器。

| CSR 项 | 结论 | 替代机制 |
|--------|------|----------|
| 配置寄存器 | 不实现 | `DATA_WIDTH` 静态参数 |
| 状态寄存器 | 不实现 | `grant_o` 和握手信号直接对外可观测 |
| 计数寄存器 | 不实现 | testbench 内部统计；SoC 集成层如需软件计数，由外层 wrapper 实现 |

# 5. 性能需求

## 5.1 延迟

`RRA3.LRS.PERF.01`：在输出寄存器为空且下游可接受的前提下，输入握手成功后 1 个 `clk_i` 周期内，输出侧产生对应 `out_valid_o/out_data_o/grant_o`。

`RRA3.LRS.PERF.02`：当输出寄存器已满且 `out_ready_i=0` 时，新的输入接受延迟由下游反压持续时间决定，模块不得丢失已保存输出拍。

## 5.2 吞吐量

`RRA3.LRS.PERF.03`：在 `out_ready_i=1` 且至少一路 `in_valid_i=1` 持续成立的前提下，启动填充后模块每周期输出 1 个数据拍。

`RRA3.LRS.PERF.04`：在 `clk_i=500 MHz`、`DATA_WIDTH=32`、下游无反压的前提下，理论有效数据吞吐量为 `500M * 32 bit = 16 Gbps`。

`RRA3.LRS.PERF.05`：在 3 路输入持续有效且下游无反压的前提下，每一路输入每 3 个输出握手周期获得 1 次服务。

## 5.3 可验证指标

| 指标 | 前提 | 期望值 | 验证方法 |
|------|------|--------|----------|
| 单拍输入到输出延迟 | 输出寄存器为空，`out_ready_i=1` | 1 cycle | TB 检查输入握手后一拍输出数据 |
| 满载吞吐 | 任意一路或多路输入持续 valid，`out_ready_i=1` | 1 beat/cycle | TB 连续计数输出握手 |
| 满载公平性 | 3 路持续 valid，`out_ready_i=1` | grant 顺序 0、1、2 循环 | TB scoreboard 检查 `grant_o` |
| 反压保持 | `out_valid_o=1` 后 `out_ready_i=0` | data/grant 不变 | TB 在反压窗口逐周期比较 |

# 6. 低功耗需求

`RRA3.LRS.LP.01`：复位后且无输入有效时，输出寄存器 valid 位保持 0，数据寄存器保持复位值，不因空闲扫描产生输出翻转。

`RRA3.LRS.LP.02`：输出反压期间，输出数据和 grant 寄存器保持不变。

`RRA3.LRS.LP.03`：v1.0 RTL 不例化门控时钟单元；综合阶段可基于寄存器使能推导时钟门控。

# 7. 资源需求

## 7.1 逻辑资源估计

`RRA3.LRS.AREA.01`：模块包含 3 路 valid 扫描组合逻辑、3 bit one-hot grant 组合逻辑、2 bit 轮询指针寄存器、1 bit 输出 valid 寄存器、3 bit grant 寄存器和 `DATA_WIDTH` bit 输出数据寄存器。

`RRA3.LRS.AREA.02`：当 `DATA_WIDTH=32` 时，状态寄存器数量为 `2 + 1 + 3 + 32 = 38 bit`，不含综合插入的优化寄存器。

## 7.2 存储资源

`RRA3.LRS.AREA.03`：模块不例化 SRAM、RAM、ROM 或深 FIFO。唯一数据存储为 1 拍输出数据寄存器。

# 8. 使用方法及限制

## 8.1 初始化流程

`RRA3.LRS.FLOW.01`：集成方须在 `clk_i` 稳定后拉低 `rst_n_i` 至少 2 个 `clk_i` 周期。

`RRA3.LRS.FLOW.02`：集成方释放 `rst_n_i` 后，等待 1 个 `clk_i` 周期再允许上游拉高 `in_valid_i`。

`RRA3.LRS.FLOW.03`：复位释放后的第一笔服务从 input0 优先级开始。

## 8.2 工作流程

| 步骤 | 操作 | 违反后果 |
|------|------|----------|
| 1 | 上游在数据可用时拉高对应 `in_valid_i[x]` 并保持数据稳定 | 数据在握手前变化会导致输出数据不可追溯 |
| 2 | 模块返回对应 `in_ready_o[x]` 后，上游在该上升沿完成输入传输 | 上游忽略 ready 会导致数据拍未被模块接受 |
| 3 | 下游在可接受输出时拉高 `out_ready_i` | 下游持续拉低 ready 会阻塞全部输入 |
| 4 | 下游在 `out_valid_o && out_ready_i` 上升沿采样 `out_data_o/grant_o` | 下游在非握手周期采样的数据无效 |

## 8.3 使用限制

`RRA3.LRS.FLOW.04`：`NUM_INPUTS` 在 v1.0 中固定为 3，集成时不得改为其他值。

`RRA3.LRS.FLOW.05`：`DATA_WIDTH` 的取值须在 1 到 1024 之间。

`RRA3.LRS.FLOW.06`：上游须遵守 valid/ready 源端稳定性约束。

`RRA3.LRS.FLOW.07`：模块不得跨时钟域直连异步输入；异步来源须在外部完成 CDC 同步或异步 FIFO 适配。

# 9. 异常处理

## 9.1 协议约束违规

| 异常 | 检测能力 | 硬件行为 | 外部处理 |
|------|----------|----------|----------|
| 上游 valid 等待 ready 期间改变 data | RTL 不检测 | 输出该周期采样到的数据 | 上游协议检查器或 TB 断言报错 |
| 下游长期 `out_ready_i=0` | RTL 不设超时计数 | 保持当前输出拍并反压全部输入 | 系统级 watchdog 负责超时策略 |
| 多路输入同时有效 | 正常场景 | round-robin 选择一路，其他路等待 | 无需处理 |
| 所有输入无效 | 正常空闲场景 | `out_valid_o=0`，不更新轮询指针 | 无需处理 |

## 9.2 多异常关系

`RRA3.LRS.ERR.01`：上游协议违规和下游反压同时发生时，模块优先保持当前输出拍；上游侧违规由外部协议检查器定位。

`RRA3.LRS.ERR.02`：模块不产生错误码、中断或 sticky 状态位。

# 10. 维护需求

## 10.1 可观测信号

`RRA3.LRS.DFX.01`：`grant_o[2:0]` 对外暴露当前输出拍来源，满足输出级来源定位。

`RRA3.LRS.DFX.02`：`in_valid_i/in_ready_o/out_valid_o/out_ready_i` 均为模块边界信号，集成层可直接采样用于挂死定位。

`RRA3.LRS.DFX.03`：RTL 内部轮询指针可作为仿真层级信号观察，综合交付不承诺将该内部信号作为顶层端口。

## 10.2 统计需求

`RRA3.LRS.DFX.04`：v1.0 顶层不提供硬件计数器端口。验证环境须统计输入握手数、输出握手数和各输入 grant 次数。

`RRA3.LRS.DFX.05`：满载公平性测试须记录每路 grant 次数，任意两路 grant 次数差值不得超过 1。

# 11. 模块内部架构参考

## 11.1 内部结构

| 内部单元 | 职责 | 关键状态 |
|----------|------|----------|
| 轮询扫描组合逻辑 | 从扫描起点开始查找首个有效输入 | `scan_start`, `sel_oh` |
| 输出寄存器 | 保存 1 拍输出 valid/data/grant | `out_valid_q`, `out_data_q`, `grant_q` |
| 指针更新逻辑 | 在输出握手后更新下一次扫描起点 | `arb_ptr_q` |

## 11.2 状态转移

`RRA3.LRS.FUNC.11`：模块控制状态可抽象为 `EMPTY` 和 `FULL`。`EMPTY` 表示输出寄存器无有效拍；`FULL` 表示输出寄存器保存待下游接受的数据拍。

`RRA3.LRS.FUNC.12`：`EMPTY` 状态下若存在有效输入且输出寄存器可写，则进入 `FULL`。`FULL` 状态下若 `out_ready_i=1` 且同周期无新输入被接受，则进入 `EMPTY`；若同周期接受新输入，则保持 `FULL`。

## 11.3 实现约束

`RRA3.LRS.FUNC.13`：RTL 采用可综合 SystemVerilog 编写，不使用 testbench-only 语法。

`RRA3.LRS.FUNC.14`：复位后 `out_valid_o=0`、`out_data_o=0`、`grant_o=0`、轮询指针为 0。

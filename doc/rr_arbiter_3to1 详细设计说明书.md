# rr_arbiter_3to1 详细设计说明书

**模块名称**：`rr_arbiter_3to1`
**设计类型**：3 选 1 round-robin valid/ready 仲裁器
**RTL 文件**：`rtl/rr_arbiter_3to1.sv`
**TB 文件**：`tb/tb_rr_arbiter_3to1.sv`
**波形文件**：`sim/dump.fsdb`
**版本**：v1.0
**日期**：2026-08-18

## 1. 设计目标

`rr_arbiter_3to1` 用于把 3 路输入 valid/ready 数据流合并成 1 路输出 valid/ready 数据流。

模块要解决的问题是：当多路输入同时有数据时，不能永远偏向 input0，而是要让 input0、input1、input2 轮流获得服务。

本设计的核心目标如下：

| 目标 | 说明 |
|------|------|
| 3 选 1 | 从 `in_valid_i[2:0]` 中选择一路有效输入 |
| round-robin 公平性 | 多路持续 valid 时，grant 顺序轮转，不固定优先级 |
| valid/ready 握手 | 输入和输出都遵循 valid/ready 协议 |
| 下游反压保护 | `out_ready_i=0` 时不得覆盖尚未被接收的输出数据 |
| 1 拍输出寄存 | 输入握手后，下一拍从输出端看到对应数据 |

## 2. 接口说明

### 2.1 顶层端口

| 信号 | 方向 | 位宽 | 含义 |
|------|------|------|------|
| `clk_i` | input | 1 | 工作时钟 |
| `rst_n_i` | input | 1 | 低有效异步复位 |
| `in_valid_i[2:0]` | input | 3 | 3 路输入 valid |
| `in_ready_o[2:0]` | output | 3 | 3 路输入 ready |
| `in_data_i[2:0][DATA_WIDTH-1:0]` | input | `3*DATA_WIDTH` | 3 路输入数据 |
| `out_valid_o` | output | 1 | 输出 valid |
| `out_ready_i` | input | 1 | 输出 ready |
| `out_data_o[DATA_WIDTH-1:0]` | output | `DATA_WIDTH` | 输出数据 |
| `grant_o[2:0]` | output | 3 | 当前输出数据来自哪一路输入 |

### 2.2 grant 编码

| `grant_o` | 含义 |
|-----------|------|
| `3'b001` | 当前输出数据来自 input0 |
| `3'b010` | 当前输出数据来自 input1 |
| `3'b100` | 当前输出数据来自 input2 |
| `3'b000` | 当前没有有效输出 |

`grant_o` 是 one-hot 编码。它描述的是当前输出拍的数据来源，不是当前输入侧即将接受哪一路。

这一点很容易混淆：

| 信号 | 表示对象 | 时间含义 |
|------|----------|----------|
| `grant_o` | 当前输出寄存器里的数据来自哪一路 | 当前正在被下游消费的数据 |
| `in_ready_o` | 当前可以接收哪一路输入 | 准备装入下一拍输出寄存器的数据 |

在满载无反压场景下，`grant_o` 和 `in_ready_o` 经常不是同一路。这不是 bug，而是因为本设计允许“当前输出被消费”和“下一笔输入被接收”在同一个时钟沿发生。

## 3. valid/ready 协议

### 3.1 输入侧握手

输入侧第 `x` 路握手条件：

```systemverilog
in_valid_i[x] && in_ready_o[x]
```

当这个条件在 `clk_i` 上升沿成立时，模块接受 `in_data_i[x]`。

上游需要遵守源端稳定性规则：当 `in_valid_i[x]=1` 且 `in_ready_o[x]=0` 时，`in_data_i[x]` 必须保持不变。

### 3.2 输出侧握手

输出侧握手条件：

```systemverilog
out_valid_o && out_ready_i
```

当这个条件在 `clk_i` 上升沿成立时，下游接受 `out_data_o` 和 `grant_o`。

本设计规定轮询指针只在输出侧握手成功后前进。也就是说，真正完成一次服务的判据是“数据被下游接收”，不是“输入被模块接收”。

## 4. 内部状态

RTL 中主要内部信号如下：

| 信号 | 类型 | 含义 |
|------|------|------|
| `out_valid_q` | 寄存器 | 输出寄存器是否保存有效数据 |
| `out_data_q` | 寄存器 | 输出数据寄存器 |
| `grant_q` | 寄存器 | 输出数据来源寄存器 |
| `arb_ptr_q` | 寄存器 | 下一次轮询扫描的起点 |
| `sel_oh` | 组合逻辑 | 本周期选择到的输入 one-hot |
| `sel_data` | 组合逻辑 | 本周期选择到的输入数据 |
| `sel_valid` | 组合逻辑 | 本周期是否找到有效输入 |
| `can_accept` | 组合逻辑 | 输出寄存器当前是否可以接收新输入 |
| `scan_start` | 组合逻辑 | 本周期实际扫描起点 |

## 5. 整体数据路径

一次完整的数据传输分两段：

| 阶段 | 发生条件 | 行为 |
|------|----------|------|
| 输入接收 | `in_valid_i[x] && in_ready_o[x]` | 模块采样 `in_data_i[x]`，同时记录来源 `x` |
| 输出发送 | `out_valid_o && out_ready_i` | 下游采样 `out_data_o` 和 `grant_o` |

因为模块中间有 1 拍输出寄存器，所以输入接收和输出发送之间隔 1 拍。输入数据不是组合直通到输出，而是先进入 `out_data_q/grant_q/out_valid_q`。

## 6. 输出寄存器设计

输出端由三个寄存器组成：

```systemverilog
logic                  out_valid_q;
logic [DATA_WIDTH-1:0] out_data_q;
logic [2:0]            grant_q;
```

对应输出赋值：

```systemverilog
assign out_valid_o = out_valid_q;
assign out_data_o  = out_data_q;
assign grant_o     = out_valid_q ? grant_q : 3'b000;
```

这样设计有两个好处：

1. 下游反压时，输出数据可以稳定保持。
2. `grant_o` 和 `out_data_o` 生命周期一致，都是当前输出拍的一部分。

如果没有输出寄存器，输入选择结果会组合直通输出。下游拉低 `out_ready_i` 时，如果上游 valid 变化，输出数据也会变化，这会违反 valid/ready 协议里的保持要求。

## 7. 何时可以接收新输入

RTL 使用 `can_accept` 表示输出寄存器是否可以写入新数据：

```systemverilog
assign can_accept = (!out_valid_q) || out_ready_i;
```

含义如下：

| `out_valid_q` | `out_ready_i` | `can_accept` | 解释 |
|---------------|---------------|--------------|------|
| 0 | 0 | 1 | 输出寄存器空，可以接收 |
| 0 | 1 | 1 | 输出寄存器空，可以接收 |
| 1 | 0 | 0 | 输出寄存器满且下游不接收，不能接收 |
| 1 | 1 | 1 | 当前输出会被消费，可以同拍接收下一笔 |

最重要的是最后一行：当 `out_valid_q=1` 且 `out_ready_i=1` 时，当前输出拍会在这个上升沿被下游拿走，所以输出寄存器可以在同一个上升沿装入下一笔输入。这就是满载场景可以达到 1 beat/cycle 的原因。

## 8. 轮询指针

### 8.1 指针含义

`arb_ptr_q` 表示下一次常规扫描从哪一路输入开始。

| `arb_ptr_q` | 扫描顺序 |
|-------------|----------|
| 0 | input0 -> input1 -> input2 |
| 1 | input1 -> input2 -> input0 |
| 2 | input2 -> input0 -> input1 |

复位后：

```systemverilog
arb_ptr_q <= 2'd0;
```

所以复位后第一优先级是 input0。

### 8.2 next_idx 函数

RTL 用 `next_idx` 实现 0、1、2 的循环：

```systemverilog
function automatic logic [1:0] next_idx(input logic [1:0] idx);
  if (idx == 2'd2) begin
    next_idx = 2'd0;
  end else begin
    next_idx = idx + 2'd1;
  end
endfunction
```

效果是：

| 输入 | 输出 |
|------|------|
| 0 | 1 |
| 1 | 2 |
| 2 | 0 |

## 9. 扫描起点 scan_start

RTL 中 `scan_start` 的逻辑是：

```systemverilog
assign grant_idx_q = onehot_to_idx(grant_q);
assign scan_start = (out_valid_q && out_ready_i) ? next_idx(grant_idx_q) : arb_ptr_q;
```

这里是整个设计最容易不理解的地方。

### 9.1 普通情况

如果当前没有输出握手：

```systemverilog
scan_start = arb_ptr_q;
```

也就是从保存的轮询指针开始扫。

### 9.2 同拍消费与回填

如果当前有输出握手：

```systemverilog
out_valid_q && out_ready_i
```

说明当前 `grant_q` 对应的数据会在这个上升沿被下游接收。为了同拍装入下一笔数据，组合选择逻辑不能继续从旧的 `arb_ptr_q` 开始扫，而要直接从当前 grant 的下一路开始扫：

```systemverilog
scan_start = next_idx(grant_idx_q);
```

这个设计避免了一个问题：满载时刚输出 input0，如果同拍又从 input0 开始选择，就会连续服务 input0，破坏轮询公平性。

## 10. 组合扫描逻辑

核心扫描逻辑如下：

```systemverilog
always_comb begin
  sel_oh    = 3'b000;
  sel_data  = '0;
  sel_valid = 1'b0;

  for (int unsigned offset = 0; offset < NUM_INPUTS; offset++) begin
    int unsigned idx;

    idx = scan_start + offset;
    if (idx >= NUM_INPUTS) begin
      idx = idx - NUM_INPUTS;
    end

    if (!sel_valid && in_valid_i[idx]) begin
      sel_oh[idx] = 1'b1;
      sel_data    = in_data_i[idx];
      sel_valid   = 1'b1;
    end
  end
end
```

可以把它理解为“从 `scan_start` 开始排队看一遍”。

例如 `scan_start=1`，循环三次：

| offset | idx 计算 | 实际检查 |
|--------|----------|----------|
| 0 | 1 + 0 = 1 | input1 |
| 1 | 1 + 1 = 2 | input2 |
| 2 | 1 + 2 = 3，回绕成 0 | input0 |

如果 `in_valid_i=3'b101`，也就是 input0 和 input2 有效，扫描顺序 input1、input2、input0 中第一个有效的是 input2，所以选择 input2。

`!sel_valid` 的作用是：一旦已经找到第一路有效输入，就不再允许后面的有效输入覆盖选择结果。

## 11. in_ready_o 生成

输入 ready 逻辑是：

```systemverilog
assign in_ready_o = can_accept ? sel_oh : 3'b000;
```

含义：

| 条件 | `in_ready_o` |
|------|--------------|
| 输出寄存器可以接收 | 等于当前选择结果 `sel_oh` |
| 输出寄存器不能接收 | 强制为 `3'b000` |

因此 `in_ready_o` 有两个性质：

1. 同一周期最多只有一个 bit 为 1。
2. 下游反压时，所有输入都被反压。

## 12. 指针更新逻辑

时序逻辑中指针更新代码是：

```systemverilog
if (out_valid_q && out_ready_i) begin
  arb_ptr_q <= next_idx(grant_idx_q);
end
```

含义是：只有当当前输出拍被下游真正接收后，才认为当前 grant 对应的输入完成了一次服务。

这也是为什么指针不跟 `in_ready_o` 直接绑定。

如果指针在输入接收时就前进，会出现一种歧义：输入虽然被模块接收了，但如果下游一直反压，数据还没真正离开仲裁器。设计上把“服务完成”定义在输出握手点，这样更符合端到端数据流。

## 13. 输出寄存器更新逻辑

输出寄存器只在 `can_accept=1` 时更新：

```systemverilog
if (can_accept) begin
  if (sel_valid) begin
    out_valid_q <= 1'b1;
    out_data_q  <= sel_data;
    grant_q     <= sel_oh;
  end else begin
    out_valid_q <= 1'b0;
    out_data_q  <= '0;
    grant_q     <= 3'b000;
  end
end
```

有三种典型场景。

### 13.1 输出为空，有输入 valid

`out_valid_q=0`，所以 `can_accept=1`。扫描逻辑找到一路 valid 后，把数据装入输出寄存器。下一拍输出 valid 拉高。

### 13.2 输出满，下游 ready

`out_valid_q=1 && out_ready_i=1`，所以 `can_accept=1`。当前输出拍被下游接收，同时模块可以接收下一路输入并写入输出寄存器。

这就是无气泡流水。

### 13.3 输出满，下游不 ready

`out_valid_q=1 && out_ready_i=0`，所以 `can_accept=0`。时序块不更新 `out_valid_q/out_data_q/grant_q`。输出稳定保持，输入 ready 全 0。

这就是反压保护。

## 14. 满载轮询示例

假设复位后 3 路输入一直有效：

```systemverilog
in_valid_i = 3'b111;
out_ready_i = 1'b1;
```

波形抽样结果：

| 时间 | `in_valid` | `in_ready` | `out_valid` | `out_ready` | `grant` | `ptr` | `scan_start` |
|------|------------|------------|-------------|-------------|---------|-------|--------------|
| 190000 | 111 | 001 | 0 | 1 | 000 | 00 | 00 |
| 195000 | 111 | 010 | 1 | 1 | 001 | 00 | 01 |
| 205000 | 111 | 100 | 1 | 1 | 010 | 01 | 10 |
| 215000 | 111 | 001 | 1 | 1 | 100 | 10 | 00 |
| 225000 | 111 | 010 | 1 | 1 | 001 | 00 | 01 |

逐拍解释：

| 时间 | 发生的事 |
|------|----------|
| 190000 | 输出寄存器空，从 input0 开始扫，`in_ready=001` 接收 input0 |
| 195000 | 输出 input0，所以 `grant=001`；同拍准备接收 input1，所以 `in_ready=010` |
| 205000 | 输出 input1，所以 `grant=010`；同拍准备接收 input2，所以 `in_ready=100` |
| 215000 | 输出 input2，所以 `grant=100`；同拍回绕准备接收 input0，所以 `in_ready=001` |

这里最关键的观察是：`grant` 是当前输出来源，`in_ready` 是下一笔输入选择。它们在满载流水下天然错一拍。

## 15. 跳过无效输入示例

复位后指针为 0，如果只有 input2 有效：

```systemverilog
in_valid_i = 3'b100;
```

扫描顺序是 input0、input1、input2。

input0 invalid，跳过。
input1 invalid，跳过。
input2 valid，选择 input2。

所以：

```systemverilog
in_ready_o = 3'b100;
```

输出下一拍：

```systemverilog
grant_o = 3'b100;
out_data_o = in_data_i[2];
```

## 16. 反压示例

当输出已经有效，但下游拉低 ready：

```systemverilog
out_valid_o = 1'b1;
out_ready_i = 1'b0;
```

此时：

```systemverilog
can_accept = 1'b0;
in_ready_o = 3'b000;
```

输出寄存器不更新，表现为：

| 信号 | 行为 |
|------|------|
| `out_valid_o` | 保持 1 |
| `out_data_o` | 保持原数据 |
| `grant_o` | 保持原来源 |
| `in_ready_o` | 全 0 |
| `arb_ptr_q` | 不前进 |

波形抽样中 320000 ps 到 350000 ps 可以看到：

| 时间 | `in_ready` | `out_valid` | `out_ready` | `grant` | `out_data` |
|------|------------|-------------|-------------|---------|------------|
| 320000 | 000 | 1 | 0 | 001 | 000000a5 |
| 350000 | 000 | 1 | 1 | 001 | 000000a5 |

这说明反压期间输出稳定，没有接收新输入。

## 17. ready one-hot 与同拍回填

`in_ready_o` 必须是 one-hot 或 0：

| 合法值 | 含义 |
|--------|------|
| `000` | 本周期不接收输入 |
| `001` | 接收 input0 |
| `010` | 接收 input1 |
| `100` | 接收 input2 |

如果出现 `011`、`101`、`110`、`111`，就表示同周期接受多路输入，这违反 3 选 1 仲裁器定义。

TB 中用 `is_onehot0()` 检查这个约束。

同拍消费/回填场景下需要注意：

| 时间 | `grant` | `in_ready` | 解释 |
|------|---------|------------|------|
| 415000 | 001 | 010 | 当前输出 input0，同时准备接收 input1 |
| 425000 | 010 | 001 | 当前输出 input1，同时从 input2 开始扫；input2 invalid，回绕接收 input0 |

所以看到 `grant != in_ready` 时，不要直接判断错误。先看当前是否 `out_valid_o && out_ready_i` 成立。如果成立，说明这是流水回填行为。

## 18. 为什么不是固定优先级

固定优先级仲裁器通常每次都按 input0、input1、input2 扫描。这样当 input0 长期 valid 时，input1 和 input2 会被饿死。

本设计每次输出成功后都会改变下一次扫描起点：

| 当前服务 | 下一次扫描起点 |
|----------|----------------|
| input0 | input1 |
| input1 | input2 |
| input2 | input0 |

因此三路都持续 valid 时，服务顺序必然是：

```text
input0, input1, input2, input0, input1, input2, ...
```

## 19. 为什么指针在输出握手后更新

这个设计把“服务完成”定义为输出被下游接收。

原因是仲裁器不是只负责从输入拿数据，它还负责把数据交给下游。如果下游反压，数据停在输出寄存器里，服务还没有真正完成。

用输出握手更新指针有两个好处：

1. 指针更新和 `grant_o` 的生命周期一致。
2. 反压期间指针不乱跳，波形更容易解释，验证也更直接。

## 20. TB 覆盖点

当前 TB 把轮询核心逻辑拆成 5 个独立任务：

| TB 任务 | 覆盖点 |
|---------|--------|
| `test_reset_initial_priority` | 复位后从 input0 开始 |
| `test_skip_invalid_inputs` | 扫描时跳过 invalid 输入 |
| `test_full_load_rotation` | 三路满载下 0、1、2 轮转 |
| `test_backpressure_freezes_output` | 反压期间输出保持，输入 ready 全 0 |
| `test_ready_onehot_and_refill` | `in_ready_o` one-hot，同拍消费/回填 |

最终仿真日志显示：

```text
TEST_CASE_PASS[1]: reset_initial_priority_input0
TEST_CASE_PASS[2]: skip_invalid_inputs_from_pointer
TEST_CASE_PASS[3]: full_load_rotation_0_1_2
TEST_CASE_PASS[4]: backpressure_freezes_output_and_ready
TEST_CASE_PASS[5]: ready_onehot_and_consume_refill
TEST_PASS: rr_arbiter_3to1 focused round-robin self-checks passed
```

## 21. 看波形的方法

打开 `sim/dump.fsdb` 后，优先看这些信号：

| 信号 | 观察目的 |
|------|----------|
| `clk_i` | 时钟边沿 |
| `rst_n_i` | 复位释放时间 |
| `in_valid_i[2:0]` | 哪些输入请求服务 |
| `in_ready_o[2:0]` | 本周期接受哪一路输入 |
| `out_valid_o` | 输出是否有效 |
| `out_ready_i` | 下游是否接收 |
| `grant_o[2:0]` | 当前输出来自哪一路 |
| `out_data_o` | 当前输出数据 |
| `dut.arb_ptr_q` | 保存的下一次扫描起点 |
| `dut.scan_start` | 当前组合扫描起点 |
| `dut.sel_oh` | 当前组合选择结果 |
| `dut.can_accept` | 输出寄存器是否允许回填 |

读波形时按这个顺序判断：

1. 看 `out_valid_o && out_ready_i` 是否成立。
2. 如果成立，当前 `grant_o` 对应的输出拍被消费。
3. 再看 `scan_start`，它通常会指向当前 grant 的下一路。
4. 看 `in_valid_i` 中从 `scan_start` 开始第一路有效输入。
5. 这一路会体现在 `in_ready_o` 和 `sel_oh` 上。
6. 下一拍 `grant_o/out_data_o` 变成刚接收的输入。

## 22. 常见误解

### 22.1 误解一：`grant_o` 和 `in_ready_o` 必须一样

不对。`grant_o` 是当前输出来源，`in_ready_o` 是当前输入接收选择。满载流水下，它们经常错开。

### 22.2 误解二：input 被 ready 后，指针立即应该变

本设计不是这样。指针只在输出握手后更新。输入 ready 表示数据进入仲裁器，输出握手表示数据离开仲裁器。

### 22.3 误解三：下游反压时还可以提前接收新输入

本设计只有 1 拍输出寄存器，没有深 FIFO。输出寄存器满且下游不 ready 时，不能接收新输入，否则会覆盖旧数据。

### 22.4 误解四：`arb_ptr_q` 和 `scan_start` 永远一样

不对。在同拍消费/回填场景，`scan_start` 会直接使用当前 grant 的下一路，而不是等待 `arb_ptr_q` 在时钟沿后更新。

## 23. 设计边界

本模块不包含以下功能：

| 功能 | 处理方式 |
|------|----------|
| 深 FIFO | 外部添加 |
| CDC | 外部同步或异步 FIFO |
| CSR 配置 | 使用参数配置 |
| 权重仲裁 | 当前仅 round-robin |
| 错误中断 | 外部协议检查器或系统 watchdog |

## 24. 小结

这个仲裁器可以用一句话概括：

当输出寄存器能接收新数据时，从轮询指针指定的位置开始扫描 3 路输入，选择第一路 valid 输入；当前输出被下游接收后，下一次扫描起点移动到当前 grant 的下一路；下游反压时保持输出并停止接收输入。

实现上最关键的是把三个概念分清：

| 概念 | RTL 信号 | 含义 |
|------|----------|------|
| 当前输出来源 | `grant_o/grant_q` | 下游正在看的数据来自哪一路 |
| 下一次扫描起点 | `arb_ptr_q` | 常规情况下从哪一路开始找 valid |
| 本周期实际扫描起点 | `scan_start` | 考虑同拍输出消费后的真实扫描起点 |

理解这三个信号之间的关系，就能理解整个轮询仲裁器。

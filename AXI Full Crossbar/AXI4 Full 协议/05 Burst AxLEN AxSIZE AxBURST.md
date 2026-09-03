# 05 Burst：AxLEN / AxSIZE / AxBURST


核心含义：主机只发送一次地址，连续传输多笔数据（多拍 beat），整组连续数据合起来叫一次 burst（突发）

AXI4 的 burst 主要由地址通道上的三组字段决定：

```text
AxLEN    -> 这次 burst 有多少个 beat
AxSIZE   -> 每个 beat 的传输粒度是多少 byte
AxBURST  -> 后续 beat 的地址怎么变化
```

| 笔记写法      | 写地址通道真实信号 | 读地址通道真实信号 | 含义             |
| --------- | --------- | --------- | -------------- |
| `AxID`    | `AWID`    | `ARID`    | transaction ID |
| `AxADDR`  | `AWADDR`  | `ARADDR`  | burst 首地址      |
| `AxLEN`   | `AWLEN`   | `ARLEN`   | burst 长度编码     |
| `AxSIZE`  | `AWSIZE`  | `ARSIZE`  | 每个 beat 的大小编码  |
| `AxBURST` | `AWBURST` | `ARBURST` | burst 地址类型     |

## Burst 的基本模型

没有 burst 时，连续访问多个 word 需要多次地址握手：

```text
地址握手 -> 数据
地址握手 -> 数据
地址握手 -> 数据
地址握手 -> 数据
```

有 burst 后，一次地址握手可以描述多个数据 beat：

```text
一次地址握手 -> beat0 -> beat1 -> beat2 -> ... -> last beat
```

写事务里是：

```text
AW 描述 burst
W  发送多个写数据 beat
B  返回整笔写事务的响应
```

读事务里是：

```text
AR 描述 burst
R  返回多个读数据 beat
```

关键点：`AxLEN/AxSIZE/AxBURST` 都在地址通道上；真正的数据在 `W` 或 `R` 通道上。

## AxLEN

`AxLEN` 表示 burst length，但它是编码值，不是直接的 beat 数。

```text
beats = AxLEN + 1
```

例子：

| `AxLEN` | 实际 beat 数 | 常见理解              |
| ------- | --------: | ----------------- |
| `0`     |    1 beat | single transfer   |
| `1`     |   2 beats | 两个数据 beat         |
| `3`     |   4 beats | 很常见的 4-beat burst |
| `15`    |  16 beats | `FIXED/WRAP` 的上限  |
| `255`   | 256 beats | AXI4 `INCR` 的上限   |

AXI4 里 `AWLEN/ARLEN` 是 8-bit，因此编码范围是 `0..255`。

但合法 beat 数还要看 `AxBURST` 类型：

| burst 类型 | AXI4 支持的 beat 数 |
|---|---|
| `FIXED` | 1 到 16 |
| `INCR` | 1 到 256 |
| `WRAP` | 2、4、8、16 |

验证时最容易写错的是把 `AxLEN` 当成 beat 数本身。正确判断最后一个 beat 要用：

```text
last_beat_index = AxLEN
beat_count      = AxLEN + 1
```

因此：

```text
WLAST/RLAST 应该出现在第 AxLEN 个 beat 上
```

这里的 beat index 从 0 开始数。

##  `AxSIZE`

`AxSIZE` 描述每个 beat 的传输大小，单位是 byte。

```text
bytes_per_beat = 2 ^ AxSIZE
```

常见编码：

| `AxSIZE` | 每 beat 字节数 | 等价宽度 |
|---|---:|---|
| `000` | 1 byte | 8-bit |
| `001` | 2 bytes | 16-bit |
| `010` | 4 bytes | 32-bit |
| `011` | 8 bytes | 64-bit |
| `100` | 16 bytes | 128-bit |
| `101` | 32 bytes | 256-bit |
| `110` | 64 bytes | 512-bit |
| `111` | 128 bytes | 1024-bit |

`AxSIZE` 不能超过数据总线宽度。

例子：如果数据总线是 32-bit，也就是 4 bytes：

```text
AWSIZE = 2 -> 2^2 = 4 bytes/beat，刚好占满总线
AWSIZE = 1 -> 2^1 = 2 bytes/beat，narrow transfer
AWSIZE = 0 -> 2^0 = 1 byte/beat，narrow transfer
AWSIZE = 3 -> 2^3 = 8 bytes/beat，非法，因为超过 32-bit 总线
```

注意，`AxSIZE` 不是 `WSTRB`。

```text
AxSIZE 决定这笔 burst 每个 beat 的传输粒度。
WSTRB  决定写数据 beat 中哪些 byte lane 真正被写。
```

所以写通道里真正落到 byte lane 的行为，要结合：

```text
AWADDR + AWSIZE + AWBURST + WSTRB
```

## 4. `AxBURST`

`AxBURST` 决定 burst 中每个 beat 的地址怎么变化。

| `AxBURST` | 类型 | 地址行为 | 典型场景 |
|---|---|---|---|
| `00` | `FIXED` | 每个 beat 地址不变 | FIFO、寄存器窗口 |
| `01` | `INCR` | 地址按 `bytes_per_beat` 递增 | 连续内存访问 |
| `10` | `WRAP` | 地址递增，到边界后回绕 | cache line fill |
| `11` | reserved | 保留 | 不应发出 |
### FIXED

所有 beat 使用同一个地址：

```text
AxADDR = 0x1000
AxLEN  = 3
AxSIZE = 2

地址序列 = 0x1000, 0x1000, 0x1000, 0x1000
```

### INCR

每个 beat 地址按 `bytes_per_beat` 增加：

```text
AxADDR  = 0x1000
AxLEN   = 3
AxSIZE  = 2        -> 4 bytes/beat
AxBURST = INCR

地址序列 = 0x1000, 0x1004, 0x1008, 0x100C
```

### WRAP

`WRAP` 也是递增，但只能在一个固定大小的窗口里递增。到窗口上边界后，地址回到窗口下边界。

先算窗口大小：

```text
beats           = AxLEN + 1
bytes_per_beat  = 2 ^ AxSIZE
wrap_size       = beats × bytes_per_beat
```

再算 wrap 边界：

```text
wrap_base  = floor(AxADDR / wrap_size) × wrap_size
wrap_limit = wrap_base + wrap_size
```

例子：

```text
AxADDR  = 0x1008
AxLEN   = 3        -> 4 beats
AxSIZE  = 2        -> 4 bytes/beat
AxBURST = WRAP

wrap_size  = 4 × 4 = 16 bytes
wrap_base  = 0x1000
wrap_limit = 0x1010

地址序列 = 0x1008, 0x100C, 0x1000, 0x1004
```

`WRAP` 的起始地址必须按 `bytes_per_beat` 对齐，而且 beat 数只能是 2、4、8、16。

## `LEN + SIZE + BURST` 

一次 burst 的“总传输规模”由 `AxLEN` 和 `AxSIZE` 决定：

```text
beats       = AxLEN + 1
bytes/beat  = 2 ^ AxSIZE
total bytes = beats × bytes/beat
```

`AxBURST` 不改变总 byte 数，它只改变这些 beat 对应的地址序列。

同一个例子：

```text
AxADDR = 0x1000
AxLEN  = 3
AxSIZE = 2
```

先得到：

```text
beats       = 4
bytes/beat  = 4
total bytes = 16
```

再看 `AxBURST`：

| `AxBURST` | 地址序列                             |
| --------- | -------------------------------- |
| `FIXED`   | `0x1000, 0x1000, 0x1000, 0x1000` |
| `INCR`    | `0x1000, 0x1004, 0x1008, 0x100C` |
| `WRAP`    | 这个起点刚好在 wrap base，上述参数下和 INCR 一样 |

如果把起点换成 `0x1008`，同样是 4-beat、4-byte/beat 的 `WRAP`：

```text
地址序列 = 0x1008, 0x100C, 0x1000, 0x1004
```

这就是 `WRAP` 比 `INCR` 更容易写错的地方。

## 6. 重要限制

AXI4 burst 里有几条必须记住的约束：

- burst 不能跨 4KB 地址边界。
- `AxSIZE` 指定的 bytes/beat 不能超过数据总线宽度。
- `INCR` burst 支持 1 到 256 个 beat。
- `FIXED` burst 支持 1 到 16 个 beat。
- `WRAP` burst 只支持 2、4、8、16 个 beat。
- `WRAP` burst 起始地址必须按 `bytes_per_beat` 对齐。
- 不能提前终止 burst；发起了多少 beat，就必须完成多少 beat。

4KB 边界检查可以按整笔 burst 的地址范围理解：

```text
start_4kb_page == end_4kb_page
```

其中 `end` 不是简单 `AxADDR + total_bytes` 在所有场景下都能粗暴代替，尤其有 unaligned、narrow、WRAP 时要按协议地址序列和 byte lane 计算。验证环境里更稳妥的做法是逐 beat、逐 byte lane 建模。



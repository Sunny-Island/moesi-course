# 第6讲：Store Buffer 与伪共享（False Sharing）

## 本讲目标

本讲是 MOESI 协议课程的最后一讲。我们将从"写操作延迟"这一实际问题出发，引入 **Store Buffer** 的概念，然后分析它带来的**内存顺序**问题、硬件解决方案（内存屏障），再到软件层面的**伪共享（False Sharing）**问题及其解决方案。最后对本课程六讲内容做一个总结。

---

## 1. 写延迟问题：为什么 S → M 这么慢？

回顾第4讲和第5讲：当一个核心要对一个处于 **Shared (S)** 状态的缓存行执行写操作时：

```
Core0: 想要写地址 A
       ↓
   Cache 检查: 地址 A 的缓存行当前是 S 状态
       ↓
   发起 BusUpgr 请求到总线
       ↓
   ┌─────────────────────────────────────────────┐
   │  等待所有其他核心返回 ACK                     │
   │  (其他核心需要先 invalidate 自己的副本)         │
   │  如果有 4 个核心, 可能要等 4 个 ACK            │
   └─────────────────────────────────────────────┘
       ↓
   收到全部 ACK 后, 状态转换: S → M
       ↓
   现在可以写入数据了
```

**实际延迟是多少？**

在 x86 架构上，一个 `S → M` 转换（BusUpgr + 等待 ACK）可能需要 **30 ~ 50 个时钟周期**，甚至更长（取决于总线争用、其他核心的负载等）。

**问题来了：CPU 流水线不能等这么久！**

```
如果每次写操作都要等 30+ 周期:

  store [x], 1     ← 流水线卡在这里, 等 30 周期
  add  r1, r2      ← 这条指令也被卡住了
  load r3, [y]     ← 这条也被卡住了
  ...

这完全不可接受。现代 CPU 流水线每周期可以发射多条指令。
等待 30 周期 = 浪费了 30 * N 条指令的执行机会。
```

**解决思路**：CPU 不能直接等待缓存完成写操作。需要一种机制让"写操作"看起来是瞬间完成的（从流水线视角），而实际写入缓存的延迟操作在后台完成。

---

## 2. Store Buffer：CPU 流水线的"写操作缓冲"

### 2.1 什么是 Store Buffer？

**Store Buffer（存储缓冲区）** 是一个位于 CPU 流水线和 L1 数据缓存之间的**小型 FIFO 队列**，容量通常为 **4 ~ 12 个条目**（早期处理器），现代处理器（2015年后，如 Intel Skylake 及以后）通常为 **56 ~ 72 个条目**。

```
┌─────────────────────────────────────────────────────────────────┐
│                         CPU Core 0                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    CPU 流水线 (Pipeline)                  │    │
│  │   fetch → decode → execute → memory → writeback         │    │
│  └───────────────────┬─────────────────────────────────────┘    │
│                      │                                          │
│          store [addr, data]                                     │
│                      │                                          │
│                      ▼                                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              STORE BUFFER (FIFO, 4~12 entries)           │    │
│  │  ┌──────────┬──────────┬──────────┬──────────┐          │    │
│  │  │ Entry 0  │ Entry 1  │ Entry 2  │ Entry 3  │  ...     │    │
│  │  │ addr=0xA │ addr=0xB │ addr=0xC │ (empty)  │          │    │
│  │  │ data=42  │ data=7   │ data=99  │          │          │    │
│  │  └──────────┴──────────┴──────────┴──────────┘          │    │
│  └───────────────────┬─────────────────────────────────────┘    │
│                      │ (后台逐条排出, 慢, 不可见)                  │
│                      ▼                                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  L1 Data Cache                           │    │
│  │   需要等待 BusUpgr ACK 才能真正写入                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                      │                                          │
│                      ▼                                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │             MOESI 状态机 / 总线接口                         │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Store Buffer 的工作机制

**写入路径（快速，流水线视角）：**

```
阶段1 (流水线内) :
  store [addr], data
      ↓
  将 {地址, 数据} 写入 Store Buffer（只要 Buffer 不满）
      ↓
  指令立即从流水线中"退休"(retire) —— 对流水线来说, 写操作已完成
      ↓
  流水线继续执行下一条指令（不会被阻塞！）
```

**排出路径（慢，后台完成）：**

```
阶段2 (后台) :
  Store Buffer 按 FIFO 顺序逐条排出
      ↓
  对每条 entry:
    a. 检查该地址的缓存行当前状态
    b. 如果是 S 状态 → 发起 BusUpgr, 等待所有 ACK
    c. ACK 到齐 → 状态变为 M → 将数据写入缓存行
    d. 如果是 E/M 状态 → 直接写入缓存行（无须总线事务）
    e. 处理完毕 → 从 Store Buffer 中移除此 entry
```

**关键点**：
- 流水线写入 Store Buffer 是 **1 个周期**（或极少数周期）。
- Store Buffer 排出到 L1 缓存可能需要 **30+ 周期**，但这发生在后台，流水线不感知。
- 这解决了"流水线停顿"问题——Store Buffer 是异步的"写入缓冲"。

---

## 3. Store Buffer 引入的内存顺序问题

Store Buffer 解决了写延迟问题，但引入了一个**更隐蔽的问题**：内存可见性顺序。

### 3.1 经典案例：两个核心的同步

考虑以下场景——这是**几乎所有"锁无关编程"Bug 的根源**：

```
共享变量 (初始值) :
  int x   = 0;   // 地址 0x1000
  int flag = 0;  // 地址 0x2000

Core 0 (生产者) :              Core 1 (消费者) :
  x = 1;          // S1          while (flag == 0) {}  // L1: 自旋等待
  flag = 1;       // S2          print(x);             // L2: 预期输出 1

直觉: 既然 Core0 先写 x 再写 flag,
      Core1 看到 flag==1 后, x 一定也是 1。
      → 输出应该是 1。

现实: 输出可能是 0！ (!!!)
```

### 3.2 逐步分析：为什么 Core1 看到了 x=0？

让我们按时间线跟踪 MOESI 状态和 Store Buffer 的内容：

```
初始状态:
  x 在 Core0 缓存: S 状态
  x 在 Core1 缓存: S 状态
  flag 在 Core0 缓存: S 状态
  flag 在 Core1 缓存: S 状态

时间线:
═══════════════════════════════════════════════════════════════════════

T1: Core0 执行 S1: x = 1
    ┌─────────────────────────────────────────────┐
    │ Core0 Store Buffer                          │
    │ ┌───────────────────────────┐               │
    │ │ Entry 0: addr=0x1000,    │               │
    │ │         data=1           │               │
    │ └───────────────────────────┘               │
    └─────────────────────────────────────────────┘
    流水线: 指令退休, 继续执行 S2
    后台:   发起 BusUpgr(0x1000), 等待 ACK... (慢!)


T2: Core0 执行 S2: flag = 1
    ┌─────────────────────────────────────────────┐
    │ Core0 Store Buffer                          │
    │ ┌───────────────────────────┐               │
    │ │ Entry 0: addr=0x1000,    │  ← 还在等待ACK │
    │ │         data=1           │               │
    │ ├───────────────────────────┤               │
    │ │ Entry 1: addr=0x2000,    │  ← 新入队      │
    │ │         data=1           │               │
    │ └───────────────────────────┘               │
    └─────────────────────────────────────────────┘


T3: Core0 后台排出 Entry 0 (x=1) —— BusUpgr ACK 终于回来了
    ✓ x 缓存行状态: S → M, 写入 data=1
    ✗ Core1 的 x 缓存行被 Invalidate

    IMPORTANT: Entry 0 排出后, Entry 1 (flag) 排到了队首
    但还要等 BusUpgr(0x2000) 的 ACK...


T4: Core1 执行 L1: while (flag == 0) —— 轮询 flag
    Core1 的 flag 缓存行仍是 S 状态, 值为 0, 继续自旋


T5: Core0 后台排出 Entry 1 (flag=1) —— BusUpgr ACK 回来
    ✓ flag 缓存行状态: S → M, 写入 data=1
    ✗ Core1 的 flag 缓存行被 Invalidate


T6: Core1 再次读 flag（之前的缓存行已失效, 重新从内存/总线获取）
    flag == 1 !!! 跳出循环

    Core1 执行 L2: print(x)
    ┌────────────────────────────────────────────────────────┐
    │ x 的缓存行在 T3 已经被 Core0 从 Core1 中 Invalidate 了   │
    │ Core1 没有 x 的副本 → 从内存读取 → 得到 x=0 (!!! Bug)   │
    │                                                        │
    │ 为什么内存中是 0?                                       │
    │ Core0 的 x=1 可能还在 WriteBack Buffer 中,           │
    │ 也可能已经写回内存。                                     │
    │                                                        │
    │ 但关键不是"内存中的值", 而是:                             │
    │ Core1 看到 flag==1 的时间点,                            │
    │ x 的 Invalidate 消息和 flag 的 Invalidate 消息           │
    │ 到达 Core1 的顺序可能是乱的,                             │
    │ 或者 Core1 在收到 flag 失效前已经读了新 flag,             │
    │ 但还没收到 x 的失效/更新。                               │
    │                                                        │
    │ 结论: Core1 输出了 x=0 —— 违背了直觉!                   │
    └────────────────────────────────────────────────────────┘
```

### 3.3 问题根源总结

```
问题的本质:

  Core0 的 Store Buffer 中, x=1 和 flag=1 的"可见时间"不同。

  x=1 存得早, 但因为 BusUpgr 延迟, 它在 Core1 视角的"可见时刻"
  可能晚于 flag=1 的可见时刻。

  从 Core1 的角度看:
    它看到了"未来的 flag" (flag=1)
    但看到了"过去的 x" (x=0)

  Store Buffer 创造了"时间穿越"的假象。
```

**这不是理论问题，这是真实的 Bug。** 任何使用普通变量做线程间同步的代码，在 x86/ARM 上都是**未定义行为**。它可能在测试时"恰好正常"，然后在生产环境随机崩溃。

---

## 4. 内存屏障（Memory Barrier / Memory Fence）

### 4.1 什么是内存屏障？

**内存屏障**是一条特殊指令，它告诉 CPU：**在此之前的 Store Buffer 必须全部排空，之后才能执行后续的 Store 指令**。

```
没有内存屏障:

  Store Buffer:
  ┌───────┬────────┬────────┬────────┐
  │ Retire│ Entry0 │ Entry1 │ Entry2 │  → 流水线不管, 后台慢慢排
  └───────┴────────┴────────┴────────┘


有内存屏障 (mfence / dmb):

  Store Buffer:
  ┌───────┬────────┬────────────────┐
  │ Retire│ Entry0 │  (不能继续入队)   │
  └───────┴────────┴────────────────┘
                                    ↑
            必须等 Entry0 排出, 才能执行 barrier 之后的 store
            流水线在此停顿, 直到 Store Buffer 排空
```

### 4.2 修复之前的 Bug

```cpp
// ============================================
// 错误版本: 无内存屏障的同步
// ============================================

// 共享变量
int x    = 0;
int flag = 0;

// Thread 0 (生产者)
void producer_broken() {
    x    = 1;   // (A)
    flag = 1;   // (B) 可能在 (A) 的 Store Buffer entry 之前排出
}

// Thread 1 (消费者)
void consumer_broken() {
    while (flag == 0) {}   // 自旋等待
    assert(x == 1);        // 可能失败！x 可能是 0！
}


// ============================================
// 正确版本: 使用 std::atomic 和 memory_order
// ============================================

#include <atomic>

std::atomic<int> x(0);
std::atomic<int> flag(0);

// Thread 0 (生产者)
void producer_correct() {
    x.store(1, std::memory_order_release);      // (A)
    // ↑ release: 保证 (A) 之前的所有写操作对后续可见
    flag.store(1, std::memory_order_release);   // (B)
    // 或者更简单地, 直接用 seq_cst
}

// Thread 1 (消费者)
void consumer_correct() {
    while (flag.load(std::memory_order_acquire) == 0) {}
    // ↑ acquire: 保证 (B) 之后的读操作能看到 (A) 的结果
    int val = x.load(std::memory_order_acquire);
    assert(val == 1);   // 一定成功！
}


// ============================================
// 最简洁版本: seq_cst (顺序一致性)
// ============================================

std::atomic<int> x(0);
std::atomic<int> flag(0);

void producer_seq_cst() {
    x.store(1);              // 默认 memory_order_seq_cst
    flag.store(1);           // 保证全局顺序一致性
}

void consumer_seq_cst() {
    while (flag.load() == 0) {}
    int val = x.load();
    assert(val == 1);        // 一定成功
}
```

### 4.3 内存顺序语义速查

| 内存顺序 | 含义 | 性能开销 |
|----------|------|----------|
| `memory_order_relaxed` | 无屏障，仅保证原子性 | 最低 |
| `memory_order_acquire` | 确保此操作之后的读写不会被重排到此操作之前 | 低 |
| `memory_order_release` | 确保此操作之前的读写不会被重排到此操作之后 | 低 |
| `memory_order_acq_rel` | acquire + release 的组合 | 中 |
| `memory_order_seq_cst` | 全局顺序一致性，最强保证 | 最高 |

```cpp
// memory_order 与硬件指令的关系 (x86):
//
// seq_cst store → mfence (或 xchg 隐式锁总线)
// release store → 通常不需要额外指令 (x86 已有较强内存模型)
// acquire load  → 通常不需要额外指令
// seq_cst load  → 通常不需要额外指令
//
// ARM 上则几乎都需要显式屏障指令 (dmb ish / dmb st)
```

### 4.4 工作原理（硬件视角）

```
Core0 执行: x.store(1, memory_order_release);

  CPU 流水线:
    1. 将 {addr=x, data=1} 写入 Store Buffer
    2. 插入"release fence"标记
    3. 标记之后的 store 指令必须等标记之前的 Store Buffer 全部排空

  Store Buffer:
    ┌───────────┬────────────────┐
    │ Entry: x=1│ Release Fence  │ ← 新来的 store 必须等 x=1 排出
    └───────────┴────────────────┘
                        ↓
    x=1 排出到 L1 缓存 (S→M via BusUpgr)
                        ↓
    现在 flag=1 才能进入 Store Buffer (或才能开始 BusUpgr)
```

---

## 5. 伪共享（False Sharing）

### 5.1 问题描述

**伪共享**是两个（或多个）不同的变量被放置在同一个缓存行（Cache Line，64 字节）中，不同核心分别修改这些**互不相关的变量**，却因为缓存一致性协议而不断互相 Invalid 对方缓存行的问题。

```
"False" 的含义: 它们不是"真正的共享数据"——
Core0 永远只读/写 counter_a, Core1 永远只读/写 counter_b，
它们之间没有任何逻辑上的关联。
但因为它们在同一个缓存行中，
MOESI 协议会"误以为"它们在共享和争用同一份数据。
```

### 5.2 详细图解

```
假设缓存行大小为 64 字节 (x86 标准)

一个缓存行:
┌──────────────────────────────────────────────────────────────────┬──┐
│                         64 bytes (地址 0x1000 ~ 0x103F)            │..│
│  ┌──────────────┬──────────────┬─────────────────────────────────┐ │  │
│  │ counter_a    │ counter_b    │    其他数据 (padding/other)       │ │  │
│  │ 8 bytes      │ 8 bytes      │           48 bytes               │ │  │
│  │ (0x1000)     │ (0x1008)     │                                  │ │  │
│  └──────────────┴──────────────┴─────────────────────────────────┘ │  │
└──────────────────────────────────────────────────────────────────┴──┘

Core0 只修改 counter_a
Core1 只修改 counter_b
```

**伪共享的 Ping-Pong 过程：**

```
初始: 两个核心的缓存行都是 S 状态


时间线 (每个核心都在循环中递增自己的 counter):

┌─────────────────────────────────────────────────────────────────────┐
│ T1: Core0 要写 counter_a                                           │
│     counter_a 在 Core0 缓存: S → BusUpgr → M                        │
│     Core1 的这个缓存行 → Invalidate (I)                             │
│     Core0 写入 counter_a = 1                                       │
├─────────────────────────────────────────────────────────────────────┤
│ T2: Core1 要写 counter_b                                           │
│     counter_b 在 Core1: I → 缓存缺失! → BusRdX (或类似) 获取 M        │
│     Core0 的这个缓存行 → Invalidate (I)  ← Core0 刚刚才拿到 M!        │
│     Core1 写入 counter_b = 1                                       │
├─────────────────────────────────────────────────────────────────────┤
│ T3: Core0 要写 counter_a (下一轮循环)                                │
│     Core0: I → 缓存缺失 → BusRdX → M                                │
│     Core1 → Invalidate                                              │
│     Core0 写入 counter_a = 2                                       │
├─────────────────────────────────────────────────────────────────────┤
│ T4: Core1 要写 counter_b (下一轮循环)                                │
│     Core1: I → 缓存缺失 → BusRdX → M                                │
│     Core0 → Invalidate                                              │
│     Core1 写入 counter_b = 2                                       │
├─────────────────────────────────────────────────────────────────────┤
│ T5, T6, T7, ... 无限 Ping-Pong                                     │
│                                                                     │
│     核心0: M──I──M──I──M──I──M──I── ...                            │
│     核心1: I──M──I──M──I──M──I──M── ...                            │
│                                                                     │
│     每次都需要 BusRdX (读 + 获取独占权) → 几十个周期的开销            │
│     两个核心完全无法并行写入！                                       │
│     counter_a 和 counter_b 是独立的变量，却被缓存协议串行化了。       │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.3 伪共享的性能影响

```
场景: 两个核心, 每个递增自己的计数器 10,000,000 次

┌─────────────────────────┬──────────────┬──────────────┐
│ 布局方式                  │ 耗时 (ms)    │ 吞吐量        │
├─────────────────────────┼──────────────┼──────────────┤
│ 同一缓存行 (伪共享)       │    ~800 ms   │  极慢          │
│ 不同缓存行 (有 padding)   │    ~80 ms    │  ~10x 提升     │
│ 单核执行                  │    ~40 ms    │  基线          │
└─────────────────────────┴──────────────┴──────────────┘

说明:
  - 伪共享版本比有 padding 的版本慢 10 倍左右
  - 两个核心"并行"执行, 实际效果接近单核
  - 绝大部分时间浪费在 BusRdX 事务和等待缓存行失效上
  - 如果核心数更多 (4核、8核), 伪共享的损失更大
```

---

## 6. 伪共享的解决方案：缓存行对齐（Padding）

### 6.1 基本原理

确保两个被不同核心独立修改的变量**落在不同的缓存行**中。最简单的方法是填充无用的字节，把变量"推开"。

```
修复前: 两个变量挤在同一个缓存行
┌──────────────────────────────────────────────────────┐
│ │ counter_a (8B) │ counter_b (8B) │ other...         │  ← 64B
└──────────────────────────────────────────────────────┘
     ↑ 同一行 → 互相 Invalid


修复后: 每个变量独占一个缓存行
┌──────────────────────────────────────────────────────┐
│ │ counter_a (8B) │ padding (56B)                     │  ← 64B (行0)
└──────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────┐
│ │ counter_b (8B) │ padding (56B)                     │  ← 64B (行1)
└──────────────────────────────────────────────────────┘
     ↑ 不同行 → 互不影响
```

### 6.2 C++ 代码示例

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <atomic>

// =====================================================
// 错误版本: 伪共享
// =====================================================
struct BadCounters {
    uint64_t counter_a;  // 偏移 0
    uint64_t counter_b;  // 偏移 8  ← 和 counter_a 同一缓存行!
};

void bench_bad() {
    BadCounters counters = {0, 0};

    const int ITERS = 10'000'000;

    auto t1 = std::thread([&]() {
        for (int i = 0; i < ITERS; i++) {
            counters.counter_a++;
        }
    });

    auto t2 = std::thread([&]() {
        for (int i = 0; i < ITERS; i++) {
            counters.counter_b++;
        }
    });

    t1.join();
    t2.join();

    std::cout << "Bad:  a=" << counters.counter_a
              << " b=" << counters.counter_b << std::endl;
}


// =====================================================
// 正确版本: 缓存行对齐 (消除伪共享)
// =====================================================

// 方法1: 使用 C++17 alignas
struct alignas(64) PaddedCounter {
    uint64_t value;
    // alignas(64) 保证整个对象起始地址是 64 字节对齐
    // 如果 value 是唯一的成员, 这个对象独占一个缓存行
};

struct GoodCounters_alignas {
    alignas(64) uint64_t counter_a;  // 独占缓存行 0
    alignas(64) uint64_t counter_b;  // 独占缓存行 1
};

// 方法2: 手动 padding
struct GoodCounters_manual {
    uint64_t counter_a;
    char     _pad1[64 - sizeof(uint64_t)];  // 填到 64 字节
    uint64_t counter_b;
    char     _pad2[64 - sizeof(uint64_t)];  // 填到 64 字节
};

// 方法3: 使用编译器/平台常量
constexpr size_t CACHE_LINE_SIZE = 64;  // x86_64

struct GoodCounters_portable {
    alignas(CACHE_LINE_SIZE) uint64_t counter_a;
    alignas(CACHE_LINE_SIZE) uint64_t counter_b;
};

// 方法4: C++17 hardware_destructive_interference_size (标准但实现不一)
#ifdef __cpp_lib_hardware_interference_size
    #include <new>
    struct GoodCounters_cpp17 {
        alignas(std::hardware_destructive_interference_size) uint64_t counter_a;
        alignas(std::hardware_destructive_interference_size) uint64_t counter_b;
    };
    // 注意: 该常量在 C++17 标准中定义, 但编译器支持不一致
    // GCC/Clang 需要 -std=c++17, MSVC 在某些版本中可能不支持
#else
    // 回退方案: 直接使用 64 字节对齐
    struct GoodCounters_cpp17 {
        alignas(64) uint64_t counter_a;
        alignas(64) uint64_t counter_b;
    };
#endif

void bench_good() {
    GoodCounters_alignas counters = {0, 0};

    const int ITERS = 10'000'000;

    auto t1 = std::thread([&]() {
        for (int i = 0; i < ITERS; i++) {
            counters.counter_a++;
        }
    });

    auto t2 = std::thread([&]() {
        for (int i = 0; i < ITERS; i++) {
            counters.counter_b++;
        }
    });

    t1.join();
    t2.join();

    std::cout << "Good: a=" << counters.counter_a
              << " b=" << counters.counter_b << std::endl;
}
```

### 6.3 哪些场景需要注意伪共享？

```
常见易出问题的场景:

1. 线程池中的 per-thread 统计数据
   struct ThreadStats {
       uint64_t tasks_completed;
       uint64_t cache_misses;
       uint64_t branch_mispredicts;
   };
   ThreadStats stats[MAX_THREADS];
   // ↑ 如果 ThreadStats 小于 64B, 相邻线程的 stats 会共享缓存行

2. 无锁队列中相邻的 head/tail 指针
   struct LockFreeQueue {
       std::atomic<size_t> head;   // 生产者写 head
       std::atomic<size_t> tail;   // 消费者写 tail
       // ↑ 同一缓存行 → 伪共享
   };

3. 内存分配器中的 per-thread 空闲链表
   struct ThreadCache {
       FreeList* free_lists[NUM_CLASSES];
       uint64_t   allocated_bytes;
       uint64_t   freed_bytes;
   };

4. 自旋锁 (spinlock) 数组中的相邻锁
   // 高并发下, 相邻锁会互相使对方缓存行失效
```

---

## 7. 现实启示

### 7.1 为什么 MESI/MOESI 是理解 std::atomic 的前提

```
高级语言 (C++):                编译器 & CPU:
────────────────────────────────────────────────────
int x = 0;                     │  普通变量
x = 1;                         │  → 可能被优化掉
                               │  → 可能乱序执行
                               │  → Store Buffer 使其不可见

std::atomic<int> x(0);         │  原子变量
x.store(1);                    │  → 编译器不会优化掉
                               │  → 插入内存屏障
                               │  → Store Buffer 被排空
                               │  → 其他核心可见

没有 MESI/MOESI 的理解, std::atomic 只是一堆"魔法关键字"。
理解了 MESI/MOESI, 你才知道"为什么需要 acquire/release"，
"为什么 seq_cst 更贵", "为什么 ARM 上 acquire 需要屏障而 x86 不需要"。
```

### 7.2 为什么锁无关编程（Lock-Free Programming）如此困难

```
用锁 (Mutex) 编程:
  lock(mutex);
  x = 1;
  flag = 1;
  unlock(mutex);
  → 锁的 acquire/release 自动插入内存屏障
  → 程序员不需要关心 Store Buffer 和缓存一致性
  → 安全但可能较慢

锁无关编程:
  // 类似我们用 std::atomic 实现一个无锁栈
  std::atomic<Node*> head;

  void push(Node* node) {
      node->next = head.load(memory_order_acquire);
      while (!head.compare_exchange_weak(
                 node->next, node,
                 memory_order_release,  // 成功
                 memory_order_acquire   // 失败
             )) {}
  }

  → 每一步都需要正确选择 memory_order
  → 选错了 → 程序"几乎总是正常", 但偶尔崩溃
  → 调试极其困难 (崩溃不可复现)
  → 不同架构上行为不同 (x86 较宽松, ARM 更严格)

  这就是为什么"Lock-Free"被视为专家领域。
  你需要同时理解:
    - MESI/MOESI (本课程)
    - Store Buffer (本讲)
    - 内存顺序模型 (本讲)
    - CPU 的乱序执行和推测执行
    - 编译器的指令重排
```

### 7.3 为什么缓存行对齐在高性能代码中如此重要

```
高性能系统中的"隐藏成本":

  场景: 高性能 Key-Value 存储 (如 Redis, RocksDB)
  → per-thread 计数器、统计信息如果没有缓存行对齐,
    看似"无锁", 实际被伪共享串行化
  → 32 核机器实际只能跑出 ~2 核的吞吐量
  → 调试工具 (perf) 会显示大量 cache-misses 但很难定位原因

  场景: 高频交易系统
  → 每条消息延迟要求 < 1 微秒
  → 伪共享带来的 50ns 延迟抖动 = 灾难
  → 必须确保关键数据结构独占缓存行

  经验法则:
  1. 每个线程独立修改的数据 → 放在独立的缓存行
  2. 读多写少的共享数据 → 允许共享（读取不会 invalidate）
  3. 使用 alignas(64) 是性能关键路径上最便宜的"保险"
  4. 用 Linux perf 工具检查 cache-misses 和 cache-line bouncing
```

---

## 8. 课程总结：MOESI 协议六讲回顾

> 从第一讲的缓存层次与局部性原理开始，我们建立了多级缓存体系（L1/L2/L3）的认知基础。第二讲深入到窥探总线与 MOESI 状态机，理解了每个缓存行在五个状态（Modified、Owned、Exclusive、Shared、Invalid）间的转换规则。第三讲探讨了 Snoop Filter 和目录协议为什么在高核心数下取代了总线窥探。第四讲和第五讲通过 AMD64 MOESI 的完整状态转移表，剖析了诸如 `RdBlk` + `RdBlkMod` → `MO` 竞争转换这类面试热点。本讲作为收官，回到软件工程师最关心的问题：**Store Buffer** 告诉我们为什么 `std::atomic` 不是多余的语法糖；**伪共享**提醒我们即使在无锁的数据结构上，硬件的缓存一致性协议仍然主宰着性能。六讲的线索是：**从晶体管的物理距离到 C++ 的 memory_order，中间隔着的并非魔法，而是一层层精密的硬件协议。** 理解它们，是写出正确且高效并发代码的唯一途径。

---

## 9. 思考题

**Q1：Store Buffer 解决了什么问题？又引入了什么问题？**

<details>
<summary>点击查看答案</summary>

**解决的问题**：写操作的延迟。当缓存行处于 S（Shared）状态时，写操作需要发起 BusUpgr 并等待所有核心返回 ACK，这可能需要 30+ 个周期。如果 CPU 流水线必须等待这个操作完成，性能将严重受损。Store Buffer 允许写操作"瞬间"完成（从流水线视角），指令立即退休，实际写入缓存在后台异步完成。

**引入的问题**：内存顺序（Memory Ordering）问题。Store Buffer 中的写入对其他核心并非立即可见。一个核心可能先看到"后写入"的变量（flag=1），却还看不到"先写入"的变量（x=1），因为 x=1 还在 Store Buffer 中等待排出。

</details>

**Q2：`std::memory_order_seq_cst` 是如何解决 Store Buffer 可见性问题的？**

<details>
<summary>点击查看答案</summary>

`memory_order_seq_cst`（顺序一致性）提供最强的内存顺序保证。在硬件层面：

1. **seq_cst store**：在写入操作后插入**全屏障**（如 x86 的 `mfence` 或带锁的 `xchg` 指令）。这条屏障强制 Store Buffer 中所有之前的 entry 必须全部排空，后续 store 才能执行。

2. **seq_cst load**：在某些平台（如 ARM）上会插入屏障，确保读取操作能看到其他核心 seq_cst store 的最新值。

3. **全局顺序**：seq_cst 还保证所有核心观察到的 seq_cst 操作的全局顺序是一致的。这是其他 memory_order 不提供的保证。

4. 回到 Store Buffer 的问题：seq_cst 保证了 `x.store(1)` 对 Store Buffer 的写入在 `flag.store(1)` 之前完成，从其他核心的视角，看到 `flag=1` 时，`x=1` 一定也已经可见了。

</details>

**Q3：在伪共享场景中，两个核心的数据是否真的"共享"了？**

<details>
<summary>点击查看答案</summary>

**不是真正的共享。** 两个核心各自操作的是互不相关的独立变量（如 Core0 只读写 counter_a，Core1 只读写 counter_b）。它们之间没有任何逻辑上的数据依赖或通信关系。

但"物理上"，这两个变量恰好位于同一个 64 字节的缓存行中。MOESI 协议以**缓存行**为一致性单位——它无法区分同一缓存行内的不同偏移量。当 Core0 修改 counter_a 时，整个缓存行（包括 counter_b 的部分）都进入 M 状态，Core1 的副本被 Invalidate。反之亦然。

这就是"伪"（False）的含义——缓存一致性协议"误以为"有共享数据争用，实际上没有。这是一个硬件层面的"误报"。

</details>

**Q4：在 x86 平台上，防止伪共享所需的最小对齐是多少？为什么？**

<details>
<summary>点击查看答案</summary>

**最小对齐：64 字节。**

**原因**：x86 架构的缓存行大小是 64 字节。MOESI/MESI 协议以缓存行为最小单位进行状态跟踪和一致性维护。如果两个变量落在不同的缓存行中，它们的状态转换是完全独立的——一个核心对缓存行 A 的修改不会影响缓存行 B。

换一个角度看：如果有两个线程各自频繁修改的变量，只要确保它们之间的地址差 >= 64 字节，它们就不会落入同一缓存行，从而消除伪共享。

在实践中：
- 使用 `alignas(64)` 确保变量起始地址是 64 字节对齐的
- 或者在两个变量之间填充足够的字节（`64 - sizeof(variable)`）
- ARM 的某些处理器缓存行可能更大（128 字节），但 64 字节对齐在大多数平台上已经足够
- C++17 提供了 `std::hardware_destructive_interference_size` 常量，但各编译器实现不一，保守做法是直接使用 64

</details>

**Q5：如果 Core0 和 Core1 反复向同一个缓存行的不同偏移位置写入数据，MOESI 状态机会经历哪些状态转换？画出这个过程。**

<details>
<summary>点击查看答案</summary>

这是伪共享的 MOESI 视角：

```
初始状态: 两个核心的缓存行都在 S (Shared) 状态
   Core0: S        Core1: S

Cycle 1: Core0 写 counter_a
   Core0: S → BusUpgr → M       (获取独占写权限)
   Core1: S → I                 (被 Invalidate)
   结果: Core0: M, Core1: I

Cycle 2: Core1 写 counter_b
   Core1: I → BusRdX → M        (缓存缺失, 从 Core0 获取脏数据 + 独占权)
   Core0: M → I                 (被 Invalidate, 脏数据写回 Core1)
   结果: Core0: I, Core1: M

Cycle 3: Core0 写 counter_a (下一轮)
   Core0: I → BusRdX → M        (缓存缺失, 从 Core1 获取脏数据 + 独占权)
   Core1: M → I                 (被 Invalidate, 脏数据写回 Core0)
   结果: Core0: M, Core1: I

Cycle 4: Core1 写 counter_b (下一轮)
   Core1: I → BusRdX → M
   Core0: M → I
   结果: Core0: I, Core1: M

...无限循环...

状态流转:
   核心0: S → M → I → M → I → M → I → ...
   核心1: S → I → M → I → M → I → M → ...

每次"方向切换"都需要一次 BusRdX 事务。
BusRdX 的延迟通常在 30~80 个周期。
在一个 3GHz 的 CPU 上, 60 个周期 = 20ns。

如果两个核心连续递增各自计数器 10^7 次,
20ns × 10^7 = 200ms 的开销仅仅来自总线事务。
加上流水线停顿和缓存缺失的级联效应, 实际延迟远不止于此。

对比无伪共享的情况:
   核心0: S → M (只一次), 之后始终 M (计数器独占)
   核心1: S → M (只一次), 之后始终 M
   总共只有 2 次状态转换, 之后各自独立运行。
```

</details>

---

*本讲是 MOESI 协议课程第六讲（最后一讲）。全课程到此结束。*

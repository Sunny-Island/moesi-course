# 第四讲：MOESI 协议 — 从 MESI 到 MOESI 的进化

---

## 目录

1. [为什么需要 MOESI — MESI 的痛点](#1-为什么需要-moesi--mesi-的痛点)
2. [Owned 状态的定义](#2-owned-状态的定义)
3. [M→O 转换详解 — MOESI 的灵魂](#3-mo-转换详解--moesi-的灵魂)
4. [O 状态的行为规则](#4-o-状态的行为规则)
5. [完整 MOESI 状态转换表](#5-完整-moesi-状态转换表)
6. [MOESI vs MESI 对比](#6-moesi-vs-mesi-对比)
7. [C++ 仿真代码](#7-c-仿真代码)
8. [面试自测题](#8-面试自测题)

---

## 1. 为什么需要 MOESI — MESI 的痛点

### 1.1 MESI 的 M→S 强制回写问题

回顾 MESI 协议中的一个关键场景：

**场景设定**：设想一个多核系统中的一块共享内存区域，用于线程间通信。Core0 持有数据在 Modified 状态，频繁修改；Core1 需要读取这块数据。

在 MESI 协议下，当 Core0 处于 M 状态，Core1 发起 BusRd（总线读请求）时，发生的动作序列是：

```
时刻 T0: Core0 = M (dirty, 内存中的值已过期)
时刻 T1: Core1 发送 BusRd
时刻 T2: Core0 嗅探到 BusRd
          → 将脏数据写回内存 (Write-Back)
          → Core0: M → S
          → Core1: I → S
时刻 T3: 内存现在是最新的
```

**核心浪费**：M→S 转换迫使 Core0 将脏数据写回内存，但此时内存并非真正的"消费者"——Core1 才是。如果数据可以从 Core0 的缓存直接传递给 Core1（cache-to-cache transfer），则内存写入完全是多余的。这就是著名的 **"无谓写回"（unnecessary write-back）** 问题。

### 1.2 具体示例量化浪费

考虑一个生产者-消费者场景：

```
内存地址 0x1000：共享计数器

Core0（生产者）：
  - 循环 1000 次，每次对 [0x1000] 执行原子加 1
  - 期间一直持有 M 状态（独占修改）

Core1（消费者）：
  - 每次 Core0 完成一批 100 次更新后读取 [0x1000] 的值
  - 共读取 10 次
```

**MESI 下的总线流量**：

| 事件 | 总线事务 | 说明 |
|------|---------|------|
| Core1 第 1 次读 | Write-Back + BusRd | Core0 M→S，强制回写内存 |
| Core0 再次修改 | BusRdX | Core0 S→M，需再次获取所有权 |
| Core1 第 2 次读 | Write-Back + BusRd | 又一次强制回写 |
| ... | ... | 每次 Core1 读都触发回写 |
| **合计** | **10 次 Write-Back + 10 次 BusRd + 9 次 BusRdX** | 大量冗余总线事务 |

**MOESI 下的总线流量**：

| 事件 | 总线事务 | 说明 |
|------|---------|------|
| Core1 第 1 次读 | BusRd + FlushOpt | Core0 M→O，数据直接供给 Core1，不写内存 |
| Core0 再次修改 | BusUpgr | Core0 O→M，仅失效其他副本，无需从内存读 |
| Core1 第 2 次读 | BusRd + FlushOpt | Core0 再次 M→O，数据供给，不写内存 |
| ... | ... | 始终不写内存 |
| **合计** | **10 次 BusRd + 10 次 FlushOpt + 9 次 BusUpgr** | **0 次内存写入** |

**结论**：MOESI 将 MESI 中 10 次不必要的内存写入全部消除。在多核系统中，这等同于节省了 10 次内存总线往返延迟（每次约 100-300 个 CPU 周期），同时大幅降低了内存带宽压力。

### 1.3 为什么 MESI 存在这个问题

MESI 的根本设计哲学是：**只有一个干净的共享副本集合，或者一个脏的独占副本。** "脏"和"共享"在 MESI 中是互斥的——一个缓存行不能既是脏的又被共享。因此当数据需要被共享时，必须先通过写回内存来"洗净"脏数据，然后所有核心以 S 状态持有干净的副本。

MOESI 打破了这一限制，引入了 **"脏且共享"** 的概念——这正是 Owned 状态。

---

## 2. Owned 状态的定义

### 2.1 O 状态的定义

**Owned 状态** 是 MOESI 协议中最重要的创新，其定义为：

> **Owned (O)**：当前缓存行是脏的（与内存不一致），被多个核心共享，且当前核心**负责**在总线请求时向其他核心提供数据。同一时刻只有一个核心可以处于 O 状态。

三个关键属性：
1. **Dirty** — 缓存行数据与内存不一致
2. **Shared** — 其他核心可能持有 S 状态的副本
3. **Responsible** — 当其他核心发送 BusRd 时，由 O 状态的核心（而非内存）负责提供数据

### 2.2 五状态对比表

| 维度 | **M (Modified)** | **O (Owned)** | **E (Exclusive)** | **S (Shared)** | **I (Invalid)** |
|------|:---:|:---:|:---:|:---:|:---:|
| **Dirty?** | 是 | 是 | 否 | 否 | — |
| **Exclusive?** | 是（独占） | 否（共享） | 是（独占） | 否（共享） | — |
| **与内存一致?** | 否 | 否 | 是 | 是 | — |
| **其他核心有副本?** | 否 | 是 | 否 | 可能 | — |
| **谁负责供给数据?** | 本核心 | **本核心** | 本核心或内存 | 内存 | — |
| **写入是否需要总线事务?** | 否（静默写入） | 是（BusUpgr） | 否（静默写入） | 是（BusUpgr） | — |
| **Evict 时是否写回?** | 是（必须写回） | 是（必须写回） | 否 | 否 | — |

### 2.3 关键洞察：M vs O

M 和 O 的唯一区别在于**是否有其他核心持有副本**：

- **M**：脏 + 独占 → 写入无需总线事务（最"强"的状态）
- **O**：脏 + 共享 → 写入前需要 BusUpgr 失效其他副本

这种设计让脏数据可以留在缓存中，直到它被真正地逐出（Evict）——这可能永远不会发生。

### 2.4 所有权模型

MOESI 引入了一种"所有权"（ownership）的概念：

```
         M ──→  O ──→  I
         ↑      ↓
         └──────┘
       (BusUpgr)

所有权流转路径：M → O → (最终 Evict 时写回内存)
```

- M 和 O 状态的核心都"拥有"这行缓存
- 当 O 状态的核心被逐出时，才最终将数据写回内存
- 这实现了**延迟写回（lazy write-back）**——尽可能推迟内存写入，直到真正必要

---

## 3. M→O 转换详解 — MOESI 的灵魂

### 3.1 场景描述

**初始条件**：
- Core0 持有缓存行 X 在 **M** 状态（dirty, exclusive）
- Core1、Core2、Core3 持有 X 在 **I** 状态（invalid）
- 内存中的 X 值是**过期的**（stale）

### 3.2 第一步：Core1 发起 BusRd

当 Core1 尝试读取 X：

```
        Core0(M)              Core1(I)              Core2(I)              内存(stale)
           |                     |                     |                      |
           |                     |--- BusRd -----------|----------------------|--→
           |                     |                     |                      |
           |←---- 嗅探 BusRd ----|                     |                      |
           |                     |                     |                      |
           |  判断: 我持有 M     |                     |                      |
           |  → 将脏数据放置     |                     |                      |
           |    到总线上         |                     |                      |
           |                     |                     |                      |
           |--- FlushOpt -------|--→                  |                      |
           |   (脏数据+地址)     |                     |                      |
           |                     |                     |                      |
           |  状态转换:          |                     |                      |
           |  M → O             |  I → S              |                      |
           |  (仍持有脏数据)     |  (拿到干净副本)     |                      |
           |  (负责任供给)       |                     |                      |
           |                     |                     |                      |
           |          内存 NOT written!               |                      |
           |          内存仍然 stale                   |                      |
```

**关键点**：
- **FlushOpt (Flush Optional)** 是一种特殊的总线消息，表示"这里有脏数据，但收到的人不需要写回内存"——它是 cache-to-cache 的数据传递
- Core0 仍然是脏的（dirty），缓存行的 Dirty Bit 保持设置
- **内存从未被更新**——这是 MOESI 的核心优化

### 3.3 第二步：Core2 也发起 BusRd

此时，Core0 持有 O，Core1 持有 S，Core2 发起读请求：

```
        Core0(O)              Core1(S)              Core2(I)              内存(stale)
           |                     |                     |                      |
           |                     |                     |--- BusRd ------------|--→
           |                     |                     |                      |
           |←---- 嗅探 BusRd ----|←---- 嗅探 BusRd ----|                      |
           |                     |                     |                      |
           |  判断: 我持有 O     |  判断: 我持有 S     |                      |
           |  → 我是 Owner,     |  → 我不负责供给     |                      |
           |    我来供给数据     |    (在 MOESI 中      |                      |
           |                     |     S 不供给数据)    |                      |
           |                     |                     |                      |
           |--- FlushOpt -------|---------------------|--→                   |
           |                     |                     |                      |
           |  状态: O → O       |  状态: S → S       |  I → S               |
           |  (依然是 Owner)    |                     |                      |
```

**关键点**：
- O 状态的 Core0 可以**无限次**为后续的读取者提供数据，每次都无需写回内存
- 在 AMD 的 MOESI 实现中，S 状态的核心不负责供给数据——只有 O 状态的核心负责
- 这就是 MOESI 的"多读"优化：一次写回都不需要，数据一直在 O 状态核心的缓存中活着

### 3.4 第三步（MESI 对比视角）：Core0 再次修改

```
        Core0(O)              Core1(S)              Core2(S)
           |                     |                     |
           |  PrWr (本地写请求)   |                     |
           |                     |                     |
           |--- BusUpgr ---------|---------------------|--→
           |  (升级请求: 失效     |                     |
           |   所有其他 S 副本)   |                     |
           |                     |                     |
           |                     |←-- 收到 BusUpgr -----|←-- 收到 BusUpgr
           |                     |  S → I              |  S → I
           |                     |                     |
           |  状态: O → M       |                     |
           |  (回到独占脏状态)    |                     |
```

**与 MESI 的对比**：

| 步骤 | MESI 做法 | MOESI 做法 |
|------|----------|-----------|
| Core1 读 | Core0 M→S，**先写回内存** | Core0 M→O，**不写回内存** |
| Core2 读 | 内存供给数据 | Core0 (O) 供给数据 |
| Core0 写 | S→M，需 BusRdX 从内存读 | O→M，只需 BusUpgr（不需要读数据） |

MOESI 节省了一次内存写入 + 一次内存读取，总共省去了两次内存总线往返。

### 3.5 完整的 M→O→M 循环时序图

下面用 ASCII 时序图展示完整的生产者-消费者循环：

```
时间 ──────────────────────────────────────────────────────────────────────→

Core0:  ----[M]--------[M→O]--------[O]--------[O→M]--------[M]--------
              |           |           |           |           |
Core1:  ----[I]--------[I→S]-------[S]--------[S→I]-------[I]--------
                           |                       |
Core2:  ----[I]--------[I]--------[I→S]-------[S]--------[S→I]------
                                       |
内存:   ================ STALE =======================================
        ↑           ↑           ↑           ↑           ↑
事件:   初始状态    Core1      Core2      Core0       (无内存
                  BusRd       BusRd      PrWr        写入)
                  Core0      Core0      BusUpgr
                  FlushOpt   FlushOpt

关键观察: 整个过程中内存从未被写入！
```

---

## 4. O 状态的行为规则

理解 O 状态在各种触发条件下的行为，是掌握 MOESI 协议的关键。

### 4.1 O + PrRd（本地读）

**规则**：O → O（保持不变）

```
当前: Core0 = O (dirty, shared, owner)
触发: Core0 执行 load 指令读取该缓存行
结果: 状态保持 O，无总线事务
原因: 本地读取不需要改变任何状态——数据已经在缓存中，其他核心的 S 副本也不受影响
```

### 4.2 O + PrWr（本地写）

**规则**：O → M（升级为 Modified）

```
当前: Core0 = O (dirty, shared, owner)，Core1 = S, Core2 = S
触发: Core0 执行 store 指令写入该缓存行
动作:
  1. Core0 在总线上发送 BusUpgr 消息
  2. Core1 嗅探到 BusUpgr → S → I（失效本地副本）
  3. Core2 嗅探到 BusUpgr → S → I（失效本地副本）
  4. 所有 S 副本被清除后，Core0 获得独占访问权
  5. Core0: O → M
  6. Core0 现在可以本地静默写入
结果: Core0 = M，其他 = I
```

为什么需要 BusUpgr 而非 BusRdX？因为 Core0 本身就持有有效数据（dirty），不需要从总线上重新读取数据。BusUpgr 只是通知其他核心失效其副本，不涉及数据传输。这比 BusRdX 更高效。

### 4.3 O + BusRd（总线读请求）

**规则**：O → O（保持，供给数据）

```
当前: Core0 = O (dirty, shared, owner)，Core1 = I
触发: Core1 发送 BusRd 请求读取
动作:
  1. Core0 嗅探到 BusRd
  2. Core0 识别自己是 Owner → 在总线上发送 FlushOpt 提供脏数据
  3. Core1: I → S（获得共享副本）
  4. Core0: O → O（仍然是 Owner，仍然是 dirty）
结果: Core0 = O，Core1 = S
```

**关键设计决策**：为什么不是 Core0 的数据供给完毕后就变成 S？因为这样会丢失 ownership。如果 Core0 变成 S，那么下次再有 BusRd 时，谁来供给数据？如果所有核心都是 S，就只能回到内存读取，导致性能下降。保留 O 状态意味着"始终有一个核心负责"。

### 4.4 O + BusRdX（总线读-独占请求）

**规则**：O → I（供给数据后失效）

```
当前: Core0 = O (dirty, shared, owner)，Core1 = I
触发: Core1 发送 BusRdX 请求读-独占（意味着 Core1 准备写入）
动作:
  1. Core0 嗅探到 BusRdX
  2. Core0 识别自己是 Owner → 在总线上提供脏数据给 Core1
  3. Core0: O → I（放弃所有权，失效本地副本）
  4. Core1: I → M（获得脏数据，独占）
  5. 内存可能收到 Core0 提供的数据（取决于实现——有些实现中内存嗅探并缓存）
结果: Core0 = I，Core1 = M
```

**与 BusRd 的区别**：BusRdX 表示请求者需要独占访问（即将写入），因此 O 状态核心必须交出所有权并失效。

### 4.5 O + Evict（逐出）

**规则**：O → I（必须写回内存）

```
当前: Core0 = O (dirty, shared, owner)，其他核心可能有 S 副本
触发: Core0 需要驱逐该缓存行（如缓存冲突、上下文切换等）
动作:
  1. Core0 将脏数据写回内存（这是自 M→O 以来第一次写回！）
  2. Core0: O → I
  3. 其他 S 核心: 保持 S（它们有干净副本，内存现在也是干净的）
结果: Core0 = I，其他 = S（如果有），内存 = 最新
```

**这是 MOESI 的核心经济学**：脏数据的写回被延迟到了 Evict 时刻，而不是在第一次被共享时。如果核心之间的共享是长期的（如生产者-消费者缓冲区），写回可能被无限推迟，或者根本不需要（如果缓冲区在程序结束前一直活跃）。

### 4.6 O 状态行为汇总表

| 触发事件 | 状态转换 | 总线事务 | 说明 |
|---------|---------|---------|------|
| **PrRd** | O → O | 无 | 本地读，不影响任何状态 |
| **PrWr** | **O → M** | BusUpgr | 升级：失效其他 S 副本后独占写入 |
| **BusRd** | O → O | FlushOpt | 作为 Owner 供给数据，保持 ownership |
| **BusRdX** | O → I | FlushOpt | 交出数据 + ownership，自身失效 |
| **BusUpgr** | O → I | 无 | 另一个核心要升级到 M，O 自己失效 |
| **Evict** | O → I | Write-Back | 最终回写内存 |

---

## 5. 完整 MOESI 状态转换表

以下表格展示了所有五种状态在每种触发条件下的转换。格式为：**下一状态 (总线事务)**。

### 5.1 处理器侧触发（Processor-side）

| 当前状态 | PrRd（本地读） | PrWr（本地写） | Evict（逐出） |
|:------:|:---:|:---:|:---:|
| **M** | M (—) | M (—) | I (Write-Back) |
| **O** | O (—) | **M (BusUpgr)** | I (Write-Back) |
| **E** | E (—) | M (—) | I (—) |
| **S** | S (—) | M (BusUpgr) | I (—) |
| **I** | *见下方总线侧* | M (BusRdX) | I (—) |

### 5.2 总线侧触发（Bus-side，嗅探到）

| 当前状态 | BusRd | BusRdX | BusUpgr | FlushOpt |
|:------:|:---:|:---:|:---:|:---:|
| **M** | **O (FlushOpt)** | I (FlushOpt) | — | — |
| **O** | **O (FlushOpt)** | I (FlushOpt) | I (—) | — |
| **E** | S (—) | I (FlushOpt) | — | — |
| **S** | S (—) | I (—) | I (—) | — |
| **I** | I (—) | I (—) | I (—) | I (—) |

> 注："—" 表示该触发通常不会在这种情况下发生，或无需特殊处理。

### 5.3 I 状态的读取路径（补充）

当缓存行处于 I 状态且发生 PrRd 时：

| 情形 | 下一状态 | 说明 |
|------|:---:|------|
| 无其他核心持有 | E | 独占-干净 |
| 有其他 E/S 核心持有 | S | 共享，数据来自其他缓存或内存 |
| 有 M/O 核心持有 | S | 共享，数据来自 M/O 核心的 FlushOpt |

---

## 6. MOESI vs MESI 对比

### 6.1 协议层面对比

| 维度 | **MESI** | **MOESI** |
|------|---------|----------|
| **状态数** | 4 (M, E, S, I) | 5 (M, O, E, S, I) |
| **"脏且共享"存在吗** | **不存在** | **存在 (O 状态)** |
| **M 被共享时的行为** | M→S，强制写回内存 | **M→O，不写回内存** |
| **cache-to-cache 脏数据传递** | 间接（需经过内存） | 直接（FlushOpt） |
| **ownership 概念** | 无（或仅 M 隐含） | 显式（M 或 O） |
| **多核共享脏数据的效率** | 低（反复写回+重新读入） | 高（O 状态持续供给） |
| **典型实现者** | Intel、ARM | **AMD (AMD64)** |
| **总线消息类型** | BusRd, BusRdX, BusUpgr, Write-Back | BusRd, BusRdX, BusUpgr, **FlushOpt**, Write-Back |

### 6.2 典型场景效率对比

| 场景 | MESI 内存事务数量 | MOESI 内存事务数量 | 节省 |
|------|:---:|:---:|:---:|
| 单生产者单消费者 (N 次交互) | 2N 次写回 + N 次 BusRdX | 0 次写回 + N 次 BusUpgr | 2N 次内存写入 |
| 单生产者多消费者 (M 个消费者) | M 次写回 | 0 次（O 供给全部） | M 次内存写入 |
| 迁移型共享 (ping-pong) | 同 MOESI（都需 BusRdX） | 同 MESI | 0 |
| 只读共享 | 都只需首次填充 | 都只需首次填充 | 0 |

### 6.3 MOESI 的成本

MOESI 并非没有代价：

| 成本项 | 说明 |
|------|------|
| **状态位数** | 需要 3 bits 编码 5 状态（vs MESI 的 2 bits x 4 状态） |
| **总线消息类型** | 引入 FlushOpt，增加总线协议复杂度 |
| **所有权追踪** | 需要 O 状态核心响应所有 BusRd |
| **S 核心写入** | S→M 仍需 BusUpgr（与 MESI 相同，无改进） |

---

## 7. C++ 仿真代码

下面的代码完整模拟了 MOESI 协议的核心行为，并与 MESI 进行对比。

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <map>
#include <cassert>
#include <iomanip>

// ============================================================
// 状态定义
// ============================================================
enum class State { M, O, E, S, I };

std::string state_to_str(State s) {
    switch(s) {
        case State::M: return "M";
        case State::O: return "O";
        case State::E: return "E";
        case State::S: return "S";
        case State::I: return "I";
    }
    return "?";
}

// ============================================================
// 总线消息类型
// ============================================================
enum class BusMsg {
    BusRd,      // 读请求 — 需要数据
    BusRdX,     // 读-独占请求 — 需要数据且要独占
    BusUpgr,    // 升级请求 — 已持有数据，只需失效他人
    FlushOpt,   // MOESI 特有：供给脏数据，但不要写回内存
    WriteBack,  // 写回内存
    None
};

std::string msg_to_str(BusMsg m) {
    switch(m) {
        case BusMsg::BusRd:     return "BusRd";
        case BusMsg::BusRdX:    return "BusRdX";
        case BusMsg::BusUpgr:   return "BusUpgr";
        case BusMsg::FlushOpt:  return "FlushOpt";
        case BusMsg::WriteBack: return "WriteBack";
        case BusMsg::None:      return "—";
    }
    return "?";
}

// ============================================================
// 统计计数器
// ============================================================
struct Stats {
    int memory_writes = 0;      // 内存写回次数
    int flushopt_count = 0;     // cache-to-cache 脏数据传递次数
    int bus_upgr_count = 0;     // BusUpgr 次数
    int bus_rdx_count = 0;      // BusRdX 次数

    void reset() { *this = Stats{}; }
    void print(const std::string& label) {
        std::cout << "  [" << label << "] MemoryWrites=" << memory_writes
                  << ", FlushOpt=" << flushopt_count
                  << ", BusUpgr=" << bus_upgr_count
                  << ", BusRdX=" << bus_rdx_count << std::endl;
    }
};

// ============================================================
// CacheLine: 模拟一个缓存行
// ============================================================
struct CacheLine {
    State state = State::I;
    bool dirty = false;   // 数据是否脏（与内存不一致）
    int data = 0;         // 模拟的数据值
    int addr = 0;         // 地址（简化）
    bool is_owner = false;// 是否是 owner（MOESI 特有）

    void reset() {
        state = State::I;
        dirty = false;
        data = 0;
        is_owner = false;
    }
};

// ============================================================
// 内存模型（简化）
// ============================================================
int memory_value = 0;  // 全局内存值

// ============================================================
// 仿真核心
// ============================================================
class MOESISimulator {
public:
    static constexpr int NUM_CORES = 4;
    std::vector<CacheLine> caches;
    int memory_data = 0;
    Stats stats;

    MOESISimulator() : caches(NUM_CORES) {
        memory_data = 0;
    }

    void reset() {
        for (auto& c : caches) c.reset();
        memory_data = 0;
        stats.reset();
    }

    // ---- 辅助函数 ----

    // 查找是否有 M 或 O 状态的缓存（owner）
    int find_owner() {
        for (int i = 0; i < NUM_CORES; ++i) {
            if (caches[i].state == State::M || caches[i].state == State::O)
                return i;
        }
        return -1;
    }

    // 查找是否有 E 状态的缓存
    int find_exclusive() {
        for (int i = 0; i < NUM_CORES; ++i) {
            if (caches[i].state == State::E) return i;
        }
        return -1;
    }

    // 失效所有 S 状态的缓存（返回失效的数量）
    int invalidate_all_S() {
        int count = 0;
        for (int i = 0; i < NUM_CORES; ++i) {
            if (caches[i].state == State::S) {
                caches[i].reset();
                count++;
            }
        }
        return count;
    }

    // 检查是否还有其他有效副本
    bool has_other_valid_copy(int exclude_core) {
        for (int i = 0; i < NUM_CORES; ++i) {
            if (i == exclude_core) continue;
            if (caches[i].state != State::I) return true;
        }
        return false;
    }

    // 获取总线数据（从 owner 或内存）
    int get_bus_data(int requestor) {
        int owner = find_owner();
        if (owner >= 0 && owner != requestor) {
            return caches[owner].data;  // 从 owner 的缓存获取
        }
        int excl = find_exclusive();
        if (excl >= 0 && excl != requestor) {
            return caches[excl].data;
        }
        return memory_data;  // 从内存获取
    }

    // ---- 核心操作 ----

    // 处理器读
    void prRd(int core_id) {
        CacheLine& c = caches[core_id];
        std::cout << "  Core" << core_id << " PrRd: " << state_to_str(c.state);

        switch (c.state) {
        case State::M:
        case State::O:
        case State::E:
        case State::S:
            // 命中，无状态变化
            std::cout << " → " << state_to_str(c.state) << " (Hit)" << std::endl;
            break;

        case State::I: {
            // 未命中，需从总线获取
            int owner = find_owner();
            if (owner >= 0) {
                // 有 M 或 O 的 owner → 获取数据，owner 供给
                c.data = caches[owner].data;
                c.dirty = false;
                c.is_owner = false;

                if (caches[owner].state == State::M) {
                    // ★ MOESI 的关键转换：M → O ★
                    caches[owner].state = State::O;
                    caches[owner].dirty = true;   // 仍然是脏的！
                    caches[owner].is_owner = true;
                    stats.flushopt_count++;
                    std::cout << " → S | Core" << owner << ": M→O (FlushOpt) ★";
                } else {
                    // owner 本来就是 O
                    stats.flushopt_count++;
                    std::cout << " → S | Core" << owner << ": O→O (FlushOpt)";
                }
                c.state = State::S;

            } else {
                // 没有 owner，从内存获取
                c.data = memory_data;
                c.dirty = false;

                if (has_other_valid_copy(core_id)) {
                    c.state = State::S;
                } else {
                    c.state = State::E;
                }
                std::cout << " → " << state_to_str(c.state) << " (from Memory)";
            }
            std::cout << std::endl;
            break;
        }
        }
    }

    // 处理器写
    void prWr(int core_id, int new_value) {
        CacheLine& c = caches[core_id];
        std::cout << "  Core" << core_id << " PrWr(" << new_value << "): "
                  << state_to_str(c.state);

        switch (c.state) {
        case State::M:
            // 静默写入
            c.data = new_value;
            std::cout << " → M (Silent)" << std::endl;
            break;

        case State::O:
            // O → M: 需要 BusUpgr 失效其他副本
            {
                int invalidated = invalidate_all_S();
                stats.bus_upgr_count++;
                c.state = State::M;
                c.is_owner = true;
                c.data = new_value;
                c.dirty = true;
                std::cout << " → M (BusUpgr, invalidated " << invalidated << " S copies) ★" << std::endl;
            }
            break;

        case State::E:
            // E → M: 静默升级
            c.state = State::M;
            c.dirty = true;
            c.data = new_value;
            std::cout << " → M (Silent upgrade)" << std::endl;
            break;

        case State::S:
            // S → M: 需要 BusUpgr
            {
                int invalidated = invalidate_all_S();
                stats.bus_upgr_count++;
                c.state = State::M;
                c.dirty = true;
                c.is_owner = true;
                c.data = new_value;
                std::cout << " → M (BusUpgr, invalidated " << invalidated << " copies)" << std::endl;
            }
            break;

        case State::I:
            // I → M: 需要 BusRdX
            {
                int owner = find_owner();
                if (owner >= 0) {
                    c.data = caches[owner].data;
                    caches[owner].reset();  // owner 失效
                    stats.flushopt_count++;
                    std::cout << " (got data from Core" << owner << " FlushOpt)";
                } else {
                    c.data = memory_data;
                }
                invalidate_all_S();
                stats.bus_rdx_count++;
                c.state = State::M;
                c.dirty = true;
                c.is_owner = true;
                c.data = new_value;
                std::cout << " → M (BusRdX)" << std::endl;
            }
            break;
        }
    }

    // 逐出（驱逐）
    void evict(int core_id) {
        CacheLine& c = caches[core_id];
        std::cout << "  Core" << core_id << " Evict: " << state_to_str(c.state);

        switch (c.state) {
        case State::M:
        case State::O:
            // 脏数据必须写回
            memory_data = c.data;
            stats.memory_writes++;
            c.reset();
            std::cout << " → I (WriteBack to Memory) ★" << std::endl;
            break;

        case State::E:
        case State::S:
            // 干净，无需写回
            c.reset();
            std::cout << " → I (Silent)" << std::endl;
            break;

        case State::I:
            std::cout << " → I (no-op)" << std::endl;
            break;
        }
    }

    // 打印所有核心状态
    void dump_state(const std::string& label) {
        std::cout << "\n=== " << label << " ===" << std::endl;
        std::cout << "  Memory = " << memory_data;
        if (memory_would_be_served_by_memory()) {
            std::cout << " (valid)";
        } else {
            std::cout << " (STALE!)";
        }
        std::cout << std::endl;
        for (int i = 0; i < NUM_CORES; ++i) {
            const auto& c = caches[i];
            std::cout << "  Core" << i << ": " << state_to_str(c.state)
                      << " | data=" << c.data
                      << " | dirty=" << (c.dirty ? "Y" : "N")
                      << " | owner=" << (c.is_owner ? "Y" : "N")
                      << std::endl;
        }
        std::cout << std::endl;
    }

private:
    // 检查内存是否是有效的（没有脏缓存行）
    bool memory_would_be_served_by_memory() {
        for (const auto& c : caches) {
            if (c.state == State::M || c.state == State::O) return false;
        }
        return true;
    }
};

// ============================================================
// 测试 1: MOESI 的 M→O 转换 — 不写回内存
// ============================================================
void test_moesi_m_to_o_transition() {
    std::cout << "\n╔══════════════════════════════════════════════════╗" << std::endl;
    std::cout << "║  测试 1: M→O 转换 — MOESI 的核心特性           ║" << std::endl;
    std::cout << "╚══════════════════════════════════════════════════╝" << std::endl;

    MOESISimulator sim;

    // 初始：Core0 获得 M 状态
    sim.prWr(0, 42);
    sim.dump_state("Core0 写入 42 → M");

    // Core1 读取：触发 M→O 转换
    sim.prRd(1);
    sim.dump_state("Core1 读取 — Core0 M→O (无内存写入!)");

    // 验证内存是否保持 stale
    assert(sim.memory_data == 0);  // 内存仍是旧值！
    std::cout << "  ★ 验证通过: 内存仍为 0 (STALE)，未被写入！" << std::endl;

    // 验证 Core0 的状态
    assert(sim.caches[0].state == State::O);
    assert(sim.caches[0].dirty == true);
    assert(sim.caches[0].data == 42);
    std::cout << "  ★ 验证通过: Core0 处于 O 状态，数据仍为 dirty" << std::endl;

    assert(sim.caches[1].state == State::S);
    assert(sim.caches[1].data == 42);
    std::cout << "  ★ 验证通过: Core1 处于 S 状态，获得数据 42" << std::endl;

    sim.stats.print("MOESI");
    std::cout << "  ★ MOESI 关键: 0 次内存写入完成共享！" << std::endl;
}

// ============================================================
// 测试 2: O 状态持续供给多个读者
// ============================================================
void test_moesi_o_servicing_multiple_reads() {
    std::cout << "\n╔══════════════════════════════════════════════════╗" << std::endl;
    std::cout << "║  测试 2: O 状态多次供给数据                     ║" << std::endl;
    std::cout << "╚══════════════════════════════════════════════════╝" << std::endl;

    MOESISimulator sim;

    // 初始
    sim.prWr(0, 99);     // Core0 → M
    sim.prRd(1);          // Core0 M→O, Core1 I→S
    sim.dump_state("初始: Core0=O, Core1=S");

    // Core2 读取：O 状态供给
    sim.prRd(2);
    sim.dump_state("Core2 读取: Core0 (O) 供给数据");

    assert(sim.caches[0].state == State::O);  // 仍是 O
    assert(sim.caches[2].state == State::S);
    assert(sim.caches[2].data == 99);
    std::cout << "  ★ 验证通过: Core0 保持 O，Core2 获得到 S" << std::endl;

    // Core3 读取：O 状态继续供给
    sim.prRd(3);
    sim.dump_state("Core3 读取: Core0 (O) 再次供给数据");

    assert(sim.caches[0].state == State::O);  // 仍然是 O
    assert(sim.caches[3].state == State::S);
    assert(sim.caches[3].data == 99);
    std::cout << "  ★ 验证通过: Core0 保持 O 状态不丢失，连续供给 3 次" << std::endl;

    // 内存仍然 stale
    assert(sim.memory_data == 0);
    std::cout << "  ★ 验证通过: 3 次共享读取，0 次内存写入！" << std::endl;

    sim.stats.print("MOESI → O 供给");
}

// ============================================================
// 测试 3: O→M 升级
// ============================================================
void test_moesi_o_to_m_upgrade() {
    std::cout << "\n╔══════════════════════════════════════════════════╗" << std::endl;
    std::cout << "║  测试 3: O→M 升级 — BusUpgr 失效 S 副本       ║" << std::endl;
    std::cout << "╚══════════════════════════════════════════════════╝" << std::endl;

    MOESISimulator sim;

    // 建立 O+S 共享状态
    sim.prWr(0, 55);     // Core0 → M
    sim.prRd(1);          // Core0 M→O, Core1 I→S
    sim.prRd(2);          // Core0 O→O, Core2 I→S
    sim.dump_state("初始: Core0=O, Core1=S, Core2=S");

    // Core0 写入：O→M，需要 BusUpgr 失效 Core1 和 Core2 的 S 副本
    sim.prWr(0, 77);
    sim.dump_state("Core0 写入 77: O→M (BusUpgr)");

    assert(sim.caches[0].state == State::M);
    assert(sim.caches[0].data == 77);
    assert(sim.caches[1].state == State::I);  // 已失效
    assert(sim.caches[2].state == State::I);  // 已失效
    std::cout << "  ★ 验证通过: Cor0 O→M, Core1/2 被 BusUpgr 失效" << std::endl;

    // 内存仍然 stale
    assert(sim.memory_data == 0);
    std::cout << "  ★ 验证通过: 完整 M→O→M 循环，内存 0 次写入" << std::endl;

    sim.stats.print("MOESI → O→M 升级");
}

// ============================================================
// 测试 4: 与 MESI 的行为对比
// ============================================================
class MESISimulator {
public:
    static constexpr int NUM_CORES = 4;
    std::vector<CacheLine> caches;
    int memory_data = 0;
    Stats stats;

    MESISimulator() : caches(NUM_CORES) {}

    void reset() {
        for (auto& c : caches) c.reset();
        memory_data = 0;
        stats.reset();
    }

    int find_modified() {
        for (int i = 0; i < NUM_CORES; ++i) {
            if (caches[i].state == State::M) return i;
        }
        return -1;
    }

    int find_exclusive() {
        for (int i = 0; i < NUM_CORES; ++i) {
            if (caches[i].state == State::E) return i;
        }
        return -1;
    }

    void invalidate_all_S() {
        for (int i = 0; i < NUM_CORES; ++i) {
            if (caches[i].state == State::S) caches[i].reset();
        }
    }

    bool has_other_valid(int exclude) {
        for (int i = 0; i < NUM_CORES; ++i) {
            if (i == exclude) continue;
            if (caches[i].state != State::I) return true;
        }
        return false;
    }

    void prRd(int core_id) {
        CacheLine& c = caches[core_id];
        std::cout << "  [MESI] Core" << core_id << " PrRd: " << state_to_str(c.state);

        switch (c.state) {
        case State::M:
        case State::E:
        case State::S:
            std::cout << " → " << state_to_str(c.state) << " (Hit)" << std::endl;
            break;

        case State::I: {
            int owner = find_modified();
            if (owner >= 0) {
                // ★ MESI 的关键区别: 必须先 Write-Back！
                c.data = caches[owner].data;
                memory_data = caches[owner].data;
                stats.memory_writes++;
                caches[owner].state = State::S;
                caches[owner].dirty = false;
                caches[owner].is_owner = false;
                c.state = State::S;
                c.dirty = false;
                std::cout << " → S | Core" << owner << ": M→S (WriteBack!) ★" << std::endl;
            } else {
                c.data = memory_data;
                c.dirty = false;
                if (has_other_valid(core_id)) {
                    c.state = State::S;
                } else {
                    c.state = State::E;
                }
                std::cout << " → " << state_to_str(c.state) << " (from Memory)" << std::endl;
            }
            break;
        }
        case State::O:
            // MESI 中没有 O 状态
            break;
        }
    }

    void prWr(int core_id, int new_value) {
        CacheLine& c = caches[core_id];
        std::cout << "  [MESI] Core" << core_id << " PrWr(" << new_value << "): "
                  << state_to_str(c.state);

        switch (c.state) {
        case State::M:
            c.data = new_value;
            std::cout << " → M (Silent)" << std::endl;
            break;
        case State::E:
            c.state = State::M;
            c.dirty = true;
            c.data = new_value;
            std::cout << " → M (Silent upgrade)" << std::endl;
            break;
        case State::S:
            {
                int invalidated = 0;
                for (int i = 0; i < NUM_CORES; ++i) {
                    if (i != core_id && caches[i].state == State::S) {
                        caches[i].reset();
                        invalidated++;
                    }
                }
                stats.bus_upgr_count++;
                c.state = State::M;
                c.dirty = true;
                c.data = new_value;
                std::cout << " → M (BusUpgr, invalidated " << invalidated << ")" << std::endl;
            }
            break;
        case State::I:
            {
                int owner = find_modified();
                if (owner >= 0) {
                    c.data = caches[owner].data;
                    memory_data = caches[owner].data;
                    stats.memory_writes++;
                    caches[owner].reset();
                } else {
                    c.data = memory_data;
                }
                invalidate_all_S();
                stats.bus_rdx_count++;
                c.state = State::M;
                c.dirty = true;
                c.data = new_value;
                std::cout << " → M (BusRdX";
                if (owner >= 0) std::cout << " + WriteBack";
                std::cout << ")" << std::endl;
            }
            break;
        case State::O:
            break;
        }
    }

    void dump_state(const std::string& label) {
        std::cout << "\n=== [MESI] " << label << " ===" << std::endl;
        std::cout << "  Memory = " << memory_data << std::endl;
        for (int i = 0; i < NUM_CORES; ++i) {
            const auto& c = caches[i];
            std::cout << "  Core" << i << ": " << state_to_str(c.state)
                      << " | data=" << c.data
                      << " | dirty=" << (c.dirty ? "Y" : "N")
                      << std::endl;
        }
        std::cout << std::endl;
    }
};

void test_moesi_vs_mesi_comparison() {
    std::cout << "\n╔══════════════════════════════════════════════════╗" << std::endl;
    std::cout << "║  测试 4: MOESI vs MESI 对比                    ║" << std::endl;
    std::cout << "╚══════════════════════════════════════════════════╝" << std::endl;

    // ---- MOESI 仿真 ----
    std::cout << "\n--- MOESI 仿真 ---" << std::endl;
    MOESISimulator moesi;
    moesi.prWr(0, 42);    // Core0 → M
    moesi.prRd(1);         // Core0 M→O, Core1 I→S    ← 无内存写入
    moesi.prWr(0, 88);    // Core0 O→M (BusUpgr)      ← 无内存写入
    moesi.evict(0);        // 最终写回

    // ---- MESI 仿真 (同样场景) ----
    std::cout << "\n--- MESI 仿真 ---" << std::endl;
    MESISimulator mesi;
    mesi.prWr(0, 42);     // Core0 → M
    mesi.prRd(1);          // Core0 M→S, Core1 I→S   ← WriteBack 写回内存
    mesi.prWr(0, 88);     // Core0 S→M (BusUpgr)
    // MESI 中 Core0 重新变为 M 后无需再写回

    // ---- 对比统计 ----
    std::cout << "\n=== 统计对比 ===" << std::endl;
    moesi.stats.print("MOESI");
    mesi.stats.print("MESI ");

    std::cout << "\n  ★ MOESI 优势: " << std::endl;
    std::cout << "    - MOESI 内存写入: " << moesi.stats.memory_writes
              << " (仅在 Evict 时写回)" << std::endl;
    std::cout << "    - MESI 内存写入:  " << mesi.stats.memory_writes
              << " (M→S 时强制写回)" << std::endl;
    std::cout << "    - FlushOpt (cache-to-cache): MOESI 使用, MESI 无此能力" << std::endl;
}

// ============================================================
// 测试 5: O 状态最终 Evict 才写回
// ============================================================
void test_moesi_o_evict_writeback() {
    std::cout << "\n╔══════════════════════════════════════════════════╗" << std::endl;
    std::cout << "║  测试 5: O 状态 Evict 时才写回内存             ║" << std::endl;
    std::cout << "╚══════════════════════════════════════════════════╝" << std::endl;

    MOESISimulator sim;

    // 构建长共享链
    sim.prWr(0, 123);     // Core0 → M
    sim.prRd(1);           // Core0 M→O, Core1 I→S
    sim.prRd(2);           // Core0 O→O, Core2 I→S
    sim.prRd(3);           // Core0 O→O, Core3 I→S
    sim.dump_state("长共享链: Core0=O, Core1/2/3=S");

    // 验证多次共享后内存仍 stale
    assert(sim.memory_data == 0);
    std::cout << "  ★ 3 次共享读取后，内存仍为 0 (STALE)" << std::endl;

    // 最终 Evict Core0
    sim.evict(0);
    sim.dump_state("Core0 Evict: O→I, WriteBack");

    // 此时内存才被更新
    assert(sim.memory_data == 123);
    std::cout << "  ★ Evict 时内存才被更新为 123" << std::endl;
    std::cout << "  ★ 延迟写回策略验证成功: 整个共享期间 0 次内存写入" << std::endl;

    sim.stats.print("MOESI → 延迟写回");
}

// ============================================================
// main
// ============================================================
int main() {
    std::cout << "╔══════════════════════════════════════════════════════════╗" << std::endl;
    std::cout << "║     MOESI Cache Coherence Protocol — Full Simulation   ║" << std::endl;
    std::cout << "╚══════════════════════════════════════════════════════════╝" << std::endl;

    test_moesi_m_to_o_transition();
    test_moesi_o_servicing_multiple_reads();
    test_moesi_o_to_m_upgrade();
    test_moesi_vs_mesi_comparison();
    test_moesi_o_evict_writeback();

    std::cout << "\n╔══════════════════════════════════════════════════════════╗" << std::endl;
    std::cout << "║            All Tests Passed                             ║" << std::endl;
    std::cout << "╚══════════════════════════════════════════════════════════╝" << std::endl;

    return 0;
}
```

### 7.1 编译与运行

```bash
# 编译
g++ -std=c++17 -O2 -o moesi_sim moesi_sim.cpp

# 运行
./moesi_sim
```

### 7.2 预期输出要点

```
测试 1: M→O 转换
  Core0 PrWr(42): I → M (BusRdX)
  Core1 PrRd: I → S | Core0: M→O (FlushOpt) ★
  内存仍为 0 (STALE)，未被写入！

测试 2: O 状态多次供给数据
  Core2 PrRd: I → S | Core0: O→O (FlushOpt)
  Core3 PrRd: I → S | Core0: O→O (FlushOpt)
  Core0 保持 O 状态，3 次共享 0 次内存写入

测试 3: O→M 升级
  Core0 PrWr(77): O → M (BusUpgr, invalidated 2 S copies)
  完整 M→O→M 循环，内存 0 次写入

测试 4: 对比
  [MOESI] MemoryWrites=0 (共享期间), FlushOpt=1
  [MESI]  MemoryWrites=1 (M→S 时强制)
```

### 7.3 代码要点说明

1. **M→O 转换的精髓**（`prRd` 中 I 状态的 case）：当 Core0 处于 M 状态且 Core1 发起读时，Core0 通过 FlushOpt 将脏数据直接供给 Core1，自身转为 O 状态（`dirty = true`，`is_owner = true`），**内存完全被跳过**。

2. **O→O 的保持**：当 Core0 处于 O 状态且 Core2 发起读时，Core0 再次通过 FlushOpt 供给数据，状态保持 O 不变。所有权不转移，保证了后续读取始终可以快速获取数据。

3. **O→M 升级**：Core0 从 O 写入时，仅需 BusUpgr 失效其他 S 副本——不需要获取数据（Core0 自己持有脏数据）。

4. **Evict 才写回**：O 状态的 `evict()` 处理中，脏数据在这时才最终写回内存。这体现了"懒写回"（lazy write-back）的设计哲学。

---

## 8. 面试自测题

### Q1: MOESI 相比 MESI 的关键优势是什么？

<details>
<summary>点击展开答案</summary>

**参考答案**：

MOESI 相比 MESI 的关键优势在于通过引入 **Owned (O) 状态** 解决了 MESI 中 **M→S 转换时强制写回内存** 的问题。在 MOESI 中，当 Modified 状态的缓存行被其他核心读取时，缓存行从 M 转为 O（而非 S），数据保持 dirty，由当前核心负责向其他读取者供给数据，**内存写入被延迟到缓存行被逐出（Evict）时**（如果那之前发生了，否则数据一直留在缓存中）。这消除了"生产者-消费者"或"读取-修改-写入"模式中大量不必要的内存总线流量，显著降低了多核系统的内存带宽压力和延迟。

核心公式：**MESI 的 M→S (Write-Back) 变为 MOESI 的 M→O (FlushOpt)**。

</details>

---

### Q2: 在 MOESI 中，Core0 处于 M 状态持有脏数据，Core1 发起 BusRd 读取该缓存行后，Core0 处于什么状态？数据仍然是脏的吗？

<details>
<summary>点击展开答案</summary>

**参考答案**：

Core0 处于 **O (Owned) 状态**。

- **状态**：O
- **Dirty**：**是**，数据仍然是脏的（与内存不一致）
- **共享**：是，Core1 持有 S 状态的副本
- **责任**：Core0 是 Owner，后续任何核心再发起 BusRd 时，由 Core0（而非内存）负责提供数据

关键洞察：**数据从未写入过内存**。Core0 的 Dirty Bit 保持设置，缓存行的值仍然与内存不一致。MOESI 的 O 状态允许"脏数据同时在多个核心间共享"，打破了 MESI 中"脏即独占"的限制。

如果面试官追问"那什么时候内存才会被更新"，回答：等到 O 状态的核心被 Evict 驱逐时，才会最终写回内存。

</details>

---

### Q3: O 状态的缓存行在什么时候最终被写回内存？

<details>
<summary>点击展开答案</summary>

**参考答案**：

O 状态的缓存行在以下两个时机之一被写回内存：

1. **Evict（逐出/驱逐）**：当 O 状态的核心因为缓存容量不足（cache miss 导致替换）、上下文切换或其他原因需要驱逐该缓存行时，脏数据被写回内存。这是 MOESI 设计中"延迟写回"的终点。

2. **BusRdX（总线读-独占）**：当另一个核心发送 BusRdX 意图独占该缓存行时，O 状态的核心在供给数据的同时，数据也会被写到内存（取决于具体实现），然后 O 转为 I。

值得注意的是，如果 O 状态的核心一直在使用该数据且从未被驱逐，则**数据可能永远不会被写回内存**——这对频繁共享的生产者-消费者数据极为有利。

</details>

---

### Q4: 当 M 或 O 状态的核心向另一个核心提供脏数据但不写回内存时，使用的是什么总线消息？

<details>
<summary>点击展开答案</summary>

**参考答案**：

使用的是 **FlushOpt（Flush Optional）** 消息。

FlushOpt 的含义是："这里有脏数据，接收方可以获得它，但**不要**将数据写回内存。"它与传统的 Write-Back 消息的区别如下：

| 消息类型 | 含义 | 数据写回内存？ | 发送方状态变化 |
|---------|------|:---:|:---:|
| **FlushOpt** | 提供脏数据（可选刷新） | **否** | M→O 或 O→O |
| **Write-Back** | 写回脏数据 | **是** | M→I 或 O→I |

FlushOpt 是 MOESI 协议中实现"不写回内存的脏数据共享"的关键机制。在某些资料中，也可能被称作 "Intervention" 或 "Cache-to-Cache Transfer"（缓存到缓存传输），但 FlushOpt 是 MOESI 协议中的正式总线条目。

在 AMD64 架构的 MOESI 实现中（如 HyperTransport 总线），FlushOpt 作为 Probe 响应的一种类型存在，发送方主动提供数据但不触发 DRAM 写入。

</details>

---

### Q5: 哪个处理器厂商以使用 MOESI 协议而闻名？

<details>
<summary>点击展开答案</summary>

**参考答案**：

**AMD（Advanced Micro Devices）**。

AMD 在其 **AMD64 架构**（Opteron、Athlon 64、Ryzen 系列等）中使用了 MOESI 协议。这在历史上是 AMD 与 Intel 在缓存一致性协议层面的重要区别之一：

| 厂商 | 协议 | 特点 |
|------|------|------|
| **AMD** | **MOESI** | 5 状态，支持 Owned，cache-to-cache 脏数据传递 |
| **Intel** | **MESIF** | 5 状态，用 Forward 状态（F）解决多 S 响应问题，不同思路 |
| **ARM** | MESI / MOESI | Cortex-A 系列使用 MOESI 变体（如 AMBA ACE 协议中的 OwnedShared） |

**扩展知识点**：Intel 的 MESIF 协议通过 **Forward (F) 状态** 解决了类似的问题——即"多个 S 核心中谁负责转发数据"。在 MESIF 中，只有一个核心持有 F 状态，它负责响应其他核心的 BusRd，从而避免多个 S 核心同时尝试供给数据的冲突。虽然思路不同，但 MOESI 的 O（处理脏数据共享）和 MESIF 的 F（处理干净数据共享的供给责任）都试图解决同一个根本问题：**在多核共享场景下减少不必要的数据传输和总线争用**。

</details>

---

## 附录：本讲的关键概念速查

| 概念 | 一句话解释 |
|------|----------|
| **M→O 转换** | MOESI 的灵魂：M 被共享时不写回内存，转为 O 并负责供给数据 |
| **FlushOpt** | "这里有脏数据，接好，别写回内存" |
| **Owned 状态** | Dirty + Shared + Responsible（脏 + 共享 + 负责） |
| **延迟写回** | 脏数据写回推迟到 Evict 时刻，而非共享时刻 |
| **所有权 (Ownership)** | M 或 O 状态的核心"拥有"数据，负责响应总线请求 |
| **BusUpgr vs BusRdX** | BusUpgr 只需失效他人（O→M），BusRdX 需获取数据+失效他人（I→M） |
| **AMD** | MOESI 协议的经典实现者 |

---

*本讲结束。MOESI 的核心思想总结为一句话：让脏数据留在缓存中，直到它被真正需要逐出。O 状态就是这把钥匙。*

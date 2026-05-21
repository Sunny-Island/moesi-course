# MOESI 缓存一致性协议

## 课程概览

本课程从硬件底层出发，系统讲解 MOESI 缓存一致性协议。

## 讲座列表

| # | 主题 | 文件 |
|---|------|------|
| 1 | Cache 层级结构与一致性问题的起源 | [lec1-cache-hierarchy.md](lec1-cache-hierarchy.md) |
| 2 | 总线架构与 Snoop 机制 | [lec2-bus-architecture.md](lec2-bus-architecture.md) |
| 3 | MESI 协议详解 | [lec3-mesi-protocol.md](lec3-mesi-protocol.md) |
| 4 | MOESI 协议 —— Owned 状态 | [lec4-moesi-protocol.md](lec4-moesi-protocol.md) |
| 5 | 硬件实现细节 | [lec5-hardware-implementation.md](lec5-hardware-implementation.md) |
| 6 | Store Buffer、内存序与 False Sharing | [lec6-store-buffer-false-sharing.md](lec6-store-buffer-false-sharing.md) |

## 学习路线

```
Lec1 → Lec2 → Lec3 → Lec4 → Lec5 → Lec6
基础    总线    MESI   MOESI   硬件    实战
```

建议按顺序学习，每讲末尾有 Quiz 自测。

## 核心内容

本教程涵盖以下重点知识：

- MESI 四个状态各自含义及转换关系
- E→M 为什么不需要总线事务
- MOESI 比 MESI 好在哪、O 状态怎么工作
- 总线仲裁和 snoop 阶段的物理过程
- 状态位存在哪里、tag SRAM 怎么并行查找
- Store Buffer 导致的内存序问题及解决方案
- False Sharing 的成因和修复

# 第8章 内存（上）：内存分配器

**一句话**：内存分配器用 mcache/mcentral/mheap 三级、按对象大小等级在 span 上切分，栈上分配不走的才走堆。

**为什么需要它**：一个 `make([]int, 100)`、`new(T)`、或者逃逸到堆的小对象，Go 是怎么给你一块内存的？为什么它比 `malloc` 快？为什么海量小对象也不容易碎？这背后是 Go 自研的内存分配器，借鉴了线程缓存（tcmalloc）思路。这一章讲清三级结构和「大小等级」这把钥匙。

**核心结构**：

| 层级 | 归属 | 管什么 |
|---|---|---|
| mcache | 每个 P 独享 | 无锁快分配，按 size class 取小块 |
| mcentral | 全局（按大小分多个） | 给 mcache 补货，管理同类 span |
| mheap | 全局唯一 | 向操作系统要大块内存，切成 span |

**关键设计**：对象按大小分等级（size class），每个等级对应固定尺寸的 span；P 本地的 mcache 无锁拿小块，不够了向 mcentral 申请 span，mcentral 不够 mheap 再向 OS 要。

```text
G 要内存 ──► P.mcache(无锁) ──按 size class 切 span──► 返回小块
   mcache 空 ──► mcentral(按等级) ──► 给 span
   mcentral 空 ──► mheap ──► 向 OS 要内存 ──► 切 span
```

**拆解**：

### 一、mcache：每个 P 的无锁快车道

每个 P 拥有一个 mcache，里面按不同大小等级缓存着若干已切好的小块。G 申请小对象时，直接在自己 P 的 mcache 里按 size class 取一块，**完全无锁**——因为 mcache 只被绑定的那个 P 访问。这是 Go 分配器快的关键：绝大多数小对象分配根本不碰全局锁。

### 二、mcentral：按大小等级的全局补给站

当 mcache 里某等级的块用完了，向 mcentral 申请。mcentral 按对象大小分成多个，每个管理某一类 span（连续页组成的内存块）。它持有空闲 span 链表和已用 span 链表，负责给各个 P 的 mcache 补货，并在 span 用尽时向 mheap 申请新 span。mcentral 之间按等级隔离，减少了跨尺寸的锁竞争。

### 三、mheap 与 span：向操作系统要内存

mheap 是全局唯一的堆管理者，负责向操作系统申请大块连续内存（通常以页为单位），再切成 span 交给 mcentral。span 是分配的基本单位，由一个或多个连续页组成，内部按某个 size class 切成等大的小块。mheap 还会缓存一部分页以备快速响应，避免每次分配都陷入系统调用。

### 四、大小等级（size class）为什么重要

把所有对象不分大小混在一起分配，会产生大量外部碎片且查找慢。Go 把对象按大小映射到有限几个等级（如 8B、16B、32B……），每个等级对应固定尺寸的块。一个对象总是向上取最近的等级，带来极小的内部浪费，却换来 O(1) 的分配和极低的碎片率。理解 size class，就理解为什么「海量小对象」在 Go 里也能稳。

**今天怎么做**（延伸，非语料原文）：`runtime.ReadMemStats` / `pprof` 的 `allocs` 剖面能看到分配次数与大小分布；`GOGC`、软内存上限等可调 GC 与堆增长行为。这些观测/调优接口属于语料时代之后的演进，仅作延伸了解。

<div class="chapend" markdown="1">

**容易误解的地方**：
- 以为所有分配都走全局锁，其实小对象走 P 本地 mcache 是无锁的。
- 以为分配器直接按对象精确尺寸切，其实按 size class 向上取整，有微小内部碎片。
- 以为 mheap 每次都找 OS 要内存，其实它会缓存页、批量管理，减少系统调用。

**记住这一句**：P 本地 mcache 无锁快分、mcentral 按等级补货、mheap 向 OS 要大块——三级加 size class 是速度的来源。

</div>

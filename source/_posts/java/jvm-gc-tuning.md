---
title: JVM 内存模型与 GC 调优实战
tags:
  - Java
  - JVM
  - GC
  - 性能调优
categories:
  - Java
cover: 'https://images.unsplash.com/photo-1517694712202-14dd9538aa97?w=1280'
description: 深入理解 JVM 内存结构，掌握 G1/ZGC 垃圾收集器的调优技巧，解决常见的内存泄漏和 GC 停顿问题。
katex: false
mermaid: false
mathjax: false
abbrlink: 41274
date: 2025-05-20 10:00:00
top_img:
---

# JVM 内存模型与 GC 调优实战

Java 应用性能调优的核心在于理解 JVM 内存模型和垃圾收集机制。

## 1. JVM 运行时数据区

```
┌─────────────────────────────────────────┐
│              JVM 内存                    │
├──────────────┬──────────────────────────┤
│   线程私有    │        线程共享           │
├──────────────┼──────────────────────────┤
│ 程序计数器    │        堆 (Heap)         │
│ 虚拟机栈      │   ┌────────────────┐    │
│ 本地方法栈    │   │   Young Gen    │    │
│              │   │  - Eden        │    │
│              │   │  - S0, S1      │    │
│              │   ├────────────────┤    │
│              │   │   Old Gen      │    │
│              │   └────────────────┘    │
│              │      方法区 / 元空间      │
└──────────────┴──────────────────────────┘
```

## 2. 常用 GC 收集器

### G1 (Garbage First)

JDK 9+ 默认收集器，将堆划分为多个 Region。

```bash
# 启用 G1
-XX:+UseG1GC
# 设置最大停顿时间目标（毫秒）
-XX:MaxGCPauseMillis=200
# 设置堆大小
-Xms4g -Xmx4g
# G1 Region 大小
-XX:G1HeapRegionSize=4m
```

### ZGC

JDK 15+ 的超低延迟收集器，停顿时间 < 1ms。

```bash
# 启用 ZGC
-XX:+UseZGC
# 设置堆大小（建议 >= 4GB）
-Xms8g -Xmx8g
# 启用分代 ZGC（JDK 21+）
-XX:+ZGenerational
```

## 3. GC 调优实操

### 3.1 查看 GC 日志

```bash
# JDK 9+ 统一日志
-Xlog:gc*,gc+age=trace,safepoint:file=gc.log:time,level,tags:filecount=5,filesize=50m
```

### 3.2 诊断工具

| 工具 | 用途 |
|------|------|
| `jstat` | GC 统计信息 |
| `jmap` | 堆内存 dump |
| `jstack` | 线程栈分析 |
| `MAT` | 内存泄漏分析 |
| `Arthas` | 在线诊断（阿里开源） |

### 3.3 常见问题排查

**Full GC 频繁**：

```bash
# 1. 查看 GC 情况
jstat -gcutil <pid> 1000

# 2. dump 堆内存
jmap -dump:format=b,file=heap.hprof <pid>

# 3. 用 MAT 分析大对象
```

**内存泄漏**：

```java
// 典型内存泄漏场景
static List<Object> cache = new ArrayList<>();

public void leak() {
    while (true) {
        cache.add(new Object()); // 永远不会被回收
    }
}
```

使用 Arthas 在线排查：

```bash
# 查看堆内存各区域大小
memory
# 查看对象创建热点
profiler start --event alloc
profiler stop --format html
```

## 4. 调优参数速查表

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `-Xms` | 初始堆大小 | 与 Xmx 相同 |
| `-Xmx` | 最大堆大小 | 物理内存的 75% |
| `-Xmn` | 年轻代大小 | 堆的 1/3 - 1/2 |
| `-XX:MetaspaceSize` | 元空间初始大小 | 256m |
| `-XX:MaxMetaspaceSize` | 元空间上限 | 512m |
| `-XX:SurvivorRatio` | Eden:S0:S1 | 8:1:1 |

## 总结

GC 调优的关键步骤：
1. 确认问题（频繁 GC / 长停顿 / OOM）
2. 收集数据（GC 日志、堆 dump）
3. 分析定位（MAT、Arthas）
4. 优化调整（参数、代码、架构）

---

> 推荐工具：
> - [Arthas](https://arthas.aliyun.com/) - 阿里开源 Java 诊断工具
> - [Eclipse MAT](https://www.eclipse.org/mat/) - Memory Analyzer Tool
> - [GCViewer](https://github.com/chewiebug/GCViewer) - GC 日志可视化工具

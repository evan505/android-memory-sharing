# ART虚拟机内存管理

## 概述
深入ART(Android Runtime)虚拟机层面，了解Java代码如何被执行和管理内存。

## 主要内容

### 2.1 ART内存结构
- [ ] Heap结构设计
- [ ] Generation划分
- [ ] Space类型与作用

### 2.2 内存分配机制
- [ ] 对象分配策略
- [ ] 大对象处理
- [ ] 内存对齐与优化

### 2.3 垃圾回收实现
- [ ] Concurrent GC
- [ ] Mark-Sweep算法
- [ ] Compacting GC

### 2.4 ART vs Dalvik
- [ ] 内存管理差异
- [ ] 性能对比
- [ ] 迁移考虑

### 2.5 调试与监控
- [ ] ART日志分析
- [ ] GC性能指标
- [ ] 内存dump分析

## 实例代码
参见 `../resources/code-examples/art-memory/`

## 相关工具
- adb logcat (GC日志)
- perfetto
- systrace
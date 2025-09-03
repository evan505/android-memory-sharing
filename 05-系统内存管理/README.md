# 系统内存管理

## 概述
深入Android系统层面的内存管理策略，理解系统如何协调所有进程的内存使用。

## 主要内容

### 5.1 Linux内存管理基础
- [ ] 页面管理机制
- [ ] 虚拟内存系统
- [ ] Copy-On-Write机制

### 5.2 Android内存管理框架
- [ ] Low Memory Killer (LMK)
- [ ] ActivityManagerService内存管理
- [ ] 进程优先级与OOM调整

### 5.3 内存回收机制
- [ ] 页面回收算法
- [ ] Swap机制（Android 11+）
- [ ] 内存压缩技术

### 5.4 系统内存策略
- [ ] 内存阈值管理
- [ ] 后台进程清理
- [ ] 内存碎片整理

### 5.5 内存性能优化
- [ ] 预加载机制
- [ ] 内存预取策略
- [ ] NUMA感知调度

### 5.6 系统内存监控
- [ ] meminfo统计信息
- [ ] vmstat性能指标
- [ ] 内存压力事件

## 实例代码
参见 `../resources/code-examples/system-memory/`

## 相关工具
- cat /proc/meminfo
- vmstat
- sar命令
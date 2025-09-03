# 进程内存空间管理

## 概述
通过Android开机启动流程，深入理解进程空间是如何创建和管理的，从系统启动到应用运行的完整内存空间演进过程。

## 主要内容

### 4.1 Android启动序列与进程空间诞生
- [ ] Bootloader阶段：内存初始化
- [ ] Kernel启动：虚拟内存系统建立
- [ ] Init进程：第一个用户空间进程
- [ ] Zygote进程：应用进程的"孵化器"

### 4.2 从Zygote到应用进程
- [ ] Zygote预加载机制
- [ ] fork()系统调用与进程空间复制
- [ ] Copy-On-Write在进程创建中的应用
- [ ] 进程空间的差异化演进

### 4.3 进程内存空间结构详解
- [ ] 虚拟地址空间布局（0x00000000 - 0xFFFFFFFF）
- [ ] 内存段划分与作用
  - 代码段(.text)：只读可执行
  - 数据段(.data/.bss)：全局变量存储
  - 堆区(heap)：动态分配内存
  - 栈区(stack)：函数调用与局部变量
  - 内存映射区：shared libraries和mmap区域

### 4.4 System Server进程特殊性
- [ ] System Server的启动过程
- [ ] 系统服务的内存空间管理
- [ ] Binder线程池内存模型
- [ ] 与应用进程的内存交互

### 4.5 应用进程的内存空间管理
- [ ] ActivityThread主线程模型
- [ ] Application类加载与内存空间
- [ ] 四大组件的内存空间关系
- [ ] ClassLoader的内存空间隔离

### 4.6 进程间内存共享机制
- [ ] Anonymous Shared Memory (ashmem)
- [ ] Binder机制的内存管理
- [ ] Shared Memory在多媒体中的应用
- [ ] 内存映射文件的进程间共享

### 4.7 进程生命周期与内存回收
- [ ] 进程优先级调整机制
- [ ] Low Memory Killer的触发条件
- [ ] 进程终止时的内存清理
- [ ] 进程重启与内存状态恢复

### 4.8 堆空间的层次结构深度解析
- [ ] 进程堆段、Native堆、ART堆的三层关系
- [ ] Zygote启动过程中的堆空间创建时序
- [ ] Android 8.0图片内存架构重大变化
- [ ] "Java堆"的真实含义：ART虚拟机在Native堆基础上的抽象

### 4.9 栈空间的双重世界对照
- [ ] 进程栈段 vs ART Java栈的本质差异
- [ ] Native栈帧与Java栈帧的结构对比
- [ ] 汇编级别与字节码级别的栈操作机制
- [ ] JNI调用时的栈切换和异常处理栈展开
- [ ] **Android特色**: 主线程使用进程原始栈，普通线程独立分配栈空间

### 4.10 实战案例：Activity加载图片的完整内存流程
- [ ] CPU与GPU内存交互的四个阶段
- [ ] ImageView vs OpenGL ES渲染对比
- [ ] BufferQueue生产者-消费者模型的关键作用
- [ ] Android 8前后图片内存分配位置的差异分析

## 详细分析文件
- `启动流程分析.md` - Android开机启动中的进程内存空间演进
- `进程内存布局详解.md` - 32位/64位进程的虚拟地址空间结构
- `堆空间层次结构深度解析.md` - 进程堆段→Native堆→ART堆的三层关系分析
- `栈空间的双重世界对照.md` - Native栈与Java栈的结构和机制深度对比
- `Android主线程与普通线程的栈差异.md` - 主线程使用进程栈vs普通线程独立栈的机制分析
- `Android8图片内存架构变化.md` - 图片像素数据从Java堆迁移到Native堆的重大变化
- `Activity加载图片的内存交互.md` - CPU/GPU内存交互实战案例

## 实例代码
参见 `../resources/code-examples/process-memory/`

## 相关工具
- procfs (/proc/[pid]/)
- dumpsys meminfo
- smaps分析
- GPU Memory Profiler
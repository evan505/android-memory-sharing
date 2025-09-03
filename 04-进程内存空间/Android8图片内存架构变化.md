# Android 8.0 图片内存架构重大变化

## 概述
Android 8.0 (API 26) 引入了一个重大的内存管理优化：将 Bitmap 的像素数据从 Java 堆移动到 Native 堆。这个变化深刻影响了图片内存的管理方式，需要从进程空间的角度来理解。

## 变化前后对比 (Android 7 vs Android 8+)

### Android 7 及之前：Bitmap 像素数据在 Java 堆

#### 内存布局
```
进程虚拟地址空间 (4GB)
├── Kernel Space (1GB)
├── Stack
├── Memory Mapping Region 
├── Native Heap (malloc分配)
│   └── JNI对象、C++对象
└── Java Heap (ART虚拟机管理)
    ├── Java对象 (Bitmap对象本身)
    └── 像素数据 (ARGB数组) ← 在这里！
```

#### 代码层面
```kotlin
// Android 7: Bitmap像素数据在Java堆中
val bitmap = BitmapFactory.decodeResource(resources, R.drawable.large_image)
// 像素数据占用Java堆空间，受dalvik.vm.heapsize限制
```

#### 内存分配调用栈
```
Java: BitmapFactory.decodeResource()
  ↓
Native: BitmapFactory_nativeDecodeResource()
  ↓  
Native: 调用 SkBitmap::allocPixels()
  ↓
ART: 在Java堆中分配像素数组
  ↓
进程空间: 像素数据位于ART管理的堆区域
```

### Android 8.0+：Bitmap 像素数据移至 Native 堆

#### 新的内存布局
```
进程虚拟地址空间 (4GB/128TB)
├── Kernel Space
├── Stack  
├── Memory Mapping Region
├── Native Heap (malloc分配)
│   ├── JNI对象、C++对象
│   └── Bitmap像素数据 ← 移动到这里！
└── Java Heap (ART虚拟机管理)  
    └── Java对象 (Bitmap对象本身，只包含元数据)
```

#### 代码层面保持不变
```kotlin
// Android 8+: 代码完全相同，但内存分配位置改变
val bitmap = BitmapFactory.decodeResource(resources, R.drawable.large_image)
// 像素数据现在在Native堆，不受Java堆大小限制
```

#### 新的内存分配调用栈  
```
Java: BitmapFactory.decodeResource()
  ↓
Native: BitmapFactory_nativeDecodeResource()
  ↓
Native: 调用 SkBitmap::allocPixels()
  ↓
Native Heap: malloc() 直接分配像素数组
  ↓
进程空间: 像素数据位于Native堆区域
```

## 进程空间中的堆层次结构详解

### 1. 进程虚拟地址空间的堆区域
```
进程空间堆区域 (Heap Segment)
┌─────────────────────────────────────┐
│           Process Heap              │
│                                     │  
│  ┌─────────────────────────────────┐│ ↑ 向高地址增长
│  │        Native Heap              ││
│  │  - malloc/free 管理             ││  
│  │  - C/C++ 对象                   ││
│  │  - JNI 分配的内存               ││
│  │  - Android 8+ Bitmap像素数据    ││
│  └─────────────────────────────────┘│
│                                     │
│  ┌─────────────────────────────────┐│
│  │        ART Java Heap            ││
│  │  - ART虚拟机管理                ││
│  │  - Java 对象                    ││
│  │  - Android 7- Bitmap像素数据    ││
│  │  - GC 自动回收                  ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
```

### 2. Zygote 启动时的堆空间划分

#### Zygote 进程启动序列
```
1. Init进程启动 → 创建进程空间，分配基础堆区域
   ↓
2. Zygote启动 → 在进程堆中申请一块区域给ART虚拟机
   ↓  
3. ART虚拟机初始化 → 在分配的区域内创建Java堆
   ↓
4. 预加载系统类 → Java对象填充到ART堆中
   ↓
5. 应用fork → 复制整个进程空间（包括堆布局）
```

#### 具体的内存申请过程
```cpp
// Zygote启动时在native代码中
void Runtime::Init() {
    // 1. 从进程堆空间申请内存给ART虚拟机
    size_t heap_size = GetMaxMemory();  // 读取dalvik.vm.heapsize
    void* heap_memory = mmap(NULL, heap_size, 
                            PROT_READ | PROT_WRITE,
                            MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    
    // 2. 在申请的内存中初始化ART堆结构
    heap_ = new Heap(heap_memory, heap_size);
    
    // 3. 这块内存从此由ART虚拟机管理，成为"Java堆"
}
```

### 3. 堆空间的所有权和管理方式

#### 进程堆 vs ART堆的关系
```
物理视角：两者都在同一个进程的堆段(Heap Segment)中
管理视角：由不同的分配器管理

进程空间视角：
┌─────────────────────────────────────────┐
│     Process Virtual Address Space       │
│  ┌───────────────────────────────────┐  │
│  │         Heap Segment              │  │
│  │                                   │  │
│  │ [Native Heap]  [ART Java Heap]    │  │  
│  │     ↑               ↑             │  │
│  │   malloc()      ART GC管理        │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘

管理者视角：
- Native Heap: libc malloc/free + jemalloc
- ART Java Heap: ART虚拟机 GC (CMS, G1等)
```

## Android 8 变化的技术细节

### 1. Bitmap类的变化

#### Android 7 的 Bitmap 结构
```cpp
// Android 7: Bitmap.java
class Bitmap {
    private long mNativeBitmap;     // 指向native Bitmap对象
    // 像素数据直接在Java堆中作为字节数组
    // private byte[] mPixels; (概念上)
}

// Native层 SkBitmap
class SkBitmap {
    void* fPixels;  // 指向Java堆中的像素数组
}
```

#### Android 8+ 的 Bitmap 结构
```cpp  
// Android 8+: Bitmap.java (外观相同)
class Bitmap {
    private long mNativeBitmap;     // 指向native Bitmap对象
    // 像素数据现在在Native堆，Java层不直接持有
}

// Native层 SkBitmap
class SkBitmap {
    void* fPixels;  // 指向Native堆中的像素数组
    
    // 新增：自定义的内存管理
    sk_sp<SkPixelRef> fPixelRef;  // 管理像素内存的生命周期
}
```

### 2. 内存分配器的变化

#### Android 7: ART堆分配器
```cpp
// ART虚拟机内部
class ArtHeapAllocator {
    void* Allocate(size_t size) {
        // 在ART管理的Java堆中分配
        return heap_->AllocObject(size);
    }
};
```

#### Android 8+: Native堆分配器
```cpp
// 使用系统malloc或jemalloc
void* AllocateBitmapPixels(size_t size) {
    // 直接从Native堆分配，绕过ART虚拟机
    return malloc(size);  // 或使用jemalloc
}
```

### 3. 垃圾回收的变化

#### Android 7: GC回收像素数据
```
GC触发 → 扫描Java堆 → 回收不可达Bitmap → 释放像素数组内存
```

#### Android 8+: 引用计数 + GC协作
```
Java Bitmap对象被GC → 调用native析构函数 → 减少像素数据引用计数 → 
引用计数为0时 → 调用free()释放Native堆内存
```

## 这个变化带来的好处

### 1. **突破Java堆大小限制**
```
Android 7:  应用可用图片内存 ≈ dalvik.vm.heapsize (通常64MB-512MB)
Android 8+: 应用可用图片内存 ≈ 系统可用物理内存 (数GB)
```

### 2. **减少GC压力**
```
Android 7:  大量图片 → Java堆快速填满 → 频繁GC → 应用卡顿
Android 8+: 像素数据不占Java堆 → GC压力大幅降低 → 应用更流畅
```

### 3. **提高内存利用效率**
```
Android 7:  Java堆碎片化严重，大图片难以分配连续内存
Android 8+: Native堆使用专业分配器(jemalloc)，内存利用率更高
```

### 4. **更好的内存共享**
```
Android 8+: 像素数据可以更容易地在进程间共享(通过ashmem)
           多个进程显示同一图片时可以共享像素内存
```

## 对开发者的影响

### 代码兼容性
```kotlin
// 应用代码完全不需要改变
val bitmap = BitmapFactory.decodeResource(resources, R.drawable.image)
// 系统自动处理内存分配位置的变化
```

### 内存监控需要调整
```kotlin
// Android 7: 监控Java堆使用
val javaHeapSize = Runtime.getRuntime().totalMemory()

// Android 8+: 需要额外监控Native堆
val debugInfo = Debug.MemoryInfo()
Debug.getMemoryInfo(debugInfo)
val nativeHeapSize = debugInfo.nativeHeap  // 包含图片内存
```

### OOM问题改善
```
Android 7:  OutOfMemoryError 经常由于图片加载触发
Android 8+: 图片引起的OOM大幅减少，主要是其他Java对象导致
```

这个架构变化体现了Android系统对内存管理的深度优化，通过重新设计堆空间的使用方式，显著提升了应用的图片处理能力和整体性能。
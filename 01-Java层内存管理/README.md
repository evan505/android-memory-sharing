# Java层内存管理：快速定位（15分钟）

## 核心问题：对象到底存在哪里？

```kotlin
fun main() {
    val user = User()  // 这个对象究竟存在内存的什么位置？
    user.processData()
}
```

## 1. 堆与栈：最基础的内存区域划分

```kotlin
class User {
    private val name: String = "开发者"  // 📍 对象数据 → Java堆
    
    fun processData() {
        val id = 12345           // 📍 局部变量 → 栈
        val user = User()        // 📍 引用在栈，对象在堆
        calculateResult(id)
    }  // ← 栈帧销毁，但堆中对象等待GC
}
```

**关键理解**：
- **栈**：方法调用和局部变量，自动管理
- **堆**：对象实例，需要GC管理

## 2. 内存分配的实际观察

打开Memory Profiler，运行这段代码：

```kotlin
fun demonstrateMemoryGrowth() {
    val largeList = mutableListOf<ByteArray>()
    repeat(50) {
        largeList.add(ByteArray(1024 * 1024))  // 每次1MB
    }
    // 📊 观察Java/Kotlin内存增长50MB
}
```

**Profiler中观察到的**：
- Java/Kotlin堆内存持续增长
- 可能触发GC事件（蓝色标记）
- 内存释放时机

## 3. 内存泄漏：最常见的问题

```kotlin
// ❌ 泄漏示例：静态引用
class DataManager {
    companion object {
        var activity: Activity? = null  // 危险！无法被GC
    }
}

// ❌ 泄漏示例：Handler
class MainActivity : AppCompatActivity() {
    private val handler = Handler()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        handler.postDelayed({
            updateUI()  // 隐式持有Activity引用
        }, 10000)
    }
}

// ✅ 解决方案：WeakReference
class SafeDataManager {
    companion object {
        private var activityRef: WeakReference<Activity>? = null
        
        fun setActivity(activity: Activity) {
            activityRef = WeakReference(activity)
        }
    }
}
```

## 4. 快速检测工具

### LeakCanary集成（30秒设置）
```kotlin
// build.gradle (app)
debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.12'
// 自动检测Activity、Fragment、ViewModel泄漏
```

### Memory Profiler关键指标
- **Java/Kotlin**：ART虚拟机堆内存 ← **我们关注的重点**
- **Native**：C/C++内存，图片像素数据
- **Graphics**：GPU显存

## 5. 引出下一层：Java堆只是冰山一角

```kotlin
// 这些对象在Memory Profiler中显示为"Java/Kotlin"堆内存
val user = User()
val bitmap = BitmapFactory.decodeResource(resources, R.drawable.image)

// 但实际上它们存在于更复杂的层次结构中：
// 🔍 Java堆 (你看到的)
//   ↓ 嵌套在
// 🔍 Native堆 (ART管理)
//   ↓ 嵌套在  
// 🔍 进程堆段 (操作系统分配)
```

**关键认知转换**：
- Memory Profiler显示的"Java堆"实际上是**ART虚拟机的Java堆**
- 这个Java堆本身**存储在Native堆中**
- Native堆又存储在**进程的堆段**中

## 💡 引入进程空间层：向下一个放大级别过渡

我们刚才看到的Java对象内存分配，只是整个内存管理的最上层。想象一下：

**📱 40x放大镜** → Java堆中的对象世界
**📱 400x放大镜** → 进程空间中的堆段结构  ← **接下来要探索的**
**📱 4000x放大镜** → 物理内存和硬件架构

现在让我们把"镜头"调到400x，深入进程空间内部...
## LiveData与RxJava的区别？如何避免LiveData的数据倒灌问题？

### **LiveData 与 RxJava 的区别**

| 特性 | LiveData | RxJava |
|------|---------|--------|
| **核心理念** | 主要用于生命周期感知的数据持有和分发 | 响应式编程框架，支持流式数据处理 |
| **生命周期感知** | 自动感知 Activity/Fragment 的生命周期，避免内存泄漏 | 默认不感知生命周期，需要手动管理订阅（可用 `AutoDispose`、`LiveDataReactiveStreams` 适配）|
| **线程切换** | 默认在主线程更新，需手动切换 | 内置 `Schedulers` 可流畅地进行线程切换 |
| **背压处理** | 不支持背压 | 支持背压 (`Flowable`) |
| **数据存储** | 仅存储最新值，旧值被覆盖 | 可以缓存、合并多个数据流 |
| **数据分发** | 仅向 **活跃的观察者** 分发 | 向所有订阅者分发（可配置 `Observable`, `Single`, `Flowable`, `Maybe`）|
| **事件传播** | 仅当有活跃观察者时触发 | 事件始终被推送，即使无观察者 |

### **如何避免 LiveData 的数据倒灌问题**
数据倒灌（Data Backflow）是指 `LiveData` 由于 **粘性事件** 机制导致 **新观察者** 可能会收到之前缓存的数据（甚至是不期望的数据）。

#### **方法 1：使用 `SingleLiveEvent`（适用于一次性事件，如 Toast、导航）**
`SingleLiveEvent` 仅向 **一个观察者** 分发事件，并且每个事件 **仅触发一次**：

```kotlin
class SingleLiveEvent<T> : MutableLiveData<T>() {
    private val pending = AtomicBoolean(false)

    override fun observe(owner: LifecycleOwner, observer: Observer<in T>) {
        super.observe(owner) { t ->
            if (pending.compareAndSet(true, false)) {
                observer.onChanged(t)
            }
        }
    }

    override fun setValue(value: T?) {
        pending.set(true)
        super.setValue(value)
    }
}
```

使用：
```kotlin
val event = SingleLiveEvent<String>()

event.observe(this) { message ->
    showToast(message)
}
```

#### **方法 2：在观察者注册前清除旧数据**
如果 `LiveData` 在观察者注册前就已经持有了旧数据，可以在注册前手动清除：
```kotlin
liveData.value = null
```
然后再进行 `observe`。

#### **方法 3：手动检查数据是否已经被处理**
如果 `LiveData` 是 `Event<T>` 类型（封装数据的类），可以手动检查数据是否已经被消费：
```kotlin
class Event<T>(private val content: T) {
    private var hasBeenHandled = false

    fun getContentIfNotHandled(): T? {
        return if (hasBeenHandled) {
            null
        } else {
            hasBeenHandled = true
            content
        }
    }
}
```
使用：
```kotlin
liveData.observe(this) { event ->
    event.getContentIfNotHandled()?.let {
        showToast(it)
    }
}
```

#### **方法 4：使用 `Transformations.distinctUntilChanged()`**
如果 `LiveData` 频繁触发相同的数据，可以使用 `distinctUntilChanged()` 过滤掉重复数据：
```kotlin
val distinctLiveData = Transformations.distinctUntilChanged(liveData)
```
这样只有当数据发生 **真正变化** 时，观察者才会收到通知。

---

**总结：**
- **单次事件**（如 Toast、导航） → `SingleLiveEvent`
- **清除旧数据** → `liveData.value = null`
- **封装 `Event<T>`** → `getContentIfNotHandled()`
- **过滤重复数据** → `Transformations.distinctUntilChanged()`

如果你的数据是 **流式数据** 或者 **需要强大的数据流处理能力**，可以考虑使用 `RxJava` 或 `Flow` 代替 `LiveData`。




## ViewModel的生命周期如何与Activity/Fragment绑定？为什么旋转屏幕后数据不丢失？

### **ViewModel 的生命周期如何与 Activity/Fragment 绑定？**  

在 Android 中，`ViewModel` 的生命周期 **与其宿主组件（Activity / Fragment）绑定**，但它的存活时间 **长于** 组件的生命周期，特别是在 **配置变更（如旋转屏幕）时不会被销毁**。  

#### **1. ViewModel 如何绑定 Activity/Fragment？**  
`ViewModel` 通过 `ViewModelProvider` 进行绑定：  

- **Activity 级别绑定**：`ViewModel` 在 **Activity 存活期间可用**，即使旋转屏幕也不会丢失数据。  
  ```kotlin
  class MyActivity : AppCompatActivity() {
      private val viewModel: MyViewModel by viewModels()
  }
  ```

- **Fragment 级别绑定**：`ViewModel` 只在 **Fragment 存活期间可用**，切换 Fragment 时会被销毁。  
  ```kotlin
  class MyFragment : Fragment() {
      private val viewModel: MyViewModel by viewModels()
  }
  ```

- **多个 Fragment 共享 ViewModel**：使用 **Activity 作用域** 让多个 Fragment 共享数据。  
  ```kotlin
  class MyFragment : Fragment() {
      private val viewModel: MyViewModel by activityViewModels()
  }
  ```

---

### **为什么旋转屏幕后数据不丢失？**
旋转屏幕后，`Activity` 会被 **销毁并重建**，但 `ViewModel` 不会丢失数据，原因如下：

1. **ViewModel 存储在 `ViewModelStoreOwner` 中**  
   - `Activity` 旋转时，**系统会创建新的 Activity，但会复用原来的 `ViewModel`**。  
   - `ViewModel` 由 `ViewModelStore` 持有，`ViewModelStore` 依赖于 `LifecycleOwner`，不会因为 `Activity` 重建而销毁。  
   - 只有当 `Activity` **彻底退出（如用户按返回键）**，`ViewModel` 才会被清除。

2. **ViewModel 只在 `onCleared()` 时销毁**  
   ```kotlin
   override fun onCleared() {
       super.onCleared()
       // 当 Activity 彻底销毁时，ViewModel 资源才会释放
   }
   ```
   只有当 `Activity` **彻底销毁** 时，`ViewModel` 才会调用 `onCleared()` 释放资源。

3. **`SavedStateHandle`（可选）保存瞬时数据**  
   `ViewModel` 不能直接保存 UI 状态（如输入框内容），但可以使用 `SavedStateHandle`，即使进程被回收，也可以恢复数据：
   ```kotlin
   class MyViewModel(state: SavedStateHandle) : ViewModel() {
       val userInput = state.getLiveData<String>("input")
   }
   ```

---

### **总结**
| 绑定方式 | 生命周期 |
|----------|----------|
| `by viewModels()`（Activity） | 绑定到 Activity，旋转屏幕后不会销毁 |
| `by viewModels()`（Fragment） | 绑定到 Fragment，Fragment 销毁时会销毁 |
| `by activityViewModels()` | 绑定到宿主 Activity，可在多个 Fragment 共享数据 |

✅ **旋转屏幕后数据不丢失的原因**：`ViewModel` **存活于 `ViewModelStore` 中，不会因 Activity 重建而销毁**，直到 `Activity` 彻底退出。

而`ViewModelStore`之所以可以跨两个activity示例保存，是依赖于`onRetainCustomNonConfigurationInstance`/`getLastCustomNonConfigurationInstance`机制，对于`ComponentActivity`则是`onRetainNonConfigurationInstance`/`getLastCustomNonConfigurationInstance`，该机制可以将自定义的对象，保存在`ActivityThread`的`mLastNonConfigurationInstances`中。
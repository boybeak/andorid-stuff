## LaunchMode

Android 中的 `launch mode` 控制着一个 Activity 如何与任务栈中的其他 Activity 进行交互。它决定了一个新的 Activity 实例是如何启动的，以及如何与现有的 Activity 关联。Android 提供了四种主要的 launch mode，分别是：

1. **standard**（默认模式）：
   - 每次启动 Activity 时，都会创建一个新的实例，并将其添加到任务栈的顶部。
   - 如果你在栈中返回到该 Activity 时，会重新创建一个新的实例，而不是复用栈中的旧实例。

2. **singleTop**：
   - 如果任务栈的顶部已经存在该 Activity 的实例，则不会创建新的实例，而是直接复用栈中的顶部实例，并通过 `onNewIntent()` 方法传递新的 Intent。
   - 如果顶部没有该 Activity 的实例，则正常创建新的实例。
   
3. **singleTask**：
   - 如果该 Activity 在任务栈中已经存在，它将被复用，onNewIntent将被调用，并且这个 Activity 之上的所有 Activity 将会被销毁。
   - 如果该 Activity 不在任务栈中，系统会创建一个新的任务栈，并将该 Activity 放在栈底。

4. **singleInstance**：
   - 这个模式与 `singleTask` 类似，但它要求该 Activity 在一个单独的任务栈中运行。即，系统确保该 Activity 在其自己的任务栈中只有一个实例。
   - 该 Activity 之上的其他所有 Activity 会被添加到一个不同的任务栈中。

### 示例：
假设你有两个 Activity，`Activity A` 和 `Activity B`，并且你在 `AndroidManifest.xml` 中为 `Activity B` 设置了不同的 launch mode。

- **standard**：每次启动 `Activity B` 都会创建一个新的实例。
- **singleTop**：如果 `Activity B` 已经位于栈顶，再次启动 `Activity B` 时不会创建新实例，而是调用 `onNewIntent()`。
- **singleTask**：如果 `Activity B` 已经在栈中，栈中 `Activity B` 之上的 Activity 会被销毁。如果没有，`Activity B` 会启动一个新的任务栈。
- **singleInstance**：`Activity B` 将始终独占一个任务栈，并且不会与其他 Activity 共享栈。

你可以根据应用的需求选择不同的 `launch mode`，来控制用户导航和 Activity 的生命周期。

## onSaveInstanceState原理

好的，我们从Framework源码层面深入分析`onSaveInstanceState()`的实现原理，重点关注触发时机、数据存储和恢复机制的关键代码路径：

---

### **一、触发时机与调用链**
#### 1. **入口调用栈**
```java
// ActivityThread.java
void handleStopActivity(IBinder token, boolean show, int configChanges...) {
    ActivityClientRecord r = mActivities.get(token);
    // 关键调用路径
    r.activity.performStop(...); // -> 触发保存状态的入口
}

// Activity.java
final void performStop(...) {
    mInstrumentation.callActivityOnSaveInstanceState(this, outState);
}

// Instrumentation.java
public void callActivityOnSaveInstanceState(Activity activity, Bundle outState) {
    activity.performSaveInstanceState(outState); // -> 调用Activity的保存方法
}
```

#### 2. **Activity层实现**
```java
// Activity.java
final void performSaveInstanceState(Bundle outState) {
    // 调用开发者重写的onSaveInstanceState()
    onSaveInstanceState(outState);
    // 保存Fragment状态（如果有）
    mFragments.saveAllState();
    // 自动保存View层级状态（核心逻辑）
    if (mWindow != null) {
        mWindow.saveHierarchyState(outState);
    }
}
```

---

### **二、View系统自动保存机制**
#### 1. **View状态保存入口**
```java
// PhoneWindow.java
@Override
public Bundle saveHierarchyState() {
    Bundle outState = new Bundle();
    View decor = mDecor;
    if (decor != null) {
        // 从DecorView开始递归保存视图树状态
        SparseArray<Parcelable> states = new SparseArray<>();
        decor.saveHierarchyState(states);
        outState.putSparseParcelableArray(WINDOW_HIERARCHY_TAG, states);
    }
    return outState;
}
```

#### 2. **View层级递归保存**
```java
// View.java
public void saveHierarchyState(SparseArray<Parcelable> container) {
    dispatchSaveInstanceState(container);
}

protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
    if (mID != NO_ID && (mViewFlags & SAVE_DISABLED_MASK) == 0) {
        // 每个View通过onSaveInstanceState()生成自己的状态包
        Parcelable state = onSaveInstanceState();
        if (state != null) {
            container.put(mID, state); // 使用View ID作为Key存储
        }
    }
    // 递归保存子View状态
    if (mChildren instanceof ArrayList) {
        ArrayList<View> children = (ArrayList<View>) mChildren;
        for (View child : children) {
            if ((child.mViewFlags & PARENT_SAVE_DISABLED_MASK) == 0) {
                child.dispatchSaveInstanceState(container);
            }
        }
    }
}
```

#### 3. **典型View的保存实现**
以`EditText`为例：
```java
// TextView.java (EditText的父类)
@Override
public Parcelable onSaveInstanceState() {
    Parcelable superState = super.onSaveInstanceState();
    SavedState ss = new SavedState(superState);
    ss.text = getText(); // 保存文本内容
    ss.selectionStart = mSelectionStart;
    ss.selectionEnd = mSelectionEnd;
    return ss;
}
```

---

### **三、数据恢复流程**
#### 1. **恢复入口**
```java
// Activity.java
protected void onCreate(@Nullable Bundle savedInstanceState) {
    if (savedInstanceState != null) {
        // 自动恢复View层级状态
        mWindow.restoreHierarchyState(savedInstanceState);
        // 调用开发者自定义恢复逻辑
        onRestoreInstanceState(savedInstanceState);
    }
}
```

#### 2. **View状态恢复细节**
```java
// View.java
public void restoreHierarchyState(SparseArray<Parcelable> container) {
    dispatchRestoreInstanceState(container);
}

protected void dispatchRestoreInstanceState(SparseArray<Parcelable> container) {
    if (mID != NO_ID) {
        Parcelable state = container.get(mID);
        if (state != null) {
            // 调用View的onRestoreInstanceState()
            onRestoreInstanceState(state);
        }
    }
    // 递归恢复子View状态
    if (mChildren instanceof ArrayList) {
        ArrayList<View> children = (ArrayList<View>) mChildren;
        for (View child : children) {
            if ((child.mViewFlags & PARENT_SAVE_DISABLED_MASK) == 0) {
                child.dispatchRestoreInstanceState(container);
            }
        }
    }
}
```

---

### **四、核心设计要点**
1. **存储容器结构**
   - 使用`SparseArray<Parcelable>`存储View层级状态
   - 通过View的ID作为键值（`mID`必须是有效的android:id）

2. **自动保存条件**
   ```xml
   <!-- 开发者可通过设置android:saveEnabled属性关闭自动保存 -->
   <View
       android:id="@+id/my_view"
       android:saveEnabled="false"/>
   ```

3. **数据序列化限制**
   - 存储对象必须实现`Parcelable`接口
   - Bundle有1MB的IPC传输限制（不同厂商有差异）

4. **生命周期时序**
   ```
   onPause() -> onSaveInstanceState() -> onStop() 
   （若Activity被销毁，则后续调用onDestroy()）
   ```

---

### **五、开发者注意事项**
1. **自定义View状态保存**
```java
// 自定义View需要重写这两个方法
@Override
public Parcelable onSaveInstanceState() {
    Bundle bundle = new Bundle();
    bundle.putParcelable("superState", super.onSaveInstanceState());
    bundle.putInt("customState", mCustomValue);
    return bundle;
}

@Override
public void onRestoreInstanceState(Parcelable state) {
    if (state instanceof Bundle) {
        Bundle bundle = (Bundle) state;
        mCustomValue = bundle.getInt("customState");
        state = bundle.getParcelable("superState");
    }
    super.onRestoreInstanceState(state);
}
```

2. **与ViewModel的配合**
   - ViewModel更适合保存大数据或需要异步加载的数据
   - onSaveInstanceState()适合保存轻量级瞬态数据

---

通过源码分析可以看到，Android通过`Window`->`DecorView`->`ViewGroup`->`View`的层级递归调用，配合ID标识系统完成整个视图树的自动状态保存。开发者自定义保存逻辑时需要注意与系统自动保存机制的协作关系。

在Android中，`onSaveInstanceState()`方法的`outState`参数是一个`Bundle`对象，它用于保存Activity的状态信息。以下从源码角度分析`outState`的产生和保存位置：


`outState`的产生
`outState`的产生过程主要涉及`ActivityThread`和`Instrumentation`类。以下是关键步骤：


• 在`ActivityThread`中创建：

• 在`ActivityThread`的`callCallActivityOnSaveInstanceState()`方法中，会为`outState`创建一个`Bundle`实例：

```java
     private void callCallActivityOnSaveInstanceState(ActivityClientRecord r) {
         r.state = new Bundle(); // 创建Bundle对象
         r.state.setAllowFds(false);
         if (r.isPersistable()) {
             r.persistentState = new PersistableBundle();
             mInstrumentation.callActivityOnSaveInstanceState(r.activity, r.state,
                                                               r.persistentState);
         } else {
             mInstrumentation.callActivityOnSaveInstanceState(r.activity, r.state);
         }
     }
     ```


• 这里的`r.state`就是`onSaveInstanceState()`方法中传入的`outState`。


• 通过`Instrumentation`调用：

• `Instrumentation`类的`callActivityOnSaveInstanceState()`方法会调用`Activity`的`performSaveInstanceState()`方法，并将`outState`传递进去：

```java
     public void callActivityOnSaveInstanceState(Activity activity, Bundle outState) {
         activity.performSaveInstanceState(outState);
     }
     ```



• 在`Activity`中填充数据：

• 在`Activity`的`onSaveInstanceState()`方法中，开发者可以将需要保存的状态信息以键值对的形式存入`outState`：

```java
     protected void onSaveInstanceState(Bundle outState) {
         super.onSaveInstanceState(outState);
         outState.putString("key", "value");
     }
     ```



`outState`的保存位置
`outState`保存的位置涉及到Android系统对Activity状态的管理，具体来说，它会被传递到ActivityManager服务中，并最终保存在系统进程中。


• 传递到ActivityManager服务：

• 在`ActivityThread`的`handleStopActivity()`方法中，会将`outState`作为参数传递给`ActivityManager.getService().activityStopped()`方法：

```java
     private void handleStopActivity(IBinder token, boolean show, int configChanges, int seq) {
         ActivityClientRecord r = mActivities.get(token);
         ...
         StopInfo info = new StopInfo();
         ...
         info.state = r.state; // 将outState赋值给StopInfo的state
         mH.post(info);
     }
     ```


• `StopInfo`类实现了`Runnable`接口，其`run()`方法会调用`ActivityManager.getService().activityStopped()`：

```java
     public void run() {
         try {
             ActivityManager.getService().activityStopped(
                     activity.token, state, persistentState, description);
         } catch (RemoteException ex) {
             ...
         }
     }
     ```



• 保存在系统进程中：

• `ActivityManager.getService()`返回的是`ActivityManagerService`的代理对象，`activityStopped()`方法会将`outState`保存在系统进程中。当Activity被重新创建时，系统会从系统进程中恢复这些状态信息，并通过`onCreate()`或`onRestoreInstanceState()`方法传递给Activity。


总结

• `outState`是在`ActivityThread`中创建的`Bundle`对象，通过`Instrumentation`传递给`Activity`的`onSaveInstanceState()`方法。

• 开发者可以在`onSaveInstanceState()`中将需要保存的状态信息存入`outState`。

• `outState`最终会被传递到`ActivityManagerService`，并保存在系统进程中。当Activity重新创建时，系统会从系统进程中恢复这些状态信息。
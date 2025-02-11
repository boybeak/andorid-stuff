## `onMeasure()`、`onLayout()`、`onDraw()`的调用顺序及自定义View的关键点

在Android自定义View的开发中，`onMeasure()`、`onLayout()`和`onDraw()`的调用顺序及关键点如下：

---

### **调用顺序**
1. **`onMeasure()`**  
   确定View的尺寸，可能被多次调用。根据父容器的约束（`MeasureSpec`）计算自身宽高，并调用`setMeasuredDimension()`保存结果。

2. **`onLayout()`**  
   确定View的位置（对ViewGroup还需布局子View）。在自定义ViewGroup中需遍历子View并调用其`layout()`方法设置位置。

3. **`onDraw()`**  
   进行实际绘制，使用`Canvas`和`Paint`绘制内容。仅在需要重绘时调用（如调用`invalidate()`）。

**顺序总结**：  
`onMeasure() → onLayout() → onDraw()`

---

### **自定义View的关键点**
1. **处理自定义属性**  
   - 在`res/values/attrs.xml`中定义属性，通过`obtainStyledAttributes()`获取XML中配置的值。

2. **正确重写`onMeasure()`**  
   - 处理`MeasureSpec`的三种模式（`EXACTLY`、`AT_MOST`、`UNSPECIFIED`）。
   - 计算内容尺寸时需考虑`padding`，避免`wrap_content`与`match_parent`效果相同。

3. **支持`padding`和`margin`**  
   - 在`onMeasure()`和`onDraw()`中处理`padding`；若为ViewGroup，需在布局子View时处理`margin`。

4. **高效绘制**  
   - 避免在`onDraw()`中创建对象（如`Paint`），应在初始化时完成。
   - 使用`clipRect()`等优化绘制性能，减少过度绘制。

5. **处理交互事件**  
   - 重写`onTouchEvent()`或结合`GestureDetector`实现触摸反馈。
   - 根据状态（按下、禁用等）更新绘制，可复写`drawableStateChanged()`。

6. **自定义ViewGroup需重写`onLayout()`**  
   - 遍历子View并调用`child.layout()`设置其位置，需结合子View的测量结果和布局参数。

7. **性能优化**  
   - 避免频繁触发`invalidate()`，尽量局部刷新（如`invalidate(rect)`）。
   - 考虑硬件加速（`LAYER_TYPE_HARDWARE`）和离屏缓冲（`LAYER_TYPE_SOFTWARE`）。

---

### **示例代码片段**
#### 1. 处理`onMeasure()`（自定义View）
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int width = calculateWidth(widthMeasureSpec);
    int height = calculateHeight(heightMeasureSpec);
    setMeasuredDimension(width, height);
}

private int calculateWidth(int measureSpec) {
    int mode = MeasureSpec.getMode(measureSpec);
    int size = MeasureSpec.getSize(measureSpec);
    int desiredWidth = ... + getPaddingLeft() + getPaddingRight(); // 含padding
    return resolveSize(desiredWidth, measureSpec); // 简化模式判断
}
```

#### 2. 自定义ViewGroup的`onLayout()`
```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    int childLeft = getPaddingLeft();
    for (int i = 0; i < getChildCount(); i++) {
        View child = getChildAt(i);
        int childWidth = child.getMeasuredWidth();
        int childHeight = child.getMeasuredHeight();
        // 处理子View的margin（需强转LayoutParams）
        MarginLayoutParams params = (MarginLayoutParams) child.getLayoutParams();
        child.layout(childLeft + params.leftMargin, 
                     getPaddingTop() + params.topMargin,
                     childLeft + childWidth + params.leftMargin,
                     getPaddingTop() + childHeight + params.topMargin);
        childLeft += childWidth + params.leftMargin + params.rightMargin;
    }
}
```

---

### **总结**
- **顺序**：测量 → 布局 → 绘制。
- **关键**：正确处理尺寸计算、支持`wrap_content`和`padding`、高效绘制、事件处理及性能优化。  
- **注意**：自定义ViewGroup需额外处理子View的测量和布局逻辑，并管理`margin`等参数。



## 事件分发中`onTouch`、`onTouchEvent`、`onClick`的执行顺序，如何解决滑动冲突？

## **触摸事件的产生和分发机制**

### **1. 触摸事件的产生**
当用户在屏幕上触摸、滑动或抬起手指时，底层硬件会捕获这些动作并传递给 Android 内核，内核将其转换为输入事件（InputEvent），并通过 **InputManagerService** 发送到 **WindowManagerService**，最终传递给应用程序的 **Activity**。

### **2. 触摸事件的分发**
触摸事件的分发由 **Activity → Window → DecorView → ViewGroup → View** 逐级传递，核心方法是 `dispatchTouchEvent(MotionEvent event)`，它决定了事件是否需要向下传递还是拦截处理。

- **Activity**
  - `dispatchTouchEvent()`：将事件传递给 `Window`
  - `onTouchEvent()`：当 `Window` 和 `View` 都不处理时，Activity 可在此处理
  
- **Window**
  - `super.dispatchTouchEvent()`：交给 `DecorView` 处理

- **DecorView（ViewGroup）**
  - `dispatchTouchEvent()`：先交给 `onInterceptTouchEvent()` 判断是否拦截事件
  - `onInterceptTouchEvent()`：返回 `true` 则拦截，返回 `false` 继续传递给子 View
  - `onTouchEvent()`：如果子 View 不消费事件，最终由 `DecorView` 处理

- **ViewGroup**
  - `dispatchTouchEvent()`：调用 `onInterceptTouchEvent()` 判断是否拦截
  - `onInterceptTouchEvent()`：
    - 返回 `true`：拦截事件，自己处理（调用 `onTouchEvent()`）
    - 返回 `false`：传递给子 View
  - `onTouchEvent()`：如果子 View 不消费事件，则 ViewGroup 处理

- **View**
  - `dispatchTouchEvent()`：直接调用 `onTouch()` 或 `onTouchEvent()`
  - `onTouch()`：优先执行，如果返回 `true`，事件被消费，不再调用 `onTouchEvent()`
  - `onTouchEvent()`：处理 `ACTION_DOWN`、`ACTION_MOVE`、`ACTION_UP` 事件，如果 `View` 处理了 `ACTION_DOWN` 并返回 `true`，后续事件才能继续传递

---

## **3. 事件处理方法的执行顺序**

1. `dispatchTouchEvent()`：**分发事件**，控制事件的传递流程
2. `onInterceptTouchEvent()`（仅 ViewGroup）：
   - 返回 `true`：拦截事件，不传递给子 View，自己处理
   - 返回 `false`：事件继续传递给子 View
3. `onTouch()`（View）：
   - 如果 `setOnTouchListener()` 存在且 `onTouch()` 返回 `true`，则事件不再传递给 `onTouchEvent()`
4. `onTouchEvent()`（View）：如果 `onTouch()` 没有消费事件，则 `onTouchEvent()` 继续处理
5. `onClick()`：当 `ACTION_UP` 被 `onTouchEvent()` 处理后，触发 `onClick()`

**执行顺序总结：**
1. `dispatchTouchEvent()`（Activity → Window → DecorView → ViewGroup → View）
2. `onInterceptTouchEvent()`（ViewGroup 判断是否拦截）
3. `onTouch()`（View 的 OnTouchListener）
4. `onTouchEvent()`（View 处理事件）
5. `onClick()`（如果 `onTouchEvent()` 处理了 `ACTION_UP`，触发点击事件）

---

## **4. 如何解决滑动冲突？**

滑动冲突指的是 **父 View 和子 View 在处理触摸事件时的冲突**，通常发生在 **嵌套滑动布局** 中，比如 `ScrollView` 内部包含 `RecyclerView`。

### **常见的滑动冲突类型**
1. **外部拦截法**（推荐，适用于 ViewGroup 控制子 View 的滑动）
   - 父 View 在 `onInterceptTouchEvent()` 中拦截事件
   - 当检测到需要滑动时拦截事件，否则传递给子 View
   ```kotlin
   override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
       when (ev.action) {
           MotionEvent.ACTION_DOWN -> {
               // 不拦截，让子 View 先接收事件
               return false
           }
           MotionEvent.ACTION_MOVE -> {
               if (需要拦截滑动) {
                   return true // 拦截，自己处理
               }
           }
       }
       return super.onInterceptTouchEvent(ev)
   }
   ```

2. **内部拦截法**（适用于子 View 先消费事件，然后决定是否交给父 View）
   - 父 View 不拦截 `ACTION_DOWN`，子 View 通过 `requestDisallowInterceptTouchEvent(true)` 请求父 View 不拦截
   ```kotlin
   override fun onTouchEvent(event: MotionEvent): Boolean {
       when (event.action) {
           MotionEvent.ACTION_MOVE -> {
               if (需要父 View 处理) {
                   parent.requestDisallowInterceptTouchEvent(false) // 让父 View 拦截
               } else {
                   parent.requestDisallowInterceptTouchEvent(true) // 子 View 处理
               }
           }
       }
       return super.onTouchEvent(event)
   }
   ```

3. **NestedScrolling 机制**
   - `NestedScrollView`、`RecyclerView` 等控件支持 `NestedScrolling` 机制，可以自动协调滑动冲突。

### **总结**
- **外部拦截法**（`onInterceptTouchEvent()`）：父 View 先判断是否拦截，适用于 `ViewPager` 包裹 `RecyclerView` 的情况
- **内部拦截法**（`requestDisallowInterceptTouchEvent(true)`）：子 View 请求父 View 不拦截，适用于 `RecyclerView` 内部嵌套 `ScrollView`
- **NestedScrolling 机制**：适用于 `NestedScrollView`、`RecyclerView` 等控件

这样可以灵活控制滑动事件，避免冲突，让用户交互更加顺畅。
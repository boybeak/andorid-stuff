## Window、WindowManager、ViewRootImpl的关系  

在 Android 系统中，`Window`、`WindowManager`和`ViewRootImpl`是与窗口和视图绘制密切相关的重要组件，它们之间的关系如下：


1.Window

• 定义：`Window`是一个抽象概念，表示一个窗口，它是应用程序与用户交互的界面载体。例如，Activity 的窗口、Dialog 的窗口等。

• 作用：`Window`是一个容器，用于承载视图（`View`）层次结构。它定义了窗口的属性，如大小、位置、背景颜色、是否可触摸等。

• 与`WindowManager`的关系：`Window`通过`WindowManager`来管理。应用程序通过调用`WindowManager`的方法（如`addView`、`removeView`等）来创建和管理窗口。`WindowManager`会根据窗口的属性和请求，将窗口添加到系统窗口管理器中，并为其分配资源。


2.WindowManager

• 定义：`WindowManager`是一个系统服务，用于管理应用程序的窗口。它负责窗口的创建、销毁、布局和显示。

• 作用：

• 管理窗口的生命周期：创建、更新、销毁窗口。

• 控制窗口的层级和显示顺序：根据窗口的类型（如 Activity 窗口、Dialog 窗口、Toast 窗口等）和优先级，决定窗口的显示顺序。

• 处理窗口的布局：根据窗口的属性（如大小、位置、是否可调整大小等），计算窗口的布局参数。

• 与`Window`的关系：`WindowManager`是`Window`的管理者。应用程序通过`WindowManager`来操作窗口，例如通过`WindowManager.addView()`方法将一个视图添加到窗口中。

• 与`ViewRootImpl`的关系：`WindowManager`在创建窗口时，会创建一个`ViewRootImpl`实例，并将窗口的根视图（`View`）与`ViewRootImpl`关联起来。`ViewRootImpl`是`WindowManager`和`View`层之间的桥梁，负责将视图绘制到窗口上。


3.Root ViewImpl

• 定义：`ViewRootImpl`是 Android 系统中用于连接窗口管理器（`WindowManager`）和视图（`View`）层次结构的桥梁。它是`View`层与窗口管理器之间的通信通道。

• 作用：

• 桥接`WindowManager`和`View`：`ViewRootImpl`将`WindowManager`的窗口管理功能与`View`的绘制功能连接起来。

• 处理视图的绘制和布局：`ViewRootImpl`负责将视图层次结构绘制到窗口上。它接收来自`WindowManager`的布局参数，并将这些参数传递给视图层次结构，从而触发视图的布局和绘制。

• 处理事件分发：`ViewRootImpl`负责将系统事件（如触摸事件、键盘事件等）分发给视图层次结构。

• 与`Window`的关系：`ViewRootImpl`是`Window`的具体实现。它负责将`Window`的根视图（`View`）绘制到屏幕上，并管理视图的布局和事件分发。

• 与`WindowManager`的关系：`ViewRootImpl`是`WindowManager`的代理。`WindowManager`在创建窗口时会创建一个`ViewRootImpl`实例，并通过它来管理窗口的视图层次结构。`ViewRootImpl`会将来自`WindowManager`的指令（如布局参数、显示隐藏等）传递给视图层次结构。


总结

• `Window`是一个抽象的窗口概念，用于承载视图。

• `WindowManager`是一个系统服务，用于管理窗口的生命周期、布局和显示。

• `ViewRootImpl`是`WindowManager`和`View`层之间的桥梁，负责将视图绘制到窗口上，并处理事件分发。

三者之间的关系可以用以下流程表示：

• 应用程序通过`WindowManager`创建一个窗口（`Window`）。

• `WindowManager`创建一个`ViewRootImpl`实例，并将窗口的根视图与`ViewRootImpl`关联。

• `ViewRootImpl`负责将视图层次结构绘制到窗口上，并管理视图的布局和事件分发。



## View的测量、布局、绘制流程（onMeasure→onLayout→onDraw）  

在 Android 中，`View`的测量（`onMeasure`）、布局（`onLayout`）和绘制（`onDraw`）是视图绘制过程中的三个核心阶段。它们按照特定的顺序执行，确保视图能够正确地显示在屏幕上。以下是详细的流程和解释：


---



1.测量阶段（`onMeasure`）

• 作用：测量视图的大小。`onMeasure`方法用于确定视图的宽和高。

• 执行时机：当视图的大小尚未确定时，或者当视图的大小需要重新计算时，`onMeasure`会被调用。

• 参数：

• `widthMeasureSpec`：宽度测量规范，是一个整数，包含宽度的大小和测量模式。

• `heightMeasureSpec`：高度测量规范，是一个整数，包含高度的大小和测量模式。

• 测量模式：

• `MeasureSpec.EXACTLY`：精确值，表示父视图已经明确指定了视图的大小，视图必须严格遵守这个大小。

• `MeasureSpec.AT_MOST`：至多值，表示视图的大小不能超过父视图指定的最大值，视图可以根据内容自适应大小。

• `MeasureSpec.UNSPECIFIED`：未指定，表示父视图对视图的大小没有限制，视图可以根据自己的需求决定大小。

• 执行逻辑：

• 视图根据`widthMeasureSpec`和`heightMeasureSpec`提供的测量规范，计算自己的宽和高。

• 视图通过调用`setMeasuredDimension(width, height)`方法，设置自己的测量尺寸。

• 示例代码：

```java
  @Override
  protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
      int width = 0;
      int height = 0;

      // 根据测量规范计算宽度
      int widthMode = MeasureSpec.getMode(widthMeasureSpec);
      int widthSize = MeasureSpec.getSize(widthMeasureSpec);
      if (widthMode == MeasureSpec.EXACTLY) {
          width = widthSize;
      } else {
          width = 100; // 假设默认宽度为100
          if (widthMode == MeasureSpec.AT_MOST) {
              width = Math.min(width, widthSize);
          }
      }

      // 根据测量规范计算高度
      int heightMode = MeasureSpec.getMode(heightMeasureSpec);
      int heightSize = MeasureSpec.getSize(heightMeasureSpec);
      if (heightMode == MeasureSpec.EXACTLY) {
          height = heightSize;
      } else {
          height = 100; // 假设默认高度为100
          if (heightMode == MeasureSpec.AT_MOST) {
              height = Math.min(height, heightSize);
          }
      }

      // 设置测量尺寸
      setMeasuredDimension(width, height);
  }
  ```



---



2.布局阶段（`onLayout`）

• 作用：确定视图的位置。`onLayout`方法用于将视图放置到屏幕上。

• 执行时机：在测量阶段完成后，当视图的大小已经确定时，`onLayout`会被调用。

• 参数：

• `changed`：一个布尔值，表示视图的布局是否发生了变化。

• `l`、`t`、`r`、`b`：表示视图在父视图中的位置，分别是左边界、上边界、右边界和下边界。

• 执行逻辑：

• 视图根据父视图提供的位置参数，计算自己的位置。

• 如果是`ViewGroup`（容器），还需要遍历子视图，调用子视图的`layout`方法，为子视图分配位置。

• 示例代码：

```java
  @Override
  protected void onLayout(boolean changed, int l, int t, int r, int b) {
      if (changed) {
          // 计算视图的宽度和高度
          int width = r - l;
          int height = b - t;

          // 假设我们有一个子视图，需要将其放置在中心位置
          View child = getChildAt(0);
          if (child != null) {
              int childWidth = child.getMeasuredWidth();
              int childHeight = child.getMeasuredHeight();

              // 计算子视图的左上角位置
              int childLeft = (width - childWidth) / 2;
              int childTop = (height - childHeight) / 2;

              // 调用子视图的 layout 方法，为其分配位置
              child.layout(childLeft, childTop, childLeft + childWidth, childTop + childHeight);
          }
      }
  }
  ```



---



3.绘制阶段（`onDraw`）

• 作用：绘制视图的内容。`onDraw`方法用于将视图的内容绘制到屏幕上。

• 执行时机：在布局阶段完成后，当视图的位置已经确定时，`onDraw`会被调用。

• 参数：

• `canvas`：`Canvas`对象，表示绘图的画布。

• 执行逻辑：

• 视图通过`Canvas`对象，绘制自己的内容。例如，绘制背景、绘制文本、绘制图形等。

• 如果是`ViewGroup`，通常不需要重写`onDraw`方法，因为子视图会自行绘制。

• 示例代码：

```java
  @Override
  protected void onDraw(Canvas canvas) {
      super.onDraw(canvas);

      // 绘制背景
      canvas.drawColor(Color.BLUE);

      // 绘制文本
      Paint paint = new Paint();
      paint.setColor(Color.WHITE);
      paint.setTextSize(50);
      canvas.drawText("Hello, World!", 50, 100, paint);

      // 绘制一个矩形
      paint.setColor(Color.RED);
      canvas.drawRect(50, 150, 200, 300, paint);
  }
  ```



---



4.测量、布局和绘制的顺序

• 测量阶段（`onMeasure`）：首先测量视图的大小。

• 布局阶段（`onLayout`）：在测量完成后，确定视图的位置。

• 绘制阶段（`onDraw`）：在布局完成后，绘制视图的内容。

整个流程是递归进行的：

• 父视图调用`measure`方法测量子视图的大小。

• 父视图调用`layout`方法为子视图分配位置。

• 父视图调用`draw`方法绘制子视图的内容。


---



5.总结

• `onMeasure`：测量视图的大小。

• `onLayout`：确定视图的位置。

• `onDraw`：绘制视图的内容。

这三个方法共同完成了视图的绘制过程，缺一不可。



## 事件分发机制（Touch事件从驱动到ViewGroup的分发逻辑）  

在 Android 中，事件分发机制是处理触摸事件（Touch 事件）的核心机制。从硬件驱动层到应用程序层的视图（`View`和`ViewGroup`），事件分发机制涉及多个层次的处理逻辑。以下是详细的流程和逻辑：


1.事件的产生与传递

1.1 硬件驱动层
当用户触摸屏幕时，硬件驱动层会检测到触摸事件，并将其封装为原始的触摸事件数据，然后传递给 Android 操作系统。


1.2 系统层
Android 系统会将硬件驱动层传递过来的触摸事件封装为`MotionEvent`对象。`MotionEvent`包含了触摸事件的所有信息，如触摸点的坐标（`X`、`Y`）、触摸动作类型（如`ACTION_DOWN`、`ACTION_MOVE`、`ACTION_UP`等）。


1.3 窗口管理器（`WindowManager`）
系统层将触摸事件传递给`WindowManager`，`WindowManager`根据事件的坐标，确定事件应该被哪个窗口（`Window`）处理。`WindowManager`会将事件传递给对应窗口的`ViewRootImpl`。


1.4`ViewRootImpl`
`ViewRootImpl`是窗口和视图层次结构的桥梁。它接收到触摸事件后，会调用`View`的`dispatchTouchEvent`方法，将事件传递给视图层次结构。


2.事件分发机制
事件分发机制主要涉及三个方法：

• `dispatchTouchEvent`：事件分发方法，决定事件是否被当前视图处理，或者继续向下传递。

• `onInterceptTouchEvent`：仅在`ViewGroup`中存在，用于决定是否拦截事件，从而决定事件是否传递给子视图。

• `onTouchEvent`：事件处理方法，用于处理触摸事件。


2.1`ViewRootImpl`的事件分发
`ViewRootImpl`接收到事件后，会调用`View`的`dispatchTouchEvent`方法。例如：

```java
boolean result = mView.dispatchTouchEvent(event);
```



2.2`View`的事件分发
`View`的`dispatchTouchEvent`方法是事件分发的核心。其逻辑如下：

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    // 如果当前 View 已经被标记为不可用（如 View 已销毁），直接返回 false
    if (event == null || !isAttachedToWindow()) {
        return false;
    }

    // 如果 View 不可点击（clickable=false），直接调用 onTouchEvent，让父视图处理
    if (!isClickable()) {
        return onTouchEvent(event);
    }

    // 如果 View 可点击，先调用 onTouchListener（如果有设置）
    if (onTouchListener != null && onTouchListener.onTouch(this, event)) {
        return true;
    }

    // 如果 onTouchListener 没有处理，再调用 onTouchEvent
    return onTouchEvent(event);
}
```



2.3`ViewGroup`的事件分发
`ViewGroup`是一个容器，可以包含多个子视图。`ViewGroup`的事件分发逻辑比普通`View`更复杂，因为它需要决定是否将事件传递给子视图。其逻辑如下：

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    // 如果 ViewGroup 拦截事件，直接调用自身的 onTouchEvent 处理事件
    if (onInterceptTouchEvent(ev)) {
        return onTouchEvent(ev);
    }

    // 如果 ViewGroup 不拦截事件，将事件传递给子视图
    for (int i = 0; i < getChildCount(); i++) {
        View child = getChildAt(i);
        if (child.dispatchTouchEvent(ev)) {
            return true; // 子视图处理了事件，返回 true
        }
    }

    // 如果子视图没有处理事件，调用 ViewGroup 自身的 onTouchEvent
    return onTouchEvent(ev);
}
```


`ViewGroup`的`onInterceptTouchEvent`方法用于决定是否拦截事件：

```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
    // 默认情况下，ViewGroup 不拦截事件
    return false;
}
```


如果`onInterceptTouchEvent`返回`true`，则表示`ViewGroup`拦截事件，事件不会传递给子视图，而是直接由`ViewGroup`的`onTouchEvent`处理。


2.4`View`的事件处理（`onTouchEvent`）
如果事件没有被拦截，最终会调用`View`的`onTouchEvent`方法。其逻辑如下：

```java
public boolean onTouchEvent(MotionEvent event) {
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            // 处理按下事件
            return true; // 表示事件被处理
        case MotionEvent.ACTION_MOVE:
            // 处理移动事件
            return true;
        case MotionEvent.ACTION_UP:
            // 处理抬起事件
            return true;
        default:
            return false; // 表示事件未被处理
    }
}
```


如果`onTouchEvent`返回`true`，表示事件被处理，事件分发流程结束；如果返回`false`，事件会继续向上传递给父视图。


3.事件分发的流程总结

• 硬件驱动层：检测触摸事件，封装为原始数据。

• 系统层：将触摸事件封装为`MotionEvent`对象。

• `WindowManager`：根据事件坐标，确定事件所属窗口。

• `ViewRootImpl`：将事件传递给窗口的根视图。

• `View`的`dispatchTouchEvent`：决定事件是否被当前视图处理，或者继续向下传递。

• 如果是`ViewGroup`，还会调用`onInterceptTouchEvent`决定是否拦截事件。

• `View`的`onTouchEvent`：最终处理事件。

• 如果事件被处理（返回`true`），事件分发结束。

• 如果事件未被处理（返回`false`），事件会继续向上传递给父视图。


4.事件分发的关键点

• 事件传递方向：

• 向下传递：`ViewRootImpl`→`ViewGroup`→子视图。

• 向上传递：子视图→`ViewGroup`→父视图。

• 拦截机制：

• `ViewGroup`的`onInterceptTouchEvent`返回`true`，表示拦截事件，事件不再传递给子视图。

• 一旦`ViewGroup`拦截事件，后续事件（如`ACTION_MOVE`、`ACTION_UP`）都会直接由`ViewGroup`处理。

• 事件处理：

• 如果`onTouchEvent`返回`true`，表示事件被处理，事件分发结束。

• 如果`onTouchEvent`返回`false`，事件会继续向上传递。


5.示例
假设有一个`ViewGroup`，包含一个子视图`View`。触摸事件的处理逻辑如下：

```java
public class MyViewGroup extends ViewGroup {
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        // 拦截所有事件
        return true;
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        // 处理事件
        return true;
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // 测量逻辑
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        // 布局逻辑
    }
}

public class MyView extends View {
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        // 处理事件
        return true;
    }
}
```


• 当触摸事件发生时，`MyViewGroup`的`onInterceptTouchEvent`返回`true`，拦截事件。

• 事件直接由`MyViewGroup`的`onTouchEvent`处理，不会传递给子视图`MyView`。

通过理解事件分发机制，可以灵活地控制触摸事件的处理逻辑，实现复杂的交互效果。
## SurfaceView与TextureView的区别及使用场景

`SurfaceView` 和 `TextureView` 都是用于在 Android 上进行高效绘制的 `View`，尤其适用于视频播放、相机预览、游戏渲染等场景。它们的主要区别如下：

## **1. 区别**
| 特性            | SurfaceView                                  | TextureView                                |
|---------------|--------------------------------------|--------------------------------------|
| **渲染方式**   | 在**独立的 Surface** 线程中渲染，使用 `SurfaceHolder` | 直接绘制到 `View` 的窗口上，使用 `Canvas` 或 `SurfaceTexture` |
| **性能**      | 更高效，适合高帧率渲染，不受 UI 线程影响 | 由于依赖 `View` 体系，可能会有额外的 GPU 处理开销 |
| **是否支持透明** | 不支持（默认背景为黑色）                    | 支持透明背景                              |
| **是否支持动画** | 不支持直接变换（如缩放、旋转、移动）          | 支持 View 动画（如 `scaleX`、`rotation`）|
| **是否可以嵌套** | 不能嵌套在 `ViewGroup` 内部（会独立显示）       | 可以嵌套在 `ViewGroup` 内部              |
| **适用于 OpenGL** | 需要 `GLSurfaceView` 进行 OpenGL 渲染          | 可以直接用于 OpenGL 渲染                 |

---

## **2. 适用场景**
| 场景              | 适合使用的组件 |
|-----------------|-----------|
| **视频播放**       | `SurfaceView`（性能更优） |
| **相机预览**       | `SurfaceView`（减少延迟） |
| **游戏渲染**       | `SurfaceView` 或 `GLSurfaceView`（减少 UI 线程干扰） |
| **需要动画的 UI**   | `TextureView`（支持 `View` 动画） |
| **需要嵌套的 UI**   | `TextureView`（可以放入 `ViewGroup`） |
| **带透明效果的 UI** | `TextureView`（支持透明背景） |

---

## **3. 具体使用**
### **使用 SurfaceView**
适用于**相机预览、视频播放等高性能渲染**：
```java
public class MySurfaceView extends SurfaceView implements SurfaceHolder.Callback {
    private SurfaceHolder holder;

    public MySurfaceView(Context context) {
        super(context);
        holder = getHolder();
        holder.addCallback(this);
    }

    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        Canvas canvas = holder.lockCanvas();
        canvas.drawColor(Color.RED);
        holder.unlockCanvasAndPost(canvas);
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {}

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {}
}
```

---

### **使用 TextureView**
适用于**需要动画、透明、嵌套的 UI**：
```java
public class MyTextureView extends TextureView implements TextureView.SurfaceTextureListener {
    public MyTextureView(Context context) {
        super(context);
        setSurfaceTextureListener(this);
    }

    @Override
    public void onSurfaceTextureAvailable(SurfaceTexture surface, int width, int height) {
        Canvas canvas = lockCanvas();
        canvas.drawColor(Color.BLUE);
        unlockCanvasAndPost(canvas);
    }

    @Override
    public void onSurfaceTextureSizeChanged(SurfaceTexture surface, int width, int height) {}

    @Override
    public boolean onSurfaceTextureDestroyed(SurfaceTexture surface) { return false; }

    @Override
    public void onSurfaceTextureUpdated(SurfaceTexture surface) {}
}
```

---

## **4. 结论**
- **优先选择 `SurfaceView`**，如果不需要动画或透明效果，`SurfaceView` 更高效。
- **如果需要动画或嵌套 UI，则使用 `TextureView`**，但它会有额外的 GPU 处理开销。

如果你的项目涉及视频播放或相机预览，建议使用 `SurfaceView`，如果需要动态 UI 变换（如缩放、旋转），则选择 `TextureView`。
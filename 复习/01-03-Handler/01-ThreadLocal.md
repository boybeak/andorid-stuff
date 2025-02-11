## ThreadLocal原理

ThreadLocal 是 Java 中用于实现线程本地变量的工具类，其核心目标是**为每个线程提供独立的变量副本**，解决多线程环境下变量共享的线程安全问题。以下是其工作原理的详细解析：

---

### 一、核心思想
ThreadLocal 通过为每个线程维护一个独立的变量副本，使得不同线程间对同一 ThreadLocal 变量的操作互不干扰。这种线程隔离机制避免了同步锁的使用，提升了并发性能。

---

### 二、关键数据结构
#### 1. `Thread` 类中的 `ThreadLocalMap`
每个线程（`Thread` 对象）内部都有一个 `ThreadLocalMap`，它是 ThreadLocal 的核心存储结构：
```java
// Thread 类的成员变量
ThreadLocal.ThreadLocalMap threadLocals = null;
```

#### 2. `ThreadLocalMap` 的内部结构
- **键（Key）**：弱引用（`WeakReference`）的 `ThreadLocal` 实例。
- **值（Value）**：线程本地变量的实际副本。
- **哈希冲突处理**：使用线性探测法（开放寻址法），而非链表。

---

### 三、工作流程
#### 1. `set(T value)` 方法
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t); // 获取当前线程的 ThreadLocalMap
    if (map != null) {
        map.set(this, value); // this 是当前 ThreadLocal 实例
    } else {
        createMap(t, value); // 初始化 ThreadLocalMap
    }
}
```
- **步骤**：
  1. 获取当前线程的 `ThreadLocalMap`。
  2. 以当前 `ThreadLocal` 实例为键，存储值到 Map 中。

#### 2. `get()` 方法
```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            return (T) e.value;
        }
    }
    return setInitialValue(); // 初始化并返回初始值
}
```
- **步骤**：
  1. 获取当前线程的 `ThreadLocalMap`。
  2. 查找当前 `ThreadLocal` 实例对应的值。
  3. 若未找到，调用 `initialValue()` 初始化默认值。

#### 3. `remove()` 方法
```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null) {
        m.remove(this); // 删除当前 ThreadLocal 对应的条目
    }
}
```
- **作用**：显式清理当前线程的变量副本，避免内存泄漏。

---

### 四、内存泄漏问题与解决
#### 1. 潜在内存泄漏
- **原因**：
  - `ThreadLocalMap` 的键是弱引用（`WeakReference<ThreadLocal>`），但值是强引用。
  - 若 `ThreadLocal` 实例被回收，键变为 `null`，但值仍被 Entry 强引用，导致无法回收。
  - 若线程长时间运行（如线程池中的线程），可能积累大量无用的 Entry。

#### 2. 解决方案
1. **显式调用 `remove()`**：在使用完毕后手动清理。
2. **自动清理机制**：
   - 在 `set()` 和 `get()` 方法中，ThreadLocalMap 会检查并清理键为 `null` 的 Entry。
   - 并非完全可靠，仍需开发者主动管理。

---

### 五、设计细节
#### 1. 哈希算法
- 使用 `AtomicInteger` 生成哈希码，步长为 `0x61c88647`（黄金分割数），减少哈希冲突。

#### 2. 弱引用的键
- 弱引用允许 `ThreadLocal` 实例在没有外部强引用时被回收，防止因线程未销毁导致的内存泄漏。

#### 3. 初始容量与扩容
- 初始容量为 16，负载因子为 2/3，扩容阈值为 `容量 * 2/3`。
- 扩容时重新计算索引，清理无效 Entry。

---

### 六、典型应用场景
1. **线程间数据隔离**：如数据库连接（`Connection`）、Session 管理。
2. **避免参数传递**：在调用链中隐式传递参数（如用户身份信息）。
3. **线程安全的工具类**：如 `SimpleDateFormat` 的线程本地副本。

---

### 七、示例代码
```java
public class ThreadLocalDemo {
    private static final ThreadLocal<Integer> counter = ThreadLocal.withInitial(() -> 0);

    public static void main(String[] args) {
        Runnable task = () -> {
            int value = counter.get();
            counter.set(value + 1);
            System.out.println(Thread.currentThread().getName() + ": " + counter.get());
            counter.remove(); // 清理
        };

        new Thread(task).start();
        new Thread(task).start();
    }
}
```

---

### 八、总结
- **核心机制**：通过每个线程内部的 `ThreadLocalMap` 存储变量副本。
- **优点**：无锁化线程安全，高性能。
- **注意事项**：及时调用 `remove()` 避免内存泄漏，避免滥用导致代码复杂度增加。

理解 ThreadLocal 的工作原理有助于正确使用其解决线程隔离问题，同时规避潜在的内存风险。


## MessageQueue原理
在Android开发中，`MessageQueue`是消息机制的核心组件之一，它与`Looper`和`Handler`协同工作，实现线程间的消息传递和处理。以下是对`MessageQueue`的详细解析：


1.MessageQueue的基本概念
`MessageQueue`是一个消息队列，用于存储和管理`Message`对象。它是一个低级类，负责维护由`Looper`分发的消息列表。消息并不是直接添加到`MessageQueue`中，而是通过与`Looper`相关联的`Handler`对象来添加的。


2.MessageQueue的数据结构
`MessageQueue`内部采用单链表结构来存储消息。每个`Message`对象都有一个`next`指针，指向下一个消息。这种结构使得消息的插入和删除操作非常高效。


3.MessageQueue的关键方法

3.1`enqueueMessage(Message msg, long when)`
该方法用于将消息插入到消息队列中。它会根据消息的时间戳`when`（消息的执行时间）将消息插入到合适的位置。如果队列为空，或者新消息的时间戳小于队列中第一个消息的时间戳，则将新消息插入到队列头部。

```java
boolean enqueueMessage(Message msg, long when) {
    synchronized (this) {
        ...
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) { //根据消息时间，查找待插入位置
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; //插入消息
            prev.next = msg;
        }
        if (needWake) {
            nativeWake(mPtr); //唤醒 looper
        }
    }
    return true;
}
```



3.2`next()`
该方法用于从消息队列中取出下一个待处理的消息。如果队列为空，或者当前没有消息需要立即处理，`next()`方法会进入等待状态，释放CPU资源。

```java
Message next() {
    ...
    int nextPollTimeoutMillis = 0;
    for (;;) {
        ...
        nativePollOnce(ptr, nextPollTimeoutMillis); //等待
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else { //取出消息
                    ...
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    ...
                    return msg;
                }
            }
            ...
        }
        ...
    }
}
```



4.MessageQueue的工作原理

4.1 消息入队
当通过`Handler`发送消息时，`Handler`会调用`MessageQueue`的`enqueueMessage()`方法，将消息插入到消息队列中。消息会根据时间戳被插入到合适的位置，以保证消息按照时间顺序处理。


4.2 消息出队
`Looper`通过调用`MessageQueue`的`next()`方法从队列中取出下一个待处理的消息。如果队列为空，`next()`方法会进入等待状态，直到有新的消息到来。


4.3 等待机制
当`MessageQueue`为空时，`next()`方法会调用`nativePollOnce()`方法，让当前线程进入休眠状态，释放CPU资源。一旦有新的消息被插入到队列中，或者达到指定的等待时间，系统会唤醒休眠的线程。


5.MessageQueue与Looper、Handler的关系

• Looper：`Looper`是一个消息循环器，负责从`MessageQueue`中取出消息，并分发给对应的`Handler`进行处理。每个线程只能有一个`Looper`，而`Looper`与`MessageQueue`是一一对应的。

• Handler：`Handler`用于发送和处理消息。它与`Looper`和`MessageQueue`绑定，通过`sendMessage()`方法将消息发送到`MessageQueue`中，并在`Looper`取出消息后调用`dispatchMessage()`方法处理消息。


6.使用场景
`MessageQueue`主要用于实现线程间的消息传递和异步任务处理。例如：

• 在主线程中更新UI。

• 在子线程中执行耗时操作，并通过`Handler`将结果发送回主线程。

• 实现延时任务和定时任务。


7.注意事项

• 线程安全：`MessageQueue`的操作是线程安全的，因为它的关键方法（如`enqueueMessage()`和`next()`）都是同步的。

• 避免内存泄漏：如果`Handler`持有外部类的引用，可能会导致内存泄漏。建议使用静态内部类或弱引用。

• 消息循环的退出：可以通过调用`Looper.quit()`或`Looper.quitSafely()`方法退出消息循环。

总之，`MessageQueue`是Android消息机制的核心组件，它通过与`Looper`和`Handler`的协同工作，实现了高效的消息传递和处理机制。

## nativePollOnce原理

`nativePollOnce`是 Android 消息机制中的一个关键 JNI 方法，用于在 Native 层实现线程的阻塞和唤醒机制。它主要通过`epoll`机制来实现高效的事件监听和线程调度。以下是`nativePollOnce`的原理和实现细节：


1.`nativePollOnce`的作用
`nativePollOnce`是`MessageQueue`的一个本地方法，用于从 Native 层监听消息队列中的事件。它的主要作用是：

• 阻塞当前线程：如果消息队列中没有待处理的消息，线程会进入阻塞状态，释放 CPU 资源。

• 等待事件发生：监听消息队列中的事件，包括新消息的加入、定时消息的到期等。

• 唤醒线程：当有事件发生时，线程会被唤醒，继续处理消息。


2.`nativePollOnce`的实现
`nativePollOnce`的实现主要依赖于`epoll`机制。以下是其实现的关键步骤：


2.1JNI 方法调用
`nativePollOnce`的 JNI 方法定义如下：

```cpp
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj, jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
```

这个方法会将调用转发到`NativeMessageQueue`的`pollOnce`方法。


2.2`NativeMessageQueue::pollOnce`方法
`NativeMessageQueue::pollOnce`方法会进一步调用`Looper`的`pollOnce`方法：

```cpp
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis);
    mPollObj = NULL;
    mPollEnv = NULL;
    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```

这里的核心逻辑是调用`Looper`的`pollOnce`方法。


2.3`Looper::pollOnce`方法
`Looper::pollOnce`方法是实现线程阻塞和唤醒的核心。它的主要逻辑如下：

```cpp
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        // 处理已有的响应事件
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) {
                // 处理没有回调函数的响应
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
                if (outFd != nullptr) *outFd = fd;
                if (outEvents != nullptr) *outEvents = events;
                if (outData != nullptr) *outData = data;
                return ident;
            }
        }

        if (result != 0) {
            return result;
        }

        // 调用 pollInner 方法，监听事件
        result = pollInner(timeoutMillis);
    }
}
```

`pollInner`方法是实现线程阻塞和唤醒的核心部分。


2.4`Looper::pollInner`方法
`pollInner`方法通过`epoll_wait`系统调用实现线程的阻塞和唤醒：

```cpp
int Looper::pollInner(int timeoutMillis) {
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
    if (eventCount == 0) {
        return POLL_TIMEOUT; // 超时
    }

    for (int i = 0; i < eventCount; i++) {
        if (eventItems[i].data.fd == mWakeEventFd) {
            // 清空 eventfd，使其重新变为可读监听的 fd
            awoken();
        } else {
            // 保存自定义 fd 触发的事件
            mResponses.push(eventItems[i]);
        }
    }

    // 处理消息队列中的消息
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            // 处理消息
            sp<MessageHandler> handler = messageEnvelope.handler;
            Message message = messageEnvelope.message;
            mMessageEnvelopes.removeAt(0);
            handler->handleMessage(message);
        } else {
            break;
        }
    }

    // 处理自定义 fd 的回调
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        response.request.callback->handleEvent(response.request.fd, response.events, response.request.data);
    }

    return POLL_WAKE;
}
```



3.`epoll`机制
`epoll`是 Linux 提供的一种高效的 I/O 事件监听机制。它通过`epoll_fd`来管理多个文件描述符（fd），并监听这些 fd 上的事件（如可读、可写、错误等）。`epoll_wait`会阻塞当前线程，直到有事件发生或超时。

在 Android 的消息机制中，`Looper`使用`epoll`来监听以下事件：

• `mWakeEventFd`：这是一个特殊的文件描述符，用于监听消息队列的变化。当有新消息加入时，`mWakeEventFd`会被写入数据，触发`epoll_wait`唤醒线程。

• 自定义文件描述符：用户可以通过`Looper`注册自定义的文件描述符，并监听其事件。


4.线程阻塞和唤醒

• 阻塞：当消息队列为空且没有其他事件发生时，`epoll_wait`会阻塞当前线程，释放 CPU 资源。

• 唤醒：当有新消息加入消息队列时，会通过`nativeWake`方法向`mWakeEventFd`写入数据，触发`epoll_wait`唤醒线程。


5.`nativePollOnce`的优势

• 高效：`epoll`机制比传统的`select`和`poll`更高效，特别是在处理大量文件描述符时。

• 低功耗：线程在没有事件时会进入阻塞状态，减少 CPU 的占用，降低功耗。

• 灵活：支持多种事件类型，包括消息队列事件和自定义文件描述符事件。


6.总结
`nativePollOnce`是 Android 消息机制中的一个关键方法，通过`epoll`机制实现了线程的高效阻塞和唤醒。它不仅支持消息队列的事件监听，还支持自定义文件描述符的事件监听，为 Android 的线程间通信和异步任务处理提供了强大的支持。
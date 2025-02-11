## LocalBroadcastManager原理

`LocalBroadcastManager`是 Android 提供的一个用于在应用内部进行广播通信的工具类。它的原理主要基于以下几点：


1.单例模式
`LocalBroadcastManager`使用单例模式确保在整个应用中只有一个实例。通过`getInstance(Context context)`方法获取实例：

```java
public static LocalBroadcastManager getInstance(Context context) {
    synchronized(mLock) {
        if (mInstance == null) {
            mInstance = new LocalBroadcastManager(context.getApplicationContext());
        }
        return mInstance;
    }
}
```


• 作用：确保所有广播的发送和接收都在同一个实例中管理，避免重复创建实例导致的资源浪费。

• 线程安全：通过`synchronized`确保线程安全。


2.数据结构
`LocalBroadcastManager`使用以下数据结构来管理广播接收器和广播记录：

• `mReceivers`：`HashMap<BroadcastReceiver, ArrayList<ReceiverRecord>>`，用于存储注册的`BroadcastReceiver`和其对应的`ReceiverRecord`。

• `mActions`：`HashMap<String, ArrayList<ReceiverRecord>>`，用于根据`Intent`的`action`快速查找对应的接收器。

• `mPendingBroadcasts`：`ArrayList<BroadcastRecord>`，用于存储待处理的广播记录。


3.广播发送与接收

发送广播
通过`sendBroadcast(Intent intent)`方法发送广播：

```java
public void sendBroadcast(Intent intent) {
    synchronized (mReceivers) {
        // 获取所有匹配的接收器
        List<ReceiverRecord> receivers = getReceivers(intent);
        if (receivers != null) {
            for (int i = 0; i < receivers.size(); i++) {
                receivers.get(i).broadcasting = false;
            }
            mPendingBroadcasts.add(new BroadcastRecord(intent, receivers));
            // 向消息队列放入消息，表示有广播可以分发
            if (!mHandler.hasMessages(MSG_EXEC_PENDING_BROADCASTS)) {
                mHandler.sendEmptyMessage(MSG_EXEC_PENDING_BROADCASTS);
            }
        }
    }
}
```


• 流程：

• 获取所有匹配的`BroadcastReceiver`。

• 将广播记录添加到`mPendingBroadcasts`中。

• 通过`Handler`发送消息，触发广播分发。


广播分发
在`Handler`的`handleMessage`方法中处理消息，调用`executePendingBroadcasts()`方法分发广播：

```java
private void executePendingBroadcasts() {
    while (true) {
        BroadcastRecord[] brs;
        synchronized (mReceivers) {
            int N = mPendingBroadcasts.size();
            if (N <= 0) {
                return;
            }
            brs = new BroadcastRecord[N];
            mPendingBroadcasts.toArray(brs);
            mPendingBroadcasts.clear();
        }
        for (int i = 0; i < brs.length; i++) {
            BroadcastRecord br = brs[i];
            for (int j = 0; j < br.receivers.size(); j++) {
                ReceiverRecord rec = br.receivers.get(j);
                if (!rec.dead) {
                    rec.receiver.onReceive(mAppContext, br.intent);
                }
            }
        }
    }
}
```


• 流程：

• 从`mPendingBroadcasts`中获取所有待处理的广播记录。

• 遍历每个广播记录，调用每个`BroadcastReceiver`的`onReceive`方法。

• 注意：广播分发在主线程中进行，因此接收器的`onReceive`方法中不能执行耗时操作，否则会导致主线程阻塞。


4.注册与注销

注册接收器
通过`registerReceiver(BroadcastReceiver receiver, IntentFilter filter)`方法注册接收器：

```java
public void registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
    synchronized (mReceivers) {
        ReceiverRecord record = new ReceiverRecord(filter, receiver);
        ArrayList<ReceiverRecord> list = mReceivers.get(receiver);
        if (list == null) {
            list = new ArrayList<>();
            mReceivers.put(receiver, list);
        }
        list.add(record);

        ArrayList<ReceiverRecord> actionList = mActions.get(filter.getAction(0));
        if (actionList == null) {
            actionList = new ArrayList<>();
            mActions.put(filter.getAction(0), actionList);
        }
        actionList.add(record);
    }
}
```


• 流程：

• 创建`ReceiverRecord`对象。

• 将接收器添加到`mReceivers`和`mActions`中。


注销接收器
通过`unregisterReceiver(BroadcastReceiver receiver)`方法注销接收器：

```java
public void unregisterReceiver(BroadcastReceiver receiver) {
    synchronized (mReceivers) {
        ArrayList<ReceiverRecord> list = mReceivers.get(receiver);
        if (list != null) {
            for (ReceiverRecord record : list) {
                record.dead = true;
            }
            mReceivers.remove(receiver);
        }
    }
}
```


• 流程：

• 标记接收器为“已注销”（`dead = true`）。

• 从`mReceivers`中移除接收器。


5.线程安全
`LocalBroadcastManager`在所有涉及`mReceivers`和`mActions`的操作中都使用了`synchronized`锁，确保线程安全：

```java
synchronized (mReceivers) {
    // 操作 mReceivers 和 mActions
}
```


• 作用：防止多线程并发操作导致的数据不一致问题。


6.效率与安全性

• 效率：`LocalBroadcastManager`通过`Handler`和主线程消息队列进行广播分发，避免了跨进程通信（IPC）的开销，因此比全局广播更高效。

• 安全性：广播仅在应用内部传播，不会泄露到外部，也不会接收外部应用的广播，避免了隐私数据泄露和安全风险。


总结
`LocalBroadcastManager`的原理基于单例模式、线程安全的数据结构（`HashMap`和`ArrayList`）、`Handler`消息机制以及高效的广播分发逻辑。它通过这些机制实现了应用内部的高效、安全的广播通信。

## 有序广播

在Android开发中，有序广播（Ordered Broadcast）是一种特殊的广播机制，它允许多个接收器按照优先级顺序接收广播消息，并且可以在接收过程中被中断。以下是有序广播的优先级策略及相关细节：


有序广播的优先级策略

• 优先级的设置

• 静态注册的接收器：在`AndroidManifest.xml`中通过`<intent-filter>`的`android:priority`属性设置优先级。例如：

```xml
     <receiver android:name=".MyReceiver">
         <intent-filter android:priority="100">
             <action android:name="com.example.MY_ACTION"/>
         </intent-filter>
     </receiver>
     ```


• 动态注册的接收器：在代码中通过`IntentFilter`的`setPriority()`方法设置优先级。例如：

```java
     IntentFilter filter = new IntentFilter("com.example.MY_ACTION");
     filter.setPriority(100);
     registerReceiver(myReceiver, filter);
     ```



• 优先级的处理规则

• 优先级范围：优先级是一个整数，默认值为0。较大的数值表示更高的优先级。

• 接收顺序：优先级高的接收器会先接收到广播消息。如果多个接收器的优先级相同，则动态注册的接收器会排在静态注册的接收器之前。

• 中断广播：高优先级的接收器可以在`onReceive()`方法中调用`abortBroadcast()`来中断广播的传播，使得低优先级的接收器无法接收到该广播。


• 动态注册与静态注册的优先级比较

• 动态注册的接收器和静态注册的接收器可以共存。当优先级相同时，动态注册的接收器会优先于静态注册的接收器接收广播。

• 动态注册的接收器的生命周期与注册它的组件（如`Activity`或`Service`）相关，而静态注册的接收器的生命周期与应用的整个生命周期相关。


有序广播的发送
发送有序广播需要使用`sendOrderedBroadcast()`方法，而不是普通的`sendBroadcast()`方法。`sendOrderedBroadcast()`方法允许设置广播的优先级和是否允许中断广播。例如：

```java
Intent intent = new Intent("com.example.MY_ACTION");
sendOrderedBroadcast(intent, null, null, null, Activity.RESULT_OK, null, null);
```



示例
假设我们有两个广播接收器，`ReceiverA`和`ReceiverB`，它们的优先级分别为100和50：

```xml
<!-- ReceiverA -->
<receiver android:name=".ReceiverA">
    <intent-filter android:priority="100">
        <action android:name="com.example.MY_ACTION"/>
    </intent-filter>
</receiver>

<!-- ReceiverB -->
<receiver android:name=".ReceiverB">
    <intent-filter android:priority="50">
        <action android:name="com.example.MY_ACTION"/>
    </intent-filter>
</receiver>
```


当发送有序广播时：

```java
Intent intent = new Intent("com.example.MY_ACTION");
sendOrderedBroadcast(intent, null, null, null, Activity.RESULT_OK, null, null);
```



• `ReceiverA`会先接收到广播，因为它具有更高的优先级。

• 如果`ReceiverA`调用了`abortBroadcast()`，则`ReceiverB`将无法接收到该广播。

• 如果`ReceiverA`没有中断广播，则`ReceiverB`会按顺序接收到广播。


总结
有序广播的优先级策略允许开发者通过设置优先级来控制广播接收器接收消息的顺序，并且可以在高优先级的接收器中中断广播的传播。这种机制在需要按顺序处理广播消息或在某些情况下阻止消息传播时非常有用。
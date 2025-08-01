[一看就会 Android框架DataBinding的使用与封装Android中DataBinding的封装 先简单的介绍 - 掘金](https://juejin.cn/post/7098510891187961869?searchId=20250724103946B0CEE6D560518C803AA9#heading-2) [用Kotlin Flow解决Android开发中的痛点问题本文将通过实际业务场景阐述如何使用Kotlin Flow解决A - 掘金](https://juejin.cn/post/7031726493906829319?searchId=20250724161730F3FCC1BCAE2C0BA197C8)

[MVVM 进阶版：MVI 架构了解一下~MVVM架构被官方推荐，成为Android开发中的主流架构。不过软件开发中没有银 - 掘金](https://juejin.cn/post/7022624191723601928?searchId=20250724211452DE3DCC033022757983EF)

[OkHttp源码之深度解析（五）——CacheInterceptor详解：缓存机制“我报名参加金石计划1期挑战——瓜分1 - 掘金](https://juejin.cn/post/7142894768539271198)

mvi 事件流驱动 代理类 吸附组件（点击事件分发拦截处理，自定义view（添加动画效果）） animation动画效果 flow流 rxjava databinding okhttp拦截器 retrofit disposable自动处理rxjava关闭 eventbus自动根据生命周期注册和取消 mmkv handler webview

桌面小组件：支持的view种类少，且不能自定义view

刷新UI：

```javascript
AppWidgetManager.getInstance(context).updateAppWidget(appWidgetIds, remoteViews)
```

点击事件：pendingintent 添加

*系统每次触发调用AppWidget的onUpdate后的都会生成一个新的对象，所以不要在AppWidgetProvider使用全局变量，因为很有可能拿到null对象*

桌面小组件的本质是一个BroadcastReceiver，因此，小组件具备BroadcastReceiver的所有特征，我们可以通过在AndroidMenifest.xml里面注册各种Action来实现通讯，另外桌面小组件在添加和删除时会有一些特定的Action的事件调用，如下：

- a.添加新的AppWidget：  
  1.1.第一个AppWidget:会先依次调用APPWIDGET_ENABLED->APPWIDGET_UPDATE-APPWIDGET_UPDATE_OPTIONS  
  1.2.添加第二个AppWidget：会先依次调用APPWIDGET_UPDATE->APPWIDGET_UPDATE_OPTIONS

- b.删除AppWidget  
  2.1.非最后一个相同AppWidget：会先调用APPWIDGET_DELETED  
  2.2.最后一个相同AppWidget：会先依次调用APPWIDGET_DELETED->APPWIDGET_DISABLED

- c.系统自动更新AppWidget：会调用APPWIDGET_UPDATE

小组件内启动前台服务

自动拆装箱：集合类型只能存储对象类型的数据，而基本数据类型不能直接作为集合元素

原子性，可见性，一致性

## 1. 事件分发流程简述

- 触摸事件（如点击、滑动）首先由 Activity 的 `dispatchTouchEvent()` 接收。
- Activity 会将事件传递给其根 View（通常是 DecorView）。
- 根 View（ViewGroup）会调用自己的 `dispatchTouchEvent()`，然后决定是否将事件分发给子 View。
- 如果是 ViewGroup，会遍历子 View，判断事件是否落在子 View 区域内，只有**命中的子 View**才会收到事件（即调用其 `dispatchTouchEvent()`）。
- 如果是普通 View（非 ViewGroup），就直接调用自己的 `onTouchEvent()`。

---

## 2. 不是所有 View 都会收到事件

- 只有**事件命中的 View**（即触摸点落在该 View 区域内）才会收到事件分发。
- 没有命中的 View，`dispatchTouchEvent()` 根本不会被调用。

---

## 3. 在 `dispatchTouchEvent()` 判断是否消费事件

- 每个收到事件的 View 的 `dispatchTouchEvent()` 都可以决定是否消费事件。
- 如果 `dispatchTouchEvent()` 返回 `true`，表示事件被消费，不再向下传递。
- 如果返回 `false`，事件会继续向上传递（比如父 View 的 `onTouchEvent()`）。

---

扩大view响应范围

## 1. **为什么要在父 View 判断？**

- Android 的事件分发机制决定了：只有触摸点落在 View 的“命中区域”内，事件才会被分发给该 View。
- 如果你想让 View 响应超出自身 bounds 的区域，**必须让父 View 先“拦截”到这些区域的事件，然后手动分发给目标 View**。

---

## 2. **常见的两种做法**

### 方式一：重写父 View 的 `dispatchTouchEvent()`

- 在父 View 的 `dispatchTouchEvent()` 中，判断触摸点是否在你希望的“扩展区域”内。
- 如果在扩展区域内，可以手动调用目标 View 的 `dispatchTouchEvent()` 或 `onTouchEvent()`，让它响应事件。
- 这种方式需要你自己管理事件的分发和消费逻辑。

**示例伪代码：**

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (isInExpandedArea(ev.getX(), ev.getY())) {
        // 手动分发给目标View
        return targetView.dispatchTouchEvent(ev);
    }
    return super.dispatchTouchEvent(ev);
}
```

### 方式二：使用 `TouchDelegate`（推荐）

- Android 官方推荐的做法。
- 只需在父 View 设置一次 TouchDelegate，之后父 View 会自动帮你把扩展区域的事件分发给目标 View。
- 代码简单，维护性好。

**示例：**

```java
Rect rect = new Rect();
targetView.getHitRect(rect);
rect.inset(-100, -100); // 四周扩展100像素
parentView.setTouchDelegate(new TouchDelegate(rect, targetView));
```

---

## 3. **在父 View 的 `onTouchEvent()` 里拦截再传递？**

- 这种方式也可行，但通常不如在 `dispatchTouchEvent()` 里处理灵活。
- `onTouchEvent()` 只会在事件没有被子 View 消费时才被调用，且事件流已经到了父 View，手动再传递给子 View 可能导致事件流不自然（比如 ACTION_DOWN/ACTION_MOVE/ACTION_UP 的顺序和归属混乱）。
- 推荐优先用 `dispatchTouchEvent()` 或 `TouchDelegate`。

---

handler -> Looper -> thread

                                       messagequeue -> idlehandler （接口）

## 1. Handler 发送消息（存入 MessageQueue）

当你调用 Handler 的 `sendMessage()`、`post()` 等方法时，实际上是将一个 Message 对象插入到 MessageQueue 中。

**核心源码流程：**

1. `Handler.sendMessage(msg)`
2. 内部调用 `sendMessageAtTime(msg, uptimeMillis)`
3. 最终调用 `MessageQueue.enqueueMessage(msg, uptimeMillis)`

**简化伪代码：**

```java
public boolean sendMessage(Message msg) {
    return sendMessageDelayed(msg, 0);
}

public boolean sendMessageDelayed(Message msg, long delayMillis) {
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    msg.target = this; // 记录是哪个Handler发的
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

**MessageQueue.enqueueMessage()**  
会把 Message 按时间顺序插入到队列（链表）中。

---

## 2. Looper 取出消息并处理

Looper 是一个循环体，它不断从 MessageQueue 取出消息，并交给对应的 Handler 处理。

**核心源码流程：**

1. Looper.loop() 不断调用 `queue.next()` 取出下一个 Message
2. 取到 Message 后，调用 `msg.target.dispatchMessage(msg)`
3. `dispatchMessage()` 内部会回调 Handler 的 `handleMessage(msg)`

**简化伪代码：**

```java
public static void loop() {
    for (;;) {
        Message msg = queue.next(); // 取出消息（阻塞等待）
        if (msg == null) {
            continue;
        }
        msg.target.dispatchMessage(msg); // target就是Handler
        msg.recycle();
    }
}
```

---

## 3. Handler 处理消息

Handler 的 `dispatchMessage()` 会根据消息类型，最终调用你的 `handleMessage()` 方法。

**简化伪代码：**

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        msg.callback.run();
    } else {
        handleMessage(msg); // 你重写的处理逻辑
    }
}
```

---

## 4. 总结流程图

```
你调用 Handler.sendMessage(msg)
        ↓
Handler 调用 MessageQueue.enqueueMessage(msg)
        ↓
MessageQueue 按时间顺序存储消息
        ↓
Looper.loop() 不断调用 queue.next() 取出消息
        ↓
取到 msg 后，调用 msg.target.dispatchMessage(msg)
        ↓
Handler.dispatchMessage(msg) → Handler.handleMessage(msg)
```

---

## 5. 关键点

- **存入**：Handler 通过 MessageQueue.enqueueMessage() 存入消息队列。
- **取出**：Looper.loop() 循环从 MessageQueue 取出消息。
- **处理**：取出的消息由 Handler 处理（handleMessage）。

---

**确保每次空闲只会处理一遍idlehandler，取出前x个任务的任务处理**

## 1. 基本概念

- **Binder**：一种基于 C/S（客户端/服务端）架构的 IPC 机制。
- **ServiceManager**：Binder 服务的注册和查询中心。
- **Binder 驱动**：内核模块 `/dev/binder`，负责跨进程通信的实际数据传递。
- **Proxy/Stub**：AIDL 自动生成的代理和桩代码，隐藏底层通信细节。

---

## 2. 通信流程图

```
Client进程
   |   | 1. 通过ServiceManager查找服务   v
ServiceManager进程
   |   | 2. 返回服务的Binder代理对象   v
Client进程
   |   | 3. 通过Binder驱动发送请求   v
Binder驱动（内核空间）
   |   | 4. 通知Server进程有请求   v
Server进程
   |   | 5. 处理请求，返回结果   v
Binder驱动
   |   | 6. 返回结果给Client   v
Client进程
```

---

## 3. 关键步骤详解

### 1. 服务注册

- 服务端（Server）实现 Binder 接口（通常继承自 `Binder` 或 AIDL 生成的 Stub）。
- 服务端通过 `ServiceManager.addService()` 注册服务。

### 2. 客户端获取服务

- 客户端（Client）通过 `ServiceManager.getService()` 获取服务的 Binder 代理对象（Proxy）。

### 3. 远程调用

- 客户端调用 Proxy 的方法，Proxy 会将请求参数打包成 Parcel，通过 Binder 驱动发送到服务端。
- Binder 驱动负责在内核空间安全地转发数据。

### 4. 服务端处理

- 服务端的 Stub 接收到请求后，解包参数，执行实际逻辑，然后将结果打包返回。

### 5. 客户端接收结果

- 客户端 Proxy 解包返回值，得到调用结果。

---

## 4. 关键技术点

- **高效**：Binder 采用内存映射（mmap）和零拷贝技术，数据传递效率高。
- **安全**：Binder 驱动会校验调用者 UID/PID，支持权限控制。
- **引用计数**：Binder 支持对象引用计数，防止内存泄漏。

---

## 5. 代码示意（AIDL）

**AIDL 文件**

```aidl
// IMyService.aidl
interface IMyService {
    void sayHello();
}
```

**服务端实现**

```java
public class MyService extends Service {
    private final IMyService.Stub mBinder = new IMyService.Stub() {
        public void sayHello() {
            // 业务逻辑
        }
    };

    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```

**客户端调用**

```java
IMyService service = IMyService.Stub.asInterface(binder);
service.sayHello();
```

## RecyclerView 如何复用视图？

### 1. 传统 ListView 的问题

- ListView 通过 `convertView` 复用视图，但机制较为简单，灵活性有限。

### 2. RecyclerView 的复用机制

RecyclerView 采用了更为完善的**“回收池”**（Recycler）机制，具体流程如下：

#### （1）ViewHolder 机制

- 每个 item 由一个 ViewHolder 对象持有，ViewHolder 负责缓存 itemView 的子控件引用，避免重复 findViewById。

#### （2）回收池（Recycler）

- RecyclerView 内部有一个 Recycler（回收池），用于存放滑出屏幕的 ViewHolder。
- 当需要显示新的 item 时，RecyclerView 会优先从回收池中取出可用的 ViewHolder 进行复用，而不是每次都新建。

#### （3）复用流程

1. **滑动时，部分 item 滑出屏幕**
   - 对应的 ViewHolder 被移除屏幕，放入回收池。
2. **需要显示新 item 时**
   - RecyclerView 先从回收池查找是否有可复用的 ViewHolder。
   - 有则复用，没有则创建新的 ViewHolder。
3. **绑定数据**
   - 调用 Adapter 的 `onBindViewHolder()`，将新数据绑定到复用的 ViewHolder 上。

#### （4）回收池的分类

- 回收池会根据 ViewType 分类存储不同类型的 ViewHolder，保证不同类型的 item 不会混用 ViewHolder。

---

### 3. 关键源码片段（伪代码）

```java
// 获取可复用的ViewHolder
ViewHolder holder = recycler.getRecycledView(viewType);
if (holder == null) {
    // 没有可用的，创建新的
    holder = adapter.onCreateViewHolder(parent, viewType);
}
// 绑定数据
adapter.onBindViewHolder(holder, position);
```

---

### 4. 优势

- **高效**：大幅减少了 View 的创建和销毁次数，提升滑动流畅度。
- **灵活**：支持多种布局（线性、网格、瀑布流等），支持动画和自定义 Item 装饰。
- **解耦**：Adapter、LayoutManager、ItemAnimator 等各司其职，易于扩展。

---

## 1. Handler 内存泄漏的本质

### 1.1 Handler 的工作原理

- Handler 通过消息队列（MessageQueue）发送和处理消息（Message）。
- 每个 Message 内部有一个 `target` 字段，指向发送它的 Handler。
- Handler 通常作为 Activity 或 Fragment 的内部类（匿名类或非静态内部类），**会隐式持有外部类的引用**。

### 1.2 泄漏的典型场景

- 如果 Handler 发送了一个延迟消息（如 `postDelayed()`），消息会在一段时间后才被处理。
- 这期间，如果 Activity 已经 finish，但消息还在队列中，**Message 持有 Handler 的引用，Handler 又持有 Activity 的引用**，导致 Activity 无法被回收，发生内存泄漏。

---

activity销毁时消除messagequeue中所有任务

## 1. 非静态内部类/匿名类持有外部类引用

- **原理**：非静态内部类和匿名类会隐式持有外部类（如 Activity、Fragment）的引用。
- **常见场景**：Handler、Runnable、Thread、AsyncTask 等作为 Activity 的内部类，任务未完成时 Activity 已销毁，导致无法回收。

**示例**：

```java
public class MyActivity extends Activity {
    private Handler handler = new Handler() {
        // 持有MyActivity引用
    };
}
```

---

## 2. 静态集合或单例持有 Context

- **原理**：静态变量生命周期与应用一致，若持有 Activity、View 等 Context，会导致其无法释放。
- **常见场景**：单例模式中保存了 Activity、View、Context 的引用。

**示例**：

```java
public class MyManager {
    private static Context context;
    public static void init(Context ctx) {
        context = ctx; // 若传入Activity，会泄漏
    }
}
```

---

## 3. 资源未关闭或未释放

- **原理**：未及时关闭的资源（如 Cursor、File、Stream、Bitmap 等）会占用内存，导致泄漏。
- **常见场景**：数据库 Cursor、IO 流、Bitmap 未回收。

**示例**：

```java
Cursor cursor = db.query(...);
// 未调用 cursor.close()
```

---

## 4. 监听器、回调未解除注册

- **原理**：注册的监听器、回调等如果未在合适时机解除，会一直持有外部对象引用。
- **常见场景**：BroadcastReceiver、EventBus、View.OnClickListener、ContentObserver 等未注销。

**示例**：

```java
// 注册广播未注销
registerReceiver(receiver, filter);
// onDestroy未调用unregisterReceiver
```

---

## 5. WebView/Bitmap/Drawable 泄漏

- **WebView**：WebView 内部持有 Context，建议用 ApplicationContext 或在销毁时调用 `webView.destroy()`。
- **Bitmap/Drawable**：大图片未及时回收，或 Drawable 持有 Activity 引用。

---

## 6. Fragment 泄漏

- Fragment 生命周期管理不当，导致 Fragment 或其 View 持有 Activity 的引用。

---

## 7. Handler/MessageQueue 泄漏

- Handler 发送延迟消息，MessageQueue 持有 Handler，Handler 持有 Activity，导致泄漏。

---

## 8. 静态 View 泄漏

- 静态变量持有 View，会导致整个 View 树无法释放。

---

## 9. 线程泄漏

- 长时间运行的 Thread、TimerTask、AsyncTask 等持有 Activity/Fragment 的引用，任务未结束时页面已销毁。

---

## 10. Context 泄漏

- 使用 ApplicationContext 可以避免很多 Context 泄漏问题，但要注意不能用 ApplicationContext 替代所有场景（如 UI 相关操作）。

---

## 总结

**常见内存泄漏场景关键词**：

- 非静态内部类/匿名类
- 静态变量/单例
- 资源未关闭
- 监听器/回调未注销
- WebView/Bitmap/Drawable
- Handler/MessageQueue
- 静态 View
- 线程未结束
- Context 泄漏

**防止内存泄漏的建议**：

- 避免非静态内部类长时间持有外部类引用
- 静态变量/单例不要持有 Activity、View 等短生命周期对象
- 及时关闭资源，注销监听器
- 使用 WeakReference、ApplicationContext 等方式降低泄漏风险
- 利用 LeakCanary 等工具检测泄漏

你问的是“监听器、回调未解除注册”这一类内存泄漏中的对象（如 BroadcastReceiver、EventBus、View.OnClickListener、ContentObserver 等），**在什么情况下才会销毁**。下面详细说明：

---

## 1. 监听器/回调的生命周期

### （1）注册后，谁持有引用？

- 你注册监听器（如 BroadcastReceiver、OnClickListener）时，系统或第三方库会**持有你传入对象的引用**。
- 只要这个引用存在，对象就不会被 GC 回收。

### （2）什么时候会销毁？

- **只有在你手动解除注册（unregister）后，系统/库才会释放对监听器对象的引用**。
- 解除注册后，如果没有其他地方引用它，监听器对象就会被 GC 回收。

---

## 2. 常见场景分析

### 2.1 BroadcastReceiver

- 注册后，系统持有引用，只有调用 `unregisterReceiver()` 后，系统才释放引用。
- 如果 Activity 已销毁但未注销，BroadcastReceiver 依然被系统引用，导致 Activity 也无法被回收。

### 2.2 EventBus

- 注册后，EventBus 持有订阅者对象的引用，只有调用 `unregister()` 后才释放。
- 否则订阅者（如 Activity/Fragment）会一直被 EventBus 持有。

### 2.3 View.OnClickListener

- View 持有 OnClickListener 的引用。
- 只有 View 被销毁（如 Activity/Fragment 的 View 树被销毁），Listener 才会被回收。
- 如果 View 是静态变量或长生命周期对象，Listener 也不会被回收。

### 2.4 ContentObserver

- 注册后，系统持有 Observer 的引用，只有调用 `unregisterContentObserver()` 后才释放。

---

## 3. 典型内存泄漏场景

- **未手动解除注册**：Activity/Fragment 已销毁，但监听器/回调还被系统或库持有引用，导致无法回收。
- **静态变量持有 View/Listener**：生命周期过长，导致泄漏。

---

## 4. 总结

**只有在你主动解除注册（unregister），或者持有监听器引用的对象（如 View、系统服务、第三方库）本身被销毁时，监听器/回调才会被销毁。**

**否则，只要有引用存在，对象就不会被 GC 回收，容易导致内存泄漏。**

---

序列号作用

## 1. 唯一标识每个字节的数据

- TCP是面向字节流的协议，数据在传输过程中会被分割成多个报文段。
- 每个报文段都有一个序列号（Sequence Number），**用于唯一标识该报文段中第一个字节的数据在整个数据流中的位置**。

---

## 2. 保证数据的有序性和完整性

- 接收方根据序列号可以将收到的数据按顺序重组，即使数据包乱序到达，也能正确还原原始数据流。
- 如果某个序列号的数据丢失，接收方可以通过ACK（确认号）通知发送方重传。

---

## 3. 防止数据包重复和失效连接请求的干扰

- 每次建立连接时，双方都会选择一个**初始序列号（ISN, Initial Sequence Number）**，这个值通常是随机生成的。
- 这样可以防止历史连接中的旧数据包被误认为是当前连接的数据，提升安全性和可靠性。

---

## 4. 支持流量控制和拥塞控制

- 通过序列号和确认号，TCP可以实现滑动窗口机制，进行流量控制和拥塞控制，保证网络传输的效率和稳定性。

---

## 5. SYN序列号在三次握手中的作用

- 在三次握手时，客户端和服务器分别发送自己的SYN序列号，告诉对方“我准备从这个序列号开始发送数据”。
- 对方收到后，用ACK确认这个序列号，双方就能正确追踪和管理后续的数据传输。

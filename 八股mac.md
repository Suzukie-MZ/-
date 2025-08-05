[一看就会 Android框架DataBinding的使用与封装Android中DataBinding的封装 先简单的介绍 - 掘金](https://juejin.cn/post/7098510891187961869?searchId=20250724103946B0CEE6D560518C803AA9#heading-2) [用Kotlin Flow解决Android开发中的痛点问题本文将通过实际业务场景阐述如何使用Kotlin Flow解决A - 掘金](https://juejin.cn/post/7031726493906829319?searchId=20250724161730F3FCC1BCAE2C0BA197C8)

[MVVM 进阶版：MVI 架构了解一下~MVVM架构被官方推荐，成为Android开发中的主流架构。不过软件开发中没有银 - 掘金](https://juejin.cn/post/7022624191723601928?searchId=20250724211452DE3DCC033022757983EF)

[OkHttp源码之深度解析（五）——CacheInterceptor详解：缓存机制“我报名参加金石计划1期挑战——瓜分1 - 掘金](https://juejin.cn/post/7142894768539271198)

[Linux 进程间通信（IPC）总结 - huansky - 博客园](https://www.cnblogs.com/huansky/p/13170125.html)

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

进程隔离：虚拟内存地址

进程间通信方式：管道通信，共享内存，消息队列

binder通信机制：client ，service，servicemanager，binder驱动

binder内存管理：mmap内存映射

设计模式 ：单列模式，工厂模式，建造者模式，观察者模式，责任链模式，recyclerview

mvc, mvp ,mvvm,mvi

## 2. 为什么只有“原位置”和“原位置+oldCap”？

### **原理分析**

假设原数组长度为 oldCap，扩容后新数组长度为 newCap = 2 × oldCap。

对于每个元素，原来的下标是： $$ index_{old} = hash & (oldCap - 1) $$

扩容后，下标变为： $$ index_{new} = hash & (newCap - 1) $$

由于 newCap = 2 × oldCap，所以 newCap - 1 = 2 × oldCap - 1 = (oldCap << 1) - 1。

#### **二进制分析**

- oldCap - 1: 低位全1（比如 oldCap=16，oldCap-1=15，二进制 0000 1111）
- newCap - 1: 比 oldCap-1 多一位1（比如 newCap=32，newCap-1=31，二进制 0001 1111）

**hash & (newCap - 1)** 其实就是比 hash & (oldCap - 1) 多考虑了一位（oldCap位）。

- 如果 hash 的第 oldCap 位是 0，index_new = index_old
- 如果 hash 的第 oldCap 位是 1，index_new = index_old + oldCap



flow

协程异常处理

sharedflow 热流，一对多

channel 无订阅者时挂起，保留事件，一对一

MVI 对比MVVM的区别主要在哪 ？state，action ，effect

1. MVVM并没有约束View层与ViewModel的交互方式，具体来说就是View层可以随意调用ViewModel中的方法，而MVI架构下ViewModel的实现对View层屏蔽，只能通过发送Intent来驱动事件。
2. MVVM架构并不强调对表征UI状态的Model值收敛，并且对能影响UI的值的修改可以散布在各个可被直接调用的方法内部。而MVI架构下，Intent是驱动UI变化的唯一来源，并且表征UI状态的值收敛在一个变量里

flow操作符

如使用flowOn操作符切换协程上下文、使用buffer、conflate操作符处理背压、使用debounce操作符实现防抖、使用combine操作符实现flow的组合等等

1. flowOn —— 切换上游执行线程

---

• 作用：把 flow 上游（生产阶段）的执行上下文切换到指定协程调度器；下游继续使用原调度器。  
• 典型场景：在 IO 线程读取文件／网络，然后在 UI 线程收集并渲染。

示例：

```kotlin
val flow = flow {
    repeat(5) {
        emit(it)
        delay(100)        // 模拟耗时 IO
    }
}.flowOn(Dispatchers.IO)  // 上游放到 IO 线程执行

scope.launch(Dispatchers.Main) {   // 下游在 Main/UI 线程
    flow.collect { value ->
        println("collect $value on ${Thread.currentThread().name}")
    }
}
```

---

2. buffer —— 生产者-消费者解耦（背压缓冲）

---

• 作用：给流加一个缓冲区，并在不同调度器之间并发执行；生产者和消费者不用严格一一对应等待。  
• 典型场景：上游很快、下游很慢，想通过缓冲区抗一下“背压”。

示例（未 buffer 与 buffer 对比）：

```kotlin
val fastProducer = flow {
    repeat(10) {
        emit(it)
        println("emit $it")
    }
}.onEach { delay(50) }              // 上游 50 ms 产生一次
  .flowOn(Dispatchers.Default)

// 无 buffer：每次 emit 都要等下游处理完成
fastProducer.collect { value ->
    delay(200)                      // 下游处理较慢
    println("collect $value")
}

// buffer：下游慢、上游可继续发，最多缓存默认 64 个
fastProducer.buffer()
    .collect { value ->
        delay(200)
        println("collect $value")
    }
```

---

3. conflate —— 最新值覆盖旧值

---

• 作用：与 buffer 类似也是解耦背压，但是只保留缓冲区的“最新”元素；中间值被丢弃。  
• 典型场景：UI 刷新、进度条：只需要最新状态，不必处理全部历史。

示例：

```kotlin
flow {
    repeat(5) {
        emit(it)           // 0,1,2,3,4
        delay(50)
    }
}
.conflate()                // 或 .buffer(Channel.CONFLATED)
.collect { value ->
    delay(150)             // 消费者慢
    println("receive $value")   // 至多打印 0, ? , 4
}
```

---

4. debounce —— 静默一段时间后才发射

---

• 作用：当上游持续快速发射时，只在“最近一次值产生后经过指定时间都没有新值”时才真正发射；常用于搜索框输入防抖。  
• 典型场景：文本输入、按钮点击、滚动监听等 UI 防抖。

示例（300 ms 防抖）：

```kotlin
val queryFlow = callbackFlow<String> {
    // 假设 editText.addTextChangedListener { trySend(it.toString()) }
}

queryFlow
    .debounce(300)          // 300 ms 内有新输入就重置计时
    .collect { keyword ->
        println("Start searching: $keyword")
    }
```

---

5. combine —— 多个 Flow 按“最新”配对

---

• 作用：把多个 Flow 的“最新值”组合成一个新值，只要任意一个 Flow 发射了新值就触发。  
• 对比：

- zip：必须两个 Flow 同时拥有“下一对”元素才发射，一一配对。
- combine：类似 RxJava 的 combineLatest，谁先来都行，用彼此的“最新”元素。

示例（用户名+密码输入校验）：

```kotlin
val usernameFlow = callbackFlow<String> { /* ... */ }
val passwordFlow = callbackFlow<String> { /* ... */ }

usernameFlow
    .combine(passwordFlow) { u, p -> u to p }  // 也可写 combine(passwordFlow) { u, p -> "$u:$p" }
    .collect { (u, p) ->
        val enable = u.isNotBlank() && p.length >= 6
        loginButton.isEnabled = enable
    }
```

---

## 对比速查

• flowOn：只负责切换“上游”协程上下文，不解决背压。  
• buffer：建立队列，完整保留队列中所有元素。  
• conflate：建立大小为 1 的“最新值”缓冲区，中间值会被丢弃。  
• debounce：基于时间窗口，只有停顿期到来才发射。  
• combine：把多个 Flow 的“最新值”实时合并

1. 协程是什么  
   • Kotlin 协程是一种轻量级的并发实现，可看作“可挂起的函数”。  
   • 相比线程，协程的创建和切换开销小得多，可在同一线程上成千上万地运行。  
   • 协程依赖于挂起函数和协程调度器，实现非阻塞式的并发逻辑。

2. 基本概念  
   • 挂起函数（suspend）：用 suspend 关键字修饰，可在协程内“挂起”当前执行并让出线程。  
   • 协程作用域（CoroutineScope）：决定协程的生命周期，常见有 GlobalScope、CoroutineScope、lifecycleScope 等。  
   • 协程构建器：
   
   - launch：返回 Job，不携带结果，侧重“做某事”。
   - async：返回 Deferred，可通过 await() 获取结果，侧重“计算某值”。
   - runBlocking：阻塞当前线程，通常仅在 main 方法或单元测试中使用。  
     • 调度器（Dispatcher）：
   - Dispatchers.Default：适合 CPU 密集型任务（默认线程池）。
   - Dispatchers.IO：适合 I/O 密集型任务，如文件/网络操作。
   - Dispatchers.Main：Android UI 线程（仅限 Android）。
   - Dispatchers.Unconfined：不受限线程，先在当前线程执行，遇到挂起后再根据挂起点的线程继续执行。

3. 典型示例

```kotlin
fun main() = runBlocking {
    val job1 = launch {
        repeat(5) { i ->
            println("launch: $i")
            delay(200) // 挂起 200 ms，而不是阻塞线程
        }
    }

    val deferred = async(Dispatchers.Default) {
        // 计算 Fibonacci(30) 之类的 CPU 密集型任务
        fib(30)
    }

    println("Fibonacci = ${deferred.await()}") // 等待结果
    job1.join() // 等待子协程结束
}

// 递归斐波那契
fun fib(n: Int): Int = if (n <= 1) n else fib(n - 1) + fib(n - 2)
```

4. 取消与异常处理  
   • 协程可通过 Job.cancel()、CoroutineScope.cancel() 等方式取消。  
   • 挂起点会定期检查协程是否被取消并抛出 CancellationException。  
   • try/catch/finally 捕获异常并可在 finally 中调用 coroutineContext.cancel()。（注意：CancellationException 通常会被忽略，需要显式处理其他异常。）

```kotlin
val job = launch {
    try {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500)
        }
    } finally {
        println("Finally, I'm cancelled")
    }
}
delay(1300)
job.cancelAndJoin()
```

5. 结构化并发  
   • 子协程默认继承父作用域，父作用域取消时会自动取消所有子协程。  
   • 使用 coroutineScope { … } 或 supervisorScope { … } 管理协程层级可确保异常正确传播和资源自动回收。

6. Flow（响应式数据流）  
   • Flow 用于异步数据流，支持背压、取消、异常处理。  
   • 常用操作符：map、filter、collect、buffer、conflate、flatMapMerge 等。

```k
fun simpleFlow() = flow {
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

runBlocking {
    simpleFlow()
        .map { it * it }
        .collect { println(it) }
}
```

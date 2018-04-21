# 常见形式
Bundle、文件共享、AIDL、Messenger、ContentProvider、Socket

# 什么是IPC

IPC是Inter-Process Communication的缩写。含义为进程间通信或跨进程通信，指的是两个进程之间进行数据交换的过程。++进程和线程是截然不同的两个概念。按照操作系统中的描述，线程是CPU调度的最小单元，同时线程是一种有限的系统资源。而进程一般是指一个执行单元，在PC和移动设备上指的是一个程序或一个应用。++ 一个进程至少有一个线程，叫主线程，在Android中叫做UI线程，在UI线程中才能操作界面元素。如果在主线程中执行大量的耗时操作，会导致界面无法响应，严重影响用户体验，在Android中有个特殊的名字叫做ANR(Application Not Responding)。

IPC并不是Android中独有的，任何一个操作系统都需要有相应的IPC机制，比如Windows中可以通过剪贴板、管道、油槽等来进行进程间通信；Linux可以通过命名管道、共享内存、信号量等来进行进程间通信。

对于Android来说，它是一种基于Linux内核的操作系统，它的通信方式并不能完全继承自Linux，相反，它有自己的进程间通信方式，在Android中最具特色的通信方式是Binder，通过Binder可以轻松的实现进程间通信，另外Android还支持Socket，通过Socket也可以实现任意两个终端之间的通信，当然可以实现进程间通信。

# 多进程存在的问题
在Android中使用多进程只有一种办法，那就是给四大组件在Manifest中指定android:process属性，也就是说我们无法给一个线程或者一个实体类指定其运行时所在的线程。我们知道Android系统会为每个应用分配UID，具有相同UID的应用才能共享数据。Android系统为每个应用分配了一个独立的虚拟机，或者说为每个进程都分配了一个独立的虚拟机，不同的虚拟机在内存分配上都有不同的地址空间，这就导致了在不同的虚拟机上访问同一个类的对象会产生过个副本。因此多有运行在不同进程中的四大组件，只要他们之间需要通过内存来进行数据共享，都会失败，这也是多进程所带来的主要问题。一般来说，多进程会造成如下几方面的问题：
1. 静态成员和单例模式完全失效
1. 线程同步机制完全失效
1. SharePreferences的可靠性下降（不支持两个进程同时执行写操作，否则会导致一定几率的数据丢失）
1. Application会多次被创建

# Serializable和Parcelable
## Serializable接口
serialVersionUID的工作机制：序列化的时候系统会把当前类的serialVersionUID写入序列化的文件中（也可能是其他中介），当反序列化的时候系统会去检测文件中的serialVersionUID，看它是否和当前类的serialVersionUID一致，如果一致就说明序列化的版本和当前类的版本相同，这个时候可以成功反序列化；否则就说明当前类和反序列化的类相比发生了某些变化，比如成员变量的数量、类型可能发生了改变，这个时候是无法正常反序列化的，因此会报出` java.io.InvalidClassException` 异常。

一般来说，我们应该手动指定serialVersionUID的值，比如1L，也可以让IDE根据当前类的结构自动去生成它的hash值，这样序列化和反序列化时两者的serialVersionUID是相同的，因此可以正常进行反序列化。如果不手动指定serialVersionUID的值反序列化时当前类有所改变，那么系统就会重新计算hash值并把它赋值给serialVersionUID，这个时候当前类的serialVersionUID就和序列化的数据中的serialVersionUID不一致，于是导致序列化失败，程序就会出现crash。


## Parcelable接口
Parcelable也是一个接口，只要实现这个接口，这个类的对象就能实现序列化并可以通过Intent和Binder传递。

Parcelable的方法说明
<html>
<!--在这里插入内容-->
<table>
  <tr>
    <th>方法</th>
    <th>功能</th>
    <th>标记位</th>
  </tr>
  <tr>
    <td>createFromParcel(Parcel in)</td>
    <td>从序列化后的对象中创建原始对象</td>
    <td></td>
  </tr>
  <tr>
    <td>newArray(int size)</td>
    <td>创建指定长度的原始对象数组</td>
    <td></td>
  </tr>
  <tr>
    <td>User(Parcel in)</td>
    <td>从序列化后的对象中创建原始对象</td>
    <td></td>
  </tr>
  <tr>
    <td>writeToParcel(Parcel out, <br/>int flags)</td>
    <td>将当前对象写入序列化结构中，其中flags标识有两种值：<br/>
    0或者1（参见右侧标记位）。位1时标识当前对象需要作为返回值返回， <br/>不能立即释放资源，几乎所有情况都是0</td>
    <td>PARCELABLE_WRITE_RETURN_VALUE</td>
  </tr>
  <tr>
    <td>describeContents</td>
    <td>返回当前对象的内容描述。如果含有文件描述符，返回1（参见右侧标<br/>记位），否则返回0，几乎所有情况都是返回0.</td>
    <td>CONTENTS_FILE_DESCRIPTOR</td>
  </tr>

</table>
</html>

## 两者的区别
Serializable是Java中的序列化接口，其使用起来方便简单但是开销大，序列化和反序列化过程需要大量的I/O操作。而Parcelable是Android中的序列化方式，因此更适合在Android平台中使用，它的缺点就是使用起来稍麻烦点，但是它的效率很高，这是Android推荐的序列化方式，因此我们应首选Parcelable。Parcelable主要用在内存序列化上，也可以使用在将对象序列化到存储设备上或者将对象序列化后通过网络传输，但是这个过程稍微复杂，因此在这两个过程下还是建议使用Serializable
# Binder
- 直观的讲，Binder是Android中的一个类，它实现了IBinder接口；
- 从IPC的角度来讲，Binder是Android中的一种跨进程通信的方式；
- Binder还可以理解为一种虚拟的物理设备，它的驱动是`/dev/binder`，该通信方式在Linux中没有；
- 从Android Framework角度来讲，Binder是ServiceManager链接各种Manager(ActivityManager, WindowManager...)和相应ManagerService的桥梁；
- 从Android应用层来讲，Binder的客户端和服务器进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象客户端就可以获取服务端提供的服务或数据，这里的服务端包括普通服务和基于AIDL的服务。

# 选用合适的IPC方式

<html>
<table border="1">
  <tr>
    <th>名称</th>
    <th>优点</th>
    <th>缺点</th>
    <th>适用场景</th>
  </tr>
  <tr>
    <td>Bundle</td>
    <td>简单易用</td>
    <td>只能传递Bundle支持的数据类型</td>
    <td>四大组件之间的进程间通信</td>
  </tr>
  <tr>
    <td>文件共享</td>
    <td>简单易用</td>
    <td>不适合高并发场景，<br/>并且无法做到进程间通信的及时性</td>
    <td>无并发访问情形，<br/>交换简单的数据实时性不高的场景</td>
  </tr>
  <tr>
    <td>AIDL</td>
    <td>功能强大，<br/>支持一对多并发通信，<br/>支持实时通信</td>
    <td>适用稍复杂，需要处理好线程通信</td>
    <td>一对多通信切有RPC需求</td>
  </tr>
  <tr>
    <td>Messenger</td>
    <td>功能一般，<br/>支持一对多串行通信，<br/>支持实时通信</td>
    <td>不能很好的处理高并发的场景，<br/>不支持RPC，<br/>数据通过Message进行传输，<br/>因此只能传输Bundle支持的数据类型</td>
    <td>低并发的一对多即时通信，<br/>无RPC需求，<br/>或者无需返回结果的RPC需求</td>
  </tr>
  <tr>
    <td>ContentProvider</td>
    <td>在数据源访问方面功能强大，<br/>支持一对多并发数据共享，<br/>可通过Call方法扩展其他操作</td>
    <td>可以理解为受约束的AIDL，<br/>主要提供数据源的CRUD操作</td>
    <td>一对多的进程间数据共享</td>
  </tr>
  <tr>
    <td>Socket</td>
    <td>功能强大，<br/>可以通过网络传输字节流，<br/>支持一对多并发实时通信</td>
    <td>实现细节稍微有些繁琐，<br/>不支持直接的RPC</td>
    <td>网络数据交换</td>
  </tr>
</table>
</html>
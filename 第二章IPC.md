# 常见形式
Bundle、文件共享、AIDL、Messenger、ContentProvider、Socket

# 什么是IPC

IPC是Inter-Process Communication的缩写。含义为进程间通信或跨进程通信，指的是两个进程之间进行数据交换的过程。++进程和线程是截然不同的两个概念。按照操作系统中的描述，线程是CPU调度的最小单元，同时线程是一种有限的系统资源。而进程一般是指一个执行单元，在PC和移动设备上指的是一个程序或一个应用。++ 一个进程至少有一个线程，叫主线程，在Android中叫做UI线程，在UI线程中才能操作界面元素。如果在主线程中执行大量的耗时操作，会导致界面无法响应，严重影响用户体验，在Android中有个特殊的名字叫做ANR(Application Not Responding)。

IPC并不是Android中独有的，任何一个操作系统都需要有相应的IPC机制，比如Windows中可以通过剪贴板、管道、油槽等来进行进程间通信；Linux可以通过命名管道、共享内存、信号量等来进行进程间通信。

对于Android来说，它是一种基于Linux内核的操作系统，它的通信方式并不能完全继承自Linux，相反，它有自己的进程间通信方式，在Android中最具特色的通信方式是Binder，通过Binder可以轻松的实现进程间通信，另外Android还支持Socket，通过Socket也可以实现任意两个终端之间的通信，当然可以实现进程间通信。


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


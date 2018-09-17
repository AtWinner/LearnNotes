> 以下文章以Android4.3的源码为基础，来自《Android源码分析实录》第5章

Android的虚拟机系统是Dalvik虚拟机，是Google等商家合作开发的Android移动设备平台的核心组件之一。DalvikVM可以支持已转为.dex格式的Java应用程序的运行，其中“.dex”格式是专为DVM设计的一种压缩格式，适合内存和处理器速度都有限的系统。现实中的大多数虚拟机都是一种基于堆栈的机器，例如JVM，而DVM则是基于寄存器的。基于栈的机器需要更多指令，而基于寄存器的机器指令更大。

# 关于Android虚拟机
Android系统的应用层是采用Java开发的，由于Java语言的跨平台特性，所以Java的代码必须运行在虚拟机中。正因为这个特性，Android系统也实现了自己的一个类似JVM但是更适合嵌入式平台的虚拟机——Dalvik。Dalvik的功能等同于JVM，为Android平台上的Java代码提供了运行环境。
## Dalvik的架构
在Android源码中，DVM的实现位于dalvik/目录下，其中dalvik/vm是虚拟机的实现部分，将会变异成libdvm.so。而dalvik/libdex将会变异成libdex.a的静态库

![Dalvik和Android应用](https://upload-images.jianshu.io/upload_images/3610640-d735e6fa5b0c1325.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

DX工具用来转换Java class称为dex格式，但不是全部。多个类型包含在一个Dex文件中，多个类型中重复的字符串和其他常数在Dex中只存放一次，以节省空间。Java字节码转换成DVM所使用的替代指令集。一个未压缩Dex文件通常是稍稍小于一个已经压缩的jar文件。

再次安装到执行设备时，Dalvik可执行文件可能会是修改过的。为了获得进一步的优化，端序（Byte Order）可能会存在一定的数据交换，简单的数据结构和函数库可内联（Linked Inline），空的类型对象可能会短路处理。

当启动Android时，DVM会监视所有程序，并且创建依序关系树，为每个程序优化代码并存储在Dalvik缓存中。Dalvik第一次加载后会生成Cache文件，以提供下次快速加载，所以第一次会比较慢。

Dalvik解释器采用预先算好的Goto地址，基于每个指令集OpCode，都固定以64字节为Memory Alignment。这样可以节省一个指令集OpCode后要进行查表的时间。为了强化功能，Dalvik还提供了Fast Interpreter（快速翻译器）。

DX是一套工具，可以将Java的.class文件转换。一个Dex文件通常会有多个.class文件。Dex有时必须进行优化，这会使文件大小增加1~4倍，以ODEX结尾。

## DVM的重要特征
在DVM中，一个应用总会定义很多类，编译完成后，即会有很多响应的Class文件，Class文件间会有不少冗余的信息；而Dex文件格式会把所有的Class文件内容整合到一个文件中。这样，除了减少整体文件尺寸、IO操作，也提高了类的查找速度。原来每个类文件中的常量池，在Dex文件中由一个常量池来管理。

每一个Android应用都运行在一个DVM实例中，而每一个虚拟机实例都是一个独立的进程空间。虚拟机的线程控制、内存分配和管理、Mutex等都依赖底层的操作系统实现。所有Android应用的线程都对应一个Linux线程，虚拟机因而可以更多的依赖操作系统的线程调度和管理机制。

不同的应用在不同的进程空间里运行，加之对不同来源的应用都使用不同的Linux用户来运行，这可以最大程度的保护应用的安全和独立运行。

Zygote是一个虚拟机进程，同时也是一个虚拟机实例的孵化器，每当系统要求执行一个Android应用程序的时候，Zygote就会Fork出一个子进程来执行该应用程序。这样做的好处是：Zygote进程是在系统启动时产生的，它会完成虚拟机的初始化、库加载、预置类库的加载和初始化等操作，而在系统需要一个新的虚拟机实例时，Zygote通过复制自身，最快速的提供一个子系统。另外，对于一些只读的系统库，所有虚拟机实例都与Zygote共享一块内存区域，大大节省了内存开销。

相对于基于堆栈的虚拟机实现，基于寄存器的虚拟机实现虽然在硬件通用性上要差一些，但是它在代码的执行效率上却更胜一筹。在基于寄存器的虚拟机里，可以更有效的减少冗余指令的分发和减少内存的读写访问。

## Dalvik的进程管理
Dalvik进程管理是依赖于Linux的进程体系结构的，如要为应用程序创建一个进程，它会使用Linux的Fork机制来复制一个进程（复制进程往往比创建进程的效率更高）。

Zygote通过Init进程启动。首先会孕育出System_Server（Android绝大多数系统服务的守护进程，它会监听socket，等待请求命令，当有一个应用程序启动时，就会像它发出请求，Zygote就会Fork出一个新的应用程序进程）。每当系统要求执行一个Android应用程序时，Zygote就会运用Linux的Fork机制产生一个子进程来执行该应用程序。

## Android的初始化流程
Linux中进程间通信的方式很多，但Dalvik使用的是信号方式来完成进程间通信。Android的初始化流程如下：

![Android初始化流程](https://upload-images.jianshu.io/upload_images/3610640-5ac3cf2c24d629f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# Dalvik的运行流程
## Dalvik虚拟机相关的可执行程序
在Android源码中，我们会发现好几处与Dalvik这个概念相关的可执行程序，正确区分这些源码可执行程序的区别，将有助于理解Framework内部结构。

名称 | 源码路径
---|---
dalvikvm | [dalvik/dalvikvm/](http://androidxref.com/4.3_r2.1/xref/dalvik/dalvikvm/)
dvz | [dalvik/dvz/](http://androidxref.com/4.3_r2.1/xref/dalvik/dvz/)
app\_process | [frameworks/base/cmds/app\_process/](http://androidxref.com/4.3_r2.1/xref/frameworks/base/cmds/app_process/)

### dalvikvm
当运行Java程序时，都是由一个虚拟机来解释Java的字节码，它将这些字节码翻译成本地CPU的指令码，然后再执行。对Java程序员来说，负责解释并执行的就是一个虚拟机。而对于Linux系统而言，这个进程只是一个普通的进程，它与一个只有一行代码的HelloWorld可执行程序没有本质区别。所以启动一个虚拟机的方法就跟启动任何一个可执行程序的方法相同，那就是在命令行下输入可执行程序的名称，并在阐述中指定要执行的Java类。dalvikvm的执行语法如下：

```
dalvikvm -cp 路径名 类名
```
由此可以看到，dalvikvm的作用就像在PC上执行的Java程序一样。

### dvz
在Dalvik虚拟机中，dvz的作用是从Zygote进程中孕育出一个新的进程，新的进程也是一个Dalvik虚拟机。在该进程与dalvikvm启动的虚拟机相比，区别在于该进程中已经预装了Framework的大部分类和资源。使用dvz的语法如下：
```
dvz -classpath 包名称 类名
```
一个apk的入口类是ActivityThread类。因为Activity类仅仅是被回调的类，所以不可以通过Activity类来启动一个APK，dvz工具仅仅用于Framework开发过程的调试阶段。

### app_process
Framework在启动时需要加载并运行如下两个特定的类文件：ZygoteInit.java和SystemServer.java。为了便于使用，系统才提供了一个app\_process进程，该进程会自动运行着两个类，从这个角度来讲，app\_process的本质是使用dalvikvm来启动ZygoteInit.java，并在启动后加载Framework中的大部分类和资源。
- dalvikvm的关键代码有两处：

``` C++
    /*
     * 第一处：通过如下代码创建一个vm对象
     * Start VM.  The current thread becomes the main thread of the VM.
     * 启动VM。 当前线程成为VM的主线程。
     */
    if (JNI_CreateJavaVM(&vm, &env, &initArgs) < 0) {
        fprintf(stderr, "Dalvik VM init failed (check log file)\n");
        goto bail;
    }
    
    /*
     * Make sure they provided a class name.  We do this after VM init
     * so that things like "-Xrunjdwp:help" have the opportunity to emit
     * a usage statement.
     * 确保他们提供了一个类名。 我们在VM init之后执行此操作，以便“-Xrunjdwp：help”有机会发出一个用法语句。
     */
    if (argIdx == argc) {
        fprintf(stderr, "Dalvik VM requires a class name\n");
        goto bail;
    }

    /*
     * We want to call main() with a String array with our arguments in it.
     * Create an array and populate it.  Note argv[0] is not included.
     * 我们想用一个带有参数的String数组调用main（）。创建一个数组并填充它。 注意不包括argv [0]。
     */
    jobjectArray strArray;
    strArray = createStringArray(env, &argv[argIdx+1], argc-argIdx-1);
    if (strArray == NULL)
        goto bail;
    
    /*
     * Find [class].main(String[]).
     */
    jclass startClass;
    jmethodID startMeth;
    char* cp;

    /* convert "com.android.Blah" to "com/android/Blah" */
    slashClass = strdup(argv[argIdx]);
    for (cp = slashClass; *cp != '\0'; cp++)
        if (*cp == '.')
            *cp = '/';

    /*第二处：创建好了JavaVM对象后，就可以使用该对象去加载指定的类了*/
    startClass = env->FindClass(slashClass);
    if (startClass == NULL) {
        fprintf(stderr, "Dalvik VM unable to locate class '%s'\n", slashClass);
        goto bail;
    }

    startMeth = env->GetStaticMethodID(startClass,
                    "main", "([Ljava/lang/String;)V");
    if (startMeth == NULL) {
        fprintf(stderr, "Dalvik VM unable to find static main(String[]) in '%s'\n",
            slashClass);
        goto bail;
    }

    /*
     * Make sure the method is public.  JNI doesn't prevent us from calling
     * a private method, so we have to check it explicitly.
     * 确保该方法是公开的。 JNI并不阻止我们调用私有方法，因此我们必须明确检查它。
     */
    if (!methodIsPublic(env, startClass, startMeth))
        goto bail;

    /*
     * Invoke main().
     */
    env->CallStaticVoidMethod(startClass, startMeth, strArray);

    if (!env->ExceptionCheck())
        result = 0;
```
- app\_process的源代码在文件  [frameworks/base/cmds/app_process/app_main.cpp](http://androidxref.com/4.3_r2.1/xref/frameworks/base/cmds/app_process/app_main.cpp)中，该文件中的关键代码有如下两处：
    - 第一处：先创建一个APPRuntime对象。
    - 第二处：调用runtime的start()方法启动指定的class。

在系统中只有一处使用了app_process，那就是在init.rc中。因为在使用时参数包含了“--zygote”以及“--start-system-server”，所以这里仅分析包含这两个参数的情况。

app\_process和dalvikvm在本质上是相同的，唯一的区别就是app\_process可以指定一些特殊的参数，这些参数有利于Framework启动特定的类，并进行一些特别的系统环境参数设置。

## 初始化Dalvik虚拟机
在Dalvik虚拟机刚刚开始运行的时候，先运行的是初始化工作，此工作的核心实现文件是Init.c。
- 使用dvmStartUp()开始虚拟机的准备工作
- 初始化跟踪显示系统，在Dalvik虚拟机的初始化过程中，使用函数dvmAllocTrackerStartup()初始化跟踪显示系统，跟踪系统主要用来生成调试系统的数据包。
- 初始化垃圾回收器，使用dvmGcStartUp()函数初始化垃圾回收器
``` C
bool dvmGcStartup()
{
    dvmInitMutex(&gDvm.gcHeapLock);
    pthread_cond_init(&gDvm.gcHeapCond, NULL);
    return dvmHeapStartup();
}
```

- 初始化线程列表和主线程环境参数，使用dvmThreadStartup()初始化线程列表和主线程环境参数。

``` C
/*
 * Initialize thread list and main thread's environment.  We need to set
 * up some basic stuff so that dvmThreadSelf() will work when we start
 * loading classes (e.g. to check for exceptions).
 */
bool dvmThreadStartup()
{
    Thread* thread;

    /* allocate a TLS slot */
    if (pthread_key_create(&gDvm.pthreadKeySelf, threadExitCheck) != 0) {
        ALOGE("ERROR: pthread_key_create failed");
        return false;
    }

    /* test our pthread lib */
    if (pthread_getspecific(gDvm.pthreadKeySelf) != NULL)
        ALOGW("WARNING: newly-created pthread TLS slot is not NULL");

    /* prep thread-related locks and conditions */
    dvmInitMutex(&gDvm.threadListLock);
    pthread_cond_init(&gDvm.threadStartCond, NULL);
    pthread_cond_init(&gDvm.vmExitCond, NULL);
    dvmInitMutex(&gDvm._threadSuspendLock);
    dvmInitMutex(&gDvm.threadSuspendCountLock);
    pthread_cond_init(&gDvm.threadSuspendCountCond, NULL);

    /*
     * Dedicated monitor for Thread.sleep().
     * TODO: change this to an Object* so we don't have to expose this
     * call, and we interact better with JDWP monitor calls.  Requires
     * deferring the object creation to much later (e.g. final "main"
     * thread prep) or until first use.
     */
    gDvm.threadSleepMon = dvmCreateMonitor(NULL);

    gDvm.threadIdMap = dvmAllocBitVector(kMaxThreadId, false);

    thread = allocThread(gDvm.mainThreadStackSize);
    if (thread == NULL)
        return false;

    /* switch mode for when we run initializers */
    thread->status = THREAD_RUNNING;

    /*
     * We need to assign the threadId early so we can lock/notify
     * object monitors.  We'll set the "threadObj" field later.
     */
    prepareThread(thread);
    gDvm.threadList = thread;

#ifdef COUNT_PRECISE_METHODS
    gDvm.preciseMethods = dvmPointerSetAlloc(200);
#endif

    return true;
}
```

- 分配内部操作方法的表格内存。

``` c
/*
 * Allocate some tables.
 */
bool dvmInlineNativeStartup()
{
    gDvm.inlinedMethods =
        (Method**) calloc(NELEM(gDvmInlineOpsTable), sizeof(Method*));
    if (gDvm.inlinedMethods == NULL)
        return false;

    return true;
}
```

- 初始化虚拟机的指令码相关的内容，使用dvmVerificationStartup()初始化虚拟机的指令码相关的内容，检查指令是否正确。
- 分配指令寄存器状态的内存，使用函数dvmRegisterMapStartup()

``` c
/*
 * Prepare some things.
 */
bool dvmRegisterMapStartup()
{
#ifdef REGISTER_MAP_STATS
    MapStats* pStats = calloc(1, sizeof(MapStats));
    gDvm.registerMapStats = pStats;
#endif
    return true;
}
```

- 分配指令寄存器状态的缓存，使用函数dvmInstanceofStartup()分配虚拟机使用的缓存。

``` c
/*
 * Allocate cache.
 */
bool dvmInstanceofStartup()
{
    gDvm.instanceofCache = dvmAllocAtomicCache(INSTANCEOF_CACHE_SIZE);
    if (gDvm.instanceofCache == NULL)
        return false;
    return true;
}
```

- 初始化虚拟机用的最基本Java库，使用dvmClassStartup()初始化。
``` c
/*
 * Initialize the bootstrap class loader.
 *
 * Call this after the bootclasspath string has been finalized.
 */
bool dvmClassStartup()
{
    /* make this a requirement -- don't currently support dirs in path */
    if (strcmp(gDvm.bootClassPathStr, ".") == 0) {
        ALOGE("ERROR: must specify non-'.' bootclasspath");
        return false;
    }

    gDvm.loadedClasses =
        dvmHashTableCreate(256, (HashFreeFunc) dvmFreeClassInnards);

    gDvm.pBootLoaderAlloc = dvmLinearAllocCreate(NULL);
    if (gDvm.pBootLoaderAlloc == NULL)
        return false;

    if (false) {
        linearAllocTests();
        exit(0);
    }

    /*
     * Class serial number.  We start with a high value to make it distinct
     * in binary dumps (e.g. hprof).
     */
    gDvm.classSerialNumber = INITIAL_CLASS_SERIAL_NUMBER;

    /*
     * Set up the table we'll use for tracking initiating loaders for
     * early classes.
     * If it's NULL, we just fall back to the InitiatingLoaderList in the
     * ClassObject, so it's not fatal to fail this allocation.
     */
    gDvm.initiatingLoaderList = (InitiatingLoaderList*)
        calloc(ZYGOTE_CLASS_CUTOFF, sizeof(InitiatingLoaderList));

    /*
     * Create the initial classes. These are the first objects constructed
     * within the nascent VM.
     */
    if (!createInitialClasses()) {
        return false;
    }

    /*
     * Process the bootstrap class path.  This means opening the specified
     * DEX or Jar files and possibly running them through the optimizer.
     */
    assert(gDvm.bootClassPath == NULL);
    processClassPath(gDvm.bootClassPathStr, true);

    if (gDvm.bootClassPath == NULL)
        return false;

    return true;
}
```

- 使用的Java类库线程类，使用函数dvmThreadObjStartup()。
- 初始化虚拟机使用的异常Java类库，使用dvmExceptionStartup()。
- 初始化虚拟机解释器使用的字符串哈希表，使用dvmStringInternStartup()。

``` c
bool dvmStringInternStartup()
{
    dvmInitMutex(&gDvm.internLock);
    gDvm.internedStrings = dvmHashTableCreate(256, NULL);
    if (gDvm.internedStrings == NULL)
        return false;
    gDvm.literalStrings = dvmHashTableCreate(256, NULL);
    if (gDvm.literalStrings == NULL)
        return false;
    return true;
}
```

- 初始化本地方法库的表，使用dvmNativeStartup()。
``` c
/*
 * Initialize the native code loader.
 */
bool dvmNativeStartup()
{
    gDvm.nativeLibs = dvmHashTableCreate(4, freeSharedLibEntry);
    if (gDvm.nativeLibs == NULL)
        return false;

    return true;
}
```

- 初始化内部本地方法，使用dvmInternalNativeStartup()。
``` c
/*
* Set up hash values on the class names.
 */
bool dvmInternalNativeStartup()
{
    DalvikNativeClass* classPtr = gDvmNativeMethodSet;

    while (classPtr->classDescriptor != NULL) {
        classPtr->classDescriptorHash =
            dvmComputeUtf8Hash(classPtr->classDescriptor);
        classPtr++;
    }

    gDvm.userDexFiles = dvmHashTableCreate(2, dvmFreeDexOrJar);
    if (gDvm.userDexFiles == NULL)
        return false;

    return true;
}
```

- 初始化JNI调用表，使用dvmJniStartup()，以便快速找到本地方法调用的入口。
- 缓存Java类库里的反射类，使用函数dvmReflectStartup()缓存Java类库里的反射类
- 剩余类的初始化工作

## 启动Zygote
详细解读Zygote启动的详细过程
### 在init.rc中配置zygote启动参数
init.rc保存在设备的根目录下，我们可以使用adb pull /init.rc

### 启动Socket服务端口
当Zygote服务从app_process启动后，会启动一个Dalvik虚拟机。因为Dalvik虚拟机执行的第一个Java类是ZygoteInit.java，座椅加下来的过程就是从类ZygoteInit中的函数main()开始的。

main()中做的第一个重要工作就是启动一个Socket服务端口，该Socket端口用于接收启动新进程的命令：
- 在静态函数registerZygoteSocket()中，完成启动Socket服务端口的功能
- 当准备好LocalServerSocket端口后，在函数main中调用runSelectLoopMode()进入非阻塞读操作，该函数会ServerSocket加入到被监测的文件描述符列表中，然后再while(true)循环中将该文件描述符添加到select的列表中，并调用ZygoteConnection类的runOnce()函数处理每一个Socket接收到的命令。
- 在SystemServer进程中创建一个Socket客户端，具体的代码是在文件Process.java中实现的，而调用Process类的工作是在AmS中的startProcessLocked()函数中实现的。
- 函数start()内部又调用了静态函数startViaZygote()，该函数的实体是使用一个本地Socket想Zygote中的Socket发送进行启动的命令，其执行流程如下：
    - 将startViaZygote()的函数参数转换为一个ArrayList<String\>列表
    - 然后再构造一个LocalSocket本地Socket接口
    - 通过LocalSocket对象构造出一个BufferedWriter对象
    - 通过该对象将ArrayList<String\>中的参数传递给Zygote的LocalServerSocket
    - 在Zygote端调用Zygote.forkAndSpecialize()函数，孕育出一个新的应用进程

[ZygoteInit代码地址](http://androidxref.com/4.3_r2.1/xref/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java)
``` java
    public static void main(String argv[]) {
        try {
            // Start profiling the zygote initialization.
            SamplingProfilerIntegration.start();

            registerZygoteSocket();
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                SystemClock.uptimeMillis());
            preload();
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                SystemClock.uptimeMillis());

            // Finish profiling the zygote initialization.
            SamplingProfilerIntegration.writeZygoteSnapshot();

            // Do an initial gc to clean up after startup
            gc();

            // Disable tracing so that forked processes do not inherit stale tracing tags from
            // Zygote.
            Trace.setTracingEnabled(false);

            // If requested, start system server directly from Zygote
            if (argv.length != 2) {
                throw new RuntimeException(argv[0] + USAGE_STRING);
            }

            if (argv[1].equals("start-system-server")) {
                startSystemServer();
            } else if (!argv[1].equals("")) {
                throw new RuntimeException(argv[0] + USAGE_STRING);
            }

            Log.i(TAG, "Accepting command socket connections");

            runSelectLoop();

            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }
```


### 加载preload-classes
在ZygoteInit的函数main中，创建完Socket服务端后还不能马上孕育新的进程，因为这个卵还没有预装Framework大部分的类和资源。

预装的类列表是在framework.jar中第一个文本文件列表，名称为preload-classes，该列表的原始定义在文本文件frameworks/base/preload-classes中，而该文件又是通过如下类生成的：
```
frameworks/base/tools/preload/WritePreloadedClassFile.java
```
生成preload-classes的方法是在Android根目录下执行如下命令
```
$java -Xss512M-cp /path/to/preload.jarWritePreloadedClassFile /path/to/.compiled 
1517 classes were loaded by more than one app.
Added 147 more to speed up applications.
1664 total classes will be preloaded.
Writing object model ...
Done!
```
上述命令中，/path/to/preload.jar是指如下文件：
```
out/host/darwin-x86/framework/preload.jar
```
- 参数“-Xss”：用于执行程序所需要的JVM栈大小，此处使用512MB，默认大小不能满足程序的运行，会抛出java.lang.StackOverflowError错误信息。
- WritePreloadedClassFile：表示要执行的具体类。

### 加载preload-resources
preload-resources是在如下文件中被定义的
```
frameworks/base/core/res/res/values/arrays.xml
```
在preload-resources中包含了两类资源，一类是drawable资源，另一类是color资源：
``` xml 
<array name="preloaded_drawables">
    <item>@drawable/toast_frame_holo</item>
    ...
</array>
<array name="preloaded_color_state_lists">
    <item>@color/primary_text_dark</item>
    ...
</array>
```
加载这些资源的功能是在函数preloadResources()中实现的，在函数中分别调用了如下两个函数来加载这两类资源：preloadDrawables()和preloadColorStateLists()。

具体的加载原理，就是把这些资源读出来，放在一个全局变量中，只要该类资源不被销毁，这些全局变量就会一直保存。

通过全局变量mResources来保存Drawable资源，该资源变量的类型是Resources类，由于在该类内部会保存一个Drawable资源列表，因此实际上是在Resources内部缓存这些Drawable资源的。保存Color资源的全局变量的功能也是mResources实现的。同样，在类Resources内部也有一个Color资源列表。

### 使用fork启动新进程
fork是Linux系统中的一个系统嗲用，其功能是复制当前进程并产生一个新的进程。除了进程id不同，新的进程将拥有与原始进程完全形同的进程信息。进程信息包括该进程所打开的文件描述符列表、所分配的内存等。当创建新进程后，两个进程将共享已经分配的内存空间，直到其中一个需要向内存中写入数据时，操作系统才负责复制一份目标地址空间，并将要写的数据写入到新的地址中，这就是“copy-on-write（仅当写的时候才复制）”机制，这种机制可以最大限度的在多个进程中共享物理内存。

在所有的操作系统中，都存在一个程序装载器，程序装载器一般会作为操作系统的一部分，并由Shell程序调用。当内核启动后，会首先启动Shell程序。

常见的Shell程序包含如下两大类：
- 命令行界面的
- 窗口界面的

Windows系统中的Shell程序就是桌面程序，Ubuntu系统中的Shell程序就是GNOME桌面程序。当启动Shell程序后，用户可以双击桌面图标启动指定的应用程序，而在操作系统内部，启动新的进程包含如下三个过程。
- 第一个过程，内核创建一个进程数据结构，用于表示将要启动的进程
- 第二个过程，内核调用程序装载器函数，从指定的程序文件读取程序代码，并将这些程序代码装载到预先设定的内存地址。
- 第三个过程，装载完毕后，内核将程序指针指向到目标程序地址的入口处开始执行指定的进程。当然，实际的过程会考虑更多的细节，不过大致的思路就是这样。

在一般情况下，没有必要复制进程，而是暗账以上3个过程创建新进程，但当满足条件时，则由于函数fork()的Linux的系统调用，Android中的Java层仅仅是对该方法进行了JNI封装而已，因此，接下来的C代码是介绍使用函数fork()的过程，以便读者对该方法有更具体的认识：

``` c
#include <sys/types.h>
#include <unistd.h>
int main() {
    pid_t pid;
    printf("pid = %d, Take camera, by subway, take air! \n", getpid());
    pid = fork();
    if(pid > 0) {
        printf("pid=%d \n", getpid());
        pid = fork();
        if(!pid) prinrf("pid=%d, 去看AA！\n", getpid());
    }
    else if(!pid) printf("pid=%d, 去看BB！ \n", getpid());
    else if(pid == -1) perror("fork");
    getchar();
}
//输出的结果如下
pid = 3927, Take camera, by subway, take air!
pid=3927
pid=3929，去看AA!
pid=3930，去看BB！
```
函数fork()的返回值与普通函数调用完全不同，具体如下：
- 当返回值大于0的时候，代表是父进程
- 当等于0的时候，代表的是被复制的进程

也就是说，父进程和子进程的代码都在该C文件中，只是不同的进程执行不同的代码，而进程的靠fork()的返回值进行区分的。

由以上的结果可以看出，第一次调用fork()时复制了一个“看AA”进程，然后再父进程中再次调用fork()复制了“看BB”的进程，三者都有各自不同的进程id。

Zygote进程是上述演示代码中的“精灵进程”。在文件[ZygoteInit.java](http://androidxref.com/4.3_r2.1/xref/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java)中复制新锦成是通过在函数runSelectLoopMode()中调用ZygoteConnection的函数runOnce()完成的，在该函数中通过调用forkAndSpecialize()函数来复制一个新的进程。

> 函数forkAndSpecialize()是一个naive方法，其内部的执行原理与上面的C代码类似，当新进程被创建好后，还需做一些完善工作。因为当Zygote复制新进程时，已经创建了一个Socket服务端，而这个服务端是不应该被新进程使用的，否则系统中会有多个进程接收并调用新进程中指定的Class文件的main()函数作为新进程的入口点。而这些正是在调用函数forkAndSpecialize()后根据返回值pid完成的。当pid等于0时，代表的是子进程，函数handleChildProc()会从指定Class文件的main()函数处开始执行。新的进程会完全脱离Zygote进程的孕育过程，称为一个真正的应用进程。
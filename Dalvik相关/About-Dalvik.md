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

## 启动SystemServer进程
SystemServer进程是Zygote孕育出的第一个进程，该进程是从ZygoteInit.java的main()函数中调用startSystemServer()开始的。与启动普通进程的差别在于，zygote类为启动SystemServer提供了专门的函数startSystemServer()，而不是普通的forkAndSpecilize()函数。同时，启动SystemServer进程后，首先要做的事情与普通进程也有所差别。

函数startSystemServer()的主要功能如下：
- 定义了一个String[]数组，数组中包含了要启动的进程的相关信息，其中最后一项指定新进程启动后装载的第一个Java类，此处即为类com.android.server.SystemServer。
- 调用forkSystemServer()从当前的Zygote进程孕育出新的进程。该函数是一个native方法，其作用于forkAndSpecilize()相似。
- 启动新进程后，在函数startSystemServerProcess()中完成如下两件事：
    - 关闭Socket服务端
    - 执行com.android.server.SystemServer类中的main()函数。

    除了这两个主要事情外，还做了一些额外的运行环境配置，这些配置主要在函数commonInit()和函数zygoteInitNative()中完成。一旦配置好SystemServer的进程环境后，就从类SystemServer中的main()函数开始执行。

### 启动各种系统服务线程
SystemServer进程在Android运行环境中扮演了“中枢”的角色，在APK应用中能够直接交互的大部分系统服务都在这个进程中运行，例如WindowManagerServer(Wms)、ActivityManagerSystemService(Ams)、PackageManagerServer(PmS)等常见的应用，这些系统服务都是以一个线程的方式存在于SystemServer进程中的。

SystemServer中的main()函数首先调用的是函数init1()，这是一个native函数，内部会进行一些Dalvik虚拟机相关的初始化工作。该函数执行完毕后，其内部会调用Java端的init2()函数，这就是为什么Java源码中没有引用你init2()的地方，主要的系统服务都是在init2()函数中完成的。

该函数首先创建了一个ServerThread对象，该对象是一个线程，然后直接运行该线程，从ServerThread的run方法内部开始真正启动各种服务线程。基本上，每个服务都有对应的Java类，从编码规范的角度来看，启动这些服务的模式可归于如下三种。
- 模式一：是指直接使用构造函数构造一个服务，由于大多数服务都对应一个线程，因此，在构造函数内部就会创建一个线程并自动运行。
- 模式二：是指服务类会提供一个getInstance()方法，通过该方法获取该服务对象，这样的好处是保证系统中仅包含一个该服务对象。
- 模式三：是指从服务类的main()方法中开始执行。

无论以上任何模式，当创建了服务对象后，有时可能还需要调用该服务类的init()函数或者systemReady()函数来完成该对象的启动，当然，这些都是服务类内部定义的。为了区分以上启动的不同，以下采用一种新的方式来描述该启动过程。

下表中列出了SystemServer中启动的所有服务，以及这些服务的启动模式。

服务类名称 | 作用描述 | 启动模式
---|---|---
EntropyService | 提供伪随机数 | 1.0
PowerManagerService | 电源管理服务 | 1.2/3
ActivityManagerService | 最核心的服务之一，管理Activity | 自定义
TelephonyRegistry | 注册电话模块的时间相应，</br>比如重启、关闭、启动等 | 1.0
PackageManagerService | 程序包管理服务 | 3.3
AccountManagerService | 账户管理服务，是指联系人账户，</br>而不是Linux系统账户 | 1.0
ContentService | ContentProvider服务，提供跨进程数据交换 | 3.0
BatteryService | 电池管理服务 | 1.0
LightsService | 自然光强度感应传感器服务 | 1.0
VibratorService | 振动器服务 | 1.0
AlarmManagerService | 定时器管理服务，提供定时提醒服务 | 1.0
WindowManagerService | Framework最核心的服务之一，负责窗口管理 | 3.3
BluetoothService | 蓝牙服务 | 1.0+
DevicePolicyManagerService | 提供一些系统级别的设置和属性 | 1.3
StatusBarManagerService | 状态栏管理服务 | 1.3
ClipboardService | 系统剪贴板服务 | 1.0
InputMethodManagerService | 输入法管理服务 | 1.0
NetStatService | 网络状态服务 | 1.0
NetworkManagementService | 网络管理服务 | NMS.</br>creat()
ConnectivityService | 网络连接管理服务 | 2.3
ThrottleService | 暂不清楚其作用 | 1.3
AccessibilityManagerService | 辅助管理程序截获所有的用户输入，</br>并根据这些输入给用户一些额外的反馈，</br>起到辅助的效果
MountService | 挂载服务，可通过该服务调用Linux层面的mount程序 | 1.0
NotificationManagerService | 通知栏管理服务， Android 中的通知栏和状态栏在一起，</br>只是界面上前者在左边，后者在右边
DeviceStorageMonitorService | 磁盘空间状态检测服务 | 1.0
LocationManagerService | 地理位置服务 | 1.3
SearchManagerService | 搜索管理服 | 1.0
DropBoxManagerService | 通过该服务访问Linux层面的Dropbox 程序 | 1.0
WallpaperManagerService|墙纸管理服务，墙纸不等同于桌面背景，在 View 系统内部，</br>墙纸可以作为任何窗口的背景|1.3
AudioService|音频管理服务|1.0
BackupManagerService|系统备份服务|1.0
AppWidgetService|Widget服务|1.3
RecognitionManagerService|身份识别服务|1.3
DiskStatsService|磁盘统计服务|1.0

AmS的启动模式如下：
- 调用函数main()返回一个Context对象，而不是AmS服务本身。
- 调用AmS.setSystemProcess()。
- 调用AmS.installProviders()。
- 调用systemReady()，当AmS执行完systemReady()后，会相继启动相关联服务的systemReady()函数，完成整体初始化。

### 启动第一个Activity
当启动以上服务线程后，AMS服务是以systemReady()调用完成最后启动的，而在AMS的函数systemReady()内部的最后一行代码发出了启动任务队列中最上面一个Activity的消息。因为在系统刚启动时，mMainStack队列中并没有任何Activity对象，所以在类ActivityStack中将调用函数startHomeActivityLocked()。
``` java
    boolean startHomeActivityLocked(int userId) {
        if (mHeadless) {
            // Added because none of the other calls to ensureBootCompleted seem to fire
            // when running headless.
            ensureBootCompleted();
            return false;
        }

        if (mFactoryTest == SystemServer.FACTORY_TEST_LOW_LEVEL
                && mTopAction == null) {
            // We are running in factory test mode, but unable to find
            // the factory test app, so just sit around displaying the
            // error message and don't try to start anything.
            return false;
        }
        Intent intent = new Intent(
            mTopAction,
            mTopData != null ? Uri.parse(mTopData) : null);
        intent.setComponent(mTopComponent);
        if (mFactoryTest != SystemServer.FACTORY_TEST_LOW_LEVEL) {
            intent.addCategory(Intent.CATEGORY_HOME);
        }
        ActivityInfo aInfo =
            resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
        if (aInfo != null) {
            intent.setComponent(new ComponentName(
                    aInfo.applicationInfo.packageName, aInfo.name));
            // Don't do this if the home app is currently being
            // instrumented.
            aInfo = new ActivityInfo(aInfo);
            aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
            ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                    aInfo.applicationInfo.uid);
            if (app == null || app.instrumentationClass == null) {
                intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
                mMainStack.startActivityLocked(null, intent, null, aInfo,
                        null, null, 0, 0, 0, null, 0, null, false, null);
            }
        }

        return true;
    }
```
在开机后，系统从哪个Activity开始执行这一动作，完全取决于mMainStack中的第一个Activity对象。如果在AMS启动时能够构造一个Activity对象（并不是说构造出一个Activity类对象），并将其放在mMainStack中，那么第一个运行的Activity就是这个Activity，这一点不像其他操作系统中通过设置一个固定程序作为第一个启动程序。


在AMS的startHomeActivityLocked()中，系统发出了一个category字段包含CATEGORY_HOME的intent。

无论是哪个应用程序，只要声明自己能够响应该intent，那么就可以被认为是Home程序，这就是为什么在Android领域中会存在各种“Home程序”的原因。系统并没有给任何程序赋予“Home”特权，而只是把这个权利交给用户。当系统中有多个程序能够响应该intent时，系统会弹出一个对话框，请求用户选择启动哪个程序，并允许用户记住该选择，从而使得以后每次按Home键后，都启动相同的Activity。这就是第一个Activity的启动过程。

## 加载class类文件
Java的源代码经过编译后，会生成“.class”格式的文件，即字节码文件。然后再Android中使用dx工具将其转换为后缀为“.jar”格式的Dex类型文件。DVM负责解释并执行编译后的字节码。在解释执行字节码之前，当然要读取文件，分析文件的内容，得到字节码，然后才能解释和执行。在整个的加载过程中，最为重要的就是对Class的加载，Class包含Method，Method又包含code。通过对Class的加载，我们可以获得所需执行的字节码。

### DexFile在内存中的映射
在Android系统中，Java源文件会被编译成“.jar”格式的Dex文件，在代码中称为Dexfile。在加载Class之前，必先读取相应的jar文件。通常我们使用read()函数来读取文件中的内容，但在Dalvik中使用mmap()函数。与read()不同的是，mmap()函数会将Dex文件映射到内存中，这样，通过普通的内存读取操作，即可访问Dexfile中的内容。

Dexfile的文件格式如图，主要由三部分组成：头部、索引、数据。通过头部可知索引的位置和树木，可知数据区的起始位置。其中classDefsOff指定ClassDef在文件中的起始位置，dataOff指定了数据在文件中的起始位置，ClassDef可理解为Class的索引。通过读取ClassDef可获知Class的基本信息，其中classDataOff指定了Class数据在数据区的位置。

![DexFile文件格式](https://upload-images.jianshu.io/upload_images/3610640-d1beb22bb48dea36.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在将Dexfile文件映射到内存后，会调用dexFileParse()函数对其进程分析，分析的结果存放于名为DexFile的数据结构中。DexFile中的baseAddr指向映射区的起始位置，pClassDefs指向ClassDefs（即class索引）的起始位置。由于在查找class时，都是使用class的名字进行查找的，所以为了加快查找速度，创建了一个hash表。在hash表中对class名字进行hash，并生成index。这些操作都是在对文件解析时完成的，这样虽然在加载过程中比较耗时，但是在运行过程中却节省大量的查找时间。

解析完毕后，接下来开始加载class文件，在此需要将加载类用ClassObject来保存，所以需要先分析与ClassObject相关的几个数据结构。

首先在文件Object.h中可以看到如下对结构体Object的定义：

``` C
typedef struct Object {
    ClassObject *clazz;
    u4 lock;
} Object;
```
通过结构体Object定义了基本类的实现，这里有如下两个变量。
- lock：对应Object对象中的锁实现，即notify wait的处理
- clazz：是结构体指针，姑且不看结构体内容，这里用了指针的定义

``` C
struct StringObject {
    ClassObject *clazz;
    u4 lock;
    u4 instanceData[i];
}
```

任何对象的内存结构体中第一行都是Object结构体，而这个结构体第一个总是一个ClassObject，第二个总是lock。按照C++中的技巧，这些结构体可以当成Object结构体使用，因此所有的类在内存中都具有对象的功能，即可以找到一个类（ClassObject），可以有一个锁（lock）。

StringObject是对String类进行管理的数据对象，ArrayObject是数据相关的管理。
### ClassObject——Class在加载后的表现形式
解析完文件后，需要加载Class的具体内容。在Dalvik中，由数据结构ClassObject负责存放加载的信息。

如下图所示，加载过程会在内存中alloc几个区域，分别存放directMethods、virtualMethods、sfields、ifields。这些信息是从Dex文件的数据区中读取的，首先会读取Class的详细信息，从中获得directMethod、virtualMethod、sfields、ifields等信息，然后再读取。

![加载过程](https://upload-images.jianshu.io/upload_images/3610640-671ec7b50fadd6d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 加载Class并生成相应的ClassObject的函数
接下来分析负责加载工作的函数findClassNoInit()。在获取Class索引时，会分为基本类库文件和用户类文件两种情况。

函数LoadClassFromDex()会先读取Class的具体数据（从ClassDataoff处），然后分别加载directMethod、virtualMethod、sfields、ifields。

为了追求效率，在加载后需要将其缓存起来，以便以后使用。其次，在查找过程中，如果是顺序查找的话会很慢，所以需要使用gDvm.loadedClasses这个Hash表来帮忙。如果一个子类需要调用超累的函数，那它当然要先加载超类了，可能的话，甚至会加载超类的超类。

### 加载基本类库文件
### 加载用户类文件
在加载用户类文件时，会先加载一个Class，然后这个Class去负责用户类文件的加载，而这个Class又会通过JNI的方式去调用findClassNoInit。具体加载过程与基本类库的加载类似。

# DVM的内存系统
内存管理是DVM中的一个重要组件，其内存管理的核心是分别实现内存分配和回收的工作，Java语言使用new操作符来分配内存，但是Java语言并没有提供任何操作来释放内存，而是通过垃圾收集机制来回收内存。对于内存管理的实现，我们通过如下三个方面加以分析：
- 内存分配
- 内存回收
- 内存管理和调试

## 如何分配内存
### 对象布局
内存管理的主要操作之一是为Java对象分配内存，所有的对象都有一个相同的头部clazz和lock。
- clazz：指向该对象的类对象，类对象用来描述该对象所属的类，这样可以很容易从一个对象获取该对象所属的类的具体信息。
- lock：是一个无符号整数，用以实现对象的同步。
- data：用于存放对象数据，根据对象的不同，数据区的大小是不同的，

### 堆
堆的DVM从操作系统分配的一块连续的虚拟内存。其中heapBase表示堆的起始地址，heapLimit表示堆的最大地址，堆大小的最大值可以通过“-Xmx”选项或dalvik.vm.heapsize指定。在原生系统中，一般dalvik.vm.heapsize的值是32MB，在MIUI中，将其设为64MB。
### 堆内存位图
在虚拟机中维护了两个对应于堆内存的位图，称为liveBits和markBits。在对象布局中，我们看到对象最小占用8个字节。在为对象分配内存时，要求必须8字节对齐。这也就是说，对象的大小会调整为8字节的倍数。堆内存位图就是用来描述堆内存的，每一个bit描述8个字节，因此堆内存的，每一个bit描述8个字节，因此堆内存位图的大小是对堆的1/64。对于MIUI的实现来说，这两个位图各占1MB。

liveBits的功能是跟踪堆中已经分配的内存，每分配一个对象时，对象的内存起始地址对应于位图中的位被设为1。

### 堆内存管理
在DVM的实现中，是通过底层的bionicC库的malloc/free操作来分配和释放内存的。库bionicC的malloc/free操作是基于DougLea的实现（dlmalloc），这是一个被广泛使用、久经考验的C内存管理库。
### dvmAllocObject
在DVM中，操作符new最终对应C函数dvmAllocObject()。
```
Object* dvmAllocObject(ClassObject* clazz, int flags)
{
    Object* newObj;

    assert(clazz != NULL);
    assert(dvmIsClassInitialized(clazz) || dvmIsClassInitializing(clazz));

    /* allocate on GC heap; memory is zeroed out */
    newObj = (Object*)dvmMalloc(clazz->objectSize, flags);
    if (newObj != NULL) {
        DVM_OBJECT_INIT(newObj, clazz);
        dvmTrackAllocation(clazz, clazz->objectSize);   /* notify DDMS */
    }

    return newObj;
}
```
为了分配内存，虚拟机尽了最大的努力，做了4次尝试。其中进行了两次垃圾收集，第一次部手机SoftReference，第二次收集SoftReference。从中我们可以看到垃圾收集的时机，实质上，在Dalvik虚拟机实现中有3个时机可以触发垃圾收集的运行：
- 程序员显式调用System.gc()
- 内存分配失败时
- 如果分配的对象大小超过384KB，运行并发标记（Concurrent Mark）

## 内存管理机制
DVM虚拟机的内存管理需要依赖于Linux的内存管理机制，DVM的内存管理的实现源码保存在vm\alloc目录下。
### 表示堆的结构体
在文件HeapSource.cpp中定义表示堆的结构体：[查看源码](http://androidxref.com/1.6/xref/dalvik/vm/alloc/HeapSource.c)

``` C
//此段代码来自Android1.6
typedef struct {
    /* The mspace to allocate from.
     * 使用dlmalloc分配的内存
     */
    mspace *msp;

    /* The bitmap that keeps track of where objects are in the heap.
     */
    HeapBitmap objectBitmap;

    /* The largest size that this heap is allowed to grow to.
     * 堆可以增长的最大值
     */
    size_t absoluteMaxSize;

    /* Number of bytes allocated from this mspace for objects,
     * including any overhead.  This value is NOT exact, and
     * should only be used as an input for certain heuristics.
     * 已经分配的字节数
     */
    size_t bytesAllocated;

    /* Number of objects currently allocated from this mspace.
     * 已分配的对象数
     */
    size_t objectsAllocated;
} Heap;
```

### 表示位图堆的结构体数据
在文件HeapBitmap.h中定义表示位图堆的结构体数据：[查看源码](http://androidxref.com/1.6/xref/dalvik/vm/alloc/HeapBitmap.h)

``` C

typedef struct {
    /* The bitmap data, which points to an mmap()ed area of zeroed
     * anonymous memory.
     * 位图数据
     */
    unsigned long int *bits;

    /* The size of the memory pointed to by bits, in bytes.
     * 位图大小
     */
    size_t bitsLen;

    /* The base address, which corresponds to the first bit in
     * the bitmap.
     * 位图对应的对象指针数组的首地址
     */
    uintptr_t base;

    /* The highest pointer value ever returned by an allocation
     * from this heap.  I.e., the highest address that may correspond
     * to a set bit.  If there are no bits set, (max < base).
     * 为使用中的最后一位被设置的对象指针地址，如果全没设置则（max < base）
     */
    uintptr_t max;
} HeapBitmap;
```

### HeapSource结构体
在DVM中，使用结构体HeapSource来管理各种Heap数据，Heap只是其中的一个子项，其在HeapSource.c中定义：
```
struct HeapSource {
    /* Target ideal heap utilization ratio; range 1..HEAP_UTILIZATION_MAX
     * 堆的使用率，范围从1到HEAP_UTILIZATION_MAX
     */
    size_t targetUtilization;

    /* Requested minimum heap size, or zero if there is no minimum.
     * 分配堆的最小尺寸
     */
    size_t minimumSize;

    /* The starting heap size.
     * 堆分配的初始尺寸
     */
    size_t startSize;

    /* The largest that the heap source as a whole is allowed to grow.
     * 允许分配的堆增长到的最大尺寸
     */
    size_t absoluteMaxSize;

    /* The desired max size of the heap source as a whole.
     * 理想的堆的最大尺寸
     */
    size_t idealSize;

    /* The maximum number of bytes allowed to be allocated from the
     * active heap before a GC is forced.  This is used to "shrink" the
     * heap in lieu of actual compaction.
     * 在卡户收集前允许堆分配的最大尺寸
     */
    size_t softLimit;

    /* The heaps; heaps[0] is always the active heap,
     * which new objects should be allocated from.
     * 堆数组，最大尺寸为3
     */
    Heap heaps[HEAP_SOURCE_MAX_HEAP_COUNT];

    /* The current number of heaps.
     * 当前堆的个数
     */
    size_t numHeaps;

    /* External allocation count.
     * 对外分配计数
     */
    size_t externalBytesAllocated;

    /* The maximum number of external bytes that may be allocated.
     * 允许外部分配的最大值
     */
    size_t externalLimit;

    /* True if zygote mode was active when the HeapSource was created.
     * 在创建这个HeapSource的时候是否是Zygote模式，确定是否有Zygote进程
     */
    bool sawZygote;
};
```

### 与mark bits相关的结构体
在MarkSweep.h中定义了与mark bits相关的结构体，[源码](http://androidxref.com/1.6/xref/dalvik/vm/alloc/MarkSweep.h)
### 结构体GcHeap
在文件HeapInternal.h中定义了Dalvik的垃圾回收机制，需要用到结构体GcHeap，[源码](http://androidxref.com/1.6/xref/dalvik/vm/alloc/HeapInternal.h)
### 初始化垃圾回收器
在文件Init.c中，通过函数dvmGcStartup()来初始化垃圾回收器：
``` C
bool dvmGcStartup(void)
{
    dvmInitMutex(&gDvm.gcHeapLock);
    return dvmHeapStartup();
}
```
### 初始化与Heap相关的信息
在文件[alloc\Heap.c](http://androidxref.com/1.6/xref/dalvik/vm/alloc/Heap.c)中，通过dvmHeapStartup()函数来初始化和Heap相关的信息，例如常见的内存分配和内存管理等工作。
``` C
bool dvmHeapStartup()
{
    GcHeap *gcHeap;

#if defined(WITH_ALLOC_LIMITS)
    gDvm.checkAllocLimits = false;
    gDvm.allocationLimit = -1;
#endif

    gcHeap = dvmHeapSourceStartup(gDvm.heapSizeStart, gDvm.heapSizeMax);
    if (gcHeap == NULL) {
        return false;
    }
    gcHeap->heapWorkerCurrentObject = NULL;
    gcHeap->heapWorkerCurrentMethod = NULL;
    gcHeap->heapWorkerInterpStartTime = 0LL;
    gcHeap->softReferenceCollectionState = SR_COLLECT_NONE;
    gcHeap->softReferenceHeapSizeThreshold = gDvm.heapSizeStart;
    gcHeap->ddmHpifWhen = 0;
    gcHeap->ddmHpsgWhen = 0;
    gcHeap->ddmHpsgWhat = 0;
    gcHeap->ddmNhsgWhen = 0;
    gcHeap->ddmNhsgWhat = 0;
#if WITH_HPROF
    gcHeap->hprofDumpOnGc = false;
    gcHeap->hprofContext = NULL;
#endif

    /* This needs to be set before we call dvmHeapInitHeapRefTable().
     */
    gDvm.gcHeap = gcHeap;

    /* Set up the table we'll use for ALLOC_NO_GC.
     */
    if (!dvmHeapInitHeapRefTable(&gcHeap->nonCollectableRefs,
                           kNonCollectableRefDefault))
    {
        LOGE_HEAP("Can't allocate GC_NO_ALLOC table\n");
        goto fail;
    }

    /* Set up the lists and lock we'll use for finalizable
     * and reference objects.
     */
    dvmInitMutex(&gDvm.heapWorkerListLock);
    gcHeap->finalizableRefs = NULL;
    gcHeap->pendingFinalizationRefs = NULL;
    gcHeap->referenceOperations = NULL;

    /* Initialize the HeapWorker locks and other state
     * that the GC uses.
     */
    dvmInitializeHeapWorkerState();

    return true;

fail:
    gDvm.gcHeap = NULL;
    dvmHeapSourceShutdown(gcHeap);
    return false;
}
```

### 创建GcHeap
在文件[alloc\HeapSource.c](http://androidxref.com/1.6/xref/dalvik/vm/alloc/HeapSource.c)中，通过函数dvmHeapSourceStartup()来创建GcHeap。源码如下：
``` C
GcHeap *
dvmHeapSourceStartup(size_t startSize, size_t absoluteMaxSize)
{
    GcHeap *gcHeap;
    HeapSource *hs;
    Heap *heap;
    mspace msp;

    assert(gHs == NULL);

    if (startSize > absoluteMaxSize) {
        LOGE("Bad heap parameters (start=%d, max=%d)\n",
           startSize, absoluteMaxSize);
        return NULL;
    }

    /* Create an unlocked dlmalloc mspace to use as
     * the small object heap source.
     */
    msp = createMspace(startSize, absoluteMaxSize, 0);
    if (msp == NULL) {
        return false;
    }

    /* Allocate a descriptor from the heap we just created.
     */
    gcHeap = mspace_malloc(msp, sizeof(*gcHeap));
    if (gcHeap == NULL) {
        LOGE_HEAP("Can't allocate heap descriptor\n");
        goto fail;
    }
    memset(gcHeap, 0, sizeof(*gcHeap));

    hs = mspace_malloc(msp, sizeof(*hs));
    if (hs == NULL) {
        LOGE_HEAP("Can't allocate heap source\n");
        goto fail;
    }
    memset(hs, 0, sizeof(*hs));

    hs->targetUtilization = DEFAULT_HEAP_UTILIZATION;
    hs->minimumSize = 0;
    hs->startSize = startSize;
    hs->absoluteMaxSize = absoluteMaxSize;
    hs->idealSize = startSize;
    hs->softLimit = INT_MAX;    // no soft limit at first
    hs->numHeaps = 0;
    hs->sawZygote = gDvm.zygote;
    if (!addNewHeap(hs, msp, absoluteMaxSize)) {
        LOGE_HEAP("Can't add initial heap\n");
        goto fail;
    }

    gcHeap->heapSource = hs;

    countAllocation(hs2heap(hs), gcHeap, false);
    countAllocation(hs2heap(hs), hs, false);

    gHs = hs;
    return gcHeap;

fail:
    destroy_contiguous_mspace(msp);
    return NULL;
}
```


### 追踪位置 
在文件alloc\HeapSource.c中，通过函数countAllocation()在Heap::object Bitmap上进行标记，以便追踪这些区域的位置。
``` C
static inline void
countAllocation(Heap *heap, const void *ptr, bool isObj)
{
    assert(heap->bytesAllocated < mspace_footprint(heap->msp));

    heap->bytesAllocated += mspace_usable_size(heap->msp, ptr) +
            HEAP_SOURCE_CHUNK_OVERHEAD;
    if (isObj) {
        heap->objectsAllocated++;
        dvmHeapBitmapSetObjectBit(&heap->objectBitmap, ptr);
    }

    assert(heap->bytesAllocated < mspace_footprint(heap->msp));
}
```

### 分配空间
在文件Heap.c中，通过函数dvmMalloc()实现空间的分配工作，[alloc\Heap.c](http://androidxref.com/1.6/xref/dalvik/vm/alloc/Heap.c)

# DVM的启动过程
Android系统中的应用程序进程是由Zygote进程孕育出来的，而Zygote进程又是由Init进程启动的。在启动Zygote进程时，会创建一个DVM实例，每当Zygote进程孕育出一个新的应用程序进程时，会将这个DVM实例复制到新的应用程序中，这样可以使每个应用程序进程都拥有一个独立的DVM实例。
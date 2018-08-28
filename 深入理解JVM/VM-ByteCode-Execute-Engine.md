# 概述
执行引擎是Java虚拟机最核心的组成部分之一。“虚拟机”是一个相对于“物理机”的概念，这两个机器都有代码执行能力，其区别是物理机的执行引擎是直接建立在处理器、硬件、指令集和操作系统层面的，而虚拟机的执行引擎则是由自己实现的，因此可以自行指定指令集与执行引擎结构，并且能够执行那些不被硬件直接支持的指令集格式。 在Java虚拟机规范中指定了虚拟机字节码执行引擎的概念模型，这个概念模型成为各种虚拟机执行引擎的统一外观（Facade）。在不同的虚拟机实现里面，执行引擎在执行Java代码的时候可能会有解释执行（通过解释器执行）和编译执行（通过即时编译器产生本地代码执行）两种选择，也可能两者兼备，甚至还可能会包含几个不同级别的编译器执行引擎。但从外观上看起来，所有的JVM的的执行引擎都是一致的：输入的是字节码文件，处理过程是字节码解析的等效过程，输出的是执行结果，下面主要从概念模型的角度来讲解虚拟机的方法调用和字节码执行。

# 运行时栈帧结构
栈帧（Stack Frame）是用于支持虚拟机进行方法调用和方法执行的数据结构，它是虚拟机运行时数据区中的虚拟机栈（Virtual Machine Stack）的栈元素。栈帧存储了方法的局部变量表、操作数栈、动态连接和方法返回地址等信息。每个方法从调用开始至执行完成的过程，都对应着一个栈帧在虚拟机栈里面从入栈到出栈的过程。

每一个栈帧都包括了局部变量表、操作数栈、动态连接、方法返回地址和一些额外的附加信息。在编译程序代码的时候，栈帧中需要多大的局部变量表，多深的操作数栈都已经完全确定了，并且写入到方法表的Code属性之中，因此一个栈帧需要分配多少内存，不会受到程序运行期变量数据的影响，而仅仅取决于具体的虚拟机实现。

一个线程中的方法调用的方法调用链可能会很长，很多方法都同时处于执行状态。对于执行引擎来说，在活动线程中，只有位于栈顶的栈帧才是有效的，称为当前栈帧（Current Stack Frame），与这个栈帧相关联的方法称为当前方法（Current Method）。执行引擎运行的所有字节码指令都只针对当前栈帧进行操作，在概念模型上，典型的栈帧结构如图所示：

![栈帧的概念结构](http://upload-images.jianshu.io/upload_images/3610640-a71988a26b8e5985.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 局部变量表
局部变量表（Local Variable Table）是一组变量值存储空间，用于存放方法参数和方法内容定义的局部变量。在Java程序编译为Class文件时，就在方法的Code属性的max_locals数据项中确定了该方法所需要分配的局部变量表的最大容量。

局部变量表的容量以变量槽（Variable Slot，下面称Slot）为最小单位，虚拟机规范中并没有明确指明了一个Slot应占用的内存空间大小，只是很有导向性地说到每个Slot都应该能存放一个boolean、byte、char、short、int、float、reference或returnAddress类型的数据，这8中数据类型，都可以使用32位或更小的内存空间来存放，但这种描述与明确指出“每个Slot占用32位长度的内存空间”是有一些差别的，它允许Slot的长度可以随着处理器、操作系统或虚拟机的不同而发生变化。只要保证64位虚拟机中使用了64位的物理地址空间去实现一个Slot，虚拟机仍要使用对齐和补白的手段让Slot在外观上看起来与32位虚拟机中的一致。

既然前面提到了JVM的数据类型，在此再简单介绍一下它们。一个Slot可以存放32位以内的数据类型，Java中占用32位以内的数据类型有boolean、byte、char、short、int、float、reference和returnAddress8种类型。第7种reference类型表示对一个对象实例的引用，虚拟机规范既没有说明它的长度，也没有说明指出这种引用应有怎样的结构。但一般来说，虚拟机实现至少都应当能通过这个引用做到两点，一是从此引用中直接或间接地查找到对象在Java堆中的数据存放的起始地址索引，二是此引用中直接或间接地查找到对象所属数据类型在方法区中的存储的类型信息，否则无法实现Java语言规范中定义的语法约束。第8种即returnAddress类型目前已经很少见了，它是为字节码指令jsr、jsr_w和ret服务的，指向了一条字节码指令的地址，很古老的JVM曾经使用这几条指令来实现异常处理，现在已经由异常表代替。

对于64位的数据类型，虚拟机会以高位对齐的方式为其分配两个连续的Slot空间。Java语言中明确的（reference类型则可能是32位也可能是64位）64位的数据类型只有long和double两种。值得一提的是，这里把long和double数据类型分割存储的做法与“long和double的非原子性协定”中把一次long和double数据类型分割存储为两次32位读写的做法有些类似。不过，由于局部变量表建立在线程的堆栈上，是线程私有的数据，无论读写两个连续的Slot是否为原子操作，都不会引起数据安全问题。

虚拟机通过索引定位的方式使用局部变量表，索引值的范围是从0开始至局部变量表最大的Slot数量。如果访问的是32位数据类型的变量，索引n就代表了使用第n个Slot，如果是64位数据类型的变量，则说明会同时使用n和n+1两个Slot。对于两个相邻的共同存放在一个64位数据的两个Slot，不允许采用任何方式单独访问其中的某一个，JVM规范中明确要求了如果遇到进行这种操作的字节码序列，虚拟机应该在加载的校验阶段抛出异常。

在方法执行时，虚拟机就使用局部变量表完成参数值变量列表的传递过程的，如果执行的是实例方法，那局部变量表中第0位所以的Slot默认是用于传递方法所属对象实例的引用，在方法中可以通过关键字this来访问到这个隐藏的参数。其余参数则按照参数表顺序排列，占用从1开始的局部变量Slot，参数表分配完毕后，再根据方法体内部定义的变量顺序和作用域分配其余的Slot。

为了尽量节省栈空间，局部变量表中的Slot是可以重用的，方法体中定义的变量，其作用域并不一定会覆盖整个方法体，如果，如果当前字节码PC计数器的值已经超出了整个变量的作用域，那这个变量对应的Slot就可以交给其他变量使用。不过，这样的设计除了节省栈帧空间之外，还会伴随着一些额外的副作用，例如，在某些情况下，Slot的复用会直接影响到系统的垃圾收集行为。代码中向内存中填充了64MB的数据，然后通知虚拟机进行垃圾回收，需要在虚拟机参数上加入“-verbose:gc”来观察垃圾收集的过程。
``` java
public static void main(String[] args) {
    byte[] placeholder = new byte[64 * 1024 * 1024];
    System.gc();
}

//运行结果如下
[GC (System.gc())  91791K->66264K(1256448K), 0.0275929 secs]
[Full GC (System.gc())  66264K->66167K(1256448K), 0.0060836 secs]
```
没有回收placeholder所占的内存能说得过去，因为在执行gc()的时候，变量placeholder还处于作用域之内，虚拟机自然不敢回收placeholder的内存。那我们把代码修改一下，如下，placeholder的作用域被限制在了花括号之内，从逻辑上将，在执行gc的时候，placeholder已经不可能再被访问了，但是执行了这段程序，会发现结果如下，还是有64MB的内存没有被回收。
``` java
public static void main(String[] args) {
    {
        byte[] placeholder = new byte[64 * 1024 * 1024];
    }
    System.gc();
}

//运行结果如下
[GC (System.gc())  91791K->66264K(1256448K), 0.0275929 secs]
[Full GC (System.gc())  66264K->66167K(1256448K), 0.0060836 secs]
```

再改成如下代码，运行后发现内存反而被正常回收了：
``` java
public static void main(String[] args) {
    {
        byte[] placeholder = new byte[64 * 1024 * 1024];
    }
    int a = 0;
    System.gc();
}

//运行结果如下
[GC (System.gc())  91791K->752K(1256448K), 0.0009265 secs]
[Full GC (System.gc())  752K->631K(1256448K), 0.0046714 secs]
```

之前的placeholder没有被回收的根本原因是：局部变量表中的Slot是否存在有关于placeholder数组对象的引用。第一次修改中，代码虽然已经离开了placeholder的作用域，但在此之后，没有任何对局部变量表的读写操作，placeholder原本所占用的Slot还没有被其他变量所复用，所以GCRoots一部分的变量表仍然保持着对它的关联。这种关联没有被及时打断，在绝大部分情况下影响都很轻微。但如果遇到一个方法，其后面的代码有一些耗时很长的操作，而前面又定义了占用大量内存、实际上已经不再使用的变量，手动将其设置为null值(用来代替那句int a=0，把变量对应的局部变量表Slot清空)便不见得是一个绝对有意义的操作，这种操作可以作为你一种在极其特殊的情况（对象占用内存大、此方法的栈帧上时间不能被回收、方法嗲用次数达不到JIT的编译条件）下的奇技来使用。

虽然在前面的代码中，将对象赋值为null是有用的，但是不应当对赋null值的操作有过多的依赖，更没有必要把它当做一个普遍的编码规范来推广。原因有两点，从编码的角度讲，以恰当的变量作用域来控制变量回收时间才是最优雅的解决方法。更关键的是，从执行的角度讲使用赋null值的操作来优化内存回收是建立在对字节码执行引擎概念模型的理解之上的。在虚拟机使用解释器执行时，通常与概念模型还比较接近，但经过JIT编译器后，才是虚拟机执行代码的主要方式，赋null值的操作在经过JIIT编译优化后就会被消除掉，这时候将变量设置为null就没有什么意义了。字节码被编译为本地代码后，对GC Roots的枚举也与解释执行期间有巨大差别，以前面的例子来看，第二种方式在gc()执行时就可以正确的回收掉内存，无须写成第三种方式。

关于局部变量表，还有一点可能会对实际开发产生影响，就是局部变量不像前面介绍的类变量那样存在“准备阶段”，我们已经知道类变量有两次赋初始值的过程，一次在准备阶段，赋予系统初始值；另一次在初始化阶段，赋予程序员定义的初始值。因此，即使在初始化阶段程序员没有为类变量赋予值也没有关系，类变量仍然具有一个确定的初始值。但局部变量就不一样，如果一个局部变量定义了但没有赋初始值就不能使用的，不要认为Java在任何情况下都存在整型变量默认为0，布尔值变量默认为false等这样的情况，下面的代码时无法被执行的，还好编译器能在编译期间检查到这一点并提示，即使编译器能通过或者手动生成字节码的方式制造出下面的代码，字节码校验的时候也会被虚拟机爱发现而导致加载失败。
``` java
public static void main(String[] args) {
    int a;
    System.out.println(a);
}
```

## 操作数栈
操作数栈也常成为操作站，它是一个后入先出的栈。同局部变量表一样，操作数栈的最大深度也在编译的时候写入到Code属性的max_stacks数据项中。操作数栈的每一个元素可以是任意的Java数据类型，包括long和double。32位数据类型所占用的栈容量为1,64位数据类型占用的栈容量为2。在方法执行的时候，操作数栈的深度都不会超过在max_stacks数据项中设定的最大值。

当一个方法刚刚开始执行的时候，这个方法的操作数栈是空的，在方法的执行过程中，会有各种字节码指令往操作数栈中写入和提取内容，也就是出栈入栈操作。例如，在做算术运算的时候是通过操作数栈来进行的，又或者在调用其他方法的时候是通过操作数栈来进行参数传递的。

操作数栈中元素的数据类型必须与字节码指令的序列严格匹配，在编译程序代码的时候，编译器要严格保证这一点，在类校验阶段的数据流分析中还要再次检验这一点。再以iadd指令为例，这个指令用于整型数加法，它在执行时，最接近栈顶的两个元素的数据类型必须为int类型，不能出现一个long和一个float使用iadd命令相加的情况。

另外，在概念模型中，两个栈帧作为虚拟机栈的元素，是完全相互独立的。但是在大多数虚拟机的实现里偶读会做一些优化处理，令两个栈帧出现一部分重叠。让下面的栈帧的部分操作数栈与上面栈帧的部分局部变量表重叠在一起。这样在进行方法调用时就可以共用一部分数据，无须进行额外的参数复制传递。

JVM的解释执行引擎称为“基于栈的执行引擎”，其中所指的“栈”就是操作数栈。

## 动态链接
每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用的是为了支持方法调用过程中的动态链接。我们知道Class文件的常量池中存在大量的符号引用，字节码中的方法调用指令就以常量池中指向方法的符号引用作为参数。这些符号引用一部分会在类加载阶段或者第一次使用的时候就转化为直接引用，这种转化称为静态解析。另外一部分将在每一次运行期间转化为直接引用，这部分为动态链接。关于这两个转化过程的详细信息，将在下面进行阐述。

## 方法返回地址
当一个方法开始执行后，只有两种方式可以退出这个方法：
- 第一种方式是执行引擎遇到任意一个方法返回的字节码指令，这时候可能会有返回值传递给上层的方法调用者（调用当前方法的方法称为调用者），是否有返回值和返回值的类型将根据遇到何方法返回指令来决定，这种退出方法的方式称为正常完成出口。
- 另一种退出方式是，在方法执行过程中遇到异常，并且这个异常没有在方法体中得到处理，无论是JVM内部产生的异常，还是代码中使用athrow字节码指令产生的异常，只要在本方法的异常表中没有搜索到匹配的异常处理器，就会导致方法退出，这种退出方法的方式称为异常完成出口（Abrupt Method Invocation Completion）。一个方法使用异常完成出口的方式退出，是不会给它的上层调用者产生任何返回值的。

无论采用哪种方式退出，在方法退出之后，都需要回到方法被调用的位置，程序才能继续执行，方法返回时可能需要在栈帧保存一些信息，用来帮助恢复它的上层方法的执行状态。一般来说，方法正常退出时，调用者的PC计数器的值可以作为返回地址，栈帧中很可能会保存这个计数器值。而方法异常退出时，返回地址是要通过异常处理器表来确定的，栈帧中一般不会保存这部分信息。

方法退出的过程实际上就等于把当前栈帧出栈，因此退出时可能执行的操作有：恢复上层方法的局部变量表和操作数栈，把返回值（如果有的话）压入调用者操作数栈中，调整PC计数器的值以指向方法调用指令后面的一条指令等。

# 方法调用
方法调用不等同于方法执行，方法调用阶段唯一的任务就是确定被调用方法的版本，即调用哪一个方法，暂时还不涉及方法内部的具体运行过程。在程序运行时，运行方法调用是最普遍、最频繁的操作，但前面已经讲过，Class文件的编译过程不包含传统编译中的连续步骤，一切方法调用在Class文件里面存储的都是符号引用，而不是方法在实际运行时内存布局中的入口地址（相当于之前说的直接引用）。这个特性给Java带来了更强大的动态扩展能力，但也使得Java方法调用过程变的相对复杂起来，需要在类加载期间，甚至到运行期间才能确定目标方法的直接引用。

## 解析
所有方法调用的目标方法在Class文件里面都是一个常量池中的符号引用，在类加载的解析阶段，会将其中的一部分符号引用转化为直接引用，这种解析能成立的前提是：方法在程序真正运行之前就有一个可确定的调用版本，并且这个方法的调用版本在运行期是不可改变的。换句话说，调用目标在程序代码写好、编译器进行编译时就必须确定下来。这类方法的调用称为解析（Resolution）。

在Java语言中符合“编译期可知，运行期不变”这个要求的方法，主要包括静态方法和私有方法两大类，前者与类型直接关联，后者在外部不可被访问，这两种方法各自的特点决定了它们都不可能通过继承或别的方式重写其他版本，因此它们都不适合在类加载阶段进行解析。

与之相应的是，在Java虚拟机里面提供了5条方法调用字节码指令，如下：
- invokestatic：调用静态方法
- invokespecial：调用实例构造器init方法、私有方法和父类方法
- involevirtual：调用所有的虚方法
- invokeinterface：调用接口方法，会在运行时再确定一个实现此接口的对象
- invokedynamic：在运行时动态解析出调用点限定符所引用的方法，然后再执行该方法，在此之前的4条调用指令，分派逻辑是固化在JVM内部的，而invokedynamic指令的分派逻辑是由用户所设定的引导方法决定的。

只要能被invokestatic和invokespecial指令调用的方法，都可以在解析阶段中确定唯一的版本调用，符合这个条件的有静态方法、私有方法、实例构造器、父类方法4类，它们在类加载的时候就会把符号引用解析为该方法的直接引用。这些方法可以称为非虚方法，与之相反，其他方法称为虚方法（除去final方法）。

Java中的非虚方法除了使用了invokestatic、invokespecial调用的方法之外还有一种，就是被final修饰的方法，虽然final方法是使用invokevirtual指令来调用，但是由于它无法被覆盖，没有其他版本，所以也无须对方法接收者进行多态选择，又或者说多态选择的结果肯定是唯一的。在Java语言规范中明确说明了final方法是一种非虚方法。

解析调用一定是个静态的过程，在编译期间就完全确定，在类装在的解析阶段就会把涉及的符号全部转变为可确定的直接引用，不会延迟到运行期再去完成。而分派（Dispatch）调用则可能是静态的也可能是动态的，根据分派依据的宗量数可分为单分派和多分派。这两类分派方式的两两组合就构成了静态单分派、静态多分派、动态单分派、动态多分派4种分派组合情况，下面我们再看看虚拟机中的方法分派是如何进行的。

## 分派
众所周知，Java是一门面向对象的语言，因为Java具备面向对象的3个基本特征：继承、封装、多态。这里讲的分派调用过程将会揭示多态性特征的一些最基本的提现，如“重载”和“重写”在JVM中是如何实现的，这里的实现当然不是语法上应该如何去写，我们关心的依然是虚拟机如何确定正确的目标方法。

### 静态分派
阅读下面代码：

``` java
public class StaticDispatch {
    static abstract class Human {

    }

    static class Man extends Human {

    }

    static class Woman extends Human {

    }

    public void sayHello(Human guy) {
        System.out.println("hello, guy");
    }

    public void sayHello(Man guy) {
        System.out.println("hello, man");
    }

    public void sayHello(Woman guy) {
        System.out.println("hello, woman");
    }


    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        StaticDispatch staticDispatch = new StaticDispatch();
        staticDispatch.sayHello(man);
        staticDispatch.sayHello(woman);
    }
}
```
这是在考察阅读者对重载的理解程度，Human man = new Man();中，我们吧Human称为变量的静态类型，或者叫做外观类型，后面的Man叫做变量的时机类型，静态类型和实际类型在程序中都可可能发生一些变化，区别是静态类型的变化仅仅在使用时发生，变量本身的静态类型不会被改变，并且最终的静态类型是在编译期可知的；而实际类型变化的结果在运行期才可确定，编译器在编译程序的时候并不知道一个对象的实际类型是什么。例如：
``` java
//实际类型变化
Human man = new Man();
man = new Woman();
//静态类型
staticDispatch.sayHello((Man) man);
staticDispatch.sayHello((Woman) man);
```

main里执行了两次sayHello()方法调用，在方法接收者已经确定是对象staticDispatch的前提下，使用哪个重载版本，就完全取决于传入参数的数量和数据类型。代码中刻意的定义了两个静态类型相同但实际类型不同的变量，但虚拟机（准确的说是编译器）在重载的时候是通过参数的静态类型而不是实际类型作为判定依据的。并且静态类型是编译期可知的。因此，在编译阶段，Javac编译器会根据参数静态类型决定使用哪个重载版本，所以选择了sayHello(Human)作为调用目标，并把这个方法符号引用写到main方法里的两条invokevirtual指令的参数中。

所有依赖静态类型来定位方法执行版本的分派动作称为静态分派。静态分派的典型方法是方法重载。静态分派发生在编译阶段，因此确定静态分析的动作实际上不是由虚拟机来执行的。另外，编译器虽然能确定出方案的重载版本，但在很多情况下这个重载版本并不是唯一的，往往能确定出方法的重载版本。产生这种模糊结论的原因是字面量不需要定义，所以字面量没有显示的静态类型，它的静态类型只能通过语言上的规则去理解和推断。

### 动态分派
动态分派和多态性的另一个重要体现，重写有着密切的关联。
``` java
public class DynamicDispatch {
    static abstract class Human {
        protected abstract void sayHello();
    }

    static class Man extends Human {
        @Override
        protected void sayHello() {
            System.out.println("man say hello");
        }
    }

    static class Woman extends Human {
        @Override
        protected void sayHello() {
            System.out.println("woman say hello");
        }
    }

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        man.sayHello();
        woman.sayHello();
        man = new Woman();
        man.sayHello();
    }
}
```
运行结果：
```
man say hello
woman say hello
woman say hello
```

这个运行结果相信不会出乎任何人的意料，我们还是要知道虚拟机如何调用到相应方法的。这显然不可能再根据静态类型来决定，因为静态类型同样都是Human的两个变量man和woman在调用sayHello()方法时执行了不同的行为，并且变量man在两次调用中执行了不同的方法。导致这个现象的原因很明显，是这两个变量的时机类型不同，JVM是如何根据类型来分派执行版本的呢？我们使用javap命令输出这段代码的字节码，从中寻找答案：

```
public static void main(java.lang.String[]);
  descriptor: ([Ljava/lang/String;)V
  flags: ACC_PUBLIC, ACC_STATIC
  Code:
    stack=2, locals=3, args_size=1
       0: new           #2                  // class DynamicDispatch$Man
       3: dup
       4: invokespecial #3                  // Method DynamicDispatch$Man."<init>":()V
       7: astore_1
       8: new           #4                  // class DynamicDispatch$Woman
      11: dup
      12: invokespecial #5                  // Method DynamicDispatch$Woman."<init>":()V
      15: astore_2
      16: aload_1
      17: invokevirtual #6                  // Method DynamicDispatch$Human.sayHello:()V
      20: aload_2
      21: invokevirtual #6                  // Method DynamicDispatch$Human.sayHello:()V
      24: new           #4                  // class DynamicDispatch$Woman
      27: dup
      28: invokespecial #5                  // Method DynamicDispatch$Woman."<init>":()V
      31: astore_1
      32: aload_1
      33: invokevirtual #6                  // Method DynamicDispatch$Human.sayHello:()V
      36: return

```
0~15行的字节码是准备动作，作用是建立man和woman的内存空间、调用Man和Woman类型的实例构造器，将这两个实例放在第1、2个布局变量表Slot中，这个动作对应了代码的：
``` java
    Human man = new Man();
    Human woman = new Woman();
```
接下来的16~21也是关键部分，16、20句分别把刚刚创建的两个对象的引用压到栈顶，这两个对象是将要执行sayHello方法的所有者，称为接收者（Receiver）；17和21句是方法调用指令，这两条调用指令单从字节码角度来看，无论是指令（都是invokevirutal）还是参数（都是常量池中第22项的常量，注释显示了这个常量是Human.sayhello的符号引用）完全一样，但是这两句指令最终执行的目标方法并不相同。原因就需要从invokevirtual指令的多态查找过程开始说起，invokevirtual指令的运行时解析过程大致可以分为以下几个步骤：
1. 找到操作数栈顶的第一个元素所指向的对象的实际类型，记作C
2. 如果在类型C中找到与常量中的描述符合简单名称都相符的方法，则进行访问权限校验，如果通过则返回这个方法的直接引用，查找过程结束；如果不通过，则返回java.lang.IllegalAccessError异常
3. 否则，按照继承关系从下往上依次对C的各个父类进行第2步的搜索和验证过程
4. 如果始终没有找到合适的方法，则抛出java.lang.AbstractMethodError异常。

由于invokevirtual指令执行的第一步就是在运行期确定接收者的实际类型，所以两次调用invokevirtual指令把常量池中的类方法符号引用解析到了不同的直接引用上，这个过程就是Java语言中方法重写的本质，我们把这种运行期根据实际类型确定方法执行版本的分派过程称为动态分派。

### 单分派与多分派
方法的接收者与方法的参数统称为方法的宗量，这个定义最早应该来源于《Java与模式》一书。根据分派基于多少种宗量，可以降分派划分为单分派和多分派两种。单分派是根据一个宗量对目标方法进行选择，多分派是根据多于一个宗量对目标方法进行选择。

``` java
public class Dispatch {
    static class QQ {}

    static class _360 {}

    public static class Father {
        public void hardChoice(QQ arg) {
            System.out.println("father choose qq");
        }

        public void hardChoice(_360 arg) {
            System.out.println("father choose 360");
        }
    }

    public static class Son extends Father {
        public void hardChoice(QQ arg) {
            System.out.println("son choose qq");
        }

        public void hardChoice(_360 arg) {
            System.out.println("son choose 360");
        }
    }

    public static void main(String[] args) {
        Father father = new Father();
        Father son = new Son();
        father.hardChoice(new _360());
        son.hardChoice(new QQ());
    }
}
```
运行结果：
```
father choose 360
son choose qq
```
在main函数中调用了两次hardChoice()方法，这两次调用的选择结果在程序输出中已经显示的很清楚了。

我们来看看编译阶段编译器的选择过程，也就是静态分派的过程。这时选择目标方法的依据有两点：一是静态类型是Father还是Son，二是方法参数是QQ还是360.这次选择结果的最终产物是产生了两条invokevirtual指令，两条指令的参数分别为常量池中指向Father.hardChoice(360)以及Father.hardChoice(QQ)方法的符号引用。因此是根据两个宗量进行选择，所以Java语言的静态分析属于多分派类型。

再看看运行阶段虚拟机的权责，也就是动态分派的过程。在执行son.hardChoice(new QQ());这段代码时，更准确的说，是在执行这句代码所对应的invokevirtual指令时，由于编译期已经决定目标方法的签名必须为hardChoice(QQ)，虚拟机此时不关心传递过来的参数到底是什么QQ，因为这时参数的静态类型、实际类型都对方法的选择不会构成任何影响，唯一可以影响虚拟机选择的因素只有此方法的接受者的时机类型是Father还是Son。因为只有一个宗量作为选择依据，所以Java语言的动态分派属于单分派。

根据上面的结论，我们可以总结一句话：**现在的Java语言是一门静态多分派、动态单分派的语言**。这个结论并不是恒久不变的，C#在3.0及之前版本与Java一样是动态单分派语言，但是在C#4.0中引入了dynamic类型后，就可以很方便的实现动态多分派。

按照目前Java语言的发展趋势，它并没有直接变为动态语言的迹象，而是通过内置动态语言（如JavaScript）执行引擎的方式来满足动态性的需求。但是JVM层面上并不是如此的，在JDK1.7中已经开始提供对动态语言的支持了，JDK1.7中新增的invokedynamic指令也成为了最复杂的一条方法调用的字节码指令，稍后笔者将专门讲解这个JDK1.7的新特性。

### 虚拟机动态分派的实现
前面介绍的分派过程，作为对虚拟机概念模型的解析基本上已经足够了，它已经解决了虚拟机在分派中“会做什么”的这个问题，但是虚拟机“具体如何做到”，可能各种虚拟机实现会有差别。

由于动态分派是非常频繁的动作，而且动态分派的方法版本选择过程需要运行时在类方法元数据中搜索合适的目标方法，因此在虚拟机的时机实现中基于性能的考虑，大部分实现都不会真正的进行如此频繁的搜索。而面对这种情况，最常用的稳定优化手段就是为类在方法区中建立一个虚方法表（Virtual Method Table，也叫itable，于此对应的，在invokeinterface执行时也会用到接口方法表，Interface Method Table，简称itable），使用虚方法表索引来代替元数据查找以提高性能。

虚方法表中存放着各个方法的实际入口地址。如果某个方法在子类中没有被重写，那子类的虚方法表里面的地址入口和父类相同方法的地址入口是一致的，都指向父类的实现入口。如果子类中重写了这个方法，子类方法表中的地址将会被替换为指向子类实现版本入口的地址。

Son重写了来自Father的全部方法，因此Son的方法表没有指向Father类型数据的箭头。但是Son和Father都没有重写来自Object的方法，所以它们的方法表中所有从Object继承来的方法都指向了Object的数据类型。

为了程序实现上的方便，具有相同签名的方法，在父类、子类的虚方法表中都应当具有一样的索引序号， 这样当类型变换时，仅需求变更查找的方法表，就可以从不同的虚方法表中按索引转换出所需的入口地址。

## 动态语言支持
JVM的字节码指令集的数量从Sun公司的第一款JVM问世至JDK7来临之前的十余年时间里，一直没有发生任何变化。随着JDK7的发布，字节码指令集终于添加了一个新成员，invokedynamic指令。这条心增加的指令是JDK7实现“动态类型语言”支持而进行的改进之一，也是为JDK8可以顺利实现Lambda表达式做技术准备。

### 动态类型语言
动态类型语言的关键特征是它的类型检查的主题过程是在运行期而不是编译期，满足这个特性的语言有很多，包括：APL、Clojure、Erlang、Groovy、JavaScript、Jython、Lisp、Lua、PHP、Prolog、Python、Ruby、Smalltalk和Tel等。相对于，在编译期就进行类型检查过程的语言（比如C++和Java等）就是最常用的静态类型语言。
``` java
public static void main(String[] args) {
    int[][][] a = new int[1][0][-1];
}
```
这段代码时可以正常编译的，但运行的时候会报NegativeArraySizeException异常。在JVM规范中明确规定了NegativeArraySizeException是一个运行时异常，通俗一点讲，运行时异常就是只要代码不运行到这一行就不会有问题。与运行时异常对应的就是连接时异常，即使会导致连接时异常的代码放在一条无法执行到的分支路径上，类加载时（Java的连接过程不在编译阶段，而在类加载阶段）也照样会跑出异常。

不过C语言会在编译期报错：
``` c
int main(void) {
    int i[1][0][-1];//GCC拒绝编译，报“size of array is negative”
    return 0;
}
```

动态和静态类型语言谁更先进呢？这个不会有确切的答案。
- 静态类型语言在编译期确定类型，最显著的好处是编译期可以提供严谨的类型检查，这样与类型相关的问题能在编码的时候就及时被发现，利于稳定性以及代码达到更大的规模。
- 动态类型语言在运行期确定类型，这可以为开发者提供更大的灵活性，某些静态类型语言中需要大量臃肿代码来实现的功能，由动态类型语言来实现可能更加清晰和简洁，也就意味着开发效率的提升。

### JDK1.7与动态类型
JDK1.7以前的字节码指令集中，4条方法调用指令（invokevirtual、invokespecial、invokestatic、invokeinterface）的第一个参数都是被调用的方法的符号引用（CONSTANT_Methodref_info或者CONSTANT_InterfaceMethodref_info常量），方法的符号引用在编译时产生，而动态类型语言只有在运行期才能确定接受者类型。这样，在JVM上实现的动态类型语言就不得不使用其他方式，比如在编译时留个占位符类型，运行时动态生成字节码实现具体类型到占位符类型的适配来实现，这样会让动态类型语言实现的复杂度增加，也可能带来额外的性能开销。尽管可以利用一些方法让这些开销变小，但这种底层问题终究是应当在虚拟机层次上去解决才最合适，因此在JVM层面上提供动态类型的直接支持就称为了JVM平台的发展趋势之一，这就是JDK1.7中invokedynamic指令以及java.lang.invoke包出现的技术背景。

### java.lang.invoke包
JDK1.7实现了JSK-292，新加入的java.lang.invoke包就是JSR-292的一个重要组成部分，这个包的目的是在之前单纯依靠符号引用来确定调用的目标方法这种方式以外，提供一种新的动态确定目标方法的机制，称为MethodHandle。拥有MethodHandle之后，Java语言也可以拥有类似函数指针或者委托的方法别名的工具了。

``` java
public class MethodHandleTest {
    static class ClassA {
        public void println(String s) {
            System.out.println(s);
        }
    }

    public static void main(String[] args) throws Throwable {
        Object obj = System.currentTimeMillis() % 2 == 0 ? System.out : new ClassA();
        //无论obj最终是哪个实现类，下面这句都能正确调用到println方法
        getPrintlnMH(obj).invokeExact("sss");
    }

    private static MethodHandle getPrintlnMH(Object receiver) throws Throwable {
        /*MethodType：代表方法类型，包含了方法的返回值methodType()的第一个参数和具体参数methodType()第二个及以后的参数。*/
        MethodType mt = MethodType.methodType(void.class, String.class);
        /*lookup()方法的作用是在指定类中查找符合给定的方法名称、方法类型，并且符合调用权限的方法句柄
        因为这里调用的是一个虚方法，按照Java语言的规则，方法第一个参数是隐式的，代表该方法的接收者，也即是this指向的对象，这个参数之前是放在参数列表中传递的，而现在提供了bindTo()方法来完成这件事情*/
        return MethodHandles.lookup().findVirtual(receiver.getClass(), "println", mt).bindTo(receiver);
    }
}
```
实际上，getPrintlnMH()方法模拟了invokevirtual指令的执行过程，只不过它的分派逻辑并非固化在Class文件的字节码上，而是通过一个具体方法来实现。而这个方法本身的返回值（MethodHandle对象），可以视作最终调用这个方法的一个“引用”。以此为基础，有了MethodHandle就可以写出类似下面的函数声明：
``` java
void sort(List list, MethodHandle methodHandle)
```

仅仅站在Java的角度来看，MethodHandle的使用方法和效果与Reflection有众多相似之处，但是，它们还有以下这些区别。
- 从本质上讲，Reflection和MethodHandle机制都是在模拟方法调用，但Reflection是在模拟Java代码层次的方法调用，而MethodHandle是在模拟字节码层次的方法调用，在MethodHandles.lookup中的3个方法——findStatic()、findVirtual()、findSpecial()正是为了对应于invokestatic、invokevirtual & invokeinterface、invokespecial这几条字节码指令的执行权限校验行为，而这些底层细节在使用Reflection API时是不需要关心的。
- Reflection中的java.lang.reflect.Method对象远比MethodHandle机制中的java.lang.invoke.MethodHandle对象所包含的信息多。前者是方法在Java一端的全面映像，包括方法的签名、描述符以及方法属性表中各个属性的Java端表示方式，还包含执行权限等运行时信息。而后者仅仅包含与执行该方法相关的信息。用通俗的话讲，Reflection是重量级的，MethodHandle是轻量级的。
- 由于MethodHandle对字节码的方法指令调用的模拟，所以理论上虚拟机在这方面做的各种优化（比如方法内联），在MethodHandle上也应可以采用类似思路去支持（但目前还不完善）。而通过反射区调用方法则不行。

MethodHandle和Reflection除了上面列举的区别外，最关键的一点在于去掉前面的“仅仅站在Java的角度来看”。Reflection的设计目标是只为Java语言服务的，而MethodHandle则设计成可以服务于所有Java虚拟机之上的语言，其中也包括Java语言。

### invokedynamic指令
从某种程度上讲，invokedynamic指令与MethodHandle机制的作用是一样的，都是为了解决原有4条“invoke*”指令方法分派规则固化在虚拟机之中的问题，把如何查找目标方法的决定权从虚拟机转嫁到具体用户代码之中，让用户有更高的自由度。而且两者的思路也是可类比的，可以把它们想象成为了达到同一个目的，一个采用上层Java代码和API实现，另一个采用字节码和Class中其他属性、常量来完成。因此，如果理解了MethodHandle，那么理解invokedynamic指令也并不难。

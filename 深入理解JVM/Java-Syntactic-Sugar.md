几乎各种语言都会或多或少的提供一些语法糖来方便程序员的代码开发，这些语法糖虽然不会提供实质性的功能改进，但是他们或能提高效率，或能提升语法的严谨性，或能减少编码出错的机会。不过也有一种观点认为语法糖并不一定是有益的，大量添加和使用语法糖，容易让程序员产生依赖，无法看清语法糖的糖衣背后，程序代码的真正面目。
## 泛型和类型擦除
泛型是在JDK1.5的一项新增特性，它的本质是参数化的类型的应用，也就是说所操作的数据类型指定为一个参数。这种参数类型可以用在类、接口和方法的创建中，分别称为泛型类、泛型接口、泛型方法。

泛型思想早在C++语言的模板中就开始萌芽，在Java语言处于还没有出现泛型版本时，只能通过Object是所有类型的父类和类型强制转换两个特点来配合实现类型泛化。例如，在哈希表的存取中，JDK1.5之前使用HashMap的get方法，返回值就是一个Object对象，由于Java语言里面所有的类型都继承与java.lang.Object，所以Object转型成任何对象都是有可能的。但是也因为有无限种可能，就只有程序员和运行期的虚拟机才知道Object到底是什么对象。在编译期间，编译器无法检查到这个Object的强制类型转换是否成功，如果仅仅依赖程序员去保障这项操作的正确性的话，会有许多ClassCastException的风险转嫁到程序运行期间。

泛型在Java和C#中的使用方式看似相同，但实际上却有着根本性的分歧。C#里面泛型无论在程序源码中、编译后的IL中（Intermediate Language，中间语言，这时候泛型是一个占位符），或是运行期的CLR中，都是切实存在的， List&lt;int> 与 List&lt;String> 就是两个不同的类型，它们在系统运行期生成，有自己的虚方法表和类型数据，这种实现称为类型膨胀，基于这种方法实现的泛型称为真实泛型。

Java中的泛型则不一样，它只在程序源码中存在，在编译后的字节码文件中，就已经替换成原来的原生类型了，并且在相应的地方插入了强制转型代码，因此，对于运行期的Java语言来说，ArrayList&lt;Integer> 和ArrayList&lt;String> 就是同一个类型，所以泛型技术实际上是Java语言的一颗语法糖，Java语言中的泛型实现方法称为类型擦除，基于这种方法实现的泛型称为伪泛型。

看一段简单的源码：
``` java
public class HH {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        list.add("1");
        list.add("2");
        list.add("3");
    }
}
```
将这段Java代码编译成Class文件，然后再用字节码反编译工具进行反编译后，会发现泛型都不见了，程序又变回了Java泛型出现之前的写法，泛型类型都变回了原生类型，如下所示：

``` java
public class HH {
    public HH() {
    }

    public static void main(String[] var0) {
        ArrayList var1 = new ArrayList();
        var1.add("1");
        var1.add("2");
        var1.add("3");
    }
}
```
当初JDK设计团队为什么选择类型擦除的方式来实现Java语言的泛型？这个不得而知，但是确实有不少人对Java语言提供的伪泛型颇有微词。在当时众多批评之中，有一些比较表面的，还有一些从性能上说泛型会由于强制转换操作和运行期缺少针对类型的优化等从而导致比C#的泛型要慢，则是完全偏离了方向，姑且不论Java泛型是不是真的会比C#泛型慢，选择从性能的角度评价用于提升语义准确性的泛型思想就不太恰当。但是在某些场景下，通过擦除法来实现泛型丧失了一些泛型思想应有的优雅：
``` java
public class HH {
    static void method(List<String> list) {

    }

    static void method(List<Integer> list) {

    }
}
```
这段代码是不能被编译的，因为List&lt;Integer>和List&lt;String>编译之后都会被擦除，变成了一样的原生类型List&lt;E>，擦除动作导致这两种方法的特征签名变得一模一样。

但是在JDK1.6的环境下（JDK1.7和JDK1.8均是不可以的 ），下面的情况也是可以被编译的。

``` java
public class HH {
    static String method(List<String> list) {
        return "";
    }

    static int method(List<Integer> list) {
        return 1;
    }

    public static void main(String[] args) {
        String s = method(new ArrayList<String>());
        int i = method(new ArrayList<Integer>());
    }
}
```
重载当然不是根据返回值确定的，之所以这次能被编译和执行，是因为两个method方法加入了不同的返回值后才能共存在一个Class文件中。Class文件方法表（method_info）的数据结构中，方法重载要求方法具备不同的特征签名，返回值并不包含在方法的特征签名之中，所以返回值不参与重在选择，但是在Class文件格式之中，只要描述符不是完全一致的两个方法就可以共存。也就是说，两个方法如果有相同的名称和特征签名，但返回值不同，那它们也是可以合法共存于一个Class文件中的。

## 自动装箱、拆箱、遍历循环
从纯技术的角度讲，自动装箱、自动拆箱和循环遍历（Foreach遍历）这些语法糖，无论是实现上还是思想上都不能和泛型相比，两种的难度和深度有很大差距
。如下展示的是自动装箱、拆箱与遍历循环。

``` java
public class HH {
    public static void main(String[] args) {
        List<Integer> arrayList = Arrays.asList(1, 2, 5, 14, 5);
        int sum = 0;
        for (Integer integer : arrayList) {
            sum += integer;
        }
        System.out.println(sum);
    }
}

```
反编译后的结果如下所示，需要注意的是，不同的反编译器反编译后的结果是不同的。
``` java
public class HH
{
    public static void main(String[] paramArrayOfString)
    {
        List localList = Arrays.asList(new Integer[] { 
                Integer.valueOf(1), 
                Integer.valueOf(2), 
                Integer.valueOf(5), 
                Integer.valueOf(14), 
                Integer.valueOf(5) });
        int var2 = 0;

        Integer var4;
        for(Iterator var3 = var1.iterator(); var3.hasNext(); var2 += var4) {
            var4 = (Integer)var3.next();
        }

        System.out.println(var2);
    }
}

```
上面的代码中包含了泛型、自动装箱、自动拆箱、循环遍历和变长参数，自动装箱和拆箱在变异后被转化成了对应的包装和还原方法，例如Integer.valueOf()，而遍历循环则把代码还原成了迭代器的实现，这也是为何遍历循环需要被遍历的类实现Iterable接口的原因。最后看看变长参数，它在调用的时候变成了一个数组类型的参数，在变长参数出现之前，程序员就是使用数组来完成类似功能的。

关于自动装箱
``` java
public static void main(String[] args) {
    Integer a = 1;
    Integer b = 2;
    Integer c = 3;
    Integer d = 3;
    Integer e = 321;
    Integer f = 321;
    Long g = 3L;
    System.out.println(c == d);
    System.out.println(e == f);
    System.out.println(c == (a + b));
    System.out.println(c.equals(a + b));
    System.out.println(g == (a + b));
    System.out.println(g.equals(a + b));
}
打印结果：
true
false
true
true
true
false
```
鉴于包装类的“==”运算在遇不到算数运算的情况下不会自动拆箱，以及它们的equals方法不处理数据转型的关系，建议在实际编码中尽量避免这样使用自动装箱和拆箱。但是打印结果的前两条还是有些匪夷所思，看一看编译后的代码，能发现一些问题：

``` java
public static void main(String[] paramArrayOfString)
{
    Integer localInteger1 = Integer.valueOf(1);
    Integer localInteger2 = Integer.valueOf(2);
    Integer localInteger3 = Integer.valueOf(3);
    Integer localInteger4 = Integer.valueOf(3);
    Integer localInteger5 = Integer.valueOf(321);
    Integer localInteger6 = Integer.valueOf(321);
    Long localLong = Long.valueOf(3L);
    int i = 3;
    System.out.println(localInteger3 == localInteger4);
    System.out.println(localInteger5 == localInteger6);
    System.out.println(localInteger3.intValue() == localInteger1.intValue() + localInteger2.intValue());
    System.out.println(localInteger3.equals(Integer.valueOf(localInteger1.intValue() + localInteger2.intValue())));
    System.out.println(localLong.longValue() == localInteger1.intValue() + localInteger2.intValue());
    System.out.println(localLong.equals(Integer.valueOf(localInteger1.intValue() + localInteger2.intValue())));
    System.out.println(i == 3);
}
```
Integer在初始化的时候，其实是调用了Integer.valueOf()方法，如下：
``` java
    /**
     * Returns an {@code Integer} instance representing the specified
     * {@code int} value.  If a new {@code Integer} instance is not
     * required, this method should generally be used in preference to
     * the constructor {@link #Integer(int)}, as this method is likely
     * to yield significantly better space and time performance by
     * caching frequently requested values.
     *
     * This method will always cache values in the range -128 to 127,
     * inclusive, and may cache other values outside of this range.
     *
     * @param  i an {@code int} value.
     * @return an {@code Integer} instance representing {@code i}.
     * @since  1.5
     */
    public static Integer valueOf(int i) {
        assert IntegerCache.high >= 127;
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
```
注释中明确指出，如果数值在-128~127中间的话，将使用缓存值，其他数值会创建新的Integer对象，因此c和d的比较其实就是Integer缓存的比较，而非对象和对象之间的比较，故返回的是true。







---
layout: post
title: Kotlin 特性学习（1）基本特性
subtitle: Kotlin 基本特性总结，读kotlin核心编程总结
image: /img/lifedoc/jijiji.jpg
tags: [Kotlin]
---

### 《kotlin核心编程》读书总结

kotlin常用表达式，if、when、try等，其中if表达式如果要达到三目运用，只需要在语句中加上else即可，例如：

```kotlin
val value = if (true) 1 else 2
```

kotlin中unit是一个类型，而不是一个关键字，例如一个方法没有显示声明返回值，那么他就返回unit类型，这个类似void的作用。

**表达式函数体**

单行加等号的函数声明称为表达式函数体，例如：

```kotlin
    // testFun 是 toString 的结果，直接调用相当于调用toString
    val testFun = toString()

    // 这个是kotlin deCompile之后的翻译，这个就相当于testFun存起来函数执行之后的结果而已
    @NotNull
    private final String testFun = this.toString();

    // 表达式函数体
    fun testF() = toString()
    fun testF2(a: Int, b : Int) = a + b

    // 高阶函数，传入函数作为参数，然后在这个函数里面执行函数
    fun useFF(c: Int, result: (Int, Int) -> Int){
        Log.d(KSTAG, "c is $c result is ${c + result(c, c)}")
    }

    // 用法，注意这里是不需要再次传入a b类型的，因为定义的哪里已经明确了为Int了
    val useFFF = useFF(2) { a ,b ->
             a*b
        }
        useFF(100) { a ,b ->
            a*b - a - b - b -b
        }
    // 用法结束

    // 翻译为Java
     public final void useFF(int c, @NotNull Function2 result) {
      Intrinsics.checkNotNullParameter(result, "result");
      Log.d("KS_TAG", "c is " + c + " result is " + (c + ((Number)result.invoke(c, c)).intValue()));
   }


   // 7 params
   fun useFF(c: Int, result: (Int, Int, Int, Int, Int, Int, Int) -> Int){
        Log.d(KSTAG, "c is $c result is ${c + result(c, c, c, c, c, c,c)}")
    }

    // trans
    public final void useFF(int c, @NotNull Function7 result) {
      Intrinsics.checkNotNullParameter(result, "result");
      Log.d("KS_TAG", "c is " + c + " result is " + (c + ((Number)result.invoke(c, c, c, c, c, c, c)).intValue()));
   }

   // 查看定义最多22个哈哈哈
   Function22
```

**kotlin数组**

```kotlin
// 拥有原生的各种Array
val intA = IntArray(10)

// 对象类型的需要泛型指定
val stringsOrNulls = arrayOfNulls<String>(10) // returns Array<String?>
val someStrings = Array<String>(5) { "it = $it" }
val otherStrings = arrayOf("a", "b", "c")

// kotlin定义
val a : Array<String?> = Array(1, {null})
// 翻译为Java
byte var6 = 1;
String[] var7 = new String[var6];

// kotlin的一个bug，在传入Java的可变参数的时候会强转为Object，这个导致我的动态代理不能用kotlin去写
// 这里可以看到args传入的时候是Object[]
@NotNull
   public Object invoke(@Nullable Object proxy, @Nullable Method method, @Nullable Object[] args) {
       
      String result = "JGP 代理之前: " + this.seller;
      boolean var5 = false;
      System.out.println(result);
      result = null;
      Object var10000;
      if (args == null) {
         Intrinsics.checkNotNull(method);
         var10000 = method.invoke(this.seller);
      } else {
         Intrinsics.checkNotNull(method);
         // 这里的invoke实际上要求的是 Object... 类型，这里应该是识别有误，把这个参数没有加kotlin的 vararg了，因此转义为 Object
         var10000 = method.invoke(this.seller, (Object)args);
      }

      Object result = var10000;
      String var8 = "JGP 代理之后: " + this.seller;
      boolean var6 = false;
      System.out.println(var8);
      Intrinsics.checkNotNull(result);
      return result;
   }
```

kotlin的数组数据在堆内存中。

**lambda表达式**

1. 一个lambda表达式必须使用 {} 包裹
2. 如果lambda表达式声明了参数部分的类型，且返回值类型支持推导，那么lambda变量就可以省略函数变量声明
3. 如果lambda返回的不是unit，那么默认返回最后一句的执行结果

```kotlin
// 参数定义里声明了类型，lambda函数里也声明
val sum : (Int, Int) -> Int = {x: Int, y: Int -> x+y } 

// 参数不声明，lambda里面声明
val sum = {x: Int, y: Int -> x+y }

// 参数声明，lambda表达式里面省略
val sum : (Int, Int) -> Int = {x, y -> x+y } 
```

**invoke方法**
```kotlin

fun foo(tt: Int) = {
    print(tt)
}

// invoke 方法显式调用，如果不调用就是拿的print方法的引用
listOf(1, 2, 3).forEach { foo(it).invoke() }

// invoke方法也可以省略，以小括号代替
listOf(1, 2, 3).forEach { foo(it)() }

```

**闭包**

```kotlin

// foo 是一个匿名函数的引用，该函数有两个参数
val foo = { x: Int, y: Int -> x + y }
// Java 意义
this.foo = (Function2)null.INSTANCE;

// 两种调用方式
foo.invoke(1, 3)
foo.(1, 3)


// fooo(x: Int) 是后面lambda表达式的引用，所以按照第一种方式比较好理解，就是首先拿到fooo的引用再去invoke他们的方法，不懂，decompile看一下
fun fooo(x: Int) = { y: Int -> x + y }
// Java 翻译
@NotNull
   public final Function1 fooo(final int x) {

       // 这里可以看到第一个fooo(1) 只到这里，可以得到Function1对象，所谓的闭包也能从这里看出来，如果我们只调用到这里只有一个Function并且包含x的对象，再次调用invoke才能激活使用这个变量，理论上是变相的延长了首次传入x的生命周期
      return (Function1)(new Function1() {
         // $FF: synthetic method
         // $FF: bridge method
         public Object invoke(Object var1) {
            return this.invoke(((Number)var1).intValue());
         }

         public final int invoke(int y) {
            return x + y;
         }
      });
   }

// 两种调用方式
fooo(1).invoke(2)
fooo(1)(2)


// object形式
fooT(object : AbstractT() {
            override fun testMethod() {
                
            }
        })

// Java翻译，只是生成一个匿名内部类
(AbstractT)this.fooT.invoke(new AbstractT() {
         public void testMethod() {
         }
      });
```

在kotlin中，匿名函数体、lambda（以及局部函数、object表达式）在语法上都存在"{}"，由这对花括号包裹的代码如果访问外部环境变量就会被称为一个 **闭包**。从上面的例子中可以看到，x是外部的变量，fooo就称为一个闭包。

lambda是最常见的一种闭包形式。

**隔离性**：

表达式有良好的隔离性，就是说通过传入的参数能够更好的使用并且不改变他们的属性和值。

**when表达式**

when表达式的参数可以省略，这种情况下分支 -> 左侧需要是返回布尔值，否则编译会出错。这个看起来也没啥卵用啊，不用为了省点语句而省，这样语义多不明显。

**中缀函数**

可以使用infix 关键字以及 to函数 定义中缀函数，中缀函数的使用方式一般为
```kotlin

// a 中缀 b， to 这种形式的定义函数为中缀函数
infix fun <A, B> A.to(that: B): Pair<A, B>

class Person {
    infix fun called(name: String) {
        println("My name is ${name}")
    }
}

// 中缀形式
val p = Person()
p called "test"

// Java 形式
Person p = new Person();
p.called("MMM");

// kotlin 定义
class Person constructor(aaa: String) {
    init {
        println("ABC")
    }
}

// 这里定义的为主函数
class Person constructor(aaa: String) {

    // 这里 this(cc) 说明只含有一个String的为主函数，因此init块全部追加到只含有String的构造函数中
    constructor(bbb: Int, cc:String) : this(cc){
        println(bbb)
    }

    init {
        println("ABC")
    }

}

// Java 翻译
public Person(@NotNull String aaa) {
      Intrinsics.checkNotNullParameter(aaa, "aaa");
      super();

      // init 语句块，追加到最后了，init 可以增加很多语句块，会顺序追加到 主构造函数 后面，但是会在从函数前面执行，因为从函数首先调用主函数。
      String var2 = "ABC";
      boolean var3 = false;
      System.out.println(var2);
}

```

**等号定义**

两个等号定义值相等，三个等号定义引用相等。

**init方法**

init 方式的构造方法会在主构造函数的最后执行，如果init 语句块在调用的时候，从函数会首先调用主函数，因此这种情况下 init 语句块在从构造函数之前调用，如果需要永远在最后，那么就需要不再class 函数体哪里定义主函数，定义为统计的函数，这样正确追加。

学到65页，未完待续。。。

### 20210822 更新

**延迟初始化：by lazy 和 lateinit**

by lazy 和 lateinit 可以延迟初始化，其中

by lazy 语法特点为：

1. 改变量必须是引用不可变的，也就是使用 val 定义，不能使用 var 定义
2. 在第一次被使用时才会进行赋值操作
3. lazy 背后是接受一个 lambda 并返回一个 Lazy<T> 实例的函数，第一次访问该属性时，会执行 lazy 对应的 lambda 表达式并记录结果，后续访问该属性时只是返回记录的结果

```kotlin
// kotlin 代码
    val sex: String by lazy { 
        if (true) "male" else "female"
    }

// 翻译的Java代码
this.sex$delegate = LazyKt.lazy((Function0)null.INSTANCE);

// 这里返回的 Lazy<T> ，默认实现是 Synchronized 形式的实现
public actual fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)

// 同步方式实现原理
private class SynchronizedLazyImpl<out T>(initializer: () -> T, lock: Any? = null) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    @Volatile private var _value: Any? = UNINITIALIZED_VALUE
    // final field is required to enable safe publication of constructed instance
    private val lock = lock ?: this

    override val value: T
        get() {
            val _v1 = _value
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }

            return synchronized(lock) {
                val _v2 = _value
                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST") (_v2 as T)
                } else {
                    val typedValue = initializer!!()
                    _value = typedValue
                    initializer = null
                    typedValue
                }
            }
        }

    override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE

    override fun toString(): String = if (isInitialized()) value.toString() else "Lazy value not initialized yet."

    private fun writeReplace(): Any = InitializedLazyImpl(value)
}

// 几种模式
public final class LazyKt$WhenMappings {
   // $FF: synthetic field
   public static final int[] $EnumSwitchMapping$0 = new int[LazyThreadSafetyMode.values().length];

   // 1 添加同步锁 2 Publication 参数不加锁 3 None 没有任何线程安全的保证
   static {
      $EnumSwitchMapping$0[LazyThreadSafetyMode.SYNCHRONIZED.ordinal()] = 1;
      $EnumSwitchMapping$0[LazyThreadSafetyMode.PUBLICATION.ordinal()] = 2;
      $EnumSwitchMapping$0[LazyThreadSafetyMode.NONE.ordinal()] = 3;
   }
}

// Publication模式的实现
private class SafePublicationLazyImpl<out T>(initializer: () -> T) : Lazy<T>, Serializable {
    @Volatile private var initializer: (() -> T)? = initializer
    @Volatile private var _value: Any? = UNINITIALIZED_VALUE
    // this final field is required to enable safe initialization of the constructed instance
    private val final: Any = UNINITIALIZED_VALUE

    override val value: T
        get() {
            val value = _value
            if (value !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return value as T
            }

            val initializerValue = initializer
            // AtomicReferenceFieldUpdater 使用这个Field Updater 去保证，内部使用了 Updater cas 去比较这个值是否是未初始化的，如果是就更新，否则不动
            // if we see null in initializer here, it means that the value is already set by another thread
            if (initializerValue != null) {
                val newValue = initializerValue()
                if (valueUpdater.compareAndSet(this, UNINITIALIZED_VALUE, newValue)) {
                    initializer = null
                    return newValue
                }
            }
            @Suppress("UNCHECKED_CAST")
            return _value as T
        }

    override fun isInitialized(): Boolean = _value !== UNINITIALIZED_VALUE

    override fun toString(): String = if (isInitialized()) value.toString() else "Lazy value not initialized yet."

    private fun writeReplace(): Any = InitializedLazyImpl(value)

    companion object {
        private val valueUpdater = java.util.concurrent.atomic.AtomicReferenceFieldUpdater.newUpdater(
            SafePublicationLazyImpl::class.java,
            Any::class.java,
            "_value"
        )
    }
}
```

**kotlin 关键字**

```kotlin
is -> instanceOf
as -> Java 的强转

和类型
 A or B

积类型
 C and D
```

**协变 和 逆变**

out 协变： 这个和Java 的 <? extends T> 语义相同，相当于都是继承于 T 的类型，是 T 的子类型
in 逆变： 这个和 Java 的 <? super T> 语义相同，限定为是 T 的父类，但是 T 的子类也可以，因为 T 的子类明显是可以转变为 T 的

PCES 原则，out 协变和 in 逆变同样支持 PCES 原则， Producer 生产者需要使用 out 协变的类型， Consumer 消费者需要使用 in 逆变类型的泛型。

**内联函数**

使用 inline 定义内联函数，内联函数执行的时候会把代码编译到使用者内部，这样就可以少一层调用栈。
例如：翻译一下with的定义

```kotlin
// with是内联，其中接收者为 T ，传入的是一个 block 方法，最后返回一个 R 的泛型
inline fun <T, R> with(receiver: T, block: T.() ->  R): R

// apply 方法定义，作用到 T 的 apply 方法，调用 T 的 block 方法，最终的返回值是 Unit ，整个 apply 执行之后还是返回 接收者本身
inline fun <T> T.apply(block: T.() -> Unit): T

// T.() 代表调用 T 身上的 lambda 方法
```

**注解定义**

注解定义使用 annotatio class 定义注解。
使用 kapt 结合 kotlin 注解可以实现基于元编程的二次编程，基于注解生成类，或者进行拦截定义特定的注解 实现各种功能。

**响应式编程和流**

Rx 某某某是响应式编程， 流的代表是kotlin的Flow以及Java的Stream，可以不用Rx 方式编程但是响应式编程明显是一个主流。响应式编程是一个思想，就是流驱动程序的运行，收到流或者数据之后进行处理，根据不同的数据做出不同的相应，或者就像我的首页数据解析流程，不同的类型进入不同的 transfer 这样可以最大的灵活性去追求变化。

**函数式编程**

函数式编程，kotlin 目前不支持纯种的函数式编程，scala 便是纯种的函数式语言，函数式编程的特点是更加数学化或者抽象化，不人类化。因此学习的时候有一定的门槛，主要是思想的转变。

**数据类 与 密封类**

数据类型用data class 声明，密封类则用 sealed class 声明，其中，data 类一般用于做bean 且他不可继承，sealed 类多用于 when 子句，使用密封类的 when 子句可以没有else 仅仅弄几个继承与 sealed 的类也可。sealed 类只能在一个文件中继承，脱离之后因为他的构造函数是private 的并不能正常继承。

```kotlin
// kotlin
sealed class Xiaoming {
    abstract fun foo()
}

data class Xiaomingzizi(val tt: Int) : Xiaoming() {
    override fun foo() {

    }

}

// Java 
// data类默认为final 定义的class 因此不能继承
public final class Xiaomingzizi extends Xiaoming {
   private final int tt;

   public void foo() {
   }

   public final int getTt() {
      return this.tt;
   }

   // 这里是调用的有DefaultConstructorMarker的构造方法，因此这里可以继承，外部默认调用的是private 那个因此有问题，使用 sealed 需在一个kt 文件中定义
   public Xiaomingzizi(int tt) {
      super((DefaultConstructorMarker)null);
      this.tt = tt;
   }

   public final int component1() {
      return this.tt;
   }

   @NotNull
   public final Xiaomingzizi copy(int tt) {
      return new Xiaomingzizi(tt);
   }

   // $FF: synthetic method
   public static Xiaomingzizi copy$default(Xiaomingzizi var0, int var1, int var2, Object var3) {
      if ((var2 & 1) != 0) {
         var1 = var0.tt;
      }

      return var0.copy(var1);
   }

   @NotNull
   public String toString() {
      return "Xiaomingzizi(tt=" + this.tt + ")";
   }

   public int hashCode() {
      return this.tt;
   }

   public boolean equals(@Nullable Object var1) {
      if (this != var1) {
         if (var1 instanceof Xiaomingzizi) {
            Xiaomingzizi var2 = (Xiaomingzizi)var1;
            if (this.tt == var2.tt) {
               return true;
            }
         }

         return false;
      } else {
         return true;
      }
   }
}
// Xiaoming.java
package com.xpj.kotlingrowth;

import kotlin.Metadata;
import kotlin.jvm.internal.DefaultConstructorMarker;
// 默认是 abstract 类定义
public abstract class Xiaoming {
   public abstract void foo();

   // 这个密封类的构造函数为private
   private Xiaoming() {
   }

   // $FF: synthetic method
   public Xiaoming(DefaultConstructorMarker $constructor_marker) {
      this();
   }
}
```


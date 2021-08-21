---
layout: post
title: Kotlin 特性学习
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


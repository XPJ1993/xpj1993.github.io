---
layout: post
title: 复习Java（二）
subtitle: 反射与动态代理学习
image: /img/lifedoc/jijiji.jpg
tags: [Java]
---

### 反射

首先是反射相关的结构。

![反射相关类](https://raw.githubusercontent.com/Pjex/images/master/fansheclasspic.awebp)


Java 反射的主要组成部分有4个：
**Class**：任何运行在内存中的所有类都是该 Class 类的实例对象，每个 Class 类对象内部都包含了本来的所有信息。记着一句话，通过反射干任何事，先找 Class 准没错！

**Field**：描述一个类的属性，内部包含了该属性的所有信息，例如数据类型，属性名，访问修饰符······

**Constructor**：描述一个类的构造方法，内部包含了构造方法的所有信息，例如参数类型，参数名字，访问修饰符······

**Method**：描述一个类的所有方法（包括抽象方法），内部包含了该方法的所有信息，与Constructor类似，不同之处是 Method 拥有返回值类型信息，因为构造方法是没有返回值的。


反射中的用法有非常非常多，常见的功能有以下这几个：

1. 在运行时获取一个类的 Class 对象
2. 在运行时构造一个类的实例化对象
3. 在运行时获取一个类的所有信息：变量、方法、构造器、注解

#### 试验类对象

```java
package com.xpj.javagrowth.javaref;

import androidx.annotation.NonNull;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * author : xpj
 * date : 8/13/21 5:55 PM
 * description :
 */
@JGTest1.TestAnn
public class JGTest1 {
    @TestAnn1
    public String tt = "te";
    @TestAnn
    private int testInt = 233;

    public JGTest1() {}

    public JGTest1(String name) {
        tt = name;
    }

    @NonNull
    @Override
    public String toString() {
        return this.getClass().getSimpleName() +
                " tt is : " + tt + " testInt is : " + testInt;
    }

    @TestAnn
    public void testM() {
        System.out.println("im in testM " + toString());
    }

    @TestAnn2
    private int testM1(String input) {
        System.out.println("input is : " + input);
        return 233;
    }

    private JGTest1Inner getInner(int age, String name) {
        return new JGTest1Inner(age, name);
    }

    class JGTest1Inner {
        private String name;
        private int age;
        JGTest1Inner(int age, String name) {
            this.age = age;
            this.name = name;
        }

        public String testInnerM() {
            return "Im in testInnerM name : " + name + " age: " + age;
        }
    }

    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE,ElementType.FIELD,ElementType.METHOD})
    public @interface TestAnn {

    }

    // 反射的时候拿不到，只有runtime生命周期的才能拿到
    @Retention(RetentionPolicy.SOURCE)
    @Target({ElementType.TYPE,ElementType.FIELD,ElementType.METHOD})
    public @interface TestAnn1 {

    }

    // 同上，只有runtime才能拿到，但是apt的时候是可以拿到的
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE,ElementType.FIELD,ElementType.METHOD})
    public @interface TestAnn2 {

    }

}

```

#### 获取类对象

```java
// 获取Class 方式1

Class cc = JGTest1.class

// 获取Class 方式2

JGTest1 jgt1 = new JGTest1();
Class cc1 = jgt1.getClass();

// 通过反射获取Class 方式3
    private Class<?> getJGTClass() throws Exception {
        return Class.forName("com.xpj.javagrowth.javaref.JGTest1");
    }

// 反射实例化 1
Object jgt1 = c1.newInstance();

// 反射实例化 2
Constructor<?> jgt2 = c1.getDeclaredConstructor(String.class);// 首先获取有对应构造参数的构造器

// 传入对应的实例化参数进行实例化
        Object jgt2s = jgt2.newInstance("呜啦啦啦");

```

#### 反射获取对象的属性

```java
    public void testFieldRef() throws Exception {
        sb.append("\n").append("testFieldRef :: -->> ");
        Class<?> c1 = getJGTClass();
        Object jgt1 = c1.newInstance();
        Field[] fds1 = c1.getDeclaredFields();
        for (Field field : fds1) {
            sb.append("\n");
            sb.append("filed: ").append(field);
        }
        sb.append("\n").append("before jgt1: ").append(jgt1.toString());
        Field stt = c1.getDeclaredField("tt");
        stt.set(jgt1, "改变之后");
        Field intt = c1.getDeclaredField("testInt");
        intt.setAccessible(true);
        intt.set(jgt1, 56666);
        sb.append("\n").append("after jgt1 is  :").append(jgt1.toString());
        sb.append("\n\n\n");
    }
```

#### 反射获取对象的方法

```java
    public void testMethodRef() throws Exception {
        sb.append("\n").append("testMethodRef :: -->> ");
        Class<?> c1 = getJGTClass();
        Object jgt1 = c1.newInstance();
        Method[] ms = c1.getDeclaredMethods();
        for (Method m : ms) {
            sb.append("\n").append("method: ").append(m);
        }

        // 拿到内部类
        Class<?> jinner = Class.forName("com.xpj.javagrowth.javaref.JGTest1$JGTest1Inner");
        Method gtinner = c1.getDeclaredMethod("getInner", int.class, String.class);
        // private 设置访问
        gtinner.setAccessible(true);
        Object jjinner = gtinner.invoke(jgt1, 52, "dead");
        // 内部类身上的方法
        Method jinnerm = jinner.getDeclaredMethod("testInnerM");
        sb.append("\n\n").append(jinnerm.invoke(jjinner)).append("\n");
        sb.append("\n").append("jjinner : ").append(jjinner);

        Method tm = c1.getDeclaredMethod("testM");
        tm.invoke(jgt1);
        Method tm1 = c1.getDeclaredMethod("testM1", String.class);
        tm1.setAccessible(true);
        sb.append("\n").append("testM1 result is : ").append(tm1.invoke(jgt1, "rrr"));
        Constructor<?> jgt2 = c1.getDeclaredConstructor(String.class);
        Object jgt2s = jgt2.newInstance("呜啦啦啦");
        sb.append("\n").append("jgt2s is >>>: ").append(jgt2s.toString());
        sb.append("\n");

    }
```

#### 反射获取对象的注解

```java
    public void testAnnRef() throws Exception {
        sb.append("\n").append("testAnnRef :: -->> ");
        Class<?> c1 = getJGTClass();
        Annotation[] anns = c1.getAnnotations();
        sb.append("\n").append("class ann:\n");
        addAnn(anns);
        Field[] fields = c1.getFields();
        for (Field field : fields) {
            sb.append("\n").append("filed ann: ").append(field);
            addAnn(field.getAnnotations());
        }
        // 默认拿不到private的方法，这也是反射效率低原因之一，他会校验很多东西。
        Method[] methods = c1.getMethods();
        for (Method method : methods) {
            sb.append("\n").append("method ann: ").append(method);
            addAnn(method.getAnnotations());
        }
        // private 的方法默认不可访问，除非指名道姓
        Method m1 = c1.getDeclaredMethod("testM1", String.class);
        sb.append("\n").append("method ann: ").append(m1);
        addAnn(m1.getAnnotations());
        sb.append("\n");
    }

    private void addAnn(Annotation[] anns) {
        sb.append("\nBEGIN ..... \n");
        for (Annotation ann : anns) {
            sb.append("\n").append("ann is : ").append(ann);
        }
        sb.append("\nEND ..... \n");
    }
```

#### 反射效率低的原因

Java 反射效率低主要原因是：

1. Method#invoke 方法会对参数做封装和解封操作
2. 需要检查方法可见性
3. 需要校验参数
4. 反射方法难以内联
5. JIT 无法优化

### 动态代理

#### 代理的含义

**静态代理**，就是对一个接口做出一个代理实现，在初始化的时候或者别的时候去把要代理的实现类注入进去，然后在调用代理实现的时候方法前后做一个代理，最终实现无侵入式的适量修改。但是，这个模式也有一些局限，就是被代理的对象多了之后或者代理之后的行为不同之后类就会膨胀，且每一次都要新增对应的代理实现或者新的。

**动态代理**：是Java提供的一个技术，就是在运行时通过固定套路模板去进行生成新的实现类去进行代理。

![类图](https://raw.githubusercontent.com/Pjex/images/master/动态代理.png)


核心方法：

```java
    @CallerSensitive
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        // Android-removed: SecurityManager calls
        /*
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }
        */

        /*
         * Look up or generate the designated proxy class.
         找到对应的Class对象
         */
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            // Android-removed: SecurityManager / permission checks.
            /*
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }
            */

            /*
            找到对应的构造函数：
            private static final Class<?>[] constructorParams =
        { InvocationHandler.class }; // 参数是这个，就是这个类
            
            这里的意思是找到以这个为参数的构造器
            */
            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                // BEGIN Android-removed: Excluded AccessController.doPrivileged call.
                /*
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
                */

                cons.setAccessible(true);
                // END Android-removed: Excluded AccessController.doPrivileged call.
            }
            // 以这些为参数创建实现类，这里是 各种接口啥的，最终生成的就是实现了这些类的实例，这个实例首先会经过 自定义实现的中介 InvocationHandler invoke实现
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```


具体使用：

```java
    public void testProxy() {
        sb = new StringBuilder();
        sb.append("\n").append("testProxy\n");
        JGPJavaHandler jgProxyHandler = new JGPJavaHandler();
        // 这个是 kotlin 实现的版本，这里因为对数组的不友好导致不能正常使用
//        JGProxyHandler jgProxyHandler = new JGProxyHandler();
        jgProxyHandler.setSeller(new JGPXiaohong());
        JGPSeller hong = (JGPSeller) Proxy.newProxyInstance(JGPSeller.class.getClassLoader(),
                new Class[]{JGPSeller.class}, jgProxyHandler);
        sb.append("\nhong, sell: ").append(hong.sell())
                .append(" sellInfo: ").append(hong.sellInfo("233"));

        jgProxyHandler.setSeller(new JGPXiaoliang());
        JGPSeller liang = (JGPSeller) Proxy.newProxyInstance(JGPSeller.class.getClassLoader(),
                new Class[]{JGPSeller.class}, jgProxyHandler);
        sb.append("\nliang:").append(liang.sell())
                .append(" sellInfo: ").append(hong.sellInfo("566"));

        // 强无敌，这里能同时代理两个，前途无量了属于是
        jgProxyHandler.setSeller(new JGPXiaoji());
        JGPSeller ji = (JGPSeller) Proxy.newProxyInstance(JGPSeller.class.getClassLoader(),
                new Class[]{JGPSeller.class, JGPConsumer.class}, jgProxyHandler);
        sb.append("\nji:").append(ji.sell())
                .append(" sellInfo: ").append(hong.sellInfo("8999"))
                .append(((JGPConsumer)ji).buy());
        sb.append("\n");
    }
```

#### 实现方式
这里是实现方式，实现了对应InnovationHandler的类，具体实现方法。

```java
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println(seller + "动态生成之前 -> " + method + " args : " + args);

        Object[] afterArgs;
        if (args != null) {
            afterArgs = new Object[args.length];
            String name = "\n";
            for (int i = 0; i < args.length; i++) {
                if (args[i] instanceof String) {
                    // 可以对某个类型进行特殊处理。
                    if (seller instanceof JGPXiaohong) {
                        name = " ->-> my name is xiaohong\n";
                    }
                    afterArgs[i] = args[i] + " translate " + name;
                } else {
                    afterArgs[i] = args[i];
                }
            }
        } else {
            afterArgs = null;
        }

        Object result = method.invoke(seller, afterArgs);
        System.out.println(seller + "动态生成之后 ->-> " + method);
        return result;
    }

```

#### 别的大佬把Proxy写到文件中读出来

通过这个文件就更加能够理解动态代理的实现了。

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;
import proxy.Person;

public final class $Proxy0 extends Proxy implements Person
{
  private static Method m1;
  private static Method m2;
  private static Method m3;
  private static Method m0;
  
  /**
  *注意这里是生成代理类的构造方法，方法参数为InvocationHandler类型，看到这，是不是就有点明白
  *为何代理对象调用方法都是执行InvocationHandler中的invoke方法，而InvocationHandler又持有一个
  *被代理对象的实例，不禁会想难道是....？ 没错，就是你想的那样。
  *
  *super(paramInvocationHandler)，是调用父类Proxy的构造方法。
  *父类持有：protected InvocationHandler h;
  *Proxy构造方法：
  *    protected Proxy(InvocationHandler h) {
  *         Objects.requireNonNull(h);
  *         this.h = h;
  *     }
  *
  */
  // 这里看到了，是根据这个构造出来的一个类。
  public $Proxy0(InvocationHandler paramInvocationHandler)
    throws 
  {
    super(paramInvocationHandler);
  }
  
  //这个静态块本来是在最后的，我把它拿到前面来，方便描述
   static
  {
    try
    {
      // 这里的方法从那里来，都是在 Proxy 里面通过 getMethodsRecursive 去统计的 methods 最后通过 这个方法 getDeclaredMethods 去统计需要代理的接口类
      //  return generateProxy(proxyName, interfaces, loader, methodsArray, exceptionsArray); 在 Android 里面是使用的 native 去完成这个 Proxy 代理类的生成的。
      //看看这儿静态块儿里面有什么，是不是找到了giveMoney方法。请记住giveMoney通过反射得到的名字m3，其他的先不管
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      m3 = Class.forName("proxy.Person").getMethod("giveMoney", new Class[0]);
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException)
    {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException)
    {
      throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
    }
  }
 
  /**
  * 
  *这里调用代理对象的giveMoney方法，直接就调用了InvocationHandler中的invoke方法，并把m3传了进去。
  *this.h.invoke(this, m3, null);这里简单，明了。
  *来，再想想，代理对象持有一个InvocationHandler对象，InvocationHandler对象持有一个被代理的对象，
  *再联系到InvacationHandler中的invoke方法。嗯，就是这样。
  */
  public final void giveMoney()
    throws 
  {
    try
    {
      this.h.invoke(this, m3, null);
      return;
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  //注意，这里为了节省篇幅，省去了toString，hashCode、equals方法的内容。原理和giveMoney方法一毛一样。


/*
我们的动态代理为啥都得实现 InvacationHandler 就是因为这个 Handler 会持有我们被代理的对象，又因为这个 Proxy$0 的实现，在调用代理对象的对应方法时，因为在静态代码块里面已经拿到我们的对应的类了，因此在调用的时候又把这个方法传递进去，那么我们在 invoke 里面去定义对应的代理前和代理后方法，那么就完成了动态代理，动态就动态在这里，这里通过生成辅助对象，然后根据 InvacationHandler 的 invoke 方法，去传入被代理的方法，完成任意接口的代理，为什么不能代理类，就是因为我们生成的类本省继承了 Proxy 但继承的原因导致我们只能代理接口。
*/
}

// 如果要看生成代码的话，可以看 Java 版本的 Proxy 生成，Android 是搞到底层了，Java 还是 Java 在做，具体的，调用 ProxyGenerator.generateProxyClass 去生成对应的类
 ProxyGenerator gen = new ProxyGenerator(name, interfaces, accessFlags);
        final byte[] classFile = gen.generateClassFile();
```

#### 动态代理总结

总结一下，这里的动态代理形式最终的核心还是用到了反射，理解了反射其实动态代理也就好理解了，反射有的优点他有，受限于反射，那么反射的缺点他也有，反射与动态代理一般是用作框架的时候比较好用，因为他们的灵活性。

但是我们也要看到，越来越多的框架实现使用了注解这个东西，注解是更加优于反射与代理的，就像kotlin就可以使用inline refield去不反射保存泛型，这样的功能配合注解能够发挥更大的功能。

**为什么动态代理只能代理接口**，根据上面写出来的Proxy0文件可以看到他是继承了Proxy类，基于Java的单继承机制，所以动态代理只能代理接口的对应类，而不能代理普通类或者抽象类。


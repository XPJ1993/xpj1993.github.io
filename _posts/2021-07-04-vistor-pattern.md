---
layout: post
title: 访问者模式
image: /img/lifedoc/jijiji.jpg
tags: [Design Patterns]
---

学习访问者模式，因为这个是ASM的使用的核心思想，ASM通过提供各种visitor去完成对方法，成员变量，类，注解等的修改和扩展，这种访问者方式和正式的有些区别，这个就是类，方法，成员变量，注解等是被动的提供了访问，我们插入对应的字节码。

wiki上面介绍访问者模式，是一种正宗的访问者，但是我们日常code的时候可能需要像ASM那样对这个做一个灵活变通，有时候，变通才能更好的写出自己的合适代码，实现自己的想法。

示例Java代码如下：

```java
interface Visitor {
    // 访问者接口，实现这个接口就可以访问对应的角色，这里就是轮子、引擎、车体、整车。
    void visit(Wheel wheel);
    void visit(Engine engine);
    void visit(Body body);
    void visit(Car car);
}

// 下面对应的四个角色都提供accept方法，在方法里实现让visitor去访问自己，然后对应的方法就可以被访问者访问了，这里的也满足了通过方法访问对应的变量的方式
class Wheel {
    private String name;
    Wheel(String name) {
        this.name = name;
    }
    String getName() {
        return this.name;
    }
    void accept(Visitor visitor) {
        visitor.visit(this);
    }
}
 
class Engine {
    private int sudu = 0;

    int getSudu() {
        return sudu;
    }

    void addSudu() {
        sudu ++;
    }

    void removeSudu() {
        if (sudu > 0) {
            sudu --;
        }
    }

    void accept(Visitor visitor) {
        visitor.visit(this);
    }
}

class Body {
    void accept(Visitor visitor) {
        visitor.visit(this);
    }
}

class Car {
    private Engine  engine = new Engine();
    private Body    body   = new Body();
    private Wheel[] wheels 
        = { new Wheel("front left"), new Wheel("front right"),
            new Wheel("back left") , new Wheel("back right")  };

    // 这个类里面的accept方法去整合别的模块被访问的方式。
    void accept(Visitor visitor) {
        visitor.visit(this);
        engine.accept(visitor);
        body.accept(visitor);
        for (int i = 0; i < wheels.length; ++ i)
            wheels[i].accept(visitor);
    }
}

class PrintVisitor implements Visitor {
    // 具体的访问方法，不同的访问者可能有自己的特色访问方式，用自己的
    public void visit(Wheel wheel) {
        System.out.println("Visiting " + wheel.getName()
                            + " wheel");
    }
    public void visit(Engine engine) {
        // 这里就是访问到之后可以修改他的状态，例如asm里面就是增加字节码或者获取对应的属性啥的
        engine.addSudu();
        System.out.println("Visiting engine sudu is : " + engine.getSudu());
        engine.addSudu();
        System.out.println("Visiting engine sudu is : " + engine.getSudu());
        engine.addSudu();
        System.out.println("Visiting engine sudu is : " + engine.getSudu());
        engine.removeSudu();
        System.out.println("Visiting engine sudu is : " + engine.getSudu());
    }
    public void visit(Body body) {
        System.out.println("Visiting body");
    }
    public void visit(Car car) {
        System.out.println("Visiting car");
    }
}

// 访问者模式，就是被访问者接受访问者的访问，访问者可以访问对应被访问者的数据
public class StudyVistor {
    public static void main(String[] args) {
        Car car = new Car();
        Visitor visitor = new PrintVisitor();
        car.accept(visitor);
    }
}
```

---
layout: post
title:  "对于Java中override的小思考"
categories: development
tag: java
---

对于以下代码的执行结果还是有点儿在我意料之外的,经过stackOverFlow的解释,对java中Override的方法有了一点新的认识,之前的理解还是不够深入.

<!--more-->
```java
public class Father {
    public void print() {
        System.out.println("father.print");
        this.print(10);
    }
    public void print(int n) {
        System.out.println("father.print n=" + n);
    }
}

public class Child extends Father {
    @Override
    public void print() {
        System.out.println("child.print");
        super.print();
    }
    @Override
    public void print(int n) {
        System.out.println("child.print n="+n);
    }
    public static void main(String[] args) {
        Child child = new Child();
        child.print();
    }
}
```

这个main函数的执行结果是什么呢?其实我本来认为应该是这样的:
```
child.print
father.print
father.print n=10
```
实际上却是:
```
child.print
father.print
child.print n=10
```

这其实是对于this这个隐藏传递的理解.
当新的child调用print时,**隐藏传递了this是一个child 实例**,当显示调用super.print()时,调用的是child,super.print(),所以会进入到Father类中的方法,但是super是不会传递的,所以当第二个调用有参数版本print时,会继续调用用child.print().

当然啦,如果Child类没有重写print(int n)这个方法,还是会打印出father.print n=10的~
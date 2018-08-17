---
layout: post
title: 从NoSuchMethodError重新认识装箱、拆箱
tags: [JVM, ]

---

在开发中遇到了这样一个问题：在A工程中引入了B工程中的一个方法，B工程中的这个方法入参是int，而A工程中在具体引用时传参是Integer类型的。某一时刻，在调用这个方法时，传入的Integer为NULL，产生了空指针异常。由于后续处理中，传参为NULL是被允许的，所以这里为了规避空指针异常，我将B工程中的这个方法的入参类型声明由int改为了Integer，然后将B工程重新打包上传（但却没有重新打包上传A工程）。

于是，产生了**NoSuchMethodError**。

其调用过程类似于下面的代码：

```java
/** A工程中的某类 **/
public class A {
    public void methodA() {
        Integer i = null;
        B b = new B();
        b.methodB(i);
    }

    public static void main(String[] args) {
        A a = new A();
        a.methodA();
    }
}

/** B工程中的某类 **/
public class B {
	/** 改动前 **/
    public void methodB(int i) {
        System.out.println(i);
    }
    
	/** 改动后，发生NoSuchMethodError **/
    public void methodB(Integer i) {
        System.out.println(i);
    }
}
```

按道理，Java会自动装箱、拆箱，原本传int类型的方法，现在改成传Integer应该不会有问题，更何况方法的调用处本身传的就是Integer类型，那为何会发生NoSuchMethodError呢？

归根结底，是因为没有弄清楚：**Java的装箱、拆箱是在源码编译成字节码的时候就发生了的**。

将上面的代码做点小小的调整：

```java
public class A {
    public void methodA() {
        Integer i = 0;
        B b = new B();
        b.methodB(i);
        b.methodC(i);
        b.methodC(0);
    }

    public static void main(String[] args) {
        A a = new A();
        a.methodA();
    }
}

public class B {
    public void methodB(int i) {
        System.out.println(i);
    }
    
    public void methodC(Integer i) {
        System.out.println(i);
    }
}
```

编译，然后**javap -c A.class**查看A类编译后所生成的字节码（这里仅截取类A中methodA的定义处的字节码）：

```java
  public void methodA();
    Code:
       0: iconst_0
       1: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
       4: astore_1
       5: new           #3                  // class autoboxingandunboxing/B
       8: dup
       9: invokespecial #4                  // Method autoboxingandunboxing/B."<init>":()V
      12: astore_2
      13: aload_2
      14: aload_1
      15: invokevirtual #5                  // Method java/lang/Integer.intValue:()I
      18: invokevirtual #6                  // Method autoboxingandunboxing/B.methodB:(I)V
      21: aload_2
      22: aload_1
      23: invokevirtual #7                  // Method autoboxingandunboxing/B.methodC:(Ljava/lang/Integer;)V
      26: aload_2
      27: iconst_0
      28: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      31: invokevirtual #7                  // Method autoboxingandunboxing/B.methodC:(Ljava/lang/Integer;)V
      34: return
```

我们可以清晰地看到，在字节码标识的**15行**处，对应源码中**b.methodB(i);**这一行代码的地方，调用了java/lang/Integer.intValue方法，这是一个拆箱操作！

由于传入的参数i是Integer类型，而methodB中的传参声明的是int类型，于是**在编译A类的时候，就对i进行了拆箱**，继而有了**18行**的B.methodB的传参为int类型。

作为对比，在**28行**处，对应源码中**b.methodC(0);**一行，则是调用了java/lang/Integer.valueOf，执行了装箱操作，将原始类型int（值0）装箱成Integer。

回到最开始的问题来。

B工程中的B方法原始传参类型是int，那么A工程在被编译的时候，就已经发生了拆箱，而且**函数的函数签名也已经确定**。这样，A工程的编译文件中调用的仍然是函数签名为**B.methodB:(I)V**的方法，B工程却因为重新编译，函数签名变为了**B.methodB:(Ljava/lang/Integer;)V**，从而导致了NoSuchMethodError错误的产生。
---
title: JVM-5-JIT编译及静态编译
date: 2017-12-12 01:17:15
tags: JVM
---

对于执行的字节码会从两处进行优化。

**第一，就是使用javac编译时。**

**第二，就是使用JIT（just-in-time）即时编译，在运行时。**

# 编译时计算：

如果在程序中出现了计算表达式，如果表达式的值能够在编译时确定，那么表达式的计算会提前到编译阶段，而不是在运行时计算。
``` java

for(int i=0;i<60*60*24*1000；i++){

}
```

我们经常会用到这样的写法60*60*24*1000，那么是不是每次循环都都要进行一次？其实不是，因为在编译的时候，对于给定的表达式会自动计算并给出结果。

字符串相加：
``` java

String info1="ab";

String info2=“a”+“b”；

String const="a";

String info3=const+"b";
```

三个值的字面量都是"ab"，那么他们相等吗，info1==info2 是相等的，理由：因为在info的“+”并为在运行时进行，在编译的时候就进行了，所以不会产生新的对象。而info3有一个未知的变量，所以“+”在运行时进行，产生了新的对象。

在变量字符串连接：

例子：
``` java

public static void addString2(String ...str1){

    String str3="";

        for(String str:str1){

            str3+=str1;        

    }

}

```
当变量字符串相加时，系统会先将字符串转化为StringBuilder，然后做append操作。但是在for循环中，是每一次循环都建立一个StringBuilder对象，这样很费系统资源，正确的做法是在循环外建立一个StringBuilder，然后做append(）操作。S

# JIT编译 

java虚拟机有三种执行方式，分别是解释执行(-Xint)、混合模式(-mixed mode )、编译执行(-Xcomp),默认是混合模式。

解释执行表示全部代码均解释执行，不做任何JIT编译，使用java -Xint -version来开启。

混合模式是根据是否会是热点代码，如果是，都会编译执行。

编译模式，所有代码均编译执行。

一般来说，编译模式的执行效率会远远高于解释模式。大家可以使用同一段代码根据-Xint、-Xcomp来比较。

# JIT编译阀值：

-client模式下，阀值是1500次

-server模式下，阀值是10000次。

使用-XX:CompileThreshold可以设置这个阀值。（-XX:CompileThreshold=500）

使用-XX:PrintCompilation可以打印出即时编译的日志。（-XX:PrintCompilation）

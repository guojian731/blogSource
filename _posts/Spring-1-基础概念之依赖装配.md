---
title: Spring-1-基础概念之依赖装配
date: 2017-12-10 18:04:36
tags: spring
---
# Spring的核心
**就是简化java开发，为了降低java开发的复杂性，当一个类中调用另一个类的时候，会存在耦合，不利于理解和测试。正常情况下，每个对象负责管理和与自己相互协作的对象(即它所依赖的对象)，这将会导致高度耦合和难以测试的代码。Spring采取了以下4中策略。**
## 基于pojo的轻量级和最小侵入性策略
	Spring通过应用上下文(Application Context)装载bean的定义并把他们组装起来。Spring应用上下文全权负责对象的创建和组装。
## 通过依赖注入和面向接口实现松耦合
	依赖注入（DI）现在属于一种策略模式，它的作用是让相互协作的软件组件保持松散耦合。
## 通过切面和惯例进行声明式编程
	面向切面编程（AOP）允许你把遍历应用各处的功能分离出来形成可重用性组件。（例如事务和日志）
## 通过切面和模版减少样式代码
	spring跟很多软件进行集成，包括mybatis，hibernate，redis等等。




## 例子：

我们现在有一个支付接口Payservice ，他只有一个方法就是支付。

	public interface Payservice {
	    void pay(Integer num);
	}

现在我们有一个实现类PhonePayServiceImpl通过手机进行支付操作

	public class PhonePayServiceImpl implements Payservice{
	    public void pay(Integer num) {
	        System.out.println("使用手机支付了"+num);
	    }
	}

那么我们现在有一个类是支付入口PayEntrance，正常情况下，我们会这么写

	public class PayEntrance {    
	    private Payservice payservice=new PhonePayServiceImpl();
	    public void doPay(){
	        payservice.pay(100);
	    }
	}

或者使用构造器

	public PayEntrance(){
	    this.payservice=new PhonePayServiceImpl();
	}
我们的支付入口PayEntrance与支付接口payservice、手机支付实现PhonePayServiceImpl紧紧的耦合在了一起，那么如果我们支付入口还需要验证密码、发送验证码等功能呢，那么依赖的对象会非常多，而且这样的代码无法进行测试。


# 依赖注入
**依赖注入就是控制反转的核心，每一个对象不负责创建和管理与它合作的对象，在spring中，交给spring容器负责。**

## 构造器注入：

	public PayEntrance(Payservice payservice){
	    this.payservice=payservice;
	}
现在我们发现，我们的支付入口并没有创建手机支付实现类，对于支付入口来说，只要我实现了payservice接口，那么我甚至不知道我具体使用了哪种支付方式（假设我们还有一个支付实现类为电脑端），支付方式对于支付入口来说无关紧要了，这就是DI最大的优点，松耦合。

## setter方法：

	public void setPayservice(Payservice payservice) {
	    this.payservice = payservice;
	}
## 接口注入：

需要新建一个接口

	public interface InjectPayService {
	    public void  injectPayService(Payservice payservice);
	}
然后继承这个接口

	public class PayEntrance implements InjectPayService{
	
	    private Payservice payservice;
	
	    public void injectPayService(Payservice payservice) {
	        this.payservice=payservice;
	    }
	}

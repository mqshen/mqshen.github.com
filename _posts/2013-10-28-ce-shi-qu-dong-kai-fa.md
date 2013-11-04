---
layout: post
title: "测试驱动开发"
description: "测试驱动开发(TDD)进入软件开发行业已经有相当长的时间了。它的基本前提是在编写真 正的功能实现代码之前先写测试代码,然后根据需要重构实现代码。"
category: ""
tags: [测试]
---
{% include JB/setup %}
测试驱动开发(TDD)进入软件开发行业已经有相当长的时间了。它的基本前提是在编写真 正的功能实现代码之前先写测试代码,然后根据需要重构实现代码。
我们认为消除恐惧和不确定性是编写测试驱动代码的重要原因。

>   恐惧会让你小心试探;
>   恐惧会让你尽量减少沟通;
>   恐惧会让你羞于得到反馈;
>   恐惧会让你脾气暴躁。

TDD可以祛除恐惧,让优秀的Java开发者变得更加自信、善于沟通、乐于接受并更加快乐。换句话说,TDD能帮你摆脱下面这种心态:

>   在开始新工作时,“我不知道从哪里开始,所以只好将就着做一些修改”;
>   在修改已有代码时,“我不知道现有代码怎么运行,所以我私下认为不能碰它们”。

TDD带来的很多好处并不会马上显现:

>   更清晰的代码——只写需要的代码;
>   更好的设计——有些开发人员管TDD叫测试驱动的设计;
>   更出色的灵活性——TDD鼓励按接口编码;
>   更快速的反馈——不会直到系统上线才知道bug的存在。


四大类伪装对象它们能简化受试代码和第三方类库中代码 的隔离,或隔离数据库之类的子系统行为.随着依赖项变得越来越复杂,伪装 对象也要变得越来越聪明。最后会介绍模拟和Mockito类库,它是一个流行的模拟工具,可以让开发人员在不受外部系统影响的环境下进行测试。

### TDD概览
TDD可以应用在多个层级上。下表列出了通常会采用TDD的四个测试层级。

| *层级* | *描述* | *例子* |
|:-------|:-------|:-------|
| 单元测试 | 通过测试验证一个类中包含的代码 | 测试BigDecimal类中的方法 |
| 集成测试 | 通过测试验证类之间的交互 | 测试Currency类以及它如何跟BigDecimal交互 |
| 系统测试 | 通过测试验证运行的系统 | 从UI到Currency类测试会计系统 |
| 系统集成测试 | 通过测试验证运行的系统,包括第三方组件 | 测试会计系统,包括它与第三方报表系统间的交互 |    
    
在单元测试中使用TDD是最容易的,如果你对TDD不熟悉,这一层就是个很好的起点。这里主要讲述如何在单元测试层中使用TDD。
TDD的三个基本步骤:红、绿、重构。

>   红,写一些不能用的测试代码(失败测试);
>   绿,尽快让测试通过(通过测试);
>   重构,消除重复(经过细化的通过测试) 。

#### 一个测试用例
假定有人要你写一个坚若磐石的方法来计算剧院门票的销售收入。剧院会计最初给出的业务规则很简单:

>   门票的底价是30美元
>   总收入=售出票数X价格
>   剧院有100个座位

为了让你了解TicketRevenue应该达到什么效果,请先看一下这些伪代码。

    estimateRevenue(int numberOfTicketsSold)
    if (numberOfTicketsSold is less than 0 OR greater than 100)
    then
        Deal with error and exit
    else
        revenue = 30 * numberOfTicketsSold;
        return revenue;
    endif

注意,千万别太深入。测试最终会驱动设计,也会部分影响实现。
接下来我们先用JUnit写一个失败单元测试。

###### 1. 编写失败测试(红)
这一步的要点是以一个会失败的测试开始。实际上,这个测试甚至无法编译,因为你还没有TicketRevenue类!
在跟会计开过一个简短的白板会议后,你意识到测试代码需要覆盖五种情况:售票数量为负数、0、1、2~100,还有大于100。

我们决定先写一个测试覆盖销售一张门票收入的情况。测试代码看起来应该如下面的代码:


{% highlight java linenos %}
    import java.math.BigDecimal;
    import static junit.framework.Assert.*;
    import org.junit.Before;
    import org.junit.Test;

    public class TicketRevenueTest {
        private TicketRevenue venueRevenue;

        private BigDecimal expectedRevenue;

        @Before
        public void setUp() {
            venueRevenue = new TicketRevenue();
        }

        @Test
        public void oneTicketSoldIsThirtyInRevenue() {
            expectedRevenue = new BigDecimal("30");
    ￼￼  ￼   assertEquals(expectedRevenue, venueRevenue.estimateTotalRevenue(1));
        }
    }
{% endhighlight %}

测试期望销售一张门票得到的收入等于30。但这个测试不能编译,因为有estimateTotalRevenue(int numberOfTicketsSold)方 法的TicketRevenue类还不存在呢。为了运行测试,可以先随便写一个让测试可以编译的实现。

{% highlight java linenos %}
    public class TicketRevenue {
        public BigDecimal estimateTotalRevenue(int i) {
            return BigDecimal.ZERO;
        }
    }
{% endhighlight %}

现在测试代码能编译了.
失败测试有了,接下来该做通过测试了(变绿)。

###### 2. 编写通过测试(绿)
这一步的要点是让测试通过,但没必要把实现做到完美。给TicketRevenue类一个更好的 estimateTotalRevenue实现(不会只返回0),可以让测试通过(变绿)。  
记住,这一阶段只要让测试通过就行,没必要追求完美。代码如下所示:
import java.math.BigDecimal;

{% highlight java linenos %}
public class TicketRevenue {
    public BigDecimal estimateTotalRevenue(int numberOfTicketsSold) {
        BigDecimal totalRevenue = BigDecimal.ZERO;
        if (numberOfTicketsSold == 1) {
            totalRevenue = new BigDecimal("30");
        }
        return totalRevenue;
    }
}
{% endhighlight %}
现在再运行测试,通过了!  
接下来就是完善前面的代码。

###### 3. 重构测试
这一步的要点是看看为了通过测试写的快速实现,确保你遵循了通行的惯例。上面的代码明显可以更清晰、更整洁。你肯定要重构,以减轻自己和他人的技术债务。
  
有了通过测试,可以放心大胆地重构。应该实现的业务逻辑不可能会被忽视。
{% highlight java linenos %}

import java.math.BigDecimal;

public class TicketRevenue {

    private final static int TICKET_PRICE = 30;

    public BigDecimal estimateTotalRevenue(int numberOfTicketsSold) {
        BigDecimal totalRevenue = BigDecimal.ZERO;
        if (numberOfTicketsSold == 1) {
            totalRevenue = new BigDecimal(TICKET_PRICE * numberOfTicketsSold);
        }
        return totalRevenue;
    }
}
{% endhighlight %}
经过这次重构,代码得到了改善,但很明显它还没有涵盖所有情况(售票数量为负值、0、 2~100和大于100)。你不能只是拼命地猜其他情况下的实现应该是什么样,而应该做更多测试驱 动的设计和实现。

#### 多个测试用例
按照TDD风格,应该继续为门票销售数量为负值、0、2~100和大于100的情况依次添加测试 用例。但还有一种办法,一次写一组测试用例也行,特别是在它们跟最初的测试有关的时候。
注意,这次仍然要遵循红—绿—重构的循环周期。在把这些用例都加上之后,你应该会得到一个带有失败测试(红)的测试类.

{% highlight java linenos %}
    import java.math.BigDecimal;
    import static junit.framework.Assert.*;
    import org.junit.Test;

    public class TicketRevenueTest {

        private TicketRevenue venueRevenue;
        private BigDecimal expectedRevenue;

        @Before
        public void setUp() {
          venueRevenue = new TicketRevenue();
        }

        @Test(expected=IllegalArgumentException.class)
        public void failIfLessThanZeroTicketsAreSold() {
          venueRevenue.estimateTotalRevenue(-1);
        }

        @Test
        public void zeroSalesEqualsZeroRevenue() {
            assertEquals(BigDecimal.ZERO, venueRevenue.estimateTotalRevenue(0));
        }

        @Test
        public void oneTicketSoldIsThirtyInRevenue() {
          expectedRevenue = new BigDecimal("30");
        ￼￼￼  assertEquals(expectedRevenue, venueRevenue.estimateTotalRevenue(1));
        }

        @Test
        public void tenTicketsSoldIsThreeHundredInRevenue() {
          expectedRevenue = new BigDecimal("300");
        ￼￼  assertEquals(expectedRevenue, venueRevenue.estimateTotalRevenue(10));
        }

        @Test(expected=IllegalArgumentException.class)
        public void failIfMoreThanOneHundredTicketsAreSold() {
            venueRevenue.estimateTotalRevenue(101);
        }
    }
{% endhighlight %}

为通过所有测试(绿)写的基本实现版
{% highlight java linenos %}

    import java.math.BigDecimal;

    public class TicketRevenue {
        public BigDecimal estimateTotalRevenue(int numberOfTicketsSold)
            throws IllegalArgumentException {
            BigDecimal totalRevenue = null;

            if (numberOfTicketsSold < 0) {
                throw new IllegalArgumentException("Must be > -1");
            }

            if (numberOfTicketsSold == 0) {
                totalRevenue = BigDecimal.ZERO;
            }

            if (numberOfTicketsSold == 1) {
                totalRevenue = new BigDecimal("30");
            }

            if (numberOfTicketsSold == 101) {
                throw new IllegalArgumentException("Must be < 101");
            }
            else {
                totalRevenue = new BigDecimal(30 * numberOfTicketsSold);
            }
            return totalRevenue;
        }
    }

{% endhighlight %}
有了刚刚完成的实现,现在你的测试就变成通过测试了。
按照TDD循环周期,现在该重构这个实现了。

{% highlight java linenos %}
    import java.math.BigDecimal;
    
    public class TicketRevenue {
        private final static int TICKET_PRICE = 30;
    
        public BigDecimal estimateTotalRevenue(int numberOfTicketsSold)
            throws IllegalArgumentException {
            if (numberOfTicketsSold < 0 || numberOfTicketsSold > 100) {
                throw new IllegalArgumentException("# Tix sold must == 1..100");
            }
            return new BigDecimal(TICKET_PRICE * numberOfTicketsSold);
        }
    
    }
{% endhighlight %}
新的TicketRevenue类更加紧凑,并且还通过了所有测试!现在你已经完成了整个红—绿 —重构循环,可以信心满满地开始实现下一个业务逻辑了。


面向对象代码的SOLID原则

| *原则* | *描述* |
|:-------|:-------|
| 单一职责原则(SRP) | 每个对象都应该做一件事,并且只做一件事 |
| 开放/封闭原则(OCP) | 对象应该是可扩展、但不可修改的 |
| 里氏替换原则(LSP) | 对象应该可以被它的子类型实例替换 |
| 接口隔离原则(ISP) | 特定的小接口更好 |
| 依赖倒置原则(DIP) | 不要依赖具体实现 |

#### JUnit
JUnit是公认的Java项目测试框架。当然,除了JUnit还有其他测试框架,比如拥有不少追随者 的TestNG,但目前JUnit还是Java测试界的主流。  
它主要有三个特性：

>   用于测试预期结果和异常的断言,比如assertEquals();
>   设置和拆卸通用测试数据的能力,比如@Before和@After;
>   运行测试套件的测试运行器。

JUnit用简单的注解模型提供了很多重要的功能。
一个基本的JUnit测试包含下面这些元素:

>   用@Before标记设置方法,在每个测试运行前准备测试数据;
>   用@After标记拆卸方法,在每个测试运行完成后拆卸测试数据;
>   测试方法本身(用@Test注解标记)。

### 测试替身
如果你继续用TDD风格编码,很快就会遇到需要引用(经常是第三方的)依赖项或子系统的 情况。在这种情况下,你肯定想把测试代码跟依赖项隔离开,以保证测试代码仅仅针对于实际构 建的代码。你肯定还想让测试代码尽可能快速运行。而调用第三方依赖项或子系统(比如数据库) 可能会花很长时间,也就是说会丧失TDD快速响应的优势(在单元测试层面尤其如此)。测试替 身(test double)就是为解决这个问题而生的。
四种测试替身

>   虚设--只传递不使用的对象。一般用于填充方法的参数列表
>   伪装--总是返回相同预设响应的对象,其中可能也有些虚设状态
>   存根--可以取代真实版本的可用版本(当然在品质和配置上达不到生产环境要求的标准)
>   模拟--可以表示一系列期望值的对象,并且可以提供预设响应

#### 虚设对象
在这四种测试替身里,虚设对象用起来最容易。记住,它是用来填充参数列表,或者填补那 些总也不会用的必填域。大多数情况下,你甚至可以传入一个空对象或null。

#### 存根对象
在使用能够做出相同响应的对象代替真实实现的情况下,就会用到存根对象。

#### 伪装替身
伪装对象可以看做是存根的升级,它所做的工作几乎和生产代码一样,但为了满足测试需求 会走些捷径。如果你想让代码的运行时环境非常接近生产环境(连接真实的第三方子系统或依赖 项),伪装替身特别有用。

#### 模拟对象
模拟对象跟前面提过的存根对象是亲戚,但存根对象一般都特别呆。比如在调用存根时它们 通常总是返回相同的结果。所以不能模拟任何与状态相关的行为。  
一个流行的模拟类库Mockito

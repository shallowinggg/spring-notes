前面一章介绍了Spring对于AOP的支持，使用@AspectJ注解或者XML切面定义。在本章中，我们讨论底层的Spring AOP APIs。对于普通的应用，我们建议通过AspectJ切点来使用Spring AOP。

### 6.1\. Pointcut API in Spring

本节介绍了Spring如何处理关键性的切点概念。

#### 6.1.1\. Concepts

Spring的切点模型确保了切点重用与增强类型无关。你可以使用相同的切点定位不同的增强。

`org.springframework.aop.Pointcut`接口是核心接口，用来为特定的类和方法定位增强。完整的接口定义如下：

```
public interface Pointcut {

    ClassFilter getClassFilter();

    MethodMatcher getMethodMatcher();

}
```

将`Pointcut`接口分为两部分允许重用类和方法匹配的部分以及细粒度的组成操作（例如与其他方法匹配器“联合”）。

`ClassFilter`接口用来对特定的目标类集合限制切点。如果`matches()`方法永远返回true，那么所有的目标类都能匹配。下面展示了`ClassFilter`接口的定义：

```
public interface ClassFilter {

    boolean matches(Class clazz);
}
```

`MethodMatcher`接口一般来说更加重要，完整的接口定义如下：

```
public interface MethodMatcher {

    boolean matches(Method m, Class targetClass);

    boolean isRuntime();

    boolean matches(Method m, Class targetClass, Object[] args);
}
```

`matches(Method, Class)`方法用来测试这个切点是否匹配目标类中给定的一个方法。当一个AOP代理被创建时执行这个检测，以避免对每一个方法调用都进行测试。如果两个参数的`matches`方法检测一个给定方法时返回`true`，并且`isRuntime()`方法也返回`true`，那么三参数的`matches`方法就会在每一个方法上调用。这可以让切点在增强逻辑执行前发现传入到目标方法中的参数。

大多数`MethodMatcher`实现类都是静态的，这意味着`isRuntime()`方法返回`false`。在这种情况下，三参数的`matches`方法永远不会被调用。

> 如果可能的话，尽量让切点是静态的，以允许AOP框架在AOP代理被创建的时候缓存切点检查的结果。

#### 6.1.2\. Operations on Pointcuts

Spring支持切点上的操作（特别是并集和交集）。

并集意味着任何一个切点可以匹配的方法。交集意味着所有切点都匹配的方法。并且一般来说是更有用的，你可以使用`org.springframework.aop.support.Pointcuts`类或者使用同一个包中的`ComposablePointcut`类的静态方法来组合切点。但是，使用AspectJ切点表达式更加简单。

#### 6.1.3\. AspectJ Expression Pointcuts

从Spring 2.0开始，Spring使用的最重要的的切点类型是`org.springframework.aop.aspectj.AspectJExpressionPointcut`。这个切点使用AspectJ提供的库来解析AspectJ切点表达式字符串。

被支持的AspectJ切点原语在[上一章](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop)中查看。

#### 6.1.4\. Convenience Pointcut Implementations

Spring提供了几个方便的切点实现类，你可以直接使用它们。其他的意在成为应用特定的切点的子类。
Others are intended to be subclassed in application-specific pointcuts.

##### Static Pointcuts

静态切点基于方法以及目标类，并且不会将方法参数考虑在内。对于大多数需求静态切点可以满足使用了，并且也是最好的选择。Spring只会检查静态切点一次，当一个方法第一次被调用时。在那之后，无需在每一次方法调用的时候都再次检查切点。

本节剩余的部分介绍了一些Spring中包含的静态切点实现类。

###### Regular Expression Pointcuts

指定静态切点的一种公认的方式就是正则表达式，除了Spring以外其他几个AOP框架也都支持这种方式。`org.springframework.aop.support.JdkRegexpMethodPointcut`是一个通用的正则表达式切点，它使用了JDK提供的正则表达式支持。

使用 `JdkRegexpMethodPointcut`类，你可以提供一系列的模式字符串。如果能匹配这些模式中的任意一个，那么切点检查返回`true`。（所以，结果实际上是这些切点的并集。）

下面的例子展示了如何使用`JdkRegexpMethodPointcut`：

```
<bean id="settersAndAbsquatulatePointcut"
        class="org.springframework.aop.support.JdkRegexpMethodPointcut">
    <property name="patterns">
        <list>
            <value>.*set.*</value>
            <value>.*absquatulate</value>
        </list>
    </property>
</bean>
```

Spring提供了一个名为`RegexpMethodPointcutAdvisor`的类，它可以让我们引用一个`Advice`（注意`Advice`是一个拦截器，before advice, throws advice以及其他）。在幕后，Spring使用了`JdkRegexpMethodPointcut`。使用`RegexpMethodPointcutAdvisor`可以简化注入，因为这个bean将切点和增强都封装了，如下所示：

```
<bean id="settersAndAbsquatulateAdvisor"
        class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
    <property name="advice">
        <ref bean="beanNameOfAopAllianceInterceptor"/>
    </property>
    <property name="patterns">
        <list>
            <value>.*set.*</value>
            <value>.*absquatulate</value>
        </list>
    </property>
</bean>
```

你可以通过任何`Advice`类型使用`RegexpMethodPointcutAdvisor`。

###### Attribute-driven Pointcuts

静态切点中一个很重要的类型就是metadata-driven切点，它使用了元数据属性的值（一般来说是源码级别的元数据）。

##### Dynamic pointcuts

动态切点检查比静态切点的开销更昂贵。它们将参数以及静态信息都纳入了考虑范围。这意味着它们必须对每一次方法调用都进行检查，并且结果无法缓存，因为参数是可变的。

主要的例子就是`control flow`切点。

###### Control Flow Pointcuts

Spring控制流切点在概念上与AspectJ的`cflow`切点相似，虽然没有它那么强大。（现在没有办法指定一个切点在另一个切点匹配的连接点下执行。）控制流切点匹配当前的调用栈。例如，如果连接点被`com.mycompany.web`包或者`SomeCaller`类中的方法调用，它可能fire。控制流切点可以使用`org.springframework.aop.support.ControlFlowPointcut`类来指定。

> 控制流切点运行时检查的开销比其他动态切点昂贵很多，在Java 1.4中，它的开销是其他动态切点的五倍。

#### 6.1.5\. Pointcut Superclasses

Spring提供一些有用的切点基类来帮助你实现你自己的切点。

因为静态切点是最有用的，你应当继承`StaticMethodMatcherPointcut`类。这只需要实现一个抽象方法（虽然你可以重写其他方法来自定义一些行为）。下面的例子展示了如何继承`StaticMethodMatcherPointcut`类：

```
class TestStaticPointcut extends StaticMethodMatcherPointcut {

    public boolean matches(Method m, Class targetClass) {
        // return true if custom criteria match
    }
}
```

也有为动态切点准备的基类，你可以使用任何增强类型来自定义切点。

#### 6.1.6\. Custom Pointcuts

因为Spring AOP中的切点是Java类而不是一种语言特性（在AspectJ中），所以你可以声明自定义切点，无论它是动态的还是静态的。Spring中的自定义切点可以是任意复杂的，但是，我们建议如果你可以的话，最好使用AspectJ切点表达式。

> Spring之后的版本可能会提供对JAC提供的“语义切点”的支持 —— 例如，“在目标对象中所有可以改变实例变量的方法”。

### 6.2\. Advice API in Spring

现在我们可以考察Spring AOP是如何处理增强的了。

#### 6.2.1\. Advice Lifecycles

每一个增强都是一个Spring bean。一个增强实例可以在所有被增强的对象共享，也可以被一个被增强的对象独占，这和per-class或per-instance增强相对应。

Per-class增强最常使用。对于通用的增强来说很合适，例如事务增强器。这不依赖于被代理对象的状态或者对其增加新状态，它们很少在其方法和参数上进行操作。

Per-instance增强适合引介，以支持混用(mixin)。在这种情况下，增强向被代理的对象增加了状态。

你可以在同一个AOP代理中混用共享和单实例增强。

#### 6.2.2\. Advice Types in Spring

Spring提供了几个增强类型，并且可以对其拓展以支持任何增强类型。本节介绍基础理论以及标准增强类型。

##### Interception Around Advice

Spring中功能最多的增强类型是环绕增强。

Spring使用了AOP联盟为环绕增强提供的接口，它使用方法拦截。想要实现环绕增强的类应当实现 `MethodInterceptor`接口，此接口定义如下所示：

```
public interface MethodInterceptor extends Interceptor {

    Object invoke(MethodInvocation invocation) throws Throwable;
}
```

`invoke()`方法的 `MethodInvocation`参数暴露了应当被调用的方法，目标连接点，AOP代理以及方法参数。`invoke()`方法应当返回调用结果：连接点的返回值。

下面的例子展示了一个简单的`MethodInterceptor`实现类：

```
public class DebugInterceptor implements MethodInterceptor {

    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("Before: invocation=[" + invocation + "]");
        Object rval = invocation.proceed();
        System.out.println("Invocation returned");
        return rval;
    }
}
```

注意此处调用了`MethodInvocation`的`proceed()`方法。这个方法调用向下推进了连接点的拦截器链，大多数的拦截器都调用了这个方法并且返回它的值。但是，`MethodInterceptor`和其他环绕增强一样，可以返回一个不同的值或者抛出一个异常来代替`proceed()`方法的真正返回结果。但是，如果没有什么好的理由你不会想要这样做。

> `MethodInterceptor`实现类提供了与其他符合AOP联盟(AOP Alliance)的AOP实现类的互用。其他的增强类型在本节剩余部分讨论，它们实现了常规的AOP概念但是是以Spring特定的方式。虽然使用大多数特定的增强类型有好处，但是如果你想要在其他AOP框架中也运行这些切面那么你要坚持使用`MethodInterceptor`环绕增强。注意切点现在还无法在框架之间互用，并且AOP联盟现在还没有定义切点接口。

##### Before Advice

一个更简单的增强类型是before advice。它不需要`MethodInvocation`对象，因为它只在进入被增强的方法之前调用。

before advice的主要优点就是不需要调用`proceed()`方法，因此不可能在拦截器链传递时失败。

下面展示了`MethodBeforeAdvice`接口：

```
public interface MethodBeforeAdvice extends BeforeAdvice {

    void before(Method m, Object[] args, Object target) throws Throwable;
}
```

(Spring的API设计允许进行字段的前置增强，)
(Spring’s API design would allow for field before advice, although the usual objects apply to field interception and it is unlikely for Spring to ever implement it.)

注意这个方法的返回类型是`void`。Before advice可以在连接点执行之前插入自定义行为，但是无法改变连接点的返回值。如果Before advice抛出了一个异常，那么会阻止拦截器链的进一步执行。这个异常会在拦截器链中反向传递。如果它是未受检异常或者存在于被调用方法的签名上，那么它会直接传递给客户端。否则，它会被AOP代理包装到一个未受检异常中。

下面的例子展示了Spring中的一个before advice，它对所有的方法调用进行计数。

```
public class CountingBeforeAdvice implements MethodBeforeAdvice {

    private int count;

    public void before(Method m, Object[] args, Object target) throws Throwable {
        ++count;
    }

    public int getCount() {
        return count;
    }
}
```

> before advice可以在任何切点中使用。

##### Throws Advice

如果连接点抛出了一个异常，那么连接点返回后调用throws advice。Spring提供了类型化的throws advice。注意这意味着`org.springframework.aop.ThrowsAdvice`不会包含任何方法，它是一个标记接口，用来标明给定的对象实现了一个或多个类型化throws advice方法，如下所示：

```
afterThrowing([Method, args, target], subclassOfThrowable)
```

只有最后一个参数是必要的。这个方法签名可以拥有一个或四个参数，这依赖于增强方法是否对方法以及参数感兴趣。下面展示了throws advice的例子：

如果抛出了一个`RemoteException`异常(包括子类异常)，那么调用下面的增强：

```
public class RemoteThrowsAdvice implements ThrowsAdvice {

    public void afterThrowing(RemoteException ex) throws Throwable {
        // Do something with remote exception
    }
}
```

与前一个增强不同，下面的例子声明了四个参数，因此它可以访问被调用的方法，方法参数以及目标对象。当抛出一个`ServletException`异常时，调用下面的增强：

```
public class ServletThrowsAdviceWithArguments implements ThrowsAdvice {

    public void afterThrowing(Method m, Object[] args, Object target, ServletException ex) {
        // Do something with all arguments
    }
}
```

最后一个例子展示了如何在一个类中使用两个方法来同时处理`RemoteException` 和 `ServletException`异常。任何数量的throws advice方法都可以在一个类中结合使用。下面展示了最后一个例子：

```
public static class CombinedThrowsAdvice implements ThrowsAdvice {

    public void afterThrowing(RemoteException ex) throws Throwable {
        // Do something with remote exception
    }

    public void afterThrowing(Method m, Object[] args, Object target, ServletException ex) {
        // Do something with all arguments
    }
}
```

> 如果throws-advice方法自己抛出了一个异常，它会覆盖原始异常（也就是会改变抛给用户的异常）。覆盖的异常一般是`RuntimeException`，它可以和任何方法参数兼容。但是，如果throws-advice方法抛出了一个受检异常，那么它必须匹配目标方法声明的异常，并且某种程度上这和特定的方法签名耦合。*不要抛出一个和目标方法签名不兼容的未声明受检异常！*

> Throws advice可以和任何切点一起使用。

##### After Returning Advice

Spring中的after returning advice必须实现`org.springframework.aop.AfterReturningAdvice`接口，如下所示：

```
public interface AfterReturningAdvice extends Advice {

    void afterReturning(Object returnValue, Method m, Object[] args, Object target)
            throws Throwable;
}
```

after returning advice可以访问被调用方法的返回值（但它不能修改），方法参数和目标对象。

下面的after returning advice对所有没有抛出异常成功调用的方法进行了计数：

```
public class CountingAfterReturningAdvice implements AfterReturningAdvice {

    private int count;

    public void afterReturning(Object returnValue, Method m, Object[] args, Object target)
            throws Throwable {
        ++count;
    }

    public int getCount() {
        return count;
    }
}
```

这个增强不能改变执行路径。如果它抛出了一个异常，它会在拦截器链中反向传递，而不是返回一个值。

> After returning advice可以和任何切点一起使用。

##### Introduction Advice

Spring将引介增强视为一种特殊的拦截增强。

引介增强需要实现`IntroductionAdvisor` 和 `IntroductionInterceptor`这两个接口：

```
public interface IntroductionInterceptor extends MethodInterceptor {

    boolean implementsInterface(Class intf);
}
```

继承自AOP联盟的`MethodInterceptor`接口的`invoke()`方法必须实现引介。也就是，如果被调用的方法属于一个引介接口，那么引介拦截器负责处理这个方法调用 —— 它不会调用`proceed()`方法。

引介增强无法和任何切点一起使用，因为它只能在类上应用，而不是在方法上。你只可以通过`IntroductionAdvisor`使用引介增强，这个接口拥有下面的方法：

```
public interface IntroductionAdvisor extends Advisor, IntroductionInfo {

    ClassFilter getClassFilter();

    void validateInterfaces() throws IllegalArgumentException;
}

public interface IntroductionInfo {

    Class[] getInterfaces();
}
```

这里没有`MethodMatcher`，因此`Pointcut`无法和引介增强关联到一起。只有类过滤器是必要的。

`getInterfaces()`方法返回这个advisor引入的接口。

`validateInterfaces()`方法在内部使用，用来查看被引入的接口是否可以被`IntroductionInterceptor`所实现。

考虑下面的例子，假设我们想要引入下面的接口到一个或多个对象中：

```
public interface Lockable {
    void lock();
    void unlock();
    boolean locked();
}
```

我们想要将被增强的对象转换为`Lockable`，不管它们的类型是什么，并且调用lock和unlock方法。如果我们调用了`lock()`方法，我们想要让所有的setter方法调用都抛出一个 `LockedException`异常。因此，我们可以增加一个提供对象不可变能力的切面，并且它们自身不知道，这是AOP的一个很好的例子。

首先，我们需要一个`IntroductionInterceptor`。此时，我们可以继承一个方便的类`org.springframework.aop.support.DelegatingIntroductionInterceptor`。我们也可以直接实现`IntroductionInterceptor`接口，但是在大多数情况下使用`DelegatingIntroductionInterceptor`最好。

`DelegatingIntroductionInterceptor`使用一个引介接口的实现类来代表引介接口，这实际上是拦截器所要做的，但是被这个委托类隐藏了。你可以使用它的构造方法的参数将任何对象设置为代表对象(delegate)，默认的delegate（当使用无参构造方法时）是`this`。在下一个例子中，delegate是`LockMixin`。给定一个delegate，`DelegatingIntroductionInterceptor`会查找这个代表对象实现的所有接口（除了`IntroductionInterceptor`），并且使用这些接口来支持引介。它的子类例如`LockMixin`可以调用`suppressInterface(Class intf)`方法来排除不应当被暴露的接口。但是，不管`IntroductionInterceptor`准备支持多少个接口，`IntroductionAdvisor`才真正控制哪些接口应当被暴露。An introduced interface conceals any implementation of the same interface by the target.

因此，`LockMixin`继承了`DelegatingIntroductionInterceptor`类并且实现了`Lockable`接口。因为它使用默认构造方法，所以delegate是它自己，父类会自动找出`Lockable`接口可以支持引介，所以我们不需要自己指定。我们可以引入任意数量的接口。

注意`locked`实例变量的使用，这在目标对象中增加了额外的状态。

下面展示了`LockMixin`类：

```
public class LockMixin extends DelegatingIntroductionInterceptor implements Lockable {

    private boolean locked;

    public void lock() {
        this.locked = true;
    }

    public void unlock() {
        this.locked = false;
    }

    public boolean locked() {
        return this.locked;
    }

    public Object invoke(MethodInvocation invocation) throws Throwable {
        if (locked() && invocation.getMethod().getName().indexOf("set") == 0) {
            throw new LockedException();
        }
        return super.invoke(invocation);
    }

}
```

一般来说，你不需要重写`invoke()`方法。`DelegatingIntroductionInterceptor`已经足够使用了（如果被调用的方法属于引介接口，那么通过反射调用`delegate`中实现的此方法，否则继续执行连接点）。在这个例子中，我们需要增加一个检查：如果在锁模式下，不能调用setter方法。

IntroductionAdvisor只需要持有一个`LockMixin`实例，并且指定被引入的接口（此处只有`Lockable`）即可。一个更复杂的例子可能会持有引介拦截器（应当被定义为prototype）的引用。在这里，`LockMixin`实例不需要进行配置，所以我们直接使用`new`来创建它。下面的例子展示了`LockMixinAdvisor`类：

```
public class LockMixinAdvisor extends DefaultIntroductionAdvisor {

    public LockMixinAdvisor() {
        super(new LockMixin(), Lockable.class);
    }
}
```

我们可以很简单的应用这个advisor，因为它不需要任何配置。（但是，没有`IntroductionAdvisor`的话无法使用`IntroductionInterceptor`）。和引介拦截器一样，advisor必须是per-instance，因为它是有状态的。对于每一个被增强的对象(`LockMixin`)我们都需要`LockMixinAdvisor`类的一个不同实例。advisor组成了被增强对象的一部分状态。

我们可以使用`Advised.addAdvisor()`方法硬编码应用这个advisor，或者使用XML配置（推荐方式）。所有在后面讨论的代理创建选择，包括“auto proxy creators”，都可以正确处理引介和有状态的混合(mixin)。

### 6.3\. The Advisor API in Spring

在Spring中，Advisor是一个只包含一个增强对象的切面，并且和一个切点表达式关联。

除了引介的特殊使用外，其他advisor都可以和增强一起使用。`org.springframework.aop.support.DefaultPointcutAdvisor`是最常用的advisor类，它可以和`MethodInterceptor`, `BeforeAdvice` 或 `ThrowsAdvice`一起使用。

可以在同一个AOP代理中混用advisor和增强类型。例如，你可以在一个代理配置中使用 interception around advice, throws advice 以及 before advice。Spring会自动创建必要的拦截器链。

### 6.4\. Using the `ProxyFactoryBean` to Create AOP Proxies

如果你为你的业务对象使用了Spring IoC容器(一个 `ApplicationContext` 或`BeanFactory`)，你会想要使用Spring的AOP `FactoryBean`实现类中的一个。（记住`FactoryBean`引入了一个间接层次，可以创建不同类型的对象）。

> Spring AOP在底层也使用了`FactoryBean`。

在Spring中创建一个AOP代理的基本方式是使用`org.springframework.aop.framework.ProxyFactoryBean`。它给予了对切点，需要应用的增强以及其顺序的完全控制。但是，如果你不需要这些控制，有更简单的选择。

#### 6.4.1\. Basics

ProxyFactoryBean`与其他Spring `FactoryBean`实现类一样，引入了一个间接层次。如果你定义了一个叫做`foo`的`ProxyFactoryBean`，引用`ProxyFactoryBean`的对象不会发现`ProxyFactoryBean`实例本身，而是发现通过`ProxyFactoryBean`的`getObject()`方法创建的对象。这个方法创建了一个AOP代理并且包装了目标对象。

使用`ProxyFactoryBean`或者其他IoC可知的类来创建AOP代理的一个好处是增强以及切点也可以被IoC管理。这个一个强大的特性，可以完成其他AOP框架很难做到的事。例如，一个增强本身可能引用了应用对象，那么它会受益于依赖注入提供的可拔插。

#### 6.4.2\. JavaBean Properties

与大多数Spring提供的`FactoryBean`实现类一样，`ProxyFactoryBean`类本身也是一个JavaBean。它的属性用作：

*   指定你想要代理的目标对象

*   指定是否使用CGLIB代理（在后面讨论，或者可以查看[JDK- and CGLIB-based proxies](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-pfb-proxy-types)）

一些核心属性继承自`org.springframework.aop.framework.ProxyConfig`(Spring中所有AOP代理工厂的父类)：

*   `proxyTargetClass`: 如果是目标类要被代理，而不是目标类的接口被代理，这个属性为`true`。如果这个属性值被设为`true`，那么就会创建CGLIB代理

*   `optimize`: 控制是否对通过CGLIB创建的代理进行激进优化。你不应该随意使用这个属性设置，除非你完全理解了相关的AOP代理如何处理优化。这只对CGLIB代理有用，对于JDK动态代理没有任何影响。

*   `frozen`: 如果一个代理配置是`frozen`，那么将不再允许改变它的配置。这是一点轻微的优化，并且当你不想让调用者在代理被创建后对代理进行操作(通过`Advised`接口)这是有用的。这个属性的默认值是`false`，所以允许进行修改(例如增加额外的增强) 。

*   `exposeProxy`: 决定当前代理是否应当被暴露在`ThreadLocal`中，这样它可以被目标对象获取。如果目标对象需要获取代理并且`exposeProxy`属性被设置为`true`，那么目标对象可以使用`AopContext.currentProxy()`方法获取代理。

其他`ProxyFactoryBean`特定的属性如下：

*   `proxyInterfaces`: 接口名称的字符串数组。如果没有接口被提供，那么对目标类使用CGLIB代理。

*   `interceptorNames`: `Advisor`，拦截器或者其他增强的名字的字符串数组。它的顺序是有意义的，在数组前面的首先使用。也就是说列表中的第一个拦截器会最先拦截调用。

    这个名字列表是在当前工厂中的bean name，包括父工厂中的bean name。你不可以在这里使用bean引用，因为这样做会导致`ProxyFactoryBean`忽略增强的单例设置。

    你可以使用`*`来附加一个拦截器名字。这样做会导致。你可以在[Using “Global” Advisors](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-global-advisors)中找到使用这个特性的一个例子。You can append an interceptor name with an asterisk (`*`). Doing so results in the application of all advisor beans with names that start with the part before the asterisk to be applied. 

*   singleton: 工厂是否应当返回一个单一的对象，不管`getObject()`方法调用多少次。一些`FactoryBean`实现类提供了这个方法，默认值是`true`。如果你想要使用有状态的增强 —— 例如，对于有状态的混合 —— 将这个属性值设为`false`来使用原型增强。

#### 6.4.3\. JDK- and CGLIB-based proxies

This section serves as the definitive documentation on how the `ProxyFactoryBean` chooses to create either a JDK-based proxy or a CGLIB-based proxy for a particular target object (which is to be proxied).

> The behavior of the `ProxyFactoryBean` with regard to creating JDK- or CGLIB-based proxies changed between versions 1.2.x and 2.0 of Spring. The `ProxyFactoryBean` now exhibits similar semantics with regard to auto-detecting interfaces as those of the `TransactionProxyFactoryBean` class. 

If the class of a target object that is to be proxied (hereafter simply referred to as the target class) does not implement any interfaces, a CGLIB-based proxy is created. This is the easiest scenario, because JDK proxies are interface-based, and no interfaces means JDK proxying is not even possible. You can plug in the target bean and specify the list of interceptors by setting the `interceptorNames` property. Note that a CGLIB-based proxy is created even if the `proxyTargetClass` property of the`ProxyFactoryBean` has been set to `false`. (Doing so makes no sense and is best removed from the bean definition, because it is, at best, redundant, and, at worst confusing.)

If the target class implements one (or more) interfaces, the type of proxy that is created depends on the configuration of the `ProxyFactoryBean`.

If the `proxyTargetClass` property of the `ProxyFactoryBean` has been set to `true`, a CGLIB-based proxy is created. This makes sense and is in keeping with the principle of least surprise. Even if the `proxyInterfaces` property of the `ProxyFactoryBean` has been set to one or more fully qualified interface names, the fact that the `proxyTargetClass` property is set to `true` causes CGLIB-based proxying to be in effect.

If the `proxyInterfaces` property of the `ProxyFactoryBean` has been set to one or more fully qualified interface names, a JDK-based proxy is created. The created proxy implements all of the interfaces that were specified in the `proxyInterfaces` property. If the target class happens to implement a whole lot more interfaces than those specified in the `proxyInterfaces` property, that is all well and good, but those additional interfaces are not implemented by the returned proxy.

If the `proxyInterfaces` property of the `ProxyFactoryBean` has not been set, but the target class does implement one (or more) interfaces, the `ProxyFactoryBean` auto-detects the fact that the target class does actually implement at least one interface, and a JDK-based proxy is created. The interfaces that are actually proxied are all of the interfaces that the target class implements. In effect, this is the same as supplying a list of each and every interface that the target class implements to the `proxyInterfaces`property. However, it is significantly less work and less prone to typographical errors.

#### 6.4.4\. Proxying Interfaces

Consider a simple example of `ProxyFactoryBean` in action. This example involves:

*   A target bean that is proxied. This is the `personTarget` bean definition in the example.

*   An `Advisor` and an `Interceptor` used to provide advice.

*   An AOP proxy bean definition to specify the target object (the `personTarget` bean), the interfaces to proxy, and the advices to apply.

The following listing shows the example:

```
<bean id="personTarget" class="com.mycompany.PersonImpl">
    <property name="name" value="Tony"/>
    <property name="age" value="51"/>
</bean>

<bean id="myAdvisor" class="com.mycompany.MyAdvisor">
    <property name="someProperty" value="Custom string property value"/>
</bean>

<bean id="debugInterceptor" class="org.springframework.aop.interceptor.DebugInterceptor">
</bean>

<bean id="person"
    class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="proxyInterfaces" value="com.mycompany.Person"/>

    <property name="target" ref="personTarget"/>
    <property name="interceptorNames">
        <list>
            <value>myAdvisor</value>
            <value>debugInterceptor</value>
        </list>
    </property>
</bean>
```

Note that the `interceptorNames` property takes a list of `String`, which holds the bean names of the interceptors or advisors in the current factory. You can use advisors, interceptors, before, after returning, and throws advice objects. The ordering of advisors is significant.

> You might be wondering why the list does not hold bean references. The reason for this is that, if the singleton property of the `ProxyFactoryBean` is set to `false`, it must be able to return independent proxy instances. If any of the advisors is itself a prototype, an independent instance would need to be returned, so it is necessary to be able to obtain an instance of the prototype from the factory. Holding a reference is not sufficient. 

The `person` bean definition shown earlier can be used in place of a `Person` implementation, as follows:

```
Person person = (Person) factory.getBean("person");
```

Other beans in the same IoC context can express a strongly typed dependency on it, as with an ordinary Java object. The following example shows how to do so:

```
<bean id="personUser" class="com.mycompany.PersonUser">
    <property name="person"><ref bean="person"/></property>
</bean>
```

The `PersonUser` class in this example exposes a property of type `Person`. As far as it is concerned, the AOP proxy can be used transparently in place of a “real” person implementation. However, its class would be a dynamic proxy class. It would be possible to cast it to the `Advised` interface (discussed later).

You can conceal the distinction between target and proxy by using an anonymous inner bean. Only the `ProxyFactoryBean`definition is different. The advice is included only for completeness. The following example shows how to use an anonymous inner bean:

```
<bean id="myAdvisor" class="com.mycompany.MyAdvisor">
    <property name="someProperty" value="Custom string property value"/>
</bean>

<bean id="debugInterceptor" class="org.springframework.aop.interceptor.DebugInterceptor"/>

<bean id="person" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="proxyInterfaces" value="com.mycompany.Person"/>
    <!-- Use inner bean, not local reference to target -->
    <property name="target">
        <bean class="com.mycompany.PersonImpl">
            <property name="name" value="Tony"/>
            <property name="age" value="51"/>
        </bean>
    </property>
    <property name="interceptorNames">
        <list>
            <value>myAdvisor</value>
            <value>debugInterceptor</value>
        </list>
    </property>
</bean>
```

Using an anonymous inner bean has the advantage that there is only one object of type `Person`. This is useful if we want to prevent users of the application context from obtaining a reference to the un-advised object or need to avoid any ambiguity with Spring IoC autowiring. There is also, arguably, an advantage in that the `ProxyFactoryBean` definition is self-contained. However, there are times when being able to obtain the un-advised target from the factory might actually be an advantage (for example, in certain test scenarios).

#### 6.4.5\. Proxying Classes

What if you need to proxy a class, rather than one or more interfaces?

Imagine that in our earlier example, there was no `Person` interface. We needed to advise a class called `Person` that did not implement any business interface. In this case, you can configure Spring to use CGLIB proxying rather than dynamic proxies. To do so, set the `proxyTargetClass` property on the `ProxyFactoryBean` shown earlier to `true`. While it is best to program to interfaces rather than classes, the ability to advise classes that do not implement interfaces can be useful when working with legacy code. (In general, Spring is not prescriptive. While it makes it easy to apply good practices, it avoids forcing a particular approach.)

If you want to, you can force the use of CGLIB in any case, even if you do have interfaces.

CGLIB proxying works by generating a subclass of the target class at runtime. Spring configures this generated subclass to delegate method calls to the original target. The subclass is used to implement the Decorator pattern, weaving in the advice.

CGLIB proxying should generally be transparent to users. However, there are some issues to consider:

*   `Final` methods cannot be advised, as they cannot be overridden.

*   There is no need to add CGLIB to your classpath. As of Spring 3.2, CGLIB is repackaged and included in the spring-core JAR. In other words, CGLIB-based AOP works “out of the box”, as do JDK dynamic proxies.

There is little performance difference between CGLIB proxying and dynamic proxies. Performance should not be a decisive consideration in this case.

#### 6.4.6\. Using “Global” Advisors

By appending an asterisk to an interceptor name, all advisors with bean names that match the part before the asterisk are added to the advisor chain. This can come in handy if you need to add a standard set of “global” advisors. The following example defines two global advisors:

```
<bean id="proxy" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target" ref="service"/>
    <property name="interceptorNames">
        <list>
            <value>global*</value>
        </list>
    </property>
</bean>

<bean id="global_debug" class="org.springframework.aop.interceptor.DebugInterceptor"/>
<bean id="global_performance" class="org.springframework.aop.interceptor.PerformanceMonitorInterceptor"/>
```

### 6.5\. Concise Proxy Definitions

Especially when defining transactional proxies, you may end up with many similar proxy definitions. The use of parent and child bean definitions, along with inner bean definitions, can result in much cleaner and more concise proxy definitions.

First, we create a parent, template, bean definition for the proxy, as follows:

```
<bean id="txProxyTemplate" abstract="true"
        class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
    <property name="transactionManager" ref="transactionManager"/>
    <property name="transactionAttributes">
        <props>
            <prop key="*">PROPAGATION_REQUIRED</prop>
        </props>
    </property>
</bean>
```

This is never instantiated itself, so it can actually be incomplete. Then, each proxy that needs to be created is a child bean definition, which wraps the target of the proxy as an inner bean definition, since the target is never used on its own anyway. The following example shows such a child bean:

```
<bean id="myService" parent="txProxyTemplate">
    <property name="target">
        <bean class="org.springframework.samples.MyServiceImpl">
        </bean>
    </property>
</bean>
```

You can override properties from the parent template. In the following example, we override the transaction propagation settings:

```
<bean id="mySpecialService" parent="txProxyTemplate">
    <property name="target">
        <bean class="org.springframework.samples.MySpecialServiceImpl">
        </bean>
    </property>
    <property name="transactionAttributes">
        <props>
            <prop key="get*">PROPAGATION_REQUIRED,readOnly</prop>
            <prop key="find*">PROPAGATION_REQUIRED,readOnly</prop>
            <prop key="load*">PROPAGATION_REQUIRED,readOnly</prop>
            <prop key="store*">PROPAGATION_REQUIRED</prop>
        </props>
    </property>
</bean>
```

Note that in the parent bean example, we explicitly marked the parent bean definition as being abstract by setting the `abstract`attribute to `true`, as described [previously](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-child-bean-definitions), so that it may not actually ever be instantiated. Application contexts (but not simple bean factories), by default, pre-instantiate all singletons. Therefore, it is important (at least for singleton beans) that, if you have a (parent) bean definition that you intend to use only as a template, and this definition specifies a class, you must make sure to set the `abstract` attribute to `true`. Otherwise, the application context actually tries to pre-instantiate it.

### 6.6\. Creating AOP Proxies Programmatically with the `ProxyFactory`

It is easy to create AOP proxies programmatically with Spring. This lets you use Spring AOP without dependency on Spring IoC.

The interfaces implemented by the target object are automatically proxied. The following listing shows creation of a proxy for a target object, with one interceptor and one advisor:

```
ProxyFactory factory = new ProxyFactory(myBusinessInterfaceImpl);
factory.addAdvice(myMethodInterceptor);
factory.addAdvisor(myAdvisor);
MyBusinessInterface tb = (MyBusinessInterface) factory.getProxy();
```

The first step is to construct an object of type `org.springframework.aop.framework.ProxyFactory`. You can create this with a target object, as in the preceding example, or specify the interfaces to be proxied in an alternate constructor.

You can add advices (with interceptors as a specialized kind of advice), advisors, or both and manipulate them for the life of the `ProxyFactory`. If you add an `IntroductionInterceptionAroundAdvisor`, you can cause the proxy to implement additional interfaces.

There are also convenience methods on `ProxyFactory` (inherited from `AdvisedSupport`) that let you add other advice types, such as before and throws advice. `AdvisedSupport` is the superclass of both `ProxyFactory` and `ProxyFactoryBean`.

> Integrating AOP proxy creation with the IoC framework is best practice in most applications. We recommend that you externalize configuration from Java code with AOP, as you should in general. 

### 6.7\. Manipulating Advised Objects

However you create AOP proxies, you can manipulate them BY using the `org.springframework.aop.framework.Advised`interface. Any AOP proxy can be cast to this interface, no matter which other interfaces it implements. This interface includes the following methods:

```
Advisor[] getAdvisors();

void addAdvice(Advice advice) throws AopConfigException;

void addAdvice(int pos, Advice advice) throws AopConfigException;

void addAdvisor(Advisor advisor) throws AopConfigException;

void addAdvisor(int pos, Advisor advisor) throws AopConfigException;

int indexOf(Advisor advisor);

boolean removeAdvisor(Advisor advisor) throws AopConfigException;

void removeAdvisor(int index) throws AopConfigException;

boolean replaceAdvisor(Advisor a, Advisor b) throws AopConfigException;

boolean isFrozen();
```

The `getAdvisors()` method returns an `Advisor` for every advisor, interceptor, or other advice type that has been added to the factory. If you added an `Advisor`, the returned advisor at this index is the object that you added. If you added an interceptor or other advice type, Spring wrapped this in an advisor with a pointcut that always returns `true`. Thus, if you added a `MethodInterceptor`, the advisor returned for this index is a `DefaultPointcutAdvisor` that returns your `MethodInterceptor` and a pointcut that matches all classes and methods.

The `addAdvisor()` methods can be used to add any `Advisor`. Usually, the advisor holding pointcut and advice is the generic `DefaultPointcutAdvisor`, which you can use with any advice or pointcut (but not for introductions).

By default, it is possible to add or remove advisors or interceptors even once a proxy has been created. The only restriction is that it is impossible to add or remove an introduction advisor, as existing proxies from the factory do not show the interface change. (You can obtain a new proxy from the factory to avoid this problem.)

The following example shows casting an AOP proxy to the `Advised` interface and examining and manipulating its advice:

```
Advised advised = (Advised) myObject;
Advisor[] advisors = advised.getAdvisors();
int oldAdvisorCount = advisors.length;
System.out.println(oldAdvisorCount + " advisors");

// Add an advice like an interceptor without a pointcut
// Will match all proxied methods
// Can use for interceptors, before, after returning or throws advice
advised.addAdvice(new DebugInterceptor());

// Add selective advice using a pointcut
advised.addAdvisor(new DefaultPointcutAdvisor(mySpecialPointcut, myAdvice));

assertEquals("Added two advisors", oldAdvisorCount + 2, advised.getAdvisors().length);
```

> It is questionable whether it is advisable (no pun intended) to modify advice on a business object in production, although there are, no doubt, legitimate usage cases. However, it can be very useful in development (for example, in tests). We have sometimes found it very useful to be able to add test code in the form of an interceptor or other advice, getting inside a method invocation that we want to test. (For example, the advice can get inside a transaction created for that method, perhaps to run SQL to check that a database was correctly updated, before marking the transaction for roll back.) 

Depending on how you created the proxy, you can usually set a `frozen` flag. In that case, the `Advised` `isFrozen()` method returns `true`, and any attempts to modify advice through addition or removal results in an `AopConfigException`. The ability to freeze the state of an advised object is useful in some cases (for example, to prevent calling code removing a security interceptor).

### 6.8\. Using the "auto-proxy" facility

So far, we have considered explicit creation of AOP proxies by using a `ProxyFactoryBean` or similar factory bean.

Spring also lets us use “auto-proxy” bean definitions, which can automatically proxy selected bean definitions. This is built on Spring’s “bean post processor” infrastructure, which enables modification of any bean definition as the container loads.

In this model, you set up some special bean definitions in your XML bean definition file to configure the auto-proxy infrastructure. This lets you declare the targets eligible for auto-proxying. You need not use `ProxyFactoryBean`.

There are two ways to do this:

*   By using an auto-proxy creator that refers to specific beans in the current context.

*   A special case of auto-proxy creation that deserves to be considered separately: auto-proxy creation driven by source-level metadata attributes.

#### 6.8.1\. Auto-proxy Bean Definitions

This section covers the auto-proxy creators provided by the `org.springframework.aop.framework.autoproxy` package.

##### `BeanNameAutoProxyCreator`

The `BeanNameAutoProxyCreator` class is a `BeanPostProcessor` that automatically creates AOP proxies for beans with names that match literal values or wildcards. The following example shows how to create a `BeanNameAutoProxyCreator` bean:

```
<bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
    <property name="beanNames" value="jdk*,onlyJdk"/>
    <property name="interceptorNames">
        <list>
            <value>myInterceptor</value>
        </list>
    </property>
</bean>
```

As with `ProxyFactoryBean`, there is an `interceptorNames` property rather than a list of interceptors, to allow correct behavior for prototype advisors. Named “interceptors” can be advisors or any advice type.

As with auto-proxying in general, the main point of using `BeanNameAutoProxyCreator` is to apply the same configuration consistently to multiple objects, with minimal volume of configuration. It is a popular choice for applying declarative transactions to multiple objects.

Bean definitions whose names match, such as `jdkMyBean` and `onlyJdk` in the preceding example, are plain old bean definitions with the target class. An AOP proxy is automatically created by the `BeanNameAutoProxyCreator`. The same advice is applied to all matching beans. Note that, if advisors are used (rather than the interceptor in the preceding example), the pointcuts may apply differently to different beans.

##### `DefaultAdvisorAutoProxyCreator`

A more general and extremely powerful auto-proxy creator is `DefaultAdvisorAutoProxyCreator`. This automagically applies eligible advisors in the current context, without the need to include specific bean names in the auto-proxy advisor’s bean definition. It offers the same merit of consistent configuration and avoidance of duplication as `BeanNameAutoProxyCreator`.

Using this mechanism involves:

*   Specifying a `DefaultAdvisorAutoProxyCreator` bean definition.

*   Specifying any number of advisors in the same or related contexts. Note that these must be advisors, not interceptors or other advices. This is necessary, because there must be a pointcut to evaluate, to check the eligibility of each advice to candidate bean definitions.

The `DefaultAdvisorAutoProxyCreator` automatically evaluates the pointcut contained in each advisor, to see what (if any) advice it should apply to each business object (such as `businessObject1` and `businessObject2` in the example).

This means that any number of advisors can be applied automatically to each business object. If no pointcut in any of the advisors matches any method in a business object, the object is not proxied. As bean definitions are added for new business objects, they are automatically proxied if necessary.

Auto-proxying in general has the advantage of making it impossible for callers or dependencies to obtain an un-advised object. Calling `getBean("businessObject1")` on this `ApplicationContext` returns an AOP proxy, not the target business object. (The “inner bean” idiom shown earlier also offers this benefit.)

The following example creates a `DefaultAdvisorAutoProxyCreator` bean and the other elements discussed in this section:

```
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>

<bean class="org.springframework.transaction.interceptor.TransactionAttributeSourceAdvisor">
    <property name="transactionInterceptor" ref="transactionInterceptor"/>
</bean>

<bean id="customAdvisor" class="com.mycompany.MyAdvisor"/>

<bean id="businessObject1" class="com.mycompany.BusinessObject1">
    <!-- Properties omitted -->
</bean>

<bean id="businessObject2" class="com.mycompany.BusinessObject2"/>
```

The `DefaultAdvisorAutoProxyCreator` is very useful if you want to apply the same advice consistently to many business objects. Once the infrastructure definitions are in place, you can add new business objects without including specific proxy configuration. You can also easily drop in additional aspects (for example, tracing or performance monitoring aspects) with minimal change to configuration.

The `DefaultAdvisorAutoProxyCreator` offers support for filtering (by using a naming convention so that only certain advisors are evaluated, which allows the use of multiple, differently configured, AdvisorAutoProxyCreators in the same factory) and ordering. Advisors can implement the `org.springframework.core.Ordered` interface to ensure correct ordering if this is an issue. The `TransactionAttributeSourceAdvisor` used in the preceding example has a configurable order value. The default setting is unordered.

### 6.9\. Using `TargetSource` Implementations

Spring offers the concept of a `TargetSource`, expressed in the `org.springframework.aop.TargetSource` interface. This interface is responsible for returning the “target object” that implements the join point. The `TargetSource` implementation is asked for a target instance each time the AOP proxy handles a method invocation.

Developers who use Spring AOP do not normally need to work directly with `TargetSource` implementations, but this provides a powerful means of supporting pooling, hot swappable, and other sophisticated targets. For example, a pooling `TargetSource`can return a different target instance for each invocation, by using a pool to manage instances.

If you do not specify a `TargetSource`, a default implementation is used to wrap a local object. The same target is returned for each invocation (as you would expect).

The rest of this section describes the standard target sources provided with Spring and how you can use them.

> When using a custom target source, your target will usually need to be a prototype rather than a singleton bean definition. This allows Spring to create a new target instance when required. 

#### 6.9.1\. Hot-swappable Target Sources

The `org.springframework.aop.target.HotSwappableTargetSource` exists to let the target of an AOP proxy be switched while letting callers keep their references to it.

Changing the target source’s target takes effect immediately. The `HotSwappableTargetSource` is thread-safe.

You can change the target by using the `swap()` method on HotSwappableTargetSource, as the follow example shows:

```
HotSwappableTargetSource swapper = (HotSwappableTargetSource) beanFactory.getBean("swapper");
Object oldTarget = swapper.swap(newTarget);
```

The following example shows the required XML definitions:

```
<bean id="initialTarget" class="mycompany.OldTarget"/>

<bean id="swapper" class="org.springframework.aop.target.HotSwappableTargetSource">
    <constructor-arg ref="initialTarget"/>
</bean>

<bean id="swappable" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="targetSource" ref="swapper"/>
</bean>
```

The preceding `swap()` call changes the target of the swappable bean. Clients that hold a reference to that bean are unaware of the change but immediately start hitting the new target.

Although this example does not add any advice (it is not necessary to add advice to use a `TargetSource`), any `TargetSource` can be used in conjunction with arbitrary advice.

#### 6.9.2\. Pooling Target Sources

Using a pooling target source provides a similar programming model to stateless session EJBs, in which a pool of identical instances is maintained, with method invocations going to free objects in the pool.

A crucial difference between Spring pooling and SLSB pooling is that Spring pooling can be applied to any POJO. As with Spring in general, this service can be applied in a non-invasive way.

Spring provides support for Commons Pool 2.2, which provides a fairly efficient pooling implementation. You need the `commons-pool` Jar on your application’s classpath to use this feature. You can also subclass`org.springframework.aop.target.AbstractPoolingTargetSource` to support any other pooling API.

> Commons Pool 1.5+ is also supported but is deprecated as of Spring Framework 4.2. 

The following listing shows an example configuration:

```
<bean id="businessObjectTarget" class="com.mycompany.MyBusinessObject"
        scope="prototype">
    ... properties omitted
</bean>

<bean id="poolTargetSource" class="org.springframework.aop.target.CommonsPool2TargetSource">
    <property name="targetBeanName" value="businessObjectTarget"/>
    <property name="maxSize" value="25"/>
</bean>

<bean id="businessObject" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="targetSource" ref="poolTargetSource"/>
    <property name="interceptorNames" value="myInterceptor"/>
</bean>
```

Note that the target object (`businessObjectTarget` in the preceding example) must be a prototype. This lets the `PoolingTargetSource` implementation create new instances of the target to grow the pool as necessary. See the [javadoc of`AbstractPoolingTargetSource`](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframeworkaop/target/AbstractPoolingTargetSource.html) and the concrete subclass you wish to use for information about its properties. `maxSize` is the most basic and is always guaranteed to be present.

In this case, `myInterceptor` is the name of an interceptor that would need to be defined in the same IoC context. However, you need not specify interceptors to use pooling. If you want only pooling and no other advice, do not set the `interceptorNames`property at all.

You can configure Spring to be able to cast any pooled object to the `org.springframework.aop.target.PoolingConfig` interface, which exposes information about the configuration and current size of the pool through an introduction. You need to define an advisor similar to the following:

```
<bean id="poolConfigAdvisor" class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
    <property name="targetObject" ref="poolTargetSource"/>
    <property name="targetMethod" value="getPoolingConfigMixin"/>
</bean>
```

This advisor is obtained by calling a convenience method on the `AbstractPoolingTargetSource` class, hence the use of `MethodInvokingFactoryBean`. This advisor’s name (`poolConfigAdvisor`, here) must be in the list of interceptors names in the `ProxyFactoryBean` that exposes the pooled object.

The cast is defined as follows:

```
PoolingConfig conf = (PoolingConfig) beanFactory.getBean("businessObject");
System.out.println("Max pool size is " + conf.getMaxSize());
```

> Pooling stateless service objects is not usually necessary. We do not believe it should be the default choice, as most stateless objects are naturally thread safe, and instance pooling is problematic if resources are cached. 

Simpler pooling is available by using auto-proxying. You can set the `TargetSource` implementations used by any auto-proxy creator.

#### 6.9.3\. Prototype Target Sources

Setting up a “prototype” target source is similar to setting up a pooling `TargetSource`. In this case, a new instance of the target is created on every method invocation. Although the cost of creating a new object is not high in a modern JVM, the cost of wiring up the new object (satisfying its IoC dependencies) may be more expensive. Thus, you should not use this approach without very good reason.

To do this, you could modify the `poolTargetSource` definition shown earlier as follows (we also changed the name, for clarity):

```
<bean id="prototypeTargetSource" class="org.springframework.aop.target.PrototypeTargetSource">
    <property name="targetBeanName" ref="businessObjectTarget"/>
</bean>
```

The only property is the name of the target bean. Inheritance is used in the `TargetSource` implementations to ensure consistent naming. As with the pooling target source, the target bean must be a prototype bean definition.

#### 6.9.4. `ThreadLocal` Target Sources

`ThreadLocal` target sources are useful if you need an object to be created for each incoming request (per thread that is). The concept of a `ThreadLocal` provides a JDK-wide facility to transparently store a resource alongside a thread. Setting up a`ThreadLocalTargetSource` is pretty much the same as was explained for the other types of target source, as the following example shows:

```
<bean id="threadlocalTargetSource" class="org.springframework.aop.target.ThreadLocalTargetSource">
    <property name="targetBeanName" value="businessObjectTarget"/>
</bean>
```

> `ThreadLocal` instances come with serious issues (potentially resulting in memory leaks) when incorrectly using them in multi-threaded and multi-classloader environments. You should always consider wrapping a threadlocal in some other class and never directly use the `ThreadLocal` itself (except in the wrapper class). Also, you should always remember to correctly set and unset (where the latter simply involves a call to `ThreadLocal.set(null)`) the resource local to the thread. Unsetting should be done in any case, since not unsetting it might result in problematic behavior. Spring’s `ThreadLocal` support does this for you and should always be considered in favor of using `ThreadLocal` instances without other proper handling code. 

### 6.10\. Defining New Advice Types

Spring AOP is designed to be extensible. While the interception implementation strategy is presently used internally, it is possible to support arbitrary advice types in addition to the interception around advice, before, throws advice, and after returning advice.

The `org.springframework.aop.framework.adapter` package is an SPI package that lets support for new custom advice types be added without changing the core framework. The only constraint on a custom `Advice` type is that it must implement the`org.aopalliance.aop.Advice` marker interface.

See the [`org.springframework.aop.framework.adapter`](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframework/aop/framework/adapter/package-frame.html) javadoc for further information.

[-[null-safety]] = Null-safety

Although Java does not let you express null-safety with its type system, the Spring Framework now provides the following annotations in the `org.springframework.lang` package to let you declare nullability of APIs and fields:

*   [`@Nullable`](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframework/lang/Nullable.html): Annotation to indicate that a specific parameter, return value, or field can be `null`.

*   [`@NonNull`](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframework/lang/NonNull.html): Annotation to indicate that a specific parameter, return value, or field cannot be `null` (not needed on parameters / return values and fields where `@NonNullApi` and `@NonNullFields` apply, respectively).

*   [`@NonNullApi`](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframework/lang/NonNullApi.html): Annotation at the package level that declares non-null as the default semantics for parameters and return values.

*   [`@NonNullFields`](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframework/lang/NonNullFields.html): Annotation at the package level that declares non-null as the default semantics for fields.

The Spring Framework itself leverages these annotations, but they can also be used in any Spring-based Java project to declare null-safe APIs and optionally null-safe fields. Generic type arguments, varargs and array elements nullability are not supported yet but should be in an upcoming release, see [SPR-15942](https://jira.spring.io/browse/SPR-15942) for up-to-date information. Nullability declarations are expected to be fine-tuned between Spring Framework releases, including minor ones. Nullability of types used inside method bodies is outside of the scope of this feature.

> Other common libraries such as Reactor and Spring Data provide null-safe APIs that use a similar nullability arrangement, delivering a consistent overall experience for Spring application developers. 

### 6.11\. Use cases

In addition to providing an explicit declaration for Spring Framework API nullability, these annotations can be used by an IDE (such as IDEA or Eclipse) to provide useful warnings related to null-safety in order to avoid `NullPointerException` at runtime.

They are also used to make Spring API null-safe in Kotlin projects, since Kotlin natively supports [null-safety](https://kotlinlang.org/docs/reference/null-safety.html). More details are available in the [Kotlin support documentation](https://docs.spring.io/spring/docs/current/spring-framework-reference/languages.html#kotlin-null-safety).

### 6.12\. JSR-305 meta-annotations

Spring annotations are meta-annotated with [JSR 305](https://jcp.org/en/jsr/detail?id=305) annotations (a dormant but wide-spread JSR). JSR-305 meta-annotations let tooling vendors like IDEA or Kotlin provide null-safety support in a generic way, without having to hard-code support for Spring annotations.

It is not necessary nor recommended to add a JSR-305 dependency to the project classpath to take advantage of Spring null-safe API. Only projects such as Spring-based libraries that use null-safety annotations in their codebase should add `com.google.code.findbugs:jsr305:3.0.2` with `compileOnly` Gradle configuration or Maven `provided` scope to avoid compile warnings.

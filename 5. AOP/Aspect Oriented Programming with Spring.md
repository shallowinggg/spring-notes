面向切面编程(AOP)提供了另一种编程架构补充面向对象编程(OOP)。OOP的核心模块单元是类，而AOP的核心模块单元是切面[aspect]。切面确保了关注点的模块化 [modularization of concerns] （例如事务管理），它涉及了多个类型和对象，关注点在AOP文献中经常使用横切(“crosscutting”)这个单词来表示。

Spring的一个核心组件就是AOP框架。虽然Spring IoC容器不依赖AOP（意味着如果你不想可以不使用AOP），但是AOP提供了一个很好的中间件解决方法来补充Spring IoC。

> Spring AOP with AspectJ pointcuts<br/>
Spring提供了简单有用的方式来编写自定义切面，可以使用 [schema-based approach](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-schema) 或者 [@AspectJ annotation style](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-ataspectj) 这两种方法。这两种方法都提供了完全的类型化增强[Advice]以及AspectJ切面语言的使用，并且仍然使用Spring AOP来织入[weaving]。<br/>
本章讨论了基于schema和@AspectJ的AOP支持，更底层的AOP支持在[下一章](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-api)讨论。

Spring框架使用了AOP的地方：

- 提供声明的[declarative]业务服务[service]，最重要的服务是[declarative transaction management](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#transaction-declarative)。
- 可以让用户自定义切面，使用AOP来补充它们OOP的使用。

> 如果你只对泛型声明的服务或者其他预先包装的中间件服务例如线程池感兴趣，那么你不需要直接使用Spring AOP，并且可以跳过本章大部分内容。

### 5.1\. AOP Concepts

让我们先定义一些核心的AOP概念以及术语，这些单词不是Spring专有的。不幸的是，AOP术语并不是特别符合直觉的。但是，如果Spring自定义一套术语的话，那么会让人更加迷惑。

- 切面(Aspect)：涉及多个类的横切逻辑[concern]的模块。一个好的例子就是企业Java应用中的事务管理。在Spring AOP中，切面使用常规类([schema-based approach](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-schema))或者使用`@Aspect`注解的常规类([@AspectJ style](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-ataspectj))来实现。

- 连接点(Join point)：程序执行的一个点，例如一个方法的执行或者一个异常的处理。在Spring AOP中，连接点总是表示一个方法的执行。

- 增强/通知(Advice)：在一个特定连接点的切面采取的动作。有不同类型的增强，包括“around”, “before” 和 “after”（增强类型在后面讨论）。许多AOP框架，包括Spring，都将增强模型化为一个拦截器并且在连接点周围控制一个拦截器链。

- 切点(Poincut)：匹配连接点的断言[predicate]。Advice与一个切点表达式关联在一起并且在任何与切点匹配的连接点上执行（例如，通过某个名字的方法的执行）。通过切点表达式匹配的连接点的概念是AOP的核心，并且Spring默认使用AspectJ切点表达式。

- 引入/引介(Introduction)：在类型的代表[behalf]上声明额外的方法或者字段。Spring AOP可以让你在任何被增强的对象上引入新的接口（以及一个对应的实现）。例如，你可以使用引介让一个bean实现`IsModified`接口来实现简单的缓存。（引介这个词在AspectJ社区中叫做内部类型声明[inter-type declaration]）

- 目标对象(Target object)：被一个或多个切面增强的对象，在其他文献也被称为“advised object”。因为Spring AOP使用运行时代理实现，所以这个对象永远都是一个被代理的对象。

- AOP代理：为了实现切面（增强方法执行等等）而通过AOP框架创建的对象。在Spring框架中，AOP代理或者是一个JDK动态代理或者是一个CGLIB代理。

- 织入(Weaving)：连接切面和其他应用类型或对象以创建一个增强对象，这可以在编译时（例如使用AspectJ编译器），加载时或者运行时进行。Spring AOP和其他纯粹的Java AOP框架一样都在运行时执行织入。

Spring AOP包括了下面的增强类型：

- 前置增强(Before advice): 在连接点(在Spring AOP中永远是方法)运行前进行增强，但这不能阻止执行流前进到连接点（除非它抛出一个异常），即被增强的方法一定会执行，除非增强代码抛出异常。

- 后置增强(After returning advice)：在连接点正常执行（例如，一个方法返回时没有抛出异常）完成后增强。

- 异常抛出增强(After throwing advice)：如果一个方法抛出异常退出，进行增强。

- finally增强(After (finally) advice)：不管连接点如何（正常或者抛出异常返回）退出，都在退出后进行增强。

- 环绕增强(Around advice)：围绕一个连接点例如方法调用进行增强，这是最强大的增强类型。Around advice可以在方法调用前后执行自定义行为，它也负责选择是否执行连接点，还可以返回一个自定义的返回值或者抛出一个异常来代替被增强的方法调用结果。

Around advice是最通用的增强类型。因为Spring AOP像AspectJ一样，提供了许多增强类型，所以我们建议你使用通用性最低的增强类型来实现需要的行为。例如，如果你只需要使用一个方法的返回值来更新缓存，那么你最好实现after returning advice而不是around advice，虽然around advice可以完成相同的功能。使用最特化的增强类型可以提供一个更简单的编程模型，这样会产生更少的潜在错误。例如，你不需要在around advice使用的`JoinPoint`上调用`proceed()`方法，因此，你不会遇到失败调用。

所有的增强参数都被静态类型化了，因此你可以使用合适类型的增强参数（例如，使用方法执行的返回值类型，而不是`Object`数组）。

通过切点匹配连接点的概念是AOP的核心，它区分了只提供拦截器的老技术。切点确保被定位的增强独立于对象继承体系。例如，你可以使用around advice来对贯穿多个对象的一组方法提供事务管理（例如在service层的所有业务操作）。

### 5.2\. Spring AOP Capabilities and Goals

Spring AOP是以纯粹的Java实现的，因此无需特殊的编译过程。Spring AOP不需要控制类加载器层次，因此适合在servlet容器或者应用服务器上使用。

Spring AOP现在只支持方法执行连接点（增强Spring bean的方法执行）。字段拦截还没有实现，虽然无需打破核心Spring AOP APIs就可以提供对于字段拦截的支持。如果你需要增强字段方法以及更新连接点，可以考虑使用例如AspectJ之类的语言。

Spring AOP与大多数其他AOP框架不同，Spring AOP的目的不是提供完整的AOP实现（虽然Spring AOP已经足够使用了）。相反，它的目的是提供在AOP实现和Spring IoC之间的封闭集成，来帮助解决企业应用中的常见问题。

因此，Spring框架的AOP一般用来和Spring IoC容器协作使用，切面可以使用常规的bean定义语法来配置（虽然可以使用更强大的自动代理功能）。这是和其他AOP实现框架之间最关键的区别，使用Spring AOP时你可能无法更简单或更有效地完成某些事，例如增强细粒度的对象（例如domain对象），AspectJ是在这种情况下最好的选择。但是，Spring AOP为企业Java应用出现的大多数问题提供了极好的解决方案。

Spring AOP从不力求和AspectJ比较来提供一个综合的AOP解决方案。我们相信所有基于代理的框架例如Spring AOP以及成熟的框架例如AspectJ都是有价值的，并且它们是互补的而不是竞争的。Spring使用AspectJ无缝集成了Spring IoC和Spring AOP，以确保所有AOP的使用都和基于Spring的应用保持一致性。这个集成不影响Spring AOP API或者AOP Alliance API，Spring AOP仍然是向后兼容的。查看[下一章](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-api)获取关于Spring AOP APIs的讨论。

> Spring框架的一个中心原则是非侵入性，因此你不会被强制要求引入框架特有的类或接口到你的业务模型中。但是，在某些地方，Spring框架可以让你引入Spring框架特有的依赖到你的代码中。给予你这种选择的根本原因是在某些环境下，可以更简单的阅读或编写某些特定功能的代码。但是，Spring框架总是（大多数）会向你提供自由选择的权利：你可以自由决定哪种最适合你的用例或者环境。<br/>
关于本章的一个这样的选择就是选择哪一个AOP框架（以及哪一种AOP风格）。你可以选择AspectJ，Spring AOP或者一起使用它们，你也可以选择使用@AspectJ注解风格或者Spring XML配置风格。本章先介绍@AspectJ注解风格不代表Spring团队更喜欢@AspectJ注解风格而不是Spring XML配置风格。<br/>
查看[Choosing which AOP Declaration Style to Use](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-choosing)获取更多关于每种风格的完整讨论。

### 5.3\. AOP Proxies

Spring AOP默认使用标准的JDK动态代理，这可以确保任何接口（或者接口集合）可以被代理。

Spring AOP也使用了CGLIB代理，因为有些时候只能代理类。默认情况下，如果业务对象没有实现接口，那么就使用CGLIB。因为对接口编程是一种更好的实践，所以业务类一般都要实现一个或多个业务接口。你也可以[强制使用CGLIB](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-proxying)，当你需要对一个没有声明在接口中的方法进行增强或者你需要将一个代理对象作为真实类型传入到方法中。

最重要的事实是Spring AOP是基于代理的，查看[Understanding AOP Proxies](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-understanding-aop-proxies)获取更多细节。

### 5.4\. @AspectJ support

@AspectJ是一种AOP风格，它使用注解将一个常规的Java类声明为切面。@AspectJ风格作为AspectJ 5发行版的一部分引入，查看[AspectJ project](https://www.eclipse.org/aspectj)获取更多信息。Spring使用了与AspectJ 5相同的注解，还使用了AspectJ提供的解析以及匹配切点的一个库。但是AOP运行时仍然是纯粹的Spring AOP，它不依赖于AspectJ编译器或织入。

> 通过AspectJ编译器和织入能够使用完整的AspectJ语言，查看[Using AspectJ with Spring Applications](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-using-aspectj)获取更多信息。

#### 5.4.1\. Enabling @AspectJ Support

为了在Spring配置中使用@AspectJ，你需要确保Spring能够配置基于@AspectJ切面的Spring AOP，还要支持能够自动代理不管是否基于这些切面增强的bean。通过自动代理，如果Spring决定一个bean被一个或多个切面增强，那么它会自动为这个bean产生一个代理来拦截方法调用，并且确保增强在必要时执行。

@AspectJ支持可以使用XML或者Java风格的配置来启用。无论使用哪一种，你都需要确保AspectJ的`aspectjweaver.jar`库在你的应用的classpath中（1.8版本及之后）。这个库可以在AspectJ发行版的`lib`目录下或者Maven Central repository中获得。

##### Enabling @AspectJ Support with Java Configuration

使用Java `@Configuration`时启用@AspectJ支持，需要增加`@EnableAspectJAutoProxy`注解，如下所示：

```
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {

}
```

##### Enabling @AspectJ Support with XML Configuration

使用基于XML的配置启用@AspectJ支持时，需要使用`aop:aspectj-autoproxy`元素，如下所示：

```
<aop:aspectj-autoproxy/>
```

这假设了你使用了[XML Schema-based configuration](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#xsd-schemas)中介绍的schema支持，查看[the AOP schema](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#xsd-schemas-aop)获取如何在`aop`命名空间下引用标签。

#### 5.4.2\. Declaring an Aspect

启用@AspectJ支持后，任何在application context中使用@AspectJ切面定义的bean都会被Spring自动检测出来并且用来配置Spring AOP。下面两个例子展示了最小化的定义。

第一个例子展示了一个bean定义，它的class指向一个拥有`@Aspect`注解的类：

```
<bean id="myAspect" class="org.xyz.NotVeryUsefulAspect">
    <!-- configure properties of the aspect here -->
</bean>
```

第二个例子展示了`NotVeryUsefulAspect`类定义，它使用了`org.aspectj.lang.annotation.Aspect`注解：

```
package org.xyz;
import org.aspectj.lang.annotation.Aspect;

@Aspect
public class NotVeryUsefulAspect {

}
```

切面（使用`@Aspect`注解的类）可以拥有方法和字段，与其他类一样。它们也可以包含切点，增强以及引介声明。

> *Autodetecting aspects through component scanning*<br/>
你可以在你的Spring XML配置中将切面类作为一个常规的bean注册，或者通过classpath扫描自动检测它们 —— 与其他Spring管理的bean一样。但是，注意classpath扫描不会检测`@Aspect`注解。因此，你需要增加一个`@Component`注解（或者一个自定义的stereotype注解，它需要符合Spring组件扫描器的规则）。

> *Advising aspects with other aspects?*<br/>
在Spring AOP中，切面本身不能成为另一个切面的目标。一个类上的`@Aspect`注解标明它是一个切面，因此会将它从自动代理的目标中排除。

#### 5.4.3\. Declaring a Pointcut

切点决定了感兴趣的连接点，因此当增强执行时我们可以控制增强哪些连接点。Spring AOP只支持Spring bean的方法执行[method execution]连接点（*在Spring AOP中，连接点总是表示一个方法的执行*），所以你可以将切点视作Spring bean中匹配的方法。一个切点声明了两个部分：包含名字以及参数的签名，精确决定我们感兴趣的方法的表达式。在@AspectJ注解风格的AOP中，切点签名通过一个常规的方法定义提供，切点表达式使用`@Pointcut`注解表示（用作切点签名的方法返回类型必须是`void`）。

下面的例子可以帮助在切点签名和切面表达式中做出明确的区分，它定义了一个叫做`anyOldTransfer`的切点，这个切点匹配任何名为 `transfer`的方法：

```
@Pointcut("execution(* transfer(..))")// the pointcut expression
private void anyOldTransfer() {}// the pointcut signature
```

组成`@Pointcut`注解值的切点表达式是一个常规的AspectJ 5切点表达式。对于AspectJ的切点语言的完整讨论，查看[AspectJ Programming Guide](https://www.eclipse.org/aspectj/doc/released/progguide/index.html) (关于拓展，查看 [AspectJ 5 Developer’s Notebook](https://www.eclipse.org/aspectj/doc/released/adk15notebook/index.html))或者讲述AspectJ的书籍 (例如 *Eclipse AspectJ*, by Colyer et. al., 或者 *AspectJ in Action*, by Ramnivas Laddad)。

##### Supported Pointcut Designators

Spring AOP支持在切点表达式中使用下面的AspectJ切点指示符(PCD)：

*   `execution`: 满足某一匹配模式的方法执行连接点。这是使用Spring AOP时最常用的切点指示符。

- `within`: 以某些类型限定（当使用Spring AOP时，指在匹配的类型中声明的方法）匹配的连接点

- `this`: 限定匹配的连接点（当使用Spring AOP时，指方法执行），当bean引用（Spring AOP代理）是给定类型的一个实例时。

- `target`: 限定匹配的连接点（当使用Spring AOP时，指方法执行），当目标对象（正在被代理的应用对象）是给定类型的实例时。

- `args`: 限定匹配的连接点（当使用Spring AOP时，指方法执行），当参数是给定类型的实例时。

- `@target`: 限定匹配的连接点（当使用Spring AOP时，指方法执行），当执行对象类拥有一个给定类型的注解时。

- `@args`: 限定匹配的连接点（当使用Spring AOP时，指方法执行），当被传入的真实参数的运行时类型拥有给定类型的注解时。

- `@within`: 以拥有给定注解的类型限定匹配的连接点（当使用Spring AOP时，指拥有给定注解的类声明的方法执行）

- `@annotation`: 限定匹配的连接点，当连接点（在Spring AOP中被执行的方法）拥有给定的注解时。

> 其他切点类型<br/>
完整的AspectJ切点语言支持了额外的切点指示符，下面这些在Spring中不支持：`call`, `get`, `set`, `preinitialization`, `staticinitialization`, `initialization`, `handler`, `adviceexecution`, `withincode`, `cflow`,`cflowbelow`, `if`, `@this` 和 `@withincode`。在Spring AOP中的切点表达式中使用这些切点指示符会抛出一个`IllegalArgumentException`异常。<br/>
Spring AOP支持的切点指示符集合可能会在未来进行拓展以支持更多AspectJ切点指示符。

因为Spring AOP只限定了匹配方法执行连接点，所以前面讨论的切点指示符比AspectJ编程指导中介绍的有更窄的定义。另外，AspectJ本身拥有基于类型的语义，并且在一个连接点中，`this` 和 `target`都引用了同一个对象：执行方法的对象。Spring AOP是一个基于代理的系统并且区分了代理对象本身（与`this`绑定在一起的的对象）和代理背后的目标对象（与`target`绑定在一起对象）。

> 由于Spring AOP框架的本质是基于代理，所以在目标对象内的方法调用不会被拦截。对于JDK代理，只有在代理上调用public接口方法才会被拦截。对于CGLIB，在代理上调用public和protected方法都会被拦截（甚至是包可见的方法）。但是，通过代理的常规交互总是应当使用public签名设计。<br/>
注意切点定义一般与被拦截的方法匹配。如果一个切点严格只能使用public方法，那么即使在CGLIB代理环境通过代理与非public方法交互，它也需要相应的定义。<br/>
如果你的拦截需求包括调用方法甚至目标类的构造方法，可以考虑使用Spring驱动的[native AspectJ weaving](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-aj-ltw)来代替基于代理的Spring AOP框架。它使用了不同的特征构造了一个不同的AOP使用模式，所以在使用之前要确保你自己要熟悉织入。

Spring AOP也支持另一个的PCD，命名`bean`。它可以让你通过一个特定名字的Spring bean或者一组命名的Spring bean（当使用通配符时）来限定连接点的匹配。`bean` PCD拥有下面的形式：

```
bean(idOrNameOfBean)
```

`idOrNameOfBean`可以是任何Spring bean的名字。通配符支持使用`*`字符，所以如果你为你的Spring bean建立了一些命名惯例，你可以编写一个`bean` PCD表达式来选择他们。与其他切点指示符的例子一样，`bean` PCD也可以使用`&&` (与), `||` (或),  `!` (非)操作符。

> `bean` PCD只在Spring AOP中支持并且它不存在于AspectJ中。它是Spring对AspectJ定义的标准PCD的拓展，因此，无法在`@Aspect`模型声明的切面中使用。<br/>
`bean` PCD可以在实例等级（以Spring bean name概念为基础）上操作而不是只在类等级上（基于织入的AOP被限制）。基于实例的切点指示符是基于代理的Spring AOP框架的特有功能并且它被封闭集成到Spring bean工厂中，可以通过名称直接找出特定的bean。

##### Combining Pointcut Expressions

你可以使用`&&,` `||` 和 `!`来结合切点表达式，你还可以通过名称引用切点表达式。下面展示了三个切点表达式：

```
@Pointcut("execution(public * *(..))")
private void anyPublicOperation() {} 

@Pointcut("within(com.xyz.someapp.trading..*)")
private void inTrading() {} 

@Pointcut("anyPublicOperation() && inTrading()")
private void tradingOperation() {} 
```

>  如果一个方法执行连接点表示任何public方法，那么匹配`anyPublicOperation`切点
> 如果一个方法执行在trading模块中，那么匹配`inTrading`切点
> 如果一个方法执行表示在trading模块中的public方法，那么匹配`tradingOperation` 切点

最佳实践是使用小的命名组件来建造更复杂的切点表达式，如前所示。当通过名称引用切点时，使用常规的Java可见性规则（你可以在同一类型中发现private切点，在继承体系中发现protected切点，在任何地方发现public切点）。可见性不会影响切点匹配。

##### Sharing Common Pointcut Definitions

当开发企业应用时，开发者经常会想要在应用中引用模块或者从切面中引用特定的操作集合。我们建议定义一个“SystemArchitecture”切面，用来获取常用的切点表达式。这样的切面一般和下面的例子相似：

```
package com.xyz.someapp;

import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;

@Aspect
public class SystemArchitecture {

    /**
     * A join point is in the web layer if the method is defined
     * in a type in the com.xyz.someapp.web package or any sub-package
     * under that.
     */
    @Pointcut("within(com.xyz.someapp.web..*)")
    public void inWebLayer() {}

    /**
     * A join point is in the service layer if the method is defined
     * in a type in the com.xyz.someapp.service package or any sub-package
     * under that.
     */
    @Pointcut("within(com.xyz.someapp.service..*)")
    public void inServiceLayer() {}

    /**
     * A join point is in the data access layer if the method is defined
     * in a type in the com.xyz.someapp.dao package or any sub-package
     * under that.
     */
    @Pointcut("within(com.xyz.someapp.dao..*)")
    public void inDataAccessLayer() {}

    /**
     * A business service is the execution of any method defined on a service
     * interface. This definition assumes that interfaces are placed in the
     * "service" package, and that implementation types are in sub-packages.
     *
     * If you group service interfaces by functional area (for example,
     * in packages com.xyz.someapp.abc.service and com.xyz.someapp.def.service) then
     * the pointcut expression "execution(* com.xyz.someapp..service.*.*(..))"
     * could be used instead.
     *
     * Alternatively, you can write the expression using the 'bean'
     * PCD, like so "bean(*Service)". (This assumes that you have
     * named your Spring service beans in a consistent fashion.)
     */
    @Pointcut("execution(* com.xyz.someapp..service.*.*(..))")
    public void businessService() {}

    /**
     * A data access operation is the execution of any method defined on a
     * dao interface. This definition assumes that interfaces are placed in the
     * "dao" package, and that implementation types are in sub-packages.
     */
    @Pointcut("execution(* com.xyz.someapp.dao.*.*(..))")
    public void dataAccessOperation() {}

}
```

当你需要一个切点表达式时，你可以在任何地方引用定义在这个切面里的切点。例如，为了让服务层开启事务，你可以如下编写：

```
<aop:config>
    <aop:advisor
        pointcut="com.xyz.someapp.SystemArchitecture.businessService()"
        advice-ref="tx-advice"/>
</aop:config>

<tx:advice id="tx-advice">
    <tx:attributes>
        <tx:method name="*" propagation="REQUIRED"/>
    </tx:attributes>
</tx:advice>
```

`<aop:config>` 和 `<aop:advisor>`元素在[Schema-based AOP Support](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-schema)中讨论。事务元素在[Transaction Management](https://docs.spring.io/spring/docs/current/spring-framework-reference/data-access.html#transaction)中讨论。

##### Examples

Spring AOP用户最有可能使用`execution`切点指示符。`execution`表达式的形式如下：

```
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?name-pattern(param-pattern)
            throws-pattern?)
```

所有的部分除了`ret-type-pattern`，`name-pattern`以及`param-pattern`外都是可选的。`ret-type-pattern`决定了方法的返回类型，以此进行连接点的匹配。`*`是`ret-type-pattern`最常用的值，它表示匹配所有的返回类型。如果指定了一个全限定类名，那么只有方法返回这个类型时才匹配。`name-pattern`匹配方法名称，你可以使用`*`通配符来匹配全部或部分方法。如果你指定了`declaring-type-pattern`，你可以在它后面增加一个`.`来将它加入到`name-pattern`组件中。`param-pattern`稍微复杂一点：`()` 匹配无参方法，而`(..)`匹配任意数量（0个或多个）参数的方法，`(*)`匹配拥有一个任意类型的参数的方法，`(*,String)`匹配拥有两个参数的方法，第一个参数可以是任意类型，但是第二个参数必须要是`String`。查看[Language Semantics](https://www.eclipse.org/aspectj/doc/released/progguide/semantics-pointcuts.html)获取更多信息。

下面的例子展示了一些常见的切点表达式：

*   任何public方法的执行：

    ```
    execution(public * *(..))
    ```

*   名称以`set`开头的方法执行：

    ```
    execution(* set*(..))
    ```

*   在`AccountService`接口中定义的方法执行：

    ```
    execution(* com.xyz.service.AccountService.*(..))
    ```

*   在`service`包中定义的方法执行：

    ```
    execution(* com.xyz.service.*.*(..))
    ```

*   在`service`包或者它的子包中定义的方法执行：

    ```
    execution(* com.xyz.service..*.*(..))
    ```

*   在`service`包中的任何连接点（在Spring AOP中只有方法执行）：
    ```
    within(com.xyz.service.*)
    ```

*   在`service`包及其子包中的任何连接点（在Spring AOP中只有方法执行）：

    ```
    within(com.xyz.service..*)
    ```

*   实现`AccountService`接口的代理中的任何连接点（在Spring AOP中只有方法执行）：

    ```
    this(com.xyz.service.AccountService)
    ```

    > 'this' 一般更多以绑定形式使用。查看 [Declaring Advice](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-advice) 获取如何在增强代码体中获取代理对象。

*   实现`AccountService`接口的目标对象中的任何连接点（在Spring AOP中只有方法执行）：

    ```
    target(com.xyz.service.AccountService)
    ```

    > 'target' 一般更多以绑定形式使用。查看 [Declaring Advice](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-advice) 获取如何在增强代码体中获取目标对象。

*   拥有一个参数并且运行时传入的参数类型为`Serializable`的任何连接点（在Spring AOP中只有方法执行）：

    ```
    args(java.io.Serializable)
    ```

    > 'args' 一般更多以绑定形式使用。查看 [Declaring Advice](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-advice) 获取如何在增强代码体中获取方法参数。
    
    注意这个例子中给定的切点与`execution(* *(java.io.Serializable))`不同。只有运行时传入的参数类型为`Serializable`才匹配args版本，如果方法签名声明了一个类型为`Serializable`的参数时匹配execution版本。

*   拥有`@Transactional`注解的目标对象的任何连接点（在Spring AOP中只有方法执行）：

    ```
    @target(org.springframework.transaction.annotation.Transactional)
    ```

    > 你也可以以绑定形式使用 '@target' ，查看 [Declaring Advice](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-advice) 获取如何在增强代码体中获取注解对象。

*   拥有`@Transactional`注解的类的任何连接点（在Spring AOP中只有方法执行）：

    ```
    @within(org.springframework.transaction.annotation.Transactional)
    ```

    > 你也可以以绑定形式使用 '@within' ，查看 [Declaring Advice](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-advice) 获取如何在增强代码体中获取注解对象。

*   拥有`@Transactional`注解的任何连接点（在Spring AOP中只有方法执行）：

    ```
    @annotation(org.springframework.transaction.annotation.Transactional)
    ```

    > 你也可以以绑定形式使用 '@annotation' ，查看 [Declaring Advice](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-advice) 获取如何在增强代码体中获取注解对象。

*   拥有一个参数并且运行时被传入的参数类型拥有`@Classified`注解的任何连接点（在Spring AOP中只有方法执行）：

    ```
    @args(com.xyz.security.Classified)
    ```

    > 你也可以以绑定形式使用 '@args' ，查看 [Declaring Advice](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-advice) 获取如何在增强代码体中获取注解对象。

*   在名为`tradeService`的Spring bean上的任何连接点（在Spring AOP中只有方法执行）：

    ```
    bean(tradeService)
    ```

*   在名称符合通配符表达式`*Service`的Spring bean上的任何连接点（在Spring AOP中只有方法执行）：
    ```
    bean(*Service)
    ```

##### Writing Good Pointcuts

AspectJ为了优化匹配性能，在编译时对切点进行了处理。检查代码并且决定是否每个连接点匹配（静态或动态）一个给定的切点是一个开销很大的过程（动态匹配意味着匹配无法从静态分析中完全决定，并且在代码运行时在代码中加入一个测试来决定是否有匹配）。当第一次遇到切点声明时，AspectJ会为匹配过程将它重写为一个优化形式。这意味着什么？本质上，切点是以DNF (Disjunctive Normal Form)形式记录的，并且切点的组件被排序，因此先对那些组件执行检查开销更小。这意味着你不需要担心要去理解各种切点指示符的性能，你可以以任何顺序声明切点。

但是，AspectJ只能够完成它被告知的工作。如果要优化匹配性能，你应当考虑正在尝试达到什么效果并且尽量在定义中为匹配缩小查询空间。已存在的指示符可以分为三组：kinded, scoping 和 contextual：

*   Kinded 指示符选择一个特定种类的连接点： `execution`, `get`, `set`, `call` 和 `handler`.

*   Scoping指示符选择一组感兴趣的连接点(可能是很多种)：  `within` and `withincode`

*   Contextual指示符基于上下文匹配（并且可以绑定）： `this`, `target`, and `@annotation`

一个编写精巧的切点应当至少包括前两种类型(kinded 和 scoping)。你可以包括contextual指示符进行基于连接点上下文的匹配或者绑定这个上下文在advice中使用。只提供一个kinded指示符或者只提供一个contextual指示符也可以工作，但是这可能会影响织入性能（时间以及空间开销），因为会进行额外的处理和分析。scoping指示符匹配速度非常快，使用它们意味着AspectJ可以非常快的排除那些不需要进一步处理的连接点，一个设计良好的切点总是应当包括一个scoping指示符。

#### 5.4.4\. Declaring Advice

增强(Advice)与切点表达式关联，并且在切点匹配的方法执行之前，之后以及周围(around)运行。切点表达式或者是对一个命名切点的简单引用，或者是一个正确的声明。

##### Before Advice

你可以使用`@Before`注解在一个切面中声明before advice：

```
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;

@Aspect
public class BeforeExample {

    @Before("com.xyz.myapp.SystemArchitecture.dataAccessOperation()")
    public void doAccessCheck() {
        // ...
    }

}
```

如果我们使用切点表达式，我们可以重写这个例子，如下所示：

```
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;

@Aspect
public class BeforeExample {

    @Before("execution(* com.xyz.myapp.dao.*.*(..))")
    public void doAccessCheck() {
        // ...
    }

}
```

##### After Returning Advice

当一个匹配的方法执行正常返回后，after returning advice执行。你可以使用`@AfterReturning`注解声明它：

```
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.AfterReturning;

@Aspect
public class AfterReturningExample {

    @AfterReturning("com.xyz.myapp.SystemArchitecture.dataAccessOperation()")
    public void doAccessCheck() {
        // ...
    }

}
```

> 你可以在同一个切面中定义多个增强声明（以及其他成员），我们在这些例子中只展示了一个增强声明，以着重显示每一种增强的效果。

有时，你需要在增强方法体内访问被增强方法返回的值，你可以使用绑定返回值的`@AfterReturning`形式来访问，如下所示：

```
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.AfterReturning;

@Aspect
public class AfterReturningExample {

    @AfterReturning(
        pointcut="com.xyz.myapp.SystemArchitecture.dataAccessOperation()",
        returning="retVal")
    public void doAccessCheck(Object retVal) {
        // ...
    }

}
```

`returning`属性中使用的名字必须和增强方法的参数名称对应。当一个方法执行返回时，返回值作为参数传入增强方法中。同时，`returning`字句会只匹配返回特定类型的方法执行（在这里返回值类型为`Object`，也就是匹配任意返回类型的方法）。

请注意当使用after returning advice时，不可能返回一个完全不同的引用。

##### After Throwing Advice

当一个匹配的方法执行抛出异常返回时，执行after throwing advice。你可以使用`@AfterThrowing`注解来声明它，如下所示：

```
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.AfterThrowing;

@Aspect
public class AfterThrowingExample {

    @AfterThrowing("com.xyz.myapp.SystemArchitecture.dataAccessOperation()")
    public void doRecoveryActions() {
        // ...
    }

}
```

一般来说，你会想要只在返回特定类型的异常时才执行增强，并且你需要在增强方法中获取被抛出的异常。你可以使用`throwing`属性，它既可以限制匹配（如果需要的话，可以使用`Throwable`作为异常类型），也可以将被抛出的异常和增强方法的参数绑定在一起。如下所示：

```
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.AfterThrowing;

@Aspect
public class AfterThrowingExample {

    @AfterThrowing(
        pointcut="com.xyz.myapp.SystemArchitecture.dataAccessOperation()",
        throwing="ex")
    public void doRecoveryActions(DataAccessException ex) {
        // ...
    }

}
```

在`throwing`属性中使用的名称必须和增强方法参数的名字相符。当一个方法抛出异常退出，这个异常会被传入到增强方法的参数中。`throwing`字句还会只匹配抛出特定类型异常的方法（在此处是`DataAccessException`）。

##### After (Finally) Advice

当一个匹配的方法退出后，执行after (finally) advice，它可以通过`@After`注解声明。after advice必须能同时处理正常返回和异常返回两种情况，它一般用来释放资源或者有类似的目的。下面展示了如何使用after finally advice：

```
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.After;

@Aspect
public class AfterFinallyExample {

    @After("com.xyz.myapp.SystemArchitecture.dataAccessOperation()")
    public void doReleaseLock() {
        // ...
    }

}
```

##### Around Advice

最后一种增强是around advice，around advice在一个匹配的方法“周围”执行。它既可以同时在方法前后工作，也能够决定什么时候，如何执行。如果你需要在方法执行前后以线程安全的方式共享状态（例如启动和停止一个计时器），那么你可以使用around advice。记住总是使用符合你的需求的通用性最低的增强（也就是如果其他增强足够使用了，那么不要使e用around advice）。

使用`@Around`注解声明around advice。增强方法的第一个参数必须是`ProceedingJoinPoint`，在增强方法的内部，调用`ProceedingJoinPoint`的`proceed`方法来执行底层方法（即被增强的方法）。`proceed`方法也可以传入`Object[]`参数，这个数组的值作为底层方法执行的参数。

> 当使用`Object[]`调用`proceed`方法时，它的行为和通过AspectJ编译器编译的`proceed`的行为有一点不同。使用传统的AspectJ语言编写的around advice，传入`proceed`方法的参数数量必须和传入around advice的参数数量（不是底层连接点使用的参数数量）相同，并且传入`proceed`方法的参数值会按参数位置替换连接点的原始参数值。使用Spring来进行这个过程则更简单，也更加符合基于代理，只执行的语义。如果你使用AspectJ编译器编译切面类，你只需要知道这里面的差异即可。有一种完全兼容Spring AOP和AspectJ的编写切面的方法，在[下一节关于增强参数](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-ataspectj-advice-params)中介绍。

下面的例子展示了如何使用around advice：

```
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.ProceedingJoinPoint;

@Aspect
public class AroundExample {

    @Around("com.xyz.myapp.SystemArchitecture.businessService()")
    public Object doBasicProfiling(ProceedingJoinPoint pjp) throws Throwable {
        // start stopwatch
        Object retVal = pjp.proceed();
        // stop stopwatch
        return retVal;
    }

}
```

around advice的返回值可以被方法的调用者发现。例如，一个简单的缓存切面可能会从缓存中返回一个值，如果缓存中存在这个值那么直接返回，否则调用`proceed()`方法。注意`proceed()`方法可能会被调用一次，多次，或者根本不在around advice的方法体内存在，这些全都是合法的。

##### Advice Parameters

Spring提供了完整的类型化增强，这意味着你可以在增强方法的签名中声明你需要的参数（比如前面的returning和throwing示例），而不是总是使用`Object[]`数组。我们将在这节看到如何在增强方法体内获取参数以及其他上下文值。首先，我们先看一下如何编写可以找到正在被增强的方法的通用增强。

###### Access to the Current `JoinPoint`

任何增强方法都可以将它的第一个参数声明为`org.aspectj.lang.JoinPoint`类型（注意around advice需要将第一个参数声明为`ProceedingJoinPoint`类型，它是`JoinPoint`的子类），`JoinPoint`接口提供了许多有用的方法：

*   `getArgs()`: 返回方法参数

*   `getThis()`: 返回代理对象

*   `getTarget()`: 返回目标对象

*   `getSignature()`: 返回正在被增强的方法的描述符

*   `toString()`: 打印正在被增强的方法的有用的描述符

查看[java文档](https://www.eclipse.org/aspectj/doc/released/runtime-api/org/aspectj/lang/JoinPoint.html)获取更多信息。

###### Passing Parameters to Advice

我们已经看过了如何绑定返回值或者异常（使用after returning advice 和 after throwing advice）。为了在增强方法中获取参数值，你可以使用`args`。如果你在一个`args`表达式中使用一个参数名来代替类型名，那么当这个增强调用时会将这个参数名对应的值作为参数传入到这个增强方法中。使用一个例子可以让这更清晰，假设你想要增强DAO操作的执行，这个操作方法将`Account`对象作为第一个参数，如果你想要在增强方法体中获取这个account，你可以像下面这样编写代码：

```
@Before("com.xyz.myapp.SystemArchitecture.dataAccessOperation() && args(account,..)")
public void validateAccount(Account account) {
    // ...
}
```

切点表达式中的`args(account,..)`有两个作用。第一，它只匹配至少有一个参数的方法，并且传入那个方法的参数是`Account`的实例。第二，它通过`account`参数在增强方法中获取真实的`Account`对象。

编写上面代码的另一种方法是声明一个当匹配连接点时“提供”`Account`对象值的切点，并且在增强方法中引用这个命名切点，如下所示：

```
@Pointcut("com.xyz.myapp.SystemArchitecture.dataAccessOperation() && args(account,..)")
private void accountDataAccessOperation(Account account) {}

@Before("accountDataAccessOperation(account)")
public void validateAccount(Account account) {
    // ...
}
```

查看AspectJ编程指导获取更多细节。

代理对象(`this`)，目标对象(`target`)以及注解(`@within`, `@target`, `@annotation` 和 `@args`)可以以相同的形式绑定。下面两个例子展示了如何使用`@Auditable`注解来匹配方法以及提取audit code：

第一个例子展示了`@Auditable`注解的定义：

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Auditable {
    AuditCode value();
}
```

第二个例子展示了匹配使用`@Auditable`注解的方法的增强：

```
@Before("com.xyz.lib.Pointcuts.anyPublicMethod() && @annotation(auditable)")
public void audit(Auditable auditable) {
    AuditCode code = auditable.value();
    // ...
}
```

###### Advice Parameters and Generics

Spring AOP可以处理类声明和方法参数中的泛型，假设你有一个下面这样的泛型：

```
public interface Sample<T> {
    void sampleGenericMethod(T param);
    void sampleGenericCollectionMethod(Collection<T> param);
}
```

你可以指定某种参数类型来限制方法的拦截，通过指定增强方法的参数类型来表示你想要拦截的方法：

```
@Before("execution(* ..Sample+.sampleGenericMethod(*)) && args(param)")
public void beforeSampleMethod(MyType param) {
    // Advice implementation
}
```

这个方法无法对泛型集合起作用，所以你无法定义下面这样的切点：

```
@Before("execution(* ..Sample+.sampleGenericCollectionMethod(*)) && args(param)")
public void beforeSampleMethod(Collection<MyType> param) {
    // Advice implementation
}
```

为了能让它起作用，我们需要检查集合中的每一个元素，但这不是合乎情理的，因为一般我们无法决定如何对待`null`值。为了达到同样的目的，你需要将参数设为`Collection<?>`并且手动检查元素类型。

###### Determining Argument Names

绑定在增强方法上的参数依赖于切点表达式中使用的名称与增强方法签名上声明的参数名称相匹配，参数名称无法通过Java反射获取，所以Spring AOP使用了下面的策略来决定参数名称：

- 如果参数名称被用户明确指定了，那么使用指定的参数名称。增强和切点注解都拥有一个可选的`argNames`属性，你可以使用它来指定被注解方法的参数名称。这些参数名称可以在运行时获取到，下面的例子展示了如何使用`argNames`属性：

    ```
    @Before(value="com.xyz.lib.Pointcuts.anyPublicMethod() && target(bean) && @annotation(auditable)",
            argNames="bean,auditable")
    public void audit(Object bean, Auditable auditable) {
        AuditCode code = auditable.value();
        // ... use code and bean
    }
    ```
    
    如果第一个参数是`JoinPoint`, `ProceedingJoinPoint` 或者 `JoinPoint.StaticPart`类型，你不需要在`argNames`属性中指定这个参数的名称。例如，如果你修改了前面的增强方法以接受一个`JoinPoint`对象，`argNames`属性不需要包含它：

    ```
    @Before(value="com.xyz.lib.Pointcuts.anyPublicMethod() && target(bean) && @annotation(auditable)",
            argNames="bean,auditable")
    public void audit(JoinPoint jp, Object bean, Auditable auditable) {
        AuditCode code = auditable.value();
        // ... use code, bean, and jp
    }
    ```

    当第一个参数类型是`JoinPoint`, `ProceedingJoinPoint` 和`JoinPoint.StaticPart`时给予它们特殊的对待，这对于那些不需要获取`JoinPoint`上下文的增强实例来说是很方便的。在这种情况下，你可以忽略`argNames`属性。例如，下面的增强方法没有声明`argNames`属性：

    ```
    @Before("com.xyz.lib.Pointcuts.anyPublicMethod()")
    public void audit(JoinPoint jp) {
        // ... use jp
    }
    ```

- 使用`'argNames'`属性有些笨拙，所以如果没有指定`'argNames'`属性，Spring AOP会查询类的debug信息并且尝试从局部变量表中决定参数名称。只要类编译时带有debug信息(至少要指定`'-g:vars'`)那么这个信息就会存在。使用这个标记编译的结果是：(1)你的代码更容易理解（逆向工程）(2）class文件稍微大一点（一般来说是微不足道的）(3)你的编译器不会进行删除未使用局部变量的优化。换句话说，使用这个标记编译时你不会遇到问题。

    > 如果一个@AspectJ切面已经使用AspectJ编译器(ajc)编译了，即使没有debug信息，你也不需要增加`argNames`属性，因为编译器已经保存了必要的信息。

*   如果代码编译时没有debug信息，Spring AOP会尝试推测参数（例如，如果在切点表达式上只有一个变量被绑定，并且增强方法只有一个参数，那么它们的关系就很明显了）。如果根据当前的信息，变量绑定是有歧义的，那么将抛出一个`AmbiguousBindingException`异常。

*   如果前面所有的策略都失败了，那么抛出一个`IllegalArgumentException`异常。

###### Proceeding with Arguments

我们在前面谈到了我们将介绍如何编写一个既兼容Spring AOP也兼容AspectJ的`proceed`方法调用，这个解决方案就是确保增强方法的签名按顺序绑定每一个方法参数。下面的例子展示了如何去做：

```
@Around("execution(List<Account> find*(..)) && " +
        "com.xyz.myapp.SystemArchitecture.inDataAccessLayer() && " +
        "args(accountHolderNamePattern)")
public Object preProcessQueryPattern(ProceedingJoinPoint pjp,
        String accountHolderNamePattern) throws Throwable {
    String newPattern = preProcess(accountHolderNamePattern);
    return pjp.proceed(new Object[] {newPattern});
}
```

在很多情况下，你都可以这样做。

##### Advice Ordering

当多个增强方法都想要在同一个连接点上运行时将会发生什么？Spring AOP遵循和AspectJ一样的优先级规则来决定增强方法执行的顺序。最高优先级的增强方法“在进入时”最先运行（所以，给定两个before advice，优先级最高的那个先运行）。当从连接点“出来”时，最高优先级的增强方法最后运行（所以，给定两个after advice，优先级最高的那个最后运行）。

当定义在两个切面类中的增强都想要在同一个连接点运行时，除了你指定了顺序，否则执行的顺序是未定义的。你可以指定优先级来控制执行的顺序，这可以以Spring自己的方式完成，通过让切面类实现`org.springframework.core.Ordered`接口或者使用`Order`注解。给定两个切面，`Ordered.getValue()`（或者注解值）返回值更小的切面拥有更高的优先级。

当两个增强方法定义在同一个切面中并且它们都需要在同一个连接点上运行时，顺序是未定义的（因为没有办法通过反射提取声明的顺序）。考虑折叠这样的增强方法到一个增强方法中或者重构增强方法到不同的切面类中，以此来对切面定义顺序。

#### 5.4.5\. Introductions

引入/引介（在AspectJ中叫做内部类型声明）可以让被增强的对象实现了一个额外的接口，并且提供一个这个接口的实现类来表示。

你可以使用`@DeclareParents`注解来定义一个引介，这个注解声明了匹配的类型拥有一个新的父类。例如，给定一个叫做`UsageTracked`的接口以及这个接口的实现类`DefaultUsageTracked`，下面的切面声明了所有service包中的类也实现了`UsageTracked`接口：

```
@Aspect
public class UsageTracking {

    @DeclareParents(value="com.xzy.myapp.service.*+", defaultImpl=DefaultUsageTracked.class)
    public static UsageTracked mixin;

    @Before("com.xyz.myapp.SystemArchitecture.businessService() && this(usageTracked)")
    public void recordUsage(UsageTracked usageTracked) {
        usageTracked.incrementUseCount();
    }

}
```

实现的接口由被注解的字段类型(此处为UsageTracked mixin)决定。`@DeclareParents`注解中的`value`属性是一个AspectJ类型模式，任何类型匹配的bean都会实现`UsageTracked`接口。注意，在前面例子中的before advice中，service bean可以直接作为`UsageTracked`接口的实现类使用。如果要硬编码获取bean，你可以像下面这样编写：

```
UsageTracked usageTracked = (UsageTracked) context.getBean("myService");
```

#### 5.4.6\. Aspect Instantiation Models

> 这是一个深入的话题，如果你刚开始接触Spring AOP，你可以先跳过本节以后再来学习。

默认情况下，ApplicationContext中的每一个切面类都只拥有一个实例，AspectJ将它称之为单例实例化模型。可以使用代替的[alternate]生命周期来定义切面。Spring支持AspectJ的`perthis` 和 `pertarget` 实例化模型（`percflow, percflowbelow,` 和 `pertypewithin`现在还不支持）。

你可以在`@Aspect`注解中指定`perthis`子句来声明一个`perthis`切面，考虑下面的例子：

```
@Aspect("perthis(com.xyz.myapp.SystemArchitecture.businessService())")
public class MyAspect {

    private int someState;

    @Before(com.xyz.myapp.SystemArchitecture.businessService())
    public void recordServiceUsage() {
        // ...
    }

}
```

在这个例子中，`'perthis'`子句的作用是为每一个执行业务服务的service对象创建一个切面实例（每一个service对象绑定到通过切点表达式匹配的连接点的`this`上）。当第一次调用service对象的方法时，创建一个切面实例。当service对象离开作用域时（即死亡），这个切面也离开了作用域。在切面实例被创建前，在它之中的增强方法不会执行。只要切面实例被创建了，在它之中声明的增强就会在匹配的连接点上执行，但是只有当service对象和这个切面连接在一起时才执行。查看AspectJ编程指导获取更多信息。

`pertarget`实例化模型的工作方式和`perthis`完全一样，但是它为匹配的连接点上的每一个目标对象创建一个切面实例。

#### 5.4.7\. An AOP Example

你已经看到了AOP所有的组成部分是如何工作的，现在我们可以将它们结合来做一些有用的事。

由于同步问题（例如死锁）业务服务的执行有时候会失败。如果这个操作被重试，它很有可能在下一次尝试成功。对于适合在这种情况下重试的业务服务，我们显然想要重试操作以避免发送给客户端一个`PessimisticLockingFailureException`异常。这是一个在服务层贯穿多个服务类的需求，使用切面来实现是理想的。

因为我们想要重试操作，所以我们需要使用around advice，这样我们可以多次调用`proceed`方法。下面展示了基本的切面实现：

```
@Aspect
public class ConcurrentOperationExecutor implements Ordered {

    private static final int DEFAULT_MAX_RETRIES = 2;

    private int maxRetries = DEFAULT_MAX_RETRIES;
    private int order = 1;

    public void setMaxRetries(int maxRetries) {
        this.maxRetries = maxRetries;
    }

    public int getOrder() {
        return this.order;
    }

    public void setOrder(int order) {
        this.order = order;
    }

    @Around("com.xyz.myapp.SystemArchitecture.businessService()")
    public Object doConcurrentOperation(ProceedingJoinPoint pjp) throws Throwable {
        int numAttempts = 0;
        PessimisticLockingFailureException lockFailureException;
        do {
            numAttempts++;
            try {
                return pjp.proceed();
            }
            catch(PessimisticLockingFailureException ex) {
                lockFailureException = ex;
            }
        } while(numAttempts <= this.maxRetries);
        throw lockFailureException;
    }

}
```

注意切面实现了`Ordered`接口，如此我们可以让这个切面的优先级高于事务增强（每次我们重试时我们都想要一个新的事务）。`maxRetries`和`order`属性都可以通过Spring配置。主要的行为发生在`doConcurrentOperation`环绕增强中，注意，我们将重试逻辑应用到了所有的`businessService()中。我们尝试执行服务操作，如果失败了那么再次尝试，直到用完所有的重试机会。

对应的Spring配置如下所示：

```
<aop:aspectj-autoproxy/>

<bean id="concurrentOperationExecutor" class="com.xyz.myapp.service.impl.ConcurrentOperationExecutor">
    <property name="maxRetries" value="3"/>
    <property name="order" value="100"/>
</bean>
```

为了重定义切面让它只应用到幂等操作中，我们可以定义下面的`Idempotent`注解：

```
@Retention(RetentionPolicy.RUNTIME)
public @interface Idempotent {
    // marker annotation
}
```

然后我们可以将这个注解应用到服务操作的实现上，同时还要重定义切点表达式以匹配`@Idempotent`操作，如下所示：

```
@Around("com.xyz.myapp.SystemArchitecture.businessService() && " +
        "@annotation(com.xyz.myapp.service.Idempotent)")
public Object doConcurrentOperation(ProceedingJoinPoint pjp) throws Throwable {
    ...
}
```

### 5.5\. Schema-based AOP Support

如果你更喜欢基于XML的形式，Spring也提供了支持，你可以使用`aop`命名空间标签来定义切面。当使用@AspectJ时支持完全相同的切点表达式以及增强类型。因此，本节我们关注新的语法并引用上一节([@AspectJ support](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-ataspectj))中的讨论来理解如何编写切点表达式和绑定增强方法参数。

为了使用本节提到的`aop`命名空间标签，你需要引用`spring-aop` schema，它在[XML Schema-based configuration](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#xsd-schemas)中介绍。查看[the AOP schema](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#xsd-schemas-aop)获取如何在`aop`命名空间引用标签。

在你的Spring配置中，所有的切面以及增强元素必须放在`<aop:config>`内（你可以在一个application context中使用多个`<aop:config>`元素）。`<aop:config>`元素包含pointcut，advisor以及aspect元素（注意这些元素必须以此顺序声明）。

> `<aop:config>`风格的配置让Spring[auto-proxying](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-autoproxy)机制的使用变得繁杂，如果你已经通过`BeanNameAutoProxyCreator`或者类似的东西进行自动代理，那么可能会引发问题（例如增强没有被织入）。建议的使用模式是只使用`<aop:config>`或者只使用`AutoProxyCreator`，永远不要混用它们。

#### 5.5.1\. Declaring an Aspect

当使用使用schema支持时，切面是一个常规的Java对象，在你的Spring application context中作为一个bean定义。通过这个对象的字段和方法来获取它的状态和行为，在XML中获取切点和增强信息。

你可以使用`<aop:aspect>`元素来声明一个切面，并且使用`ref`属性来引用协作bean，如下所示：

```
<aop:config>
    <aop:aspect id="myAspect" ref="aBean">
        ...
    </aop:aspect>
</aop:config>

<bean id="aBean" class="...">
    ...
</bean>
```

帮助[back]这个切面的bean（在此处为`aBean`）可以像其他Spring bean一样配置以及进行依赖注入。

#### 5.5.2\. Declaring a Pointcut

你可以在`<aop:config>`元素中定义一个命名的切点，使得切点定义在多个切面和增强器之间共享。

表示在服务层中的任何业务服务类的执行的切点可以如下定义：

```
<aop:config>

    <aop:pointcut id="businessService"
        expression="execution(* com.xyz.myapp.service.*.*(..))"/>

</aop:config>
```

注意切点本身使用了和[@AspectJ support](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-ataspectj)一样的AspectJ切点表达式。如果你使用了基于schema的声明形式，你可以引用在@Aspect类中定义的命名切点。另一种定义上面切点的方式如下所示：

```
<aop:config>

    <aop:pointcut id="businessService"
        expression="com.xyz.myapp.SystemArchitecture.businessService()"/>

</aop:config>
```

假设你有一个[Sharing Common Pointcut Definitions](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-common-pointcuts)中介绍的`SystemArchitecture` 切面。

在切面中声明一个切点和声明一个顶层切点很相似，如下所示：

```
<aop:config>

    <aop:aspect id="myAspect" ref="aBean">

        <aop:pointcut id="businessService"
            expression="execution(* com.xyz.myapp.service.*.*(..))"/>

        ...

    </aop:aspect>

</aop:config>
```

与@AspectJ切面非常相似，使用基于schema定义形式声明的切点也可以采集连接点上下文。例如，下面的切点将`this`对象作为连接点上下文并且将它传入到增强中：

```
<aop:config>

    <aop:aspect id="myAspect" ref="aBean">

        <aop:pointcut id="businessService"
            expression="execution(* com.xyz.myapp.service.*.*(..)) &amp;&amp; this(service)"/>

        <aop:before pointcut-ref="businessService" method="monitor"/>

        ...

    </aop:aspect>

</aop:config>
```

这个增强方法必须能够接受采集到的连接点上下文，通过一个名称相符的参数，如下所示：

```
public void monitor(Object service) {
    ...
}
```

当结合切点子表达式时，`&&`在XML文档中是不好对付的，所以你可以使用`and`, `or`,  `not`关键词来代替`&&`, `||`,  `!`。例如，前面的切点可以这样编写：

```
<aop:config>

    <aop:aspect id="myAspect" ref="aBean">

        <aop:pointcut id="businessService"
            expression="execution(* com.xyz.myapp.service.*.*(..)) and this(service)"/>

        <aop:before pointcut-ref="businessService" method="monitor"/>

        ...
    </aop:aspect>
</aop:config>
```

注意以这种方式定义的切点通过它们的`id`来引用并且不能用来组成更复杂的切点。因此，基于schema支持的命名切点比@AspectJ形式的切点受到了更多的限制。

#### 5.5.3\. Declaring Advice

基于schema的AOP支持使用和@AspectJ一样的五种增强，并且它们拥有完全相同的语义：

##### Before Advice

before advice在匹配的方法执行之前运行，在`<aop:aspect>`中使用`<aop:before>`元素来声明它，如下所示：

```
<aop:aspect id="beforeExample" ref="aBean">

    <aop:before
        pointcut-ref="dataAccessOperation"
        method="doAccessCheck"/>

    ...

</aop:aspect>
```

在这里，`dataAccessOperation`是在顶层(`<aop:config>`)定义的切点的`id`，如果要定义一个内联的切点，使用`pointcut`属性来代替`pointcut-ref`属性，如下所示：

```
<aop:aspect id="beforeExample" ref="aBean">

    <aop:before
        pointcut="execution(* com.xyz.myapp.dao.*.*(..))"
        method="doAccessCheck"/>

    ...

</aop:aspect>
```

正如我们在@AspectJ风格的讨论中提到的，使用命名的切点可以让你的代码可读性更好。

`method`属性标明了提供增强体的方法(`doAccessCheck`)，这个方法必须定义在包含这个增强的切面所引用的bean中(`ref="aBean"`)。在数据访问操作被执行之前（通过切点表达式匹配的方法执行连接点），切面bean的`doAccessCheck`方法被调用。

##### After Returning Advice

当匹配的方法正常完成执行后，运行after returning advice，它以和before advice相同的方式在`<aop:aspect>`中时声明，如下所示：

```
<aop:aspect id="afterReturningExample" ref="aBean">

    <aop:after-returning
        pointcut-ref="dataAccessOperation"
        method="doAccessCheck"/>

    ...

</aop:aspect>
```

在@AspectJ风格中，你可以在增强方法中获取返回值。为了在基于schema的AOP中获取，你需要使用`returning`属性来指定返回值应当传入的参数名，如下所示：

```
<aop:aspect id="afterReturningExample" ref="aBean">

    <aop:after-returning
        pointcut-ref="dataAccessOperation"
        returning="retVal"
        method="doAccessCheck"/>

    ...

</aop:aspect>
```

`doAccessCheck`方法必须声明一个名为`retVal`的参数，它和`@AfterReturning`一样，参数类型会限定连接点的匹配。例如，你可以像下面这样声明一个方法签名：

```
public void doAccessCheck(Object retVal) {...
```

##### After Throwing Advice

当一个匹配方法抛出异常退出执行时，运行after throwing advice。在`<aop:aspect>`中使用`aop:after-throwing`元素来声明它，如下所示：

```
<aop:aspect id="afterThrowingExample" ref="aBean">

    <aop:after-throwing
        pointcut-ref="dataAccessOperation"
        method="doRecoveryActions"/>

    ...

</aop:aspect>
```

和@AspectJ风格一眼，你可以在增强方法中获取抛出的异常。你可以使用`throwing`属性来指定异常应当传入的参数名称，如下所示：

```
<aop:aspect id="afterThrowingExample" ref="aBean">

    <aop:after-throwing
        pointcut-ref="dataAccessOperation"
        throwing="dataAccessEx"
        method="doRecoveryActions"/>

    ...

</aop:aspect>
```

`doRecoveryActions`方法必须声明一个名为`dataAccessEx`的参数，它和`@AfterThrowing`一样，参数类型可以限定连接点的匹配。例如，可以像下面这样声明方法签名：

```
public void doRecoveryActions(DataAccessException dataAccessEx) {...
```

##### After (Finally) Advice

不管匹配的方法如何退出，都会执行after (finally) advice。你可以使用`aop:after`元素来声明它，如下所示：

```
<aop:aspect id="afterFinallyExample" ref="aBean">

    <aop:after
        pointcut-ref="dataAccessOperation"
        method="doReleaseLock"/>

    ...

</aop:aspect>
```

##### Around Advice

最后一种增强是around advice。环绕增强在一个匹配的方法“周围”运行，它可以选择在这个方法前后都执行，以及决定何时，如何执行。如果你需要在方法执行前后以线程安全的方式共享状态（例如启动和停止一个计时器），那么你可以使用around advice。记住总是使用符合你的需求的通用性最低的增强（也就是如果其他增强足够使用了，那么不要使用around advice）。

你可以使用`aop:around`元素来声明around advice。增强方法的第一个参数必须是`ProceedingJoinPoint`，在增强方法的内部，调用`ProceedingJoinPoint`的`proceed`方法来执行底层方法。`proceed`方法也可以传入`Object[]`参数，这个数组的值作为方法执行的参数，查看[Around Advice](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-ataspectj-around-advice)获取更多信息。下面展示了如何在XML中声明around advice：

```
<aop:aspect id="aroundExample" ref="aBean">

    <aop:around
        pointcut-ref="businessService"
        method="doBasicProfiling"/>

    ...

</aop:aspect>
```

`doBasicProfiling`增强的实现与@AspectJ示例完全相同（当然要去除注释），如下所示：

```
public Object doBasicProfiling(ProceedingJoinPoint pjp) throws Throwable {
    // start stopwatch
    Object retVal = pjp.proceed();
    // stop stopwatch
    return retVal;
}
```

##### Advice Parameters

基于schema的声明形式和@AspectJ一样支持完全的类型化增强 —— 通过增强方法的参数名称来匹配切点参数。查看[Advice Parameters](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-ataspectj-advice-params)获取更多细节。如果你想要为增强方法明确指定参数名称（而不是依赖于前面提到的检测策略），你可以使用`arg-names` 属性来完成，它和注解形式的`argNames`属性（查看[Determining Argument Names](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-ataspectj-advice-params-names)）一样。下面的例子展示了如何在XML中指定参数名称：

```
<aop:before
    pointcut="com.xyz.lib.Pointcuts.anyPublicMethod() and @annotation(auditable)"
    method="audit"
    arg-names="auditable"/>
```

`arg-names`属性接受以逗号分隔的参数名称列表。

下面的例子展示了与多个强类型参数协作的around advice：

```
package x.y.service;

public interface PersonService {

    Person getPerson(String personName, int age);
}

public class DefaultFooService implements FooService {

    public Person getPerson(String name, int age) {
        return new Person(name, age);
    }
}
```

下面的是切面类。注意`profile(..)`方法接受了多个强类型参数，第一个参数是用来执行方法调用的ProceedingJoinPoint。这个参数的存在表示`profile(..)`是作为环绕增强使用的，如下所示：

```
package x.y;

import org.aspectj.lang.ProceedingJoinPoint;
import org.springframework.util.StopWatch;

public class SimpleProfiler {

    public Object profile(ProceedingJoinPoint call, String name, int age) throws Throwable {
        StopWatch clock = new StopWatch("Profiling for '" + name + "' and '" + age + "'");
        try {
            clock.start(call.toShortString());
            return call.proceed();
        } finally {
            clock.stop();
            System.out.println(clock.prettyPrint());
        }
    }
}
```

最后，下面展示了相应的XML配置：

```
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop https://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- this is the object that will be proxied by Spring's AOP infrastructure -->
    <bean id="personService" class="x.y.service.DefaultPersonService"/>

    <!-- this is the actual advice itself -->
    <bean id="profiler" class="x.y.SimpleProfiler"/>

    <aop:config>
        <aop:aspect ref="profiler">

            <aop:pointcut id="theExecutionOfSomePersonServiceMethod"
                expression="execution(* x.y.service.PersonService.getPerson(String,int))
                and args(name, age)"/>

            <aop:around pointcut-ref="theExecutionOfSomePersonServiceMethod"
                method="profile"/>

        </aop:aspect>
    </aop:config>

</beans>
```

考虑下面的驱动脚本：

```
import org.springframework.beans.factory.BeanFactory;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import x.y.service.PersonService;

public final class Boot {

    public static void main(final String[] args) throws Exception {
        BeanFactory ctx = new ClassPathXmlApplicationContext("x/y/plain.xml");
        PersonService person = (PersonService) ctx.getBean("personService");
        person.getPerson("Pengo", 12);
    }
}
```

运行这个Boot类，我们可以在标准输出得到下面这样的输出结果：

```
StopWatch 'Profiling for 'Pengo' and '12'': running time (millis) = 0
-----------------------------------------
ms     %     Task name
-----------------------------------------
00000  ?  execution(getFoo)
```

##### Advice Ordering

当有多个增强需要在同一个连接点执行时，执行的顺序规则在[Advice Ordering](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-ataspectj-advice-ordering)中查看。切面之间的优先级可以通过实现`Ordered`接口或者增加`Order`注解来决定。

#### 5.5.4\. Introductions

引入/引介（在AspectJ中叫做内部类型声明）可以让切面声明实现给定接口的增强对象，并且提供一个那个接口的实现来代表那些对象。

你可以在`aop:aspect`中使用`aop:declare-parents`元素来定义一个引介，你可以用`aop:declare-parents`声明匹配的类型拥有一个新的父类。例如，给定一个叫做`UsageTracked`的接口和这个接口的实现`DefaultUsageTracked`，下面的切面声明了所有service接口的实现类也实现了`UsageTracked`接口：

```
<aop:aspect id="usageTrackerAspect" ref="usageTracking">

    <aop:declare-parents
        types-matching="com.xzy.myapp.service.*+"
        implement-interface="com.xyz.myapp.service.tracking.UsageTracked"
        default-impl="com.xyz.myapp.service.tracking.DefaultUsageTracked"/>

    <aop:before
        pointcut="com.xyz.myapp.SystemArchitecture.businessService()
            and this(usageTracked)"
            method="recordUsage"/>

</aop:aspect>
```

`usageTracking` bean的类应当包含下面这样的方法：

```
public void recordUsage(UsageTracked usageTracked) {
    usageTracked.incrementUseCount();
}
```

要被实现的接口通过`implement-interface`属性决定，`types-matching`属性的值是一个AspectJ类型模式。任何类型匹配的bean都会实现`UsageTracked`接口。注意，在示例中的before advice中，service bean可以直接作为`UsageTracked`接口的实现类使用。如果想要硬编码获取一个bean，可以像下面这样编写：

```
UsageTracked usageTracked = (UsageTracked) context.getBean("myService");
```

#### 5.5.5\. Aspect Instantiation Models

schema定义的切面唯一支持的实例化模型是单例模型，其他实例化模型可能会在未来支持。

#### 5.5.6\. Advisors

“advisors”的概念来自于定义在Spring中的AOP支持，并且在AspectJ中没有一个直接对应的概念。一个advisor就像一个小的自我包含的切面，它只拥有一个增强。这个增强本身通过一个bean来表示并且必须实现[Advice Types in Spring](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-api-advice-types)中介绍的advice接口之一。Advisor可以利用AspectJ切点表达式。

Spring使用`<aop:advisor>`元素来支持advisor的概念，你一般会看到它和事务增强协作使用，事务在Spring中也有自己的命名空间。下面的例子展示了一个advisor：

```
<aop:config>

    <aop:pointcut id="businessService"
        expression="execution(* com.xyz.myapp.service.*.*(..))"/>

    <aop:advisor
        pointcut-ref="businessService"
        advice-ref="tx-advice"/>

</aop:config>

<tx:advice id="tx-advice">
    <tx:attributes>
        <tx:method name="*" propagation="REQUIRED"/>
    </tx:attributes>
</tx:advice>
```

与前面例子中使用的`pointcut-ref`属性一样，你也可以使用`pointcut`属性来定义一个内联的切点表达式。

为了定义advisor的优先级使得advice可以参与排序，你可以使用`Ordered`注解的`order`属性。

#### 5.5.7\. An AOP Schema Example

本节展示了[An AOP Example](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-ataspectj-example)中的同步锁失败重试示例，并使用schema支持将它重写。

由于同步问题（例如死锁）业务服务的执行有时候会失败。如果这个操作被重试，它很有可能在下一次尝试成功。对于适合在这种情况下重试的业务服务，我们显然想要重试操作以避免客户端发现`PessimisticLockingFailureException`异常。这是一个明显在服务层贯穿多个服务的需求，使用切面来实现是理想的。

因为我们想要重试操作，所以我们需要使用around advice，如此以至于我们可以多次调用`proceed`方法。下面展示了基础的切面实现（使用schema支持的常规Java类）：

```
public class ConcurrentOperationExecutor implements Ordered {

    private static final int DEFAULT_MAX_RETRIES = 2;

    private int maxRetries = DEFAULT_MAX_RETRIES;
    private int order = 1;

    public void setMaxRetries(int maxRetries) {
        this.maxRetries = maxRetries;
    }

    public int getOrder() {
        return this.order;
    }

    public void setOrder(int order) {
        this.order = order;
    }

    public Object doConcurrentOperation(ProceedingJoinPoint pjp) throws Throwable {
        int numAttempts = 0;
        PessimisticLockingFailureException lockFailureException;
        do {
            numAttempts++;
            try {
                return pjp.proceed();
            }
            catch(PessimisticLockingFailureException ex) {
                lockFailureException = ex;
            }
        } while(numAttempts <= this.maxRetries);
        throw lockFailureException;
    }

}
```

注意切面实现了`Ordered`接口，如此我们可以让这个切面的优先级高于事务增强（每次我们重试时我们都想要一个新的事务）。`maxRetries`和`order`属性都可以通过Spring配置。主要的行为发生在`doConcurrentOperation`环绕增强中，注意，我们将重试逻辑应用到了所有的`businessService()中。我们尝试执行服务操作，如果失败了那么再次尝试，直到用完所有的重试机会。

> 这个类与@AspectJ示例中使用的相同，但是去除了注解。

对应的Spring配置如下所示：

```
<aop:config>

    <aop:aspect id="concurrentOperationRetry" ref="concurrentOperationExecutor">

        <aop:pointcut id="idempotentOperation"
            expression="execution(* com.xyz.myapp.service.*.*(..))"/>

        <aop:around
            pointcut-ref="idempotentOperation"
            method="doConcurrentOperation"/>

    </aop:aspect>

</aop:config>

<bean id="concurrentOperationExecutor"
    class="com.xyz.myapp.service.impl.ConcurrentOperationExecutor">
        <property name="maxRetries" value="3"/>
        <property name="order" value="100"/>
</bean>
```

注意，我们假设所有业务服务都是幂等的。如果事实并非如此，我们可以改进切面使得它只重试幂等操作，通过引入一个`@Idempotent`注解并且使用这个注解来标记服务操作的实现方法，如下所示：

```
@Retention(RetentionPolicy.RUNTIME)
public @interface Idempotent {
    // marker annotation
}
```

当改变了切面使它只重试幂等操作时，还需要改进切点表达式使它只匹配`@Idempotent`注解，如下所示：

```
<aop:pointcut id="idempotentOperation"
        expression="execution(* com.xyz.myapp.service.*.*(..)) and
        @annotation(com.xyz.myapp.service.Idempotent)"/>
```

### 5.6\. Choosing which AOP Declaration Style to Use

一旦你决定了切面是实现你的需求的最好方法，那么你如何决定使用Spring AOP还是AspectJ，以及使用AspectJ语言风格，@AspectJ注解风格还是Spring XML风格呢？这受很多的因素影响，包括应用需求，开发组件以及团队对AOP的熟悉程度。

#### 5.6.1\. Spring AOP or Full AspectJ?

使用可以工作的最简单的方式。Spring AOP比完整的AspectJ更简单，因为不需要引入AspectJ编译器/织入到你的开发过程中。如果你只需要对Spring bean中的操作进行增强，Spring AOP是正确的选择。如果你需要增强不被Spring容器管理的对象（例如domain对象），那么你需要使用AspectJ。如果你想要增强简单的方法执行以外的连接点（例如字段获取和设置连接点），那么你也需要使用AspectJ。

当你使用AspectJ时，你可以选择使用AspectJ语言语法（也叫做代码样式[code style]）或者@AspectJ注解语法。如果你不使用Java 5及以上的版本，那么你可以选择使用代码样式。如果切面在你的设计中扮演了重要的角色，并且你能够使用Eclipse的[AspectJ Development Tools (AJDT)](https://www.eclipse.org/ajdt/)插件，那么AspectJ语言语法是最好的选择。它是最简洁也是最简单的，因为这个语言主要就是设计用来编写切面的。如果你不使用Eclipse或者只需要使用很少的切面，它们在你的应用中并不是主要的，那么你可以考虑使用@AspectJ注解，你可以继续你的IDE上编程常规的Java，并且在你的构造脚本(build script)中增加一个切面织入即可。

#### 5.6.2\. @AspectJ or XML for Spring AOP?

如果你选择使用Spring AOP，你可以选择使用@AspectJ或者XML风格，这有很多权衡取舍需要考虑。

对于Spring用户来说一般更熟悉XML风格，并且它可以通过POJO来进行帮助。当将AOP作为一个组件来配置业务服务时，XML是一个好的选择（一个好的测试就是你是否考虑切点表达式作为配置的一部分，并且你想要独立修改它）。当使用XML风格时，从你的配置中可以清楚地看出切面存在于你的系统中。

XML风格有两个缺点 。第一，它不能在一个地方完全概括(encapsulate)需求的实现。[DRY原则](https://zh.wikipedia.org/wiki/%E4%B8%80%E6%AC%A1%E4%B8%94%E4%BB%85%E4%B8%80%E6%AC%A1)规定了系统中的每一部分，都必须有一个单一的、明确的、权威的代表。当使用XML时，需求的实现分隔在了bean class和配置文件的XML元素中。当你使用@AspectJ时，这些信息被封装到了一个模块中：切面。第二，XML比起@AspectJ多了一点限制：它只支持单例实例化模型，并且它无法在XML中组合命名切点。例如，使用@AspectJ你可以编写下面这样的切点：

```
@Pointcut("execution(* get*())")
public void propertyAccess() {}

@Pointcut("execution(org.xyz.Account+ *(..))")
public void operationReturningAnAccount() {}

@Pointcut("propertyAccess() && operationReturningAnAccount()")
public void accountPropertyAccess() {}
```

而在XML中你只能声明前两个切点：

```
<aop:pointcut id="propertyAccess"
        expression="execution(* get*())"/>

<aop:pointcut id="operationReturningAnAccount"
        expression="execution(org.xyz.Account+ *(..))"/>
```

XML的缺点就是你不能组合这两个切点定义来定义`accountPropertyAccess`切点。

@AspectJ支持其他的实例化模型以及更丰富的切点组合。它可以将切面定义在一个模块单元中，它还有一个好处：@AspectJ切面不仅可以被Spring AOP还可以被AspectJ所理解。所以，如果你在以后需要AspectJ的功能来实现额外的需求，那么你可以很容易将代码迁移到AspectJ中。总的来说，Spring团队更喜欢使用@AspectJ来定义切面，除了简单的业务服务配置。

### 5.7\. Mixing Aspect Types

可以在相同的配置中使用auto-proxying支持，schema定义的`<aop:aspect>`切面，`<aop:advisor>`声明的advisor甚至其他风格的代理和拦截器和@AspectJ混用。所有这些实现使用了相同的底层机制并且可以同时存在。

### 5.8\. Proxying Mechanisms

Spring AOP使用JDK动态代理或者CGLIB来为一个给定的目标对象创建代理。JDK动态代理是JDK内建的，而CGLIB是一个开源的类定义库（它被包装到了`spring-core`）中。

如果要被代理的对象实现了一个或多个接口，那么会使用JDK动态代理，所有被目标对象实现的接口都会被代理。如果目标对象没有实现接口，那么使用CGLIB来创建代理。

如果你想要强制使用CGLIB代理（例如，代理定义在目标对象中的每一个方法，而不只是接口实现的方法），你可以做到。但是，你应当考虑下面的问题：

- 使用CGLIB时，`final`方法无法被增强，因为它们无法在运行时产生的子类中被重写。

- 从Spring 4.0开始，你的代理对象的构造函数不再会被调用两次，因为CGLIB代理实例通过Objenesis创建。只有当你的JVM不允许越过构造方法时，你才会看到两次调用，并且Spring AOP支持会打印相应的debug信息。

为了强制使用CGLIB代理，需要将`<aop:config>`元素的`proxy-target-class`属性设置为true，如下所示：

```
<aop:config proxy-target-class="true">
    <!-- other beans defined here... -->
</aop:config>
```

当你使用@AspectJ自动代理支持来强制CGLIB代理时，需要设置`<aop:aspectj-autoproxy>`元素的`proxy-target-class`属性为true，如下所示：

```
<aop:aspectj-autoproxy proxy-target-class="true"/>
```

> 多个`<aop:config/>`元素在运行时会被折叠成一个统一的自动代理创建器，它会应用所有`<aop:config/>`中（一般来自不同的XML文件）*最强的*代理设置。这对于`<tx:annotation-driven/>` 和 `<aop:aspectj-autoproxy/>`元素来说也是一样的。<br/>
要清楚，在`<tx:annotation-driven/>`, `<aop:aspectj-autoproxy/>` 或 `<aop:config/>`元素中使用`proxy-target-class="true"`会让*它们三个都*使用CGLIB代理。

#### 5.8.1\. Understanding AOP Proxies

Spring AOP是基于代理的。在你编写自己的切面类或者使用任何Spring框架提供的基于Spring AOP的切面类之前理解它的语义是极其重要的。

考虑第一个场景，你有一个标准的，未被代理的，没有任何特殊的对象引用，如下所示：

```
public class SimplePojo implements Pojo {

    public void foo() {
        // this next method invocation is a direct call on the 'this' reference
        this.bar();
    }

    public void bar() {
        // some logic...
    }
}
```

如果你在它的对象引用上调用一个方法，这个方法会直接在这个对象引用上调用，如下面的图片所示：

![aop proxy plain pojo call](http://upload-images.jianshu.io/upload_images/13068256-97f307247ad2418a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
public class Main {

    public static void main(String[] args) {
        Pojo pojo = new SimplePojo();
        // this is a direct method call on the 'pojo' reference
        pojo.foo();
    }
}
```

当这个引用拥有一个代理时会发生轻微的改变，考虑下面的图以及代码片段：

![aop proxy call](http://upload-images.jianshu.io/upload_images/13068256-ec4e0b1d608ad3e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
public class Main {

    public static void main(String[] args) {
        ProxyFactory factory = new ProxyFactory(new SimplePojo());
        factory.addInterface(Pojo.class);
        factory.addAdvice(new RetryAdvice());

        Pojo pojo = (Pojo) factory.getProxy();
        // this is a method call on the proxy!
        pojo.foo();
    }
}
```

要理解的核心事物是在类`Main`中的`main(..)`方法中的客户端代码拥有一个代理的引用，这意味着在这个对象引用上调用方法会在代理上调用。因此，代理会将所有相关的方法调用委托给拦截器（增强）。但是，一旦调用最终到达了目标对象（此时是`SimplePojo`引用），那么方法会在它自己上调用，例如`this.bar()` 或 `this.foo()`会在`this`引用上被调用，而不是在代理上。这有很重要的影响，它意味着自我调用不会导致相关联的增强被执行。

所以我们改如何处理呢？最好的方法是重构你的代码，不要出现自我调用。这需要你做出一些工作，但是它是最好的，侵入性最低的方法。第二种方法绝对是令人难以容忍的，我们对于指出它很犹豫，因为它是非常难以容忍的。你可以（这点对我们来说是痛苦的）在你的类中将下面的逻辑绑定到Spring AOP种，如下所示：

```
public class SimplePojo implements Pojo {

    public void foo() {
        // this works, but... gah!
        ((Pojo) AopContext.currentProxy()).bar();
    }

    public void bar() {
        // some logic...
    }
}
```

这会让你的代码完全和Spring AOP耦合，并且它让类自己知道它正在一个AOP上下文中使用。当代理被创建时还需要一些额外的配置，如下所示：

```
public class Main {

    public static void main(String[] args) {
        ProxyFactory factory = new ProxyFactory(new SimplePojo());
        factory.adddInterface(Pojo.class);
        factory.addAdvice(new RetryAdvice());
        factory.setExposeProxy(true);

        Pojo pojo = (Pojo) factory.getProxy();
        // this is a method call on the proxy!
        pojo.foo();
    }
}
```

最后，必须指出AspectJ没有自我调用的问题，因为它不是一个基于代理的AOP框架。

### 5.9\. Programmatic Creation of @AspectJ Proxies

除了在你的配置中使用`<aop:config>`或者`<aop:aspectj-autoproxy>`来声明切面外，你还可以硬编码创建增强目标对象的代理。关于Spring AOP API的全部细节，查看[下一章](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-api)。在这里，我们想要关注使用@AspectJ切面来自动创建代理的能力。

你可以使用`org.springframework.aop.aspectj.annotation.AspectJProxyFactory`类来为目标对象创建一个代理，通过使用一个或多个@AspectJ切面类来增强。这个类的基本使用是非常简单的，如下所示：

```
// create a factory that can generate a proxy for the given target object
AspectJProxyFactory factory = new AspectJProxyFactory(targetObject);

// add an aspect, the class must be an @AspectJ aspect
// you can call this as many times as you need with different aspects
factory.addAspect(SecurityManager.class);

// you can also add existing aspect instances, the type of the object supplied must be an @AspectJ aspect
factory.addAspect(usageTracker);

// now get the proxy object...
MyInterfaceType proxy = factory.getProxy();
```

查看[javadoc](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframework/aop/aspectj/annotation/AspectJProxyFactory.html) 获取更多信息。

### 5.10\. Using AspectJ with Spring Applications

到现在为止我们介绍的所有东西都是纯粹的Spring AOP。在本节，我们关注如何使用AspectJ编译器和织入来代替Spring AOP，如果你需要使用Spring AOP之外提供的功能。

Spring使用了一个小的AspectJ切面库，它可以在`spring-aspects.jar`中独立存在。如果你想要使用它里面的切面类，你需要将它增加到你的classpath中。[Using AspectJ to Dependency Inject Domain Objects with Spring](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-atconfigurable) 和 [Other Spring aspects for AspectJ](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-ajlib-other)讨论了这个库的内容以及你如何使用它。[Configuring AspectJ Aspects by Using Spring IoC](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-aj-configure)讨论了如何依赖注入使用AspectJ编译器织入的切面类。最后，[Load-time Weaving with AspectJ in the Spring Framework](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-aj-ltw)为使用AspectJ的Spring应用提供了装载时(load-time)织入。

#### 5.10.1\. Using AspectJ to Dependency Inject Domain Objects with Spring

Spring容器会实例化和配置在你的application context中定义的bean。你可以请求bean工厂来配置一个已经存在的对象，给定包含配置的bean定义的名字即可。`spring-aspects.jar`包含了注解驱动的切面类，它使用这个功能来依赖注入任何对象。这个支持可以用来创建不属于容器控制的对象。Domain对象一般应用这个策略，因为它一般使用`new`操作符硬编码创建或者使用一个ORM组件作为数据库查询的结果。

`@Configurable`注解标记一个类是Spring驱动的配置。最简单的使用就是将它纯粹作为一个标记注解，如下所示：

```
package com.xyz.myapp.domain;

import org.springframework.beans.factory.annotation.Configurable;

@Configurable
public class Account {
    // ...
}
```

当作为一个标记接口使用时，Spring会配置一个被注解类型（在这里是`Account`）的新实例，通过使用与这个类的全限定名(`com.xyz.myapp.domain.Account`)相同名字的bean定义（一般是原型作用域）。因为这个bean的默认名字是它的类型的全限定名，声明这个原型定义的方便方法就是可以不指定`id`属性，如下所示：

```
<bean class="com.xyz.myapp.domain.Account" scope="prototype">
    <property name="fundsTransferService" ref="fundsTransferService"/>
</bean>
```

如果你想要明确指定这个原型bean的名字，你可以直接在注解中指定，如下所示：

```
package com.xyz.myapp.domain;

import org.springframework.beans.factory.annotation.Configurable;

@Configurable("account")
public class Account {
    // ...
}
```

Spring会寻找名为`account`的bean定义并且使用它作为定义来配置新的`Account`实例。

你可以使用自动注入来避免指定一个bean定义。为了让Spring应用自动注入，你需要使用`@Configurable`注解的`autowire`属性。你可以指定 `@Configurable(autowire=Autowire.BY_TYPE)` 或 `@Configurable(autowire=Autowire.BY_NAME`，分别表示通过类型自动注入和通过名称自动注入。作为代替，在你的`@Configurable` bean的字段或者方法上使用`@Autowired`或`@Inject`注解来指定明确的注解驱动的依赖注入更加好（查看 [Annotation-based Container Configuration](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-annotation-config)获取更多信息）。

最后，你可以使用`dependencyCheck`属性在新创建和配置的对象上启用Spring依赖检查（例如 `@Configurable(autowire=Autowire.BY_NAME,dependencyCheck=true)`）。如果这个属性设置为`true`，Spring会在所有的属性（除了基本类型和集合）配置完之后验证它们是否已经被设置了。

注意使用这个注解就其本身而言没有任何意义，而是`spring-aspects.jar`库中的`AnnotationBeanConfigurerAspect`类对这个注解进行操作。实质上，使用`@Configurable`注解的类型在它的新对象初始化返回后，才根据这个注解的属性使用Spring配置这个新实例化的对象。在此上下文中，初始化表示一个新实例化的对象（例如，使用`new`操作符实例化的对象）或者经历了反序列化的`Serializable`对象（例如，通过[readResolve()](https://docs.oracle.com/javase/8/docs/api/java/io/Serializable.html)方法）。

>  前一段中的一个关键词是“实质上”。对于大多数情况，“在新对象初始化返回后”的精确语义是考究的(fine)。在这个上下文中，“初始化之后”意味着依赖在这个对象构造后被注入，这意味着依赖无法在类的构造方法中获取。如果你想要在构造方法执行之前注入依赖并且因此可以在构造方法中获取依赖，你需要定义`@Configurable`声明，如下所示：

```
@Configurable(preConstruction = true)
```

> 你可以在 [AspectJ Programming Guide](https://www.eclipse.org/aspectj/doc/next/progguide/index.html) 中的 [附录](https://www.eclipse.org/aspectj/doc/next/progguide/semantics-joinPoints.html) 中获取关于各种切点类型的语义的更多信息。

为了让它工作，被注解的类必须使用AspectJ织入器来织入。你可以使用build-time Ant或者Maven(查看[AspectJ Development Environment Guide](https://www.eclipse.org/aspectj/doc/released/devguide/antTasks.html))或者加载时织入(查看 [Load-time Weaving with AspectJ in the Spring Framework](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-aj-ltw))来完成。`AnnotationBeanConfigurerAspect`本身需要使用Spring类配置（为了获取bean工厂的引用，并使用这个引用来配置新对象）。如果你是用基于Java的配置，你可以在`@Configuration`类上增加`@EnableSpringConfigured`注解，如下所示：

```
@Configuration
@EnableSpringConfigured
public class AppConfig {
}
```

如果你更喜欢基于XML的配置，Spring的[`context` 命名空间](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#xsd-schemas-context)定义了一个方便使用的`context:spring-configured`元素，如下所示：

```
<context:spring-configured/>
```

`@Configurable`对象实例在切面被配置之前创建的话，会产生一个发给debug日志的信息并且这个对象的配置不会使用。一个例子是在Spring配置中的bean在初始化的时候创建domain对象。在这种情况下，你可以使用`depends-on`属性来手动指定这个bean依赖于切面类。下面的例子展示了如何使用`depends-on`属性：

```
<bean id="myService"
        class="com.xzy.myapp.service.MyService"
        depends-on="org.springframework.beans.factory.aspectj.AnnotationBeanConfigurerAspect">

    <!-- ... -->

</bean>
```

> 不要通过bean configurer aspect 来激活`@Configurable`，除非你真的打算在运行时依赖它的语义。特别的，确保你不要在一个常规bean的类上使用`@Configurable`注解，这样做会导致两次初始化，一次通过容器一次通过切面类。

##### Unit Testing `@Configurable` Objects

@Configurable支持的一个目标是可以对domain对象进行独立的单元测试，无需使用大量臃肿的属性寻找代码。如果@Configurable类型没有使用AspectJ织入，这个注解在单元测试时没有任何作用，你可以在测试中像往常一样设置对象的属性。如果`@Configurable`类使用AspectJ织入了，你仍然可以像往常一样在容器外进行单元测试，但是每当你创建一个`@Configurable`对象时你都会看到一条警告信息，表示它还没有被Spring所配置。

##### Working with Multiple Application Contexts

`AnnotationBeanConfigurerAspect`用来实现对`@Configurable`的支持，它是一个AspectJ单例切面。单例切面的作用域与`static`成员一样：每一个定义这个类型的类加载器只有一个切面实例。这意味着，如果你在同一个类加载器继承体系中定义了多个application context，你需要考虑在哪里定义`@EnableSpringConfigured`以及将`spring-aspects.jar`放在哪个classpath中。

考虑一个标准的Spring web应用配置，它有一个共享的定义了常规业务服务类和支持服务类的所有类的父application context，并且每一个servlet都有一个子application context（包含了那个servlet特有的定义）。这些context在同一个类加载器体系中共存，并且`AnnotationBeanConfigurerAspect`只可以持有它们中的一个的引用。在这种情况下，我们建议在共享的(父)application context上定义`@EnableSpringConfigured`，因为它定义了你可能想要注入到domain对象中的service。不过你无法将定义在子(servlet特有的)context中的bean注入到@Configurable对象中(一般来说这不是你想要做的)。

当在同一个容器中部署多个web应用时，确保每一个web应用使用它自己的类加载器来加载`spring-aspects.jar`中的类（例如，将`spring-aspects.jar`放在`'WEB-INF/lib'`中）。如果`spring-aspects.jar`被增加到容器级的classpath中（因此使用共享的父类加载器来加载），那么所有的web应用将会共享同一个切面实例（这可能不是你想要的）。

#### 5.10.2\. Other Spring aspects for AspectJ

除了`@Configurable`切面外，`spring-aspects.jar`还包含了一个AspectJ切面类，你可以使用它来进行Spring事务管理，通过在方法上使用`@Transactional`注解。这主要面向想要在Spring容器外使用Spring框架的事务支持的用户。

拦截`@Transactional`注解的切面类是`AnnotationTransactionAspect`。当你使用这个切面时，你必须在实现类（或者这个类的方法）上注解，而不是在这个类实现的接口上。AspectJ遵循Java的规则，在接口上使用的注解不会被继承。

在类上使用`@Transactional`注解会为这个类的所有public操作执行指定了默认的事务语义。

在方法上使用`@Transactional`注解会覆盖类注解的默认事务语义（如果类也指定了`@Transactional`）。任何可见性的方法都可以被注解，包括private方法。直接对非public方法注解是对这类方法进行事务划分的唯一办法。

> 从Spring 4.2开始，`spring-aspects`提供了一个和标准`javax.transaction.Transactional`注解功能完全相同的切面类，查看`JtaAnnotationTransactionAspect`获取更多信息。

对于想要使用Spring配置和事务管理支持但不想（或不能）使用注解的AspectJ开发者，`spring-aspects.jar`也包含了`abstract`切面类，你可以继承它来提供你自己的切面定义。查看`AbstractBeanConfigurerAspect`和`AbstractTransactionAspect`切面类的源代码获取更多信息。下面的片段展示了如何编写一个切面类来配置所有使用prototype bean定义的domain类：

```
public aspect DomainObjectConfiguration extends AbstractBeanConfigurerAspect {

    public DomainObjectConfiguration() {
        setBeanWiringInfoResolver(new ClassNameBeanWiringInfoResolver());
    }

    // the creation of a new bean (any object in the domain model)
    protected pointcut beanCreation(Object beanInstance) :
        initialization(new(..)) &&
        SystemArchitecture.inDomainModel() &&
        this(beanInstance);
}
```

#### 5.10.3\. Configuring AspectJ Aspects by Using Spring IoC

当你在Spring应用中使用AspectJ切面时，会想要能够在Spring中配置这类切面。AspectJ runtime负责切面创建，并且使用Spring配置AspectJ创建的切面，这依赖于那个切面使用的AspectJ的实例化模型(`per-xxx`子句)。

AspectJ切面主要都是单例切面，配置这类切面是很简单的。你可以像往常一样引用切面类型来创建一个bean定义，并且设置`factory-method="aspectOf"`属性。这确保了Spring请求AspectJ来获取切面实例而不是自己去创建一个实例。如下所示：

```
<bean id="profiler" class="com.xyz.profiler.Profiler"
        factory-method="aspectOf"> 

    <property name="profilingStrategy" ref="jamonProfilingStrategy"/>
</bean>
```

非单例的切面很难配置。但是，可以创建一个原型bean定义并且使用`spring-aspects.jar`库中的`@Configurable`在它们被AspectJ创建的时候配置切面实例。

如果你有一些想要使用AspectJ织入（例如，对domian对象使用加载时织入）的@AspectJ切面，还有一些想要使用Spring AOP的@AspectJ切面，并且这些切面都在Spring中配置，你需要告诉Spring AOP @AspectJ自动代理器哪些@AspectJ切面类应该使用自动代理，可以在`<aop:aspectj-autoproxy/>`中使用一个或多个`<include/>`元素来告诉它。每一个`<include/>`元素指定一个名称模式，并且只有名字与至少一个模式相匹配的bean才会用作Spring AOP自动代理。如下所示：

```
<aop:aspectj-autoproxy>
    <aop:include name="thisBean"/>
    <aop:include name="thatBean"/>
</aop:aspectj-autoproxy>
```

> 不要被`<aop:aspectj-autoproxy/>`元素的名字所误导。使用它会创建Spring AOP代理，@AspectJ风格的切面在这里声明，但是AspectJ runtime没有参与其中。

#### 5.10.4\. Load-time Weaving with AspectJ in the Spring Framework

Load-time weaving (LTW)将织入AspectJ切面的过程放在了应用的class文件中，当它们被加载到JVM的时候。本节的关注点是在特定的context中配置以及使用LTW。本节并不是对LTW的笼统介绍，对于LTW的完整信息以及如何只用AspectJ配置LTW(Spring不参与其中)，查看[LTW section of the AspectJ Development Environment Guide](https://www.eclipse.org/aspectj/doc/released/devguide/ltw.html)。

Spring框架对AspectJ LTW的改进是在织入过程中确保更细粒度的控制。'Vanilla' AspectJ LTW在使用Java (5+) agent时会受到影响，当启动JVM时可以指定一个VM参数来打开它。因此，JVM层面的设置在某些情况下可能会很好，但是一般来说都太粗粒度了。Spring LTW可以让你在每一个`ClassLoader`上打开LTW，这是更细粒度的并且在"单JVM多应用"环境下（例如标准的应用服务器环境）有更大的意义。

更深入的，[在某些环境下](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-aj-ltw-environments)，这个支持确保启用LWT不会对应用服务器的启动脚本做出任何改变，比如增加`-javaagent:path/to/aspectjweaver.jar`或者（我们在本节最后讨论）`-javaagent:path/to/spring-instrument.jar`。开发者可以配置application context来启用LTW，而不是依赖于控制部署配置的管理员，例如启动脚本。

让我们首先查看一个使用Spring的AspectJ LTW简单例子，查看[Petclinic sample application](https://github.com/spring-projects/spring-petclinic)获取完整的例子。
Now that the sales pitch is over, let us first walk through a quick example of AspectJ LTW that uses Spring, followed by detailed specifics about elements introduced in the example. For a complete example, see the [Petclinic sample application](https://github.com/spring-projects/spring-petclinic).

##### A First Example

假设你是一个被分配来诊断系统性能问题的应用开发者，首先我们打算使用一个简单的能让我们快速获取到性能指标性能剖析切面类，然后我们在特定的区域应用这个细粒度的性能剖析工具。

> 此处的例子使用了XML配置，你也可以使用[Java configuration](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-java)来配置@AspectJ。你可以使用 `@EnableLoadTimeWeaving`注解来代替`<context:load-time-weaver/>`，查看[下面](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-aj-ltw-spring)获取更多信息。

下面的例子展示了一个性能剖析切面，它是一个基于时间的性能剖析器，使用了@AspectJ风格的切面声明：

```
package foo;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.util.StopWatch;
import org.springframework.core.annotation.Order;

@Aspect
public class ProfilingAspect {

    @Around("methodsToBeProfiled()")
    public Object profile(ProceedingJoinPoint pjp) throws Throwable {
        StopWatch sw = new StopWatch(getClass().getSimpleName());
        try {
            sw.start(pjp.getSignature().getName());
            return pjp.proceed();
        } finally {
            sw.stop();
            System.out.println(sw.prettyPrint());
        }
    }

    @Pointcut("execution(public * foo..*.*(..))")
    public void methodsToBeProfiled(){}
}
```

我们还需要创建一个`META-INF/aop.xml`文件，来告知AspectJ weaver我们想要将`ProfilingAspect`织入到我们的类中。下面展示了`aop.xml`文件：

```
<!DOCTYPE aspectj PUBLIC "-//AspectJ//DTD//EN" "https://www.eclipse.org/aspectj/dtd/aspectj.dtd">
<aspectj>

    <weaver>
        <!-- only weave classes in our application-specific packages -->
        <include within="foo.*"/>
    </weaver>

    <aspects>
        <!-- weave in just this aspect -->
        <aspect name="foo.ProfilingAspect"/>
    </aspects>

</aspectj>
```

现在我们看一下Spring的配置部分，我们需要配置一个`LoadTimeWeaver` (在后面解释)。这个LTW是一个必不可少的组件，它负责将一个或多个`META-INF/aop.xml`文件中的切面配置织入到应用类中。一件好的事情是它不需要很多配置（你可以指定更多选项，在后面介绍），如下所示：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <!-- a service object; we will be profiling its methods -->
    <bean id="entitlementCalculationService"
            class="foo.StubEntitlementCalculationService"/>

    <!-- this switches on the load-time weaving -->
    <context:load-time-weaver/>
</beans>
```

现在所有必需的工具（切面类，`META-INF/aop.xml`文件以及Spring配置）都出现了，我们可以在`main(..)`方法中创建下面的驱动类来展示LTW的活动：

```
package foo;

import org.springframework.context.support.ClassPathXmlApplicationContext;

public final class Main {

    public static void main(String[] args) {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml", Main.class);

        EntitlementCalculationService entitlementCalculationService =
                (EntitlementCalculationService) ctx.getBean("entitlementCalculationService");

        // the profiling aspect is 'woven' around this method execution
        entitlementCalculationService.calculateEntitlement();
    }
}
```

我们还有最后一件事要做。本节的介绍说过可以在每个`ClassLoader`上启动LTW，并且这是正确的。但是，对于这个例子，我们使用Java agent(Spring提供的)来打开LTW。我们使用下面的命令来运行`Main`类：

```
java -javaagent:C:/projects/foo/lib/global/spring-instrument.jar foo.Main
```

`-javaagent`是指定以及启用[agents to instrument programs that run on the JVM](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)的标志。Spring框架使用了一个这样的agent，`InstrumentationSavingAgent`，它被封装在`spring-instrument.jar`库中。

`Main`程序的执行输出和下面的差不多。（我在`calculateEntitlement()`实现中加入了一条`Thread.sleep(..)`语句，因此性能剖析器可以获取一个结果而不是0ms，`01234`ms不是AOP的开销）。下面展示了运行性能剖析器时的输出：

```
Calculating entitlement

StopWatch 'ProfilingAspect': running time (millis) = 1234
------ ----- ----------------------------
ms     %     Task name
------ ----- ----------------------------
01234  100%  calculateEntitlement
```

因为LTW使用了成熟的AspectJ框架，我们没有被限制只能增强Spring bean。下面的例子运行结果相同：

```
package foo;

import org.springframework.context.support.ClassPathXmlApplicationContext;

public final class Main {

    public static void main(String[] args) {
        new ClassPathXmlApplicationContext("beans.xml", Main.class);

        EntitlementCalculationService entitlementCalculationService =
                new StubEntitlementCalculationService();

        // the profiling aspect will be 'woven' around this method execution
        entitlementCalculationService.calculateEntitlement();
    }
}
```

注意，在前面的程序中，我们启动了Spring容器并且在Spring上下文外创建了`StubEntitlementCalculationService`的一个新实例，性能剖析增强仍然被织入了。

诚然，这个例子是简单的。但是，Spring对LTW的基本支持在前面的例子中都出现了，本节剩余部分解释了背后的细节。

> 示例中使用的`ProfilingAspect`可能是基础的，但是却是相当有用的。开发者可以在开发的时候使用它检测性能，并且当应用被部署到UAT或者生产环境中时可以很容易排除它。

##### Aspects

你在LTW中使用的切面必须要是AspectJ切面。你要么使用AspectJ语言本身来编写切面类，要么使用@AspectJ注解来编写切面类。你的切面类可以同时是合法的AspectJ和Spring AOP切面。更深入的，被编译的切面类需要能够在classpath中获取到。

##### 'META-INF/aop.xml'

AspectJ LTW基础设施可以使用在Java classpath（直接存在，或者一般来说在jar中国）中的一个或多个`META-INF/aop.xml`文件来配置。

这个文件的结构以及内容可以在[AspectJ reference documentation](https://www.eclipse.org/aspectj/doc/released/devguide/ltw-configuration.html)中查看，因为`aop.xml`文件是一个100%的AspectJ，所以我们不在这里作更深入的讨论。

##### Required libraries (JARS)

你需要下面的库来使用Spring框架对AspectJ LTW的支持：

*   `spring-aop.jar`
*   `aspectjweaver.jar`

如果你使用[Spring-provided agent to enable instrumentation](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-aj-ltw-environment-generic)，你还需要：

*   `spring-instrument.jar`

##### Spring Configuration

Spring LTW的核心组件是`LoadTimeWeaver`接口（它在`org.springframework.instrument.classloading`包中），并且Spring的发行版中存在几个它的实现类。`LoadTimeWeaver`负责在运行时向`ClassLoader`增加一个或多个`java.lang.instrument.ClassFileTransformers`，它对所有感兴趣的应用打开了大门，其中之一恰好是切面LTW。

> 如果你对运行时的class文件转换不熟悉，在查看下面的内容之前先看一下`java.lang.instrument`包的API文档。虽然这个文档不是详尽的，但是至少你可以看到核心的接口以及类。

为一个特定的`ApplicationContext`配置一个`LoadTimeWeaver`就像增加一行代码那样容易。（注意你需要使用一个`ApplicationContext`作为你的Spring容器 —— 一般来说，`BeanFactory`不足以使用，因为LTW支持使用`BeanFactoryPostProcessors`来实现）。

为了启用Spring框架的LTW支持，你需要配置一个`LoadTimeWeaver`，这一般可以使用`@EnableLoadTimeWeaving`注解来完成，如下所示：

```
@Configuration
@EnableLoadTimeWeaving
public class AppConfig {
}
```

如果你更喜欢基于XML的配置，你可以使用`<context:load-time-weaver/>`元素，注意这个元素定义在`context`命名空间中。下面的例子展示了如何使用`<context:load-time-weaver/>`：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:load-time-weaver/>

</beans>
```

前面的配置自动定义以及注册多个LTW指定的基础设施bean，例如`LoadTimeWeaver`和`AspectJWeavingEnabler`。默认的`LoadTimeWeaver`实现是`DefaultContextLoadTimeWeaver`类，它尝试装饰一个被自动检测的`LoadTimeWeaver`。`LoadTimeWeaver`的准确类型依赖于你的运行时环境，它可以被自动检测。下面的表总结了各种`LoadTimeWeaver`的实现类：

| Runtime Environment | `LoadTimeWeaver` implementation |
| --- | --- |
| 在[Apache Tomcat](https://tomcat.apache.org/)中运行 | `TomcatLoadTimeWeaver` |
| 在[GlassFish](https://glassfish.dev.java.net/) (限定EAR部署)中运行 | `GlassFishLoadTimeWeaver` |
| 在Red Hat的 [JBoss AS](https://www.jboss.org/jbossas/) or [WildFly](https://www.wildfly.org/)上运行 | `JBossLoadTimeWeaver` |
| 在IBM的 [WebSphere](https://www-01.ibm.com/software/webservers/appserv/was/)上运行 | `WebSphereLoadTimeWeaver` |
| 在Oracle的 [WebLogic](https://www.oracle.com/technetwork/middleware/weblogic/overview/index-085209.html)上运行 | `WebLogicLoadTimeWeaver` |
| 使用Spring `InstrumentationSavingAgent` (`java -javaagent:path/to/spring-instrument.jar`)启动JVM | `InstrumentationLoadTimeWeaver` |
| 回调，底层的ClassLoader遵循惯例（命名的`addTransformer`和一个可选的`getThrowawayClassLoader`方法） | `ReflectiveLoadTimeWeaver` |

注意这个表只列出了当你使用`DefaultContextLoadTimeWeaver`时可以被自动检测的`LoadTimeWeavers`，你也可以明确指定使用哪一个`LoadTimeWeaver`实现类。

使用Java配置明确指定一个`LoadTimeWeaver`时，需要实现`LoadTimeWeavingConfigurer`接口并且重写它的`getLoadTimeWeaver()`方法，下面的例子指定了`ReflectiveLoadTimeWeaver`：

```
@Configuration
@EnableLoadTimeWeaving
public class AppConfig implements LoadTimeWeavingConfigurer {

    @Override
    public LoadTimeWeaver getLoadTimeWeaver() {
        return new ReflectiveLoadTimeWeaver();
    }
}
```

如果你使用基于XML的配置，你可以指定一个全限定类名作为`<context:load-time-weaver/>`元素`weaver-class`属性的值。同样，下面的例子指定了`ReflectiveLoadTimeWeaver`：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:load-time-weaver
            weaver-class="org.springframework.instrument.classloading.ReflectiveLoadTimeWeaver"/>

</beans>
```

通过配置定义以及注册的`LoadTimeWeaver`可以在之后从Spring容器中提取出来，它的bean名称为`loadTimeWeaver`。记住`LoadTimeWeaver`只作为Spring LTW基础设施的机制存在，用来增加一个或多个`ClassFileTransformers`。进行LTW行为的`ClassFileTransformers`实际上是`ClassPreProcessorAgentAdapter`类（在`org.aspectj.weaver.loadtime`包中）。查看`ClassPreProcessorAgentAdapter`类的文档获取更多信息，因为这超出了本文的范围。

这个配置还有最后一个属性需要讨论： `aspectjWeaving`属性（当你使用XML时为`aspectj-weaving`）。这个属性控制LTW是否启用。它接受可以接受三个值，默认值为`autodetect`。下标总结了这三个值：

| Annotation Value | XML Value | Explanation |
| --- | --- | --- |
| `ENABLED` | `on` | 启用AspectJ织入，并且切面在合适的加载时间织入 |
| `DISABLED` | `off` | 关闭LTW。在加载时没有切面会被织入 |
| `AUTODETECT` | `autodetect` | 如果Spring LTW可以找打至少一个 `META-INF/aop.xml`文件，那么启用AspectJ织入，否则关闭它。这个值是默认值If the  |

##### Environment-specific Configuration

最后一节包含了你可能需要的额外设置以及配置，当你在例如应用服务器和web容器这样的环境下使用Spring的LTW时。

###### Tomcat, JBoss, WebSphere, WebLogic

Tomcat, JBoss/WildFly, IBM WebSphere应用服务器以及Oracle WebLogic服务器都提供了一个app `ClassLoader`，Spring LTW会利用那些类加载器来提供AspectJ织入。你可以简单的启用加载时织入，正如[前面介绍的](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-using-aspectj)。特别的，你不需要修改JVM启动脚本来增加`-javaagent:path/to/spring-instrument.jar`。

注意在JBoss中，你需要关闭app server scanning来阻止它在应用真正启动前加载类。一个快的应变方法是向你的artifact中加入一个名为`WEB-INF/jboss-scanning.xml`的文件，此文件包含了下面的内容：

```
<scanning xmlns="urn:jboss:scanning:1.0"/>
```

###### Generic Java Applications

当在一个不支持特定`LoadTimeWeaver`实现类的环境下需要类设施时，JVM agent是通用的解决方案。对于这种情况，Spring提供了`InstrumentationLoadTimeWeaver`类，它需要Spring特定的（但是非常通用）JVM agent, `spring-instrument.jar`，通过`@EnableLoadTimeWeaving` 和 `<context:load-time-weaver/>`设置来自动检测。

为了使用它，你必须使用Spring agent启动虚拟机并提供下面的JVM选项：

```
-javaagent:/path/to/spring-instrument.jar
```

注意它需要修改JVM启动脚本，这可能会在应用服务器环境下不可用（依赖于你的服务器和操作原则）。也就是说，对于one-app-per-JVM部署的应用例如独立的Spring Boot应用，你一般可以随意控制整个JVM设置。

### 5.11\. Further Resources

关于AspectJ的更多信息可以在[AspectJ website](https://www.eclipse.org/aspectj)中找到。

*Eclipse AspectJ* [Adrian Colyer et. al. (Addison-Wesley, 2005)] 提供了对AspectJ语言的详尽介绍。

*AspectJ in Action* 第二版 [Ramnivas Laddad (Manning, 2009)] 非常推荐，这本书着重于AspectJ，但是其中也包括了许多通用的AOP主题。

### 2.1\. Introduction

Java虽然有标准的`java.net.URL`类和处理各种URL前缀的标准处理器，但是不幸的是，这对获取底层资源来说还不足够使用。例如，没有用来从classpath或者`ServletContext`中获取资源的标准化`URL`实现。虽然可以为专门的`URL`前缀注册一个处理器（与已存在的处理器相似，例如`http:`），但是一般来说相当复杂，并且`URL`接口还缺少了一些必要的功能，例如检查资源是否存在。

### 2.2\. The Resource Interface

Spring提供了一个`Resource`接口，目的在于为获取底层资源提供一个功能性更强的接口抽象。下面展示了`Resource`接口的定义：

```
public interface Resource extends InputStreamSource {

    boolean exists();

    boolean isOpen();

    URL getURL() throws IOException;

    File getFile() throws IOException;

    Resource createRelative(String relativePath) throws IOException;

    String getFilename();

    String getDescription();

}
```

正如`Resource`接口的定义所示，它继承了`InputStreamSource`接口。下面是`InputStreamSource`接口的定义：

```
public interface InputStreamSource {

    InputStream getInputStream() throws IOException;

}
```

`Resource`接口最重要的方法描述如下：

*   `getInputStream()`: 定位以及打开资源，返回读取这个资源的`InputStream`。每一次调用都返回一个最新的`InputStream`，调用者负责关闭这个输入流。
*   `exists()`: 返回一个`boolean`，表示这个资源是否在物理层面上真实存在。
*   `isOpen()`: 返回一个`boolean`，表示这个资源是否代表一个open stream的句柄。如果为`true`，那么`InputStream`不能被多次读取，只能读取一次并且需要关闭以避免资源泄漏。所有普通的Resource实现默认都返回`false`。
*   `getDescription()`: 返回这个资源的描述符，当使用这个resource时用作错误输出。它一般是这个resource的URL的全限定名。

其他一些方法可以让你获取一个`URL` 或 `File`对象表示这个资源（如果底层实现兼容并且支持这个功能）。 

Spring本身也广泛的使用了`Resource`抽象，在许多方法的签名中作为参数类型。一些Spring APIs中的方法（例如各种`ApplicationContext`实现类的构造方法）可以接受未修饰的`String`路径，用来一个创建合适的`Resource`，或者在`String`路径上增加一个特殊的前缀，让调用者指定创建的`Resource`实现类型。

虽然`Resource`接口在Spring中大量使用，但是实际上在你的代码中它也可以作为一个非常有用的工具类用来获取资源，此时你的代码可以不关心Spring的其他部分。虽然这会让你的代码和Spring耦合，但是它只和这一小部分工具类耦合，来作为`URL`的一个功能性更强的代替品，等同于使用其他库。

> `Resource`抽象没有在功能性上进行替换，它对`URL`进行了包装。例如，`UrlResource`包装了一个URL并且使用这个被包装的`UrlResource`来完成工作。

### 2.3\. Built-in Resource Implementations

Spring包含了下面的`Resource`实现类：

*   [`UrlResource`](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#resources-implementations-urlresource)
*   [`ClassPathResource`](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#resources-implementations-classpathresource)
*   [`FileSystemResource`](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#resources-implementations-filesystemresource)
*   [`ServletContextResource`](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#resources-implementations-servletcontextresource)
*   [`InputStreamResource`](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#resources-implementations-inputstreamresource)
*   [`ByteArrayResource`](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#resources-implementations-bytearrayresource)

#### 2.3.1. UrlResource

`UrlResource`包装了`java.net.URL`，并且可以用来获取任何可以从URL访问的对象，例如文件，一个HTTP target，一个FTP target或者其他。所有的URL有一个标准化的`String`表示，合适的标准化前缀被用来表示一个URL类型：`file:`表示获取文件系统，`http:`表示从HTTP协议获取资源，`ftp:`表示从FTP获取资源，以及其他类型。

`UrlResource`可以通过Java代码来明确调用`UrlResource`的构造方法进行创建，但是当你使用一个可以表示url路径的`String`参数来调用API方法时也可以隐式创建。对于后一种方法，Spring内部有一个JavaBean `PropertyEditor`会最终决定创建哪一种类型的`Resource`。如果一个路径字符串包含了知名的前缀（例如`classpath:`），那么就为这个前缀创建一个专门的`Resource`。但是，如果无法识别这个前缀，它会假设这个字符串是一个标准的URL字符串并且创建一个`UrlResource`。

```
// jdk 9及以上版本才可以使用transfer()方法
@Test
public void testUrlResource() throws Exception {
    UrlResource resource = new UrlResource("https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#context-functionality-resources");
    try (InputStream stream = resource.getInputStream()) {
        stream.transferTo(System.out);
    }
}
```
输出片段如下：
```
explicitly declared callback
method. In the following example, the cache is pre-populated upon initialization and
cleared upon destruction:</p>
</div>
<div class="exampleblock">
<div class="content">
<div class="listingblock">
<div class="content">
......
```

#### 2.3.2. ClassPathResource

这个类表示应当从classpath中获取资源。它或者使用一个线程上下文的classloader，或者一个给定的classloader，或者一个给定的加载资源的类。

这个`Resource`实现支持将资源解析为`java.io.File`，如果这个classpath资源处于文件系统中而不是处于一个还没拓展[expand]到文件系统的jar（通过servlet引擎）中。为了能定位资源，各种`Resource`实现类都支持将资源解析为一个`java.net.URL`。

`ClassPathResource`可以通过调用`ClassPathResource`的构造方法来创建，但是当你使用一个可以表示为classpath路径的`String`参数来调用API方法时也可以隐式创建。对于后一种方法，一个JavaBean `PropertyEditor`会识别这个特殊的前缀`classpath:`，此时创建一个`ClassPathResource`。

```
@Test
public void testClassPathResource() throws Exception {
    ClassPathResource resource = new ClassPathResource("spring-config.xml");
    if(resource.isFile()) {
        File file = resource.getFile();
        System.out.println(file.getName());
    }

    try (InputStream stream = resource.getInputStream()) {
        stream.transferTo(System.out);
    }
}
```
输出片段如下：
```
spring-config.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans 
            http://www.springframework.org/schema/beans/spring-beans.xsd 
            http://www.springframework.org/schema/aop 
            http://www.springframework.org/schema/aop/spring-aop.xsd 
            http://www.springframework.org/schema/context 
            http://www.springframework.org/schema/context/spring-context.xsd">
    <!-- test for 1.4.1. Dependency Injection#Dependency Resolution Process#Circular dependencies -->
    <!--
    <bean id="beanA" class="com.shallowinggg.ioc1.entity.CircularBeanA">
        <constructor-arg name="beanB" ref="beanB" />
    </bean>

......
```

#### 2.3.3. FileSystemResource

这是为`java.io.File` 和 `java.nio.file.Path`句柄使用实现的`Resource`子类，它支持将资源解析为`File` 和 `URL`。

```
@Test
public void testFileSystemResource() throws Exception {
    FileSystemResource resource = new FileSystemResource("F:/ftpshare/1.txt");
    try (InputStream stream = resource.getInputStream()) {
        stream.transferTo(System.out);
    }
}
```

#### 2.3.4. ServletContextResource

这是为`ServletContext`资源实现的`Resource`子类，在相关的web应用根目录下解析相对路径。

它支持流获取以及URL获取，但是当web应用存档被伸展并且资源物理存在于文件系统上时只允许`java.io.File`获取。是否被伸展到文件系统或者直接从JAR中获取或者像一个数据库（这是可能的）依赖于Servlet容器。

#### 2.3.5. InputStreamResource

`InputStreamResource`是为一个给定的`InputStream`实现的`Resource`。它应当只在没有合适的`Resource`实现类时才使用。特别的，如果可能的话尽量使用`ByteArrayResource`或者其他基于文件的`Resource`实现。

与其他`Resource`实现不同，它是一个已经打开的资源的描述符。因此，当调用`isOpen()`方式时会返回`true`。如果你需要在某处保存资源描述符或者你需要多次读取一个流，那么不要使用它。

#### 2.3.6. ByteArrayResource

这是为一个给定的字节数组实现的`Resource`，它为一个给定的字节数组创建一个`ByteArrayInputStream`。

当从字节数组中加载内容并且不采用一个单次使用的`InputStreamResource`时，可以使用它。

### 2.4\. The ResourceLoader

`ResourceLoader`接口的作用是返回`Resource`实例。下面展示了`ResourceLoader`接口的定义：

```
public interface ResourceLoader {

    Resource getResource(String location);

}
```

所有的application context都实现了`ResourceLoader`接口。因此，所有的pplication context都可以用来获取`Resource`实例。

当你在一个特定的application context上调用`getResource()`方法，并且位置路径字符串没有一个特定的前缀时，你得到的`Resource`类型依赖这个特定的application context。例如，假定下面的代码片段在`ClassPathXmlApplicationContext`实例中执行：

```
Resource template = ctx.getResource("some/resource/path/myTemplate.txt");
```

由于application context的实际类型是`ClassPathXmlApplicationContext`，这段代码将会返回`ClassPathResource`。如果使用`FileSystemXmlApplicationContext`实例执行相同的代码，那么将会返回一个`FileSystemResource`。对于`WebApplicationContext`，它将返回`ServletContextResource`。对于每一个context返回合适的对象。

因此，你可以从特定的application context中以合适的形式加载资源。

另一方面，你也可以强制使用`ClassPathResource`，不管application context的类型是什么，通过指定`classpath:`前缀即可，如下所示：

```
Resource template = ctx.getResource("classpath:some/resource/path/myTemplate.txt");
```

与此相似，你可以通过指定标准的`java.net.URL`前缀来强制使用`UrlResource`，下面的例子使用了`file` 和 `http`前缀：

```
Resource template = ctx.getResource("file:///some/resource/path/myTemplate.txt");
```

```
Resource template = ctx.getResource("https://myhost.com/resource/path/myTemplate.txt");
```

下面的表总结了将`String`对象转换为`Resource`对象的策略：

| 前缀 | 示例 | 说明 |
| --- | --- | --- |
| classpath: | `classpath:com/myapp/config.xml` | 从classpath中加载 |
| file: | `file:///data/config.xml` | 从文件系统中加载，作为一个`URL`。查看[`FileSystemResource`附加说明](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#resources-filesystemresource-caveats)获取更多信息。 |
| http: | `https://myserver/logo.png` | 加载为一个`URL` |
| (none) | `/data/config.xml` | 依赖于底层的`ApplicationContext`类型 |

```
@Test
public void testResourceLoader() throws Exception {
    ApplicationContext context = new ClassPathXmlApplicationContext();
    System.out.println("-------------------file-------------------");
    Resource resource = context.getResource("file:F:/ftpshare/1.txt");
    try (InputStream stream = resource.getInputStream()) {
        stream.transferTo(System.out);
    }

    System.out.println("\n-------------------url-------------------");
    resource = context.getResource("https://www.baidu.com/");
    try (InputStream stream = resource.getInputStream()) {
        stream.transferTo(System.out);
    }

    System.out.println("\n-------------------classpath-------------------");
    resource = context.getResource("spring-config.xml");
    try (InputStream stream = resource.getInputStream()) {
        stream.transferTo(System.out);
    }

    System.out.println("\n");
    context = new FileSystemXmlApplicationContext();
    resource = context.getResource("F:/ftpshare/1.txt");
    try (InputStream stream = resource.getInputStream()) {
        stream.transferTo(System.out);
    }
}
```

### 2.5\. The `ResourceLoaderAware` interface

`ResourceLoaderAware`接口是一个特殊的回调接口，用来标识希望被提供一个`ResourceLoader`引用的元件。下面展示了`ResourceLoaderAware`接口的定义：

```
public interface ResourceLoaderAware {

    void setResourceLoader(ResourceLoader resourceLoader);
}
```

当一个类实现了`ResourceLoaderAware`接口并且被部署到一个application context时（作为一个Spring管理的bean），它会被application context识别为一个`ResourceLoaderAware`。然后application context就会调用`setResourceLoader(ResourceLoader)`方法，将它自己作为参数提供给这个方法（记住，所有Spring中的application context都实现了`ResourceLoader`接口）。

因为`ApplicationContext`是一个`ResourceLoader`，所以bean也可以实现`ApplicationContextAware`接口并且使用被提供的application context来直接加载资源。但是，一般来说，如果你需要的话最好使用特定的`ResourceLoader`接口。这样你的代码只会和资源加载接口（可以视作是一个工具接口）耦合而不是和整个Spring `ApplicationContext` 接口耦合。

在应用组件中，你也可以依赖自动注入`ResourceLoader`来代替实现`ResourceLoaderAware`接口。传统的`constructor`和`byType`自动注入模型（在[Autowiring Collaborators](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-factory-autowire)中查看更多信息）都可以提供一个`ResourceLoader`作为构造方法或者setter方法的参数。如果想要更多的灵活性（包括自动注入字段以及多参方法的功能），可以考虑使用基于注解的自动注入特性。在这种情况下，只要字段，构造方法或者方法带有`@Autowired`注解，那么`ResourceLoader`就会被注入到类型为`ResourceLoader`的字段或参数中，查看[Using `@Autowired`](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-autowired-annotation)获取更多信息。

```
public class MyResourceLoaderAwareImpl implements ResourceLoaderAware {
    private ResourceLoader resourceLoader;

    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }

    public ResourceLoader getResourceLoader() {
        return resourceLoader;
    }
}
```
```
@Bean
public MyResourceLoaderAwareImpl resourceLoaderAware(ResourceLoader loader) {
    MyResourceLoaderAwareImpl aware = new MyResourceLoaderAwareImpl();
    aware.setResourceLoader(loader);
    return aware;
}
```
```
@Test
public void testResourceLoaderAware() {
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(SpringConfig.class);
    MyResourceLoaderAwareImpl aware = context.getBean(MyResourceLoaderAwareImpl.class);
    System.out.println(context);
    System.out.println(aware.getResourceLoader());
    Resource resource = aware.getResourceLoader().getResource("file:///F:/ftpshare/1.txt");
    try (InputStream stream = resource.getInputStream()) {
        stream.transferTo(System.out);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```
输出片段如下：
```
org.springframework.context.annotation.AnnotationConfigApplicationContext@659a969b, started on Wed Apr 10 11:56:55 CST 2019
org.springframework.context.annotation.AnnotationConfigApplicationContext@659a969b, started on Wed Apr 10 11:56:55 CST 2019
......
```
从中可以发现application context将自己作为参数注入到MyResourceLoaderAwareImpl的resourceLoader属性中。当然，你也可以使用`@Autowired`注解进行自动注入。

### 2.6\. Resources as Dependencies

如果bean本身打算通过某种动态过程来决定以及提供资源路径，那么使用`ResourceLoader`接口来加载资源是可行的。例如，考虑某种模板的加载，需要的资源依赖于用户的职责。不过如果资源是静态的，那么可以完全排除`ResourceLoader`接口的使用，让bean暴露一个`Resource`属性，并且将资源注入到这个属性中即可。

由于所有的application context注册以及使用一个特殊的JavaBean `PropertyEditor`，它可以将`String`路径转换为`Resource`对象，所以注入这种属性是很简单的。因此，如果`myBean`拥有一个`Resource`类型的属性`template`，那么只需要使用一个简单的字符串来配置这个属性就行了，如下所示：

```
<bean id="myBean" class="...">
    <property name="template" value="some/resource/path/myTemplate.txt"/>
</bean>
```

注意资源路径没有指定前缀。因为application context本身可以作为`ResourceLoader`使用，所以资源可以通过`ClassPathResource`, `FileSystemResource` 或者 `ServletContextResource`加载，具体则依赖context的类型。

如果你需要强制指定使用哪种类型的`Resource`，那么你可以使用一个前缀，。下面的例子展示了如何强制使用`ClassPathResource` 和 `UrlResource`（第二个用来获取文件系统中的文件）：

```
<property name="template" value="classpath:some/resource/path/myTemplate.txt">
```

```
<property name="template" value="file:///some/resource/path/myTemplate.txt"/>
```

### 2.7 Application Contexts and Resource Paths

本节介绍了如何使用资源来创建application context，包括使用XML的捷径，使用通配符以及其他细节。

#### 2.7.1\. Constructing Application Contexts

application context 构造方法一般可以接受一个字符串数组作为资源位置路径，例如组成bean定义的XML配置文件。

当位置路径没有指定前缀时，从路径中构建的`Resource`类型依赖于application context的实际类型。例如，考虑一个创建`ClassPathXmlApplicationContext`的例子：

```
ApplicationContext ctx = new ClassPathXmlApplicationContext("conf/appContext.xml");
```

它将从classpath中加载，因为使用了`ClassPathResource`类。但是，考虑一个创建`FileSystemXmlApplicationContext`的例子：

```
ApplicationContext ctx = new FileSystemXmlApplicationContext("conf/appContext.xml");
```

现在它从文件系统的某个位置（此时相对于当前工作目录）中加载。

注意，可以使用一个特定的前缀（例如classpath或者一个标准的URL前缀）来覆盖默认用来加载定义的`Resource`类型，考虑下面的例子：

```
ApplicationContext ctx = new FileSystemXmlApplicationContext("classpath:conf/appContext.xml");
```

它使用`FileSystemXmlApplicationContext`从classpath中加载bean定义。但是，它还是一个`FileSystemXmlApplicationContext`。如果后面将它作为`ResourceLoader`使用，所有没有指定前缀的路径仍然会被作为文件系统路径对待。

##### Constructing `ClassPathXmlApplicationContext` Instances — Shortcuts

为了能够进行方便的实例化，`ClassPathXmlApplicationContext`额外暴露了几个构造方法。基本思想是提供一个只包含XML文件名字（没有主要的路径信息）的字符串数组以及一个`Class`，然后`ClassPathXmlApplicationContext`会从被提供的`Class`中提取出路径信息。

考虑下面的目录层次：

```
com/
  foo/
    services.xml
    daos.xml
    MessengerService.class
```

下面的例子展示了一个`ClassPathXmlApplicationContext`实例如何通过配置文件`services.xml` 和 `daos.xml`进行实例化的：

```
ApplicationContext ctx = new ClassPathXmlApplicationContext(
    new String[] {"services.xml", "daos.xml"}, MessengerService.class);
```

查看[`ClassPathXmlApplicationContext`](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframework/jca/context/SpringContextResourceAdapter.html) java文档获取关于构造方法使用的更多细节信息。

#### 2.7.2\. Wildcards in Application Context Constructor Resource Paths

提供给application context构造方法的资源路径值可以是简单的路径（正如前面展示的），每一个路径都和一个目标`Resource`有着一对一的映射关系。但是，它也可以包含一个特殊的"classpath\*:"前缀或者Ant风格的正则表达式（通过Spring的`PathMatcher`工具类进行匹配）。后面两种形式都是有效的通配符。

当你需要进行component-style的应用装配时可以使用这个机制。所有的组件都可以将context定义片段“发布[publish]”到一个知名的位置路径上，并且，当最终的application context使用以`classpath*:`为前缀的相同路径创建时，所有的组件都会被自动找到。

注意通配符是application context构造方法（或者直接使用`PathMatcher`工具类）中资源路径使用的特例并且它会在运行时进行解析。它和`Resource`类型本身无关，你不能使用`classpath*:`前缀来构造`Resource`，因为一个`Resource`一次只会指向一个资源。

##### Ant-style Patterns

路径位置可以包含Ant风格的模式，如下所示：

```
/WEB-INF/*-context.xml
com/mycompany/**/applicationContext.xml
file:C:/some/path/*-context.xml
classpath:com/mycompany/**/applicationContext.xml
```

当路径位置包含Ant风格的模式时，解析器会使用一个更灵活的过程来尝试解析通配符。它为这个路径中最后一个非通配符段（例如对于`classpath:com/mycompany/**/applicationContext.xml`，最后一个非通配符段为`classpath:com/mycompany`）产生一个`Resource`并且从中获取一个URL。如果这个URL不是一个`jar:`或者容器指定的变种（例如WebLogic中的`zip:`，WebSphere中的`wsjar`等等），那么会从中获取一个`java.io.File`并且遍历文件系统来解析通配符。如果它是一个jar URL，那么解析器会从中获取一个`java.net.JarURLConnection`或者手动解析jar URL，然后遍历jar文件解析通配符。

###### Implications on Portability

如果指定的路径已经是一个file URL（因为`ResourceLoader`是一个文件系统类型的`ResourceLoader`），那么通配符保证以完全轻便的[portable]方式工作。

如果指定的路径是一个classpath位置，解析器必须通过`Classloader.getResource()`调用获取URL中最后一个非通配符段。因为这只是路径的一个节点（不是路径末端的文件），所以实际上它没有定义（查看`ClassLoader` java文档）返回哪种URL。事实上，总是使用`java.io.File`来代表目录（classpath资源解析为文件系统的位置）或者某种jar URL（classpath资源解析为jar的位置）。

如果从最后一个非通配符段中获取了一个jar URL，解析器必须能从中获取一个`java.net.JarURLConnection`或者手动解析这个jar URL，以能够遍历jar的内容并且解析通配符。这可以在大多数环境下工作但是在少数环境下会失败，我们强烈建议从jar中解析通配符获取的资源，在你使用它之前先在你的环境下测试。

##### The `classpath*:` Prefix

当构造一个基于XML的application context时，位置字符串可以使用特殊的`classpath*:`前缀，如下所示：

```
ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath*:conf/appContext.xml");
```

这个特殊的前缀指定了所有符合给定名字的classpath资源都必须被获取（本质上通过调用`ClassLoader.getResources(…​)`发生），然后合并它们来组成最终的application context。

> 通配符classpath依赖于底层classloader的`getResources()`方法。如今的大多数应用服务都提供了它们自己的classloader实现，所以这个方法的行为可能不同，尤其是处理jar文件时。可以使用classloader从classpath上的一个jar中加载一个文件来简单测试是否`classpath*`能正常工作，例如`getClass().getClassLoader().getResources("<someFileInsideTheJar>")`，使用名字相同但位置不同的文件来进行这个测试。如果返回了一个不合适的结果，那么检查可能会影响classloader行为的应用服务文档。

你可以将`classpath*:`与`PathMatcher`模式结合使用（例如`classpath*:META-INF/*-beans.xml`）。此时，解析策略是相当简单的：在最后一个非通配符段上调用`ClassLoader.getResources()`来获取所有在classloader层次中匹配的资源，然后对每个资源应用前面描述的`PathMatcher`解析策略来解析通配符子路径。

##### Other Notes Relating to Wildcards

注意当`classpath*:`和Ant风格的模式结合使用时，在Ant模式前至少有一个根目录时才会可靠的工作，除非真正的目标文件处于文件系统中。这意味着例如`classpath*:*.xml`这样的模式可能不会从jar文件的根目录下提取文件而是从拓展根目录中提取。

Spring提取classpath条目的能力由JDK的 `ClassLoader.getResources()`方法产生，这个方法对于一个空的字符串只会返回文件系统位置。Spring会执行`URLClassLoader`运行时配置和jar文件中的`java.class.path` manifests，但是这不保证是一个轻便的行为。

> 扫描classpath包需要在classpath中存在相关的目录。当你使用Ant构建JARs时，不要激活JAR任务的files-only开关。同时，classpath目录可能由于某些环境的安全机制没有暴露出来 —— 例如JDK 1.7.0_45及更高版本的独立应用（需要在你的manifests中设置'Trusted-Library'）。查看 [https://stackoverflow.com/questions/19394570/java-jre-7u45-breaks-classloader-getresources](https://stackoverflow.com/questions/19394570/java-jre-7u45-breaks-classloader-getresources) 获取更多信息）。<br/>
在JDK 9的模块路径下(Jigsaw)，Spring的classpath扫描一般会按预期工作。将资源放到一个专用的目录下是非常推荐的，以避免当扫描jar文件根目录时遇到前面的portability问题。


如果要被搜索的root包在多个classpath中可获得，那么使用`classpath:`的Ant风格resource不保证会找到匹配的资源，考虑下面的例子：

```
com/mycompany/package1/service-context.xml
```

现在考虑使用Ant风格的路径来寻找这个文件：

```
classpath:com/mycompany/**/service-context.xml
```

这个资源可能只处于一个位置，但是当一个像前面例子那样的路径被用来解析时，解析器会在通过`getResource("com/mycompany");`返回的URL上工作。如果包节点存在于多个classloader位置中，那么真正的最终资源可能不在当前这个classloader中。因此，此时你应当使用 `classpath*:` 来和Ant模式配合，它会搜索所有包含那个root package的classpath位置。

#### 2.7.3. `FileSystemResource` Caveats

不隶属于`FileSystemApplicationContext`的`FileSystemResource`（不将`FileSystemApplicationContext`作为`ResourceLoader`使用）会如你期待的那样对待绝对和相对路径。相对路径相对于目前的工作目录，而绝对路径相对于文件系统的根目录。

但是由于历史兼容性，当使用`FileSystemApplicationContext`作为`ResourceLoader`时这将会发生改变。`FileSystemApplicationContext`会强制所有附属的`FileSystemResource`实例将路径视为相对路径，不管它们是否以一个斜线开头。实际上，这意味着下面两个例子是等同的：

```
ApplicationContext ctx = new FileSystemXmlApplicationContext("conf/context.xml");
```

```
ApplicationContext ctx = new FileSystemXmlApplicationContext("/conf/context.xml");
```

下面的例子也是等同的（即使它们的形式不一样，一个是绝对路径一个是相对路径）：

```
FileSystemXmlApplicationContext ctx = ...;
ctx.getResource("some/resource/path/myTemplate.txt");
```

```
FileSystemXmlApplicationContext ctx = ...;
ctx.getResource("/some/resource/path/myTemplate.txt");
```

实际上，如果你需要使用绝对的文件系统路径，你应当避免在`FileSystemResource` 或 `FileSystemXmlApplicationContext`中使用绝对路径，而是通过`file:` URL前缀来强制一个`UrlResource`，如下所示：

```
// actual context type doesn't matter, the Resource will always be UrlResource
ctx.getResource("file:///some/resource/path/myTemplate.txt");
```

```
// force this FileSystemXmlApplicationContext to load its definition via a UrlResource
ApplicationContext ctx =
    new FileSystemXmlApplicationContext("file:///conf/context.xml");
```

##### 示例

```
@Test
public void testFileSystemResourceCaveats() throws Exception {
    ApplicationContext context = new FileSystemXmlApplicationContext();
    Resource resource = context.getResource("/src/java");
    System.out.println(resource.getURL());
    resource = context.getResource("src/java");
    System.out.println(resource.getURL());
}

@Test
public void testFileSystemResourceCaveats2() throws Exception {
    FileSystemResource resource = new FileSystemResource("/src/java");
    System.out.println(resource.getURL());
    resource = new FileSystemResource("src/java");
    System.out.println(resource.getURL());
}
```

输出如下：
```
测试1
file:/C:/Users/81287/IdeaProjects/SSMDocument/spring-module/src/java
file:/C:/Users/81287/IdeaProjects/SSMDocument/spring-module/src/java

测试2
file:/C:/src/java
file:/C:/Users/81287/IdeaProjects/SSMDocument/spring-module/src/java
```

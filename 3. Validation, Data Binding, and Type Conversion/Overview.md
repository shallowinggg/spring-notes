将验证作为业务逻辑的一部分有利有弊，Spring为验证[validation]（以及数据绑定[Data binding]）提供了一种设计。特别的，验证不应当和web层绑定到一起，并且应该容易实现本地化，同时还要能在需要时加上新的验证器。考虑到这些需求，Spring提出了一个`Validator`接口，它可以在应用的每个层次使用。

数据绑定可以实现用户输入和应用的domain model（或者任何你用来处理用户输入的对象）之间的动态绑定。Spring提供了恰当的名字`DataBinder`来完成这项工作。`Validator` 和`DataBinder`组成了`validation`包，它可以在多个地方使用，不只是MVC框架中。

`BeanWrapper`是Spring框架中的一个基本概念，它在很多地方使用。但是，你一般不会直接用到`BeanWrapper`。因为这是一个文档，所以我们认为需要先做一些解释说明。我们在这一章节说明`BeanWrapper`，因为当你尝试将数据和对象绑定时，你很多可能要使用它。

Spring的`DataBinder`和底层的`BeanWrapper`都使用了`PropertyEditorSupport`实现类来解析以及格式化属性值。`PropertyEditor`和`PropertyEditorSupport`类型是JavaBean的一部分，它们也将在这节解释。Spring 3新增了一个`core.convert`包并且提出了泛型类型转换功能以及一个高层的“format”包来格式化UI字段值。你可以使用这些包作为`PropertyEditorSupport`实现类的简化替代，它们也在这节解释。

>JSR-303/JSR-349 Bean Validation<br/>
从Spring 4.0开始，Spring支持了Bean Validation 1.0 (JSR-303)以及Bean Validation 1.1 (JSR-349)，并且将它们适配到了Spring的`Validator`接口中。<br/>
应用可以选择打开Bean Validation，查看[Spring Validation](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#validation-beanvalidation)，并且可以只使用它来应对所有验证的需求。<br/>
应用还可以为每一个`DataBinder`实例注册额外的Spring `Validator`实例，查看[Configuring a DataBinder](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#validation-binder)。当不使用注解来增加验证逻辑时这是有用的。

### 3.1\. Validation by Using Spring’s Validator Interface

Spring提供了一个`Validator`接口，你可以使用它来验证对象。`Validator`接口在工作时使用`Errors`对象配合工作，当遇到错误时验证器可以使用`Errors`对象来报告验证错误信息。

考虑下面的例子：

```
public class Person {

    private String name;
    private int age;

    // the usual getters and setters...
}
```

下面通过实现了`org.springframework.validation.Validator`接口的两个方法来对`Person`类进行验证：

*   `supports(Class)`: 这个`Validator`能否验证被提供的`Class`实例？
*   `validate(Object, org.springframework.validation.Errors)`: 验证给定的对象，如果出现验证错误，使用给定的`Errors`对象注册错误信息。

实现一个`Validator`是相当直接的，特别是当你使用Spring提供的`ValidationUtils`帮助类时。下面为`Person`实例实现了一个`Validator`：

```
public class PersonValidator implements Validator {

    /**
     * This Validator validates *only* Person instances
     */
    public boolean supports(Class clazz) {
        return Person.class.equals(clazz);
    }

    public void validate(Object obj, Errors e) {
        ValidationUtils.rejectIfEmpty(e, "name", "name.empty");
        Person p = (Person) obj;
        if (p.getAge() < 0) {
            e.rejectValue("age", "negativevalue");
        } else if (p.getAge() > 110) {
            e.rejectValue("age", "too.darn.old");
        }
    }
}
```

`ValidationUtils`类中的静态方法`rejectIfEmpty(..)`可以用来驳斥[reject]`name`属性，如果它是`null`或者是一个空的字符串。查看[ValidationUtils](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframework/validation/ValidationUtils.html) java文档获取更多信息。

虽然可以为一个丰富对象[rich object]实现了一个`Validator`类来验证其中的每一个嵌套对象，但是最好为每个嵌套对象都实现了一个自己的`Validator`，以封装它们专有的验证逻辑。一个简单的丰富对象的例子就是`Customer`，它包含两个`String`属性（用来表示名字）以及一个复合的`Address`对象。`Address`对象可以独立于`Customer`对象使用，所以需要实现了一个确切的`AddressValidator`。如果你想要你的`CustomerValidator`复用`AddressValidator`类中的逻辑，但是不是通过复制粘贴这种方式，你可以在`CustomerValidator`中依赖注入或者实例化一个`AddressValidator`，如下所示：

```
public class CustomerValidator implements Validator {

    private final Validator addressValidator;

    public CustomerValidator(Validator addressValidator) {
        if (addressValidator == null) {
            throw new IllegalArgumentException("The supplied [Validator] is " +
                "required and must not be null.");
        }
        if (!addressValidator.supports(Address.class)) {
            throw new IllegalArgumentException("The supplied [Validator] must " +
                "support the validation of [Address] instances.");
        }
        this.addressValidator = addressValidator;
    }

    /**
     * This Validator validates Customer instances, and any subclasses of Customer too
     */
    public boolean supports(Class clazz) {
        return Customer.class.isAssignableFrom(clazz);
    }

    public void validate(Object target, Errors errors) {
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "firstName", "field.required");
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "surname", "field.required");
        Customer customer = (Customer) target;
        try {
            errors.pushNestedPath("address");
            ValidationUtils.invokeValidator(this.addressValidator, customer.getAddress(), errors);
        } finally {
            errors.popNestedPath();
        }
    }
}
```

验证错误可以被验证器报告到`Errors`对象中。当使用Spring Web MVC时，你可以使用`<spring:bind/>`标签来检查[inspect]错误信息，但是你也可以自己检查`Errors`对象。查看[java 文档](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframeworkvalidation/Errors.html)获取更多信息。

### 3.2\. Resolving Codes to Error Messages

本节介绍输出关于验证错误的信息。在前面展示的例子中，我们拒绝了`name` 和 `age`字段。如果我们想要使用`MessageSource`来输出错误信息，我们可以在拒绝字段时使用错误代码。当你调用（间接或直接，例如使用`ValidationUtils`类）`rejectValue`或者`Errors`接口中任意一个`reject`方法时，底层实现不仅会注册你提供的错误代码，还会注册许多额外的错误代码。默认情况下会使用`MessageCodesResolver`，它不仅会注册你提供的错误信息，还会注册包含你提供给reject方法的字段名称的错误信息。因此，如果你使用`rejectValue("age", "too.darn.old")`拒绝一个字段，除了`too.darn.old`以外，Spring还会注册`too.darn.old.age` 以及 `too.darn.old.age.int`（第一个包含了字段名称，第二个包含了字段类型）。当定位错误信息时它可以用来帮助开发者。

关于`MessageCodesResolver`以及默认的策略，可以在[MessageCodesResolver](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframework/validation/MessageCodesResolver.html) 和 [DefaultMessageCodesResolver](https://docs.spring.io/spring-framework/docs/5.1.6.RELEASE/javadoc-api/org/springframework/validation/DefaultMessageCodesResolver.html) java文档中查看更多信息。

### 3.3\. Bean Manipulation and the `BeanWrapper`

`org.springframework.beans`包使用JavaBean标准。JavaBean是指只有一个默认无参构造方法以及遵循命名规范（例如，一个名为`bingoMadness`的属性拥有一个setter方法`setBingoMadness(..)`以及一个getter方法`getBingoMadness()`）的类。关于JavaBean的更多信息，查看[javabeans](https://docs.oracle.com/javase/8/docs/api/java/beans/package-summary.html)。

在`org.springframework.beans`包中一个相当重要的类是`BeanWrapper`接口以及它的实现类(`BeanWrapperImpl`)。`BeanWrapper`提供了设置和获取属性值（一次或批量），获取属性描述符以及查询属性是可读还是可写的能力。同时，`BeanWrapper`提供了对嵌套属性的支持，可以设置子属性的属性。`BeanWrapper`还提供了增加标准JavaBean `PropertyChangeListeners` 和 `VetoableChangeListeners`的能力，无需在目标类上增加代码。最后，`BeanWrapper`还支持设置属性下标。`BeanWrapper`一般不会被应用代码直接使用，而是通过`DataBinder` 和 `BeanFactory`间接使用。

`BeanWrapper`工作的方式是通过它的名字：它包装了一个bean，在这个bean上执行操作，例如设置以及提取属性。

#### 3.3.1\. Setting and Getting Basic and Nested Properties

可以使用`setPropertyValue`, `setPropertyValues`, `getPropertyValue`和 `getPropertyValues`方法来设置以及获取属性。Spring java文档中描述了更多的信息，JavaBean标准对表明[indicate]对象属性也有规定。下面的表展示了这些规定的例子：


| 表达式 | 说明 |
| --- | --- |
| `name` | 表示`name`属性与`getName()`  `isName()`  `setName(..)`方法相关|
| `account.name` | 表示`account`属性的嵌套属性`name`与`getAccount().setName()`  `getAccount().getName()`方法有关 |
| `account[2]` | 表示索引属性`account`的**第三个**元素。索引属性可以是`array`, `list`或其他可排序集合类型 |
| `account[COMPANYNAME]` | 表示`Map`属性`account`通过`COMPANYNAME`键索引的映射值 |

（如果你不打算直接使用`BeanWrapper`，那么下一节中的内容对你来说并不重要。如果你只使用`DataBinder`，`BeanFactory`以及它们的默认实现，你可以跳过[PropertyEditors](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-beans-conversion)这一节。）

下面的两个例子展示了使用`BeanWrapper`来获取以及设置属性：

```
public class Company {

    private String name;
    private Employee managingDirector;

    public String getName() {
        return this.name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Employee getManagingDirector() {
        return this.managingDirector;
    }

    public void setManagingDirector(Employee managingDirector) {
        this.managingDirector = managingDirector;
    }
}
```

```
public class Employee {

    private String name;

    private float salary;

    public String getName() {
        return this.name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public float getSalary() {
        return salary;
    }

    public void setSalary(float salary) {
        this.salary = salary;
    }
}
```

下面的代码片段展示了如何提取以及操纵已经实例化的`Companies` 和 `Employees`对象属性：

```
BeanWrapper company = new BeanWrapperImpl(new Company());
// setting the company name..
company.setPropertyValue("name", "Some Company Inc.");
// ... can also be done like this:
PropertyValue value = new PropertyValue("name", "Some Company Inc.");
company.setPropertyValue(value);

// ok, let's create the director and tie it to the company:
BeanWrapper jim = new BeanWrapperImpl(new Employee());
jim.setPropertyValue("name", "Jim Stravinsky");
company.setPropertyValue("managingDirector", jim.getWrappedInstance());

// retrieving the salary of the managingDirector through the company
Float salary = (Float) company.getPropertyValue("managingDirector.salary");
```

#### 3.3.2\. Built-in `PropertyEditor` Implementations

Spring使用`PropertyEditor`来进行`Object` 和 `String`之间的类型转换。它可以巧妙地以不同的方式来表示对象属性。例如，`Date`可以以一种人性化的方式(比如 `String`: `'2007-14-09'`)来表示，我们还可以将它从人类可阅读的形式转换回原始的日期。这个行为可以通过注册`java.beans.PropertyEditor`类型的自定义编辑器[custom editor]来完成。在`BeanWrapper`或者在一个特定的IoC容器中注册自定义编辑器，并告知编辑器如何转换一个指定类型的属性。关于`PropertyEditor`的更多信息，查看[the javadoc of the `java.beans` package from Oracle](https://docs.oracle.com/javase/8/docs/api/java/beans/package-summary.html)。

下面展示Spring中属性编辑器的用途：
- 使用`PropertyEditor`实现类来设置bean中的属性。当你在XML中使用`String`作为某些bean的属性值时，Spring（如果相关属性的setter方法拥有一个`Class`类型的参数）会使用`ClassEditor`来尝试将参数解析为一个`Class`对象。
- Spring MVC使用你在`CommandController`中手动绑定的所有`PropertyEditor`实现来解析HTTP请求参数。

Spring拥有许多内置的`PropertyEditor`实现，它们都处于`org.springframework.beans.propertyeditors`包中。大多数（但不是全部）默认都会注册到`BeanWrapperImpl`中。虽然Spring自己有许多属性编辑器，但你也可以注册你自己的来覆盖默认的编辑器。下面的表展示了各种Spring提供的`PropertyEditor`：

| 类 | 说明 |
| --- | --- |
| `ByteArrayPropertyEditor` | 处理字节数组。将字符串转换为它们相应的字节表示，默认注册到`BeanWrapperImpl`中。 |
| `ClassEditor` | 解析表示Class的字符串。如果没有找到这个类，那么抛出`IllegalArgumentException`异常，默认注册到`BeanWrapperImpl`中。 |
| `CustomBooleanEditor` | 为`Boolean`属性定制的属性编辑器。默认注册到`BeanWrapperImpl`中，但是可以注册一个自定义的编辑器来覆盖它。 |
| `CustomCollectionEditor` | 为集合定制的属性编辑器，将`Collection`转换为一个给定的`Collection`类型。 |
| `CustomDateEditor` | 为`java.util.Date`定制的属性编辑器，支持一个自定义的`DateFormat`。默认**不会**注册它，必须指定需要的格式并且手动注册 |
| `CustomNumberEditor` | 为`Number`子类定制的属性编辑器，例如`Integer`, `Long`, `Float` 或 `Double`。默认注册到`BeanWrapperImpl`中，但是可以注册一个自定义的编辑器来覆盖它。 |
| `FileEditor`| 将字符串解析为`java.io.File` 对象，默认注册到`BeanWrapperImpl`中 |
| `InputStreamEditor` | 接受一个字符串并且产生（通过中间的`ResourceEditor` 和`Resource`）一个 `InputStream` ，因此在XML中`InputStream` 属性可以直接使用字符串来设置。注意，使用完不会自动为你关闭 `InputStream`，默认注册到`BeanWrapperImpl`中 |
| `LocaleEditor` | 将字符串转换为`Locale`对象，反之亦然（字符串格式为`[country][variant]`，与`Locale`的`toString()`方法相同）。 默认注册到`BeanWrapperImpl`中 |
| `PatternEditor` | 将字符串解析为`java.util.regex.Pattern`，反之亦然 |
| `PropertiesEditor` | 将字符串（使用定义在`java.util.Properties`类中的格式）转换为`Properties` 对象。默认注册到`BeanWrapperImpl`中 |
| `StringTrimmerEditor` | 整理[trim]字符串的属性编辑器，允许将一个空的字符串转换为`null`值。默认**不会**注册它，需要用户手动注册 |
| `URLEditor` | 将字符串表示的URL转换为一个真正的`URL`对象，默认注册到`BeanWrapperImpl`中 |

Spring使用`java.beans.PropertyEditorManager`来设置属性编辑器的查询路径。查询路径包括`sun.bean.editors`，它包含例如`Font`, `Color`以及大多数基本类型的`PropertyEditor`实现类。注意，标准的JavaBean基础设施会自动发现（无需你手动注册）`PropertyEditor`类，如果它们在同一个包中并且编辑器类以`Editor`结尾。例如下面的包结构，`SomethingEditor`会被自动识别并且作为`Something`类型的属性的`PropertyEditor`使用。

```
com
  chank
    pop
      Something
      SomethingEditor // the PropertyEditor for the Something class
```

注意，你也可以使用标准的`BeanInfo` JavaBeans机制（查看[这里](https://docs.oracle.com/javase/tutorial/javabeans/advanced/customization.html)获取更多信息）。下面的例子使用了`BeanInfo`机制来为相关类的属性明确注册一个或多个`PropertyEditor`实例：

```
com
  chank
    pop
      Something
      SomethingBeanInfo // the BeanInfo for the Something class
```

下面的Java源代码展示了`SomethingBeanInfo`类，它将`CustomNumberEditor`和`Something`类的`age`属性联系在一起：

```
public class SomethingBeanInfo extends SimpleBeanInfo {

    public PropertyDescriptor[] getPropertyDescriptors() {
        try {
            final PropertyEditor numberPE = new CustomNumberEditor(Integer.class, true);
            PropertyDescriptor ageDescriptor = new PropertyDescriptor("age", Something.class) {
                public PropertyEditor createPropertyEditor(Object bean) {
                    return numberPE;
                };
            };
            return new PropertyDescriptor[] { ageDescriptor };
        }
        catch (IntrospectionException ex) {
            throw new Error(ex.toString());
        }
    }
}
```

##### Registering Additional Custom `PropertyEditor` Implementations

当使用字符串设置bean属性时，Spring IoC容器最终会使用标准的JavaBeans `PropertyEditor`实现类来将字符串转换为复杂类型的属性。Spring会提前实例化许多自定义的`PropertyEditor`实现类（例如，将使用字符串表达的类名转换为一个`Class`对象）。另外，依托Java标准的JavaBeans `PropertyEditor`查找机制，`PropertyEditor`可以放在和转换类同一个包中，并且它的名字需要与转换类对应，这样它能够被自动寻找。

如果需要注册其他自定义的`PropertyEditor`，有如下几种方式。大多数手动注册的方式都是不方便的或者不推荐的，通过使用`ConfigurableBeanFactory`接口的`registerCustomEditor()`方法。另外一个机制（稍微方便一点）是使用一个名为`CustomEditorConfigurer`的bean factory后置处理器。虽然可以使用通过`BeanFactory`使用bean factory后置处理器，但是`CustomEditorConfigurer`拥有一个嵌套的属性设置，所以我们强烈建议你使用`ApplicationContext`，这样你可以以和其他bean一样的方式部署，它会被自动检测以及应用。

注意所有的BeanFactory和ApplicationContext都会自动使用许多内建的属性编辑器，通过使用`BeanWrapper`来处理属性转换，`BeanWrapper`注册的标准的属性编辑器在前面已经列出了。另外，`ApplicationContexts`也可以覆盖或者增加额外的编辑器来处理资源查找。

标准的JavaBeans `PropertyEditor`实例被用来将字符串表示的属性值转换为属性的真实类型。你可以使用`CustomEditorConfigurer`来方便的增加额外的`PropertyEditor`到`ApplicationContext`中。

考虑下面的例子，它定义了一个名为`ExoticType`的类，以及另一个名为`DependsOnExoticType`的类：

```
package example;

public class ExoticType {

    private String name;

    public ExoticType(String name) {
        this.name = name;
    }
}

public class DependsOnExoticType {

    private ExoticType type;

    public void setType(ExoticType type) {
        this.type = type;
    }
}
```

我们想要使用一个字符串来设置type属性，通过`PropertyEditor`将字符串转换为一个真正的`ExoticType`实例。如下所示：

```
<bean id="sample" class="example.DependsOnExoticType">
    <property name="type" value="aNameForExoticType"/>
</bean>
```

`PropertyEditor`实现如下所示：

```
// converts string representation to ExoticType object
package example;

public class ExoticTypeEditor extends PropertyEditorSupport {

    public void setAsText(String text) {
        setValue(new ExoticType(text.toUpperCase()));
    }
}
```

最后，下面的例子展示了如何使用`CustomEditorConfigurer`来注册一个新的自定义的`PropertyEditor`：

```
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
    <property name="customEditors">
        <map>
            <entry key="example.ExoticType" value="example.ExoticTypeEditor"/>
        </map>
    </property>
</bean>
```

###### Using PropertyEditorRegistrar

另一种使用Spring容器注册属性编辑器的机制是创建以及使用`PropertyEditorRegistrar`。当你需要在几种不同的情况下使用相同的属性编辑器集合时，这个接口是有用的。你可以编写一个对应的registrar并且在每种情况下复用它。`PropertyEditorRegistrar`实例与`PropertyEditorRegistry`接口协力工作，Spring `BeanWrapper` (和 `DataBinder`)实现了这个接口。当与`CustomEditorConfigurer`协力工作时，使用`PropertyEditorRegistrar`实例是相当方便的，`CustomEditorConfigurer`暴露了一个叫做`setPropertyEditorRegistrars(..)`的方法。以这种形式增加到`CustomEditorConfigurer`中的`PropertyEditorRegistrar`实例可以被`DataBinder`和Spring MVC controller使用。更深入的，它避免了自定义编辑器使用的同步：`PropertyEditorRegistrar`会为每个尝试创建的bean创建一个新的`PropertyEditor`实例。

下面的例子展示了如何创建你自己的`PropertyEditorRegistrar`实现：

```
package com.foo.editors.spring;

public final class CustomPropertyEditorRegistrar implements PropertyEditorRegistrar {

    public void registerCustomEditors(PropertyEditorRegistry registry) {

        // it is expected that new PropertyEditor instances are created
        registry.registerCustomEditor(ExoticType.class, new ExoticTypeEditor());

        // you could register as many custom property editors as are required here...
    }
}
```

`org.springframework.beans.support.ResourceEditorRegistrar`是`PropertyEditorRegistrar`的一个示例实现。注意在它之中如何实现`registerCustomEditors(..)`方法，它为每一个属性编辑器都创建一个新的实例。

下面展示了如何配置`CustomEditorConfigurer`，并且注入一个`CustomPropertyEditorRegistrar`到它之中：

```
<bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
    <property name="propertyEditorRegistrars">
        <list>
            <ref bean="customPropertyEditorRegistrar"/>
        </list>
    </property>
</bean>

<bean id="customPropertyEditorRegistrar"
    class="com.foo.editors.spring.CustomPropertyEditorRegistrar"/>
```

最后，使用`PropertyEditorRegistrars`与数据绑定的`Controllers`协力工作是非常方便的。下面的例子在`initBinder(..)`方法中使用了`PropertyEditorRegistrar`：

```
public final class RegisterUserController extends SimpleFormController {

    private final PropertyEditorRegistrar customPropertyEditorRegistrar;

    public RegisterUserController(PropertyEditorRegistrar propertyEditorRegistrar) {
        this.customPropertyEditorRegistrar = propertyEditorRegistrar;
    }

    protected void initBinder(HttpServletRequest request,
            ServletRequestDataBinder binder) throws Exception {
        this.customPropertyEditorRegistrar.registerCustomEditors(binder);
    }

    // other methods to do with registering a User
}
```

这种形式的`PropertyEditor`注册可以导致简明的代码（`initBinder(..)`的实现只有一行）并且使得`PropertyEditor`的注册代码封装到一个类中，然后可以在许多`Controllers`共享使用。

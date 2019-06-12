The Spring Expression Language (简称 “SpEL”)是一个强有力的表达式语言，它支持在运行时查询以及操作对象。这个语言的语法和Unified EL相似，但是提供了额外的特性，尤其在方法调用和基础的字符串模板功能上。

虽然已经有了其他几种Java表达式语言存在 —— 比如OGNL, MVEL 以及 JBoss EL —— 但是Spring Expression Language被创建在整个Spring框架之中使用，它是提供给Spring社区的一个支持性最好的表达式语言。它的语言特性由Spring的项目需求驱动，包括在Eclipse-based Spring Tool Suite中的code completion support组件需求。也就是说，SpEL是技术无关的API，如果有需求的话，可以整合其他表达式语言的实现。

虽然SpEL在Spring中作为表达式执行的基础，但是它没有和Spring直接绑定在一起并且可以独立使用，本章中的许多例子使用SpEL时，就好像它是一个独立的表达式语言一样。不过这需要创建几个启动设施类，例如解析器。大多数Spring用户不需要处理这些设施，相反，只需要编写执行的表达式字符串即可。一个典型用例是整合SpEL到使用XML或者基于注解的bean定义中，查看 [Expression support for defining bean definitions](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-beandef) 获得更多信息。

本章介绍了表达式语言的特性，API，语法。在有些例子中，使用`Inventor` 和 `Society` 类作为表达式执行的目标对象，这些类的声明以及用来注入的数据在本章最后列举。

表达式语言支持下面的功能：

- 字面[Literal]表达式
- Boolean以及相关操作
- 正则表达式
- 类表达式
- 访问properties, arrays, lists 和 maps
- 方法调用
- 关系运算符
- 赋值
- 调用构造方法
- bean引用
- 数组构造
- 内联list
- 内联map
- 三元运算符
- 变量
- 用户定义的函数
- 集合映射[projection]
- 集合选择
- 模板表达式

### 4.1\. Evaluation

本节介绍了SpEL接口以及表达式语言的简单使用，完整的语言规范可以查看 [Language Reference](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-language-ref) 。

下面的代码使用了SpEL API来执行字面量字符串表达式  `Hello World` :

```
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("'Hello World'"); 
String message = (String) exp.getValue();
```

```
message变量的值是 'Hello World'
```

你最常使用的SpEL类和接口位于`org.springframework.expression`包以及它的子包中，例如`spel.support`。

`ExpressionParser`接口负责解析表达式字符串。在前面的例子中，表达式是一个字符串字面量，通过外层的单引号标识。`Expression`接口负责执行前面定义的表达式字符串，当调用`parser.parseExpression` 和 `exp.getValue`方法时，它可能会抛出两个异常，`ParseException` 和 `EvaluationException`。

SpEL支持很多特性，例如调用方法，访问属性以及调用构造方法。

下面的例子展示了如何调用方法，我们使用一个字符串字面量并调用它的`concat`方法：

```
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("'Hello World'.concat('!')"); 
String message = (String) exp.getValue();
// message="Hello World!"
```

下面的例子使用了`String`类的属性`bytes`：

```
ExpressionParser parser = new SpelExpressionParser();

// invokes 'getBytes()'
Expression exp = parser.parseExpression("'Hello World'.bytes"); 
byte[] bytes = (byte[]) exp.getValue();
```

SpEL也支持嵌套属性，通过标准的点标记法（例如`prop1.prop2.prop3`）访问以及设置属性值，可以直接获取public域的属性，否则需要提供一个getter方法。下面的例子展示了如何获取一个字面量的长度：

```
ExpressionParser parser = new SpelExpressionParser();

// invokes 'getBytes().length'
Expression exp = parser.parseExpression("'Hello World'.bytes.length"); 
int length = (Integer) exp.getValue();
```

可以调用字符串的构造方法来代替使用字符串字面量，如下所示：

```
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("new String('hello world').toUpperCase()"); 
String message = exp.getValue(String.class);
```

注意泛型方法的使用：`public <T> T getValue(Class<T> desiredResultType)`，使用这个方法可以避免将表达式的值转换为需要的目标类型。如果值无法被转换为类型`T`，那么就会抛出`EvaluationException`异常，或者通过注册的类型转换器转换。

SpEL的一般更多通过一个特定的对象实例（称作根对象）执行表达式字符串。下面的例子展示了如何提取`Inventor`类实例的`name`属性以及创建一个布尔条件表达式：

```
// Create and set a calendar
GregorianCalendar c = new GregorianCalendar();
c.set(1856, 7, 9);

// The constructor arguments are name, birthday, and nationality.
Inventor tesla = new Inventor("Nikola Tesla", c.getTime(), "Serbian");

ExpressionParser parser = new SpelExpressionParser();

Expression exp = parser.parseExpression("name"); 
String name = (String) exp.getValue(tesla);
// name == "Nikola Tesla"

exp = parser.parseExpression("name == 'Nikola Tesla'");
boolean result = exp.getValue(tesla, Boolean.class);
// result == true
```

#### 4.1.1\. Understanding *EvaluationContext*

当执行一个表达式来解析属性，方法或者字段以及帮助执行类型转换时，可以使用`EvaluationContext`接口。Spring提供了这个接口的两个实现类。

- `SimpleEvaluationContext`: 暴露必要的SpEL语言特性的子集和配置选项，对于不需要SpEL语言语法完整拓展或者需要被限制的表达式类别[categories]的用户，可以使用它。排除的部分包括但不限于数据绑定表达式以及基于属性的过滤器。
- `StandardEvaluationContext`: 暴露完整的SpEL语言特性的子集和配置选项。你可以使用它来指定默认的根对象以及配置每个可获得的与执行相关的策略。

`SimpleEvaluationContext`被设计用来支持SpEL语言语法的子集。它排除了Java类型引用，构造方法以及bean引用。它还需要你明确选择表达式中属性和方法的支持等级。默认情况下，静态工厂方法`create()`只能够读取属性。你还可以获取一个builder来精确配置需求支持等级，使用下面中的一个或多个等级结合：

- 只支持自定义`PropertyAccessor`（没有反射）
- 只读的数据绑定属性
- 读写的数据绑定属性

> 注意：此处Spring文档疑似存在问题，这里提到的被排除的部分特性`SimpleEvaluationContext`支持使用。

##### Type Conversion

默认情况下，SpEL使用Spring core(`org.springframework.core.convert.ConversionService`)中的conversion service。这个conversion service 拥有许多内建的转换器，用来支持常见的转换，但是它也完全支持拓展，因此你可以增加自定义的转换器。另外，它是泛型。这意味着，当你在表达式中使用泛型类型时，SpEL对遇到的所有对象都会尝试将它转换为它的实际类型。

这在实际中意味着什么？假设使用`setValue()`方法赋值，用来设置一个`List`属性。这个属性的真实类型是`List<Boolean>`，那么SpEL会在设置这个属性之前识别出这个列表中的元素需要被转换为`Boolean`类型。如下所示：

```
class Simple {
    public List<Boolean> booleanList = new ArrayList<Boolean>();
}

Simple simple = new Simple();
simple.booleanList.add(true);

EvaluationContext context = SimpleEvaluationContext.forReadOnlyDataBinding().build();

// "false" is passed in here as a String. SpEL and the conversion service
// will recognize that it needs to be a Boolean and convert it accordingly.
parser.parseExpression("booleanList[0]").setValue(context, simple, "false");

// b is false
Boolean b = simple.booleanList.get(0);
```

#### 4.1.2\. Parser Configuration

可以使用一个解析器配置对象(`org.springframework.expression.spel.SpelParserConfiguration`)来配置SpEL表达式解析器，配置对象控制了某些表达式组件的行为。例如，如果你在数组或集合中使用下标并且这个下标对应的元素是`null`，你可以自动创建这个元素，当使用表达式来构建属性引用链的时候这是有用的。如果你在数组或集合中使用下标并且这个下标越界了，你可以自动扩展数组或列表使其容纳这个下标。下面的例子展示了如何自动扩容列表以及自动创建元素：

```
class Demo {
    public List<String> list;
}

// Turn on:
// - auto null reference initialization
// - auto collection growing
SpelParserConfiguration config = new SpelParserConfiguration(true,true);

ExpressionParser parser = new SpelExpressionParser(config);

Expression expression = parser.parseExpression("list[3]");

Demo demo = new Demo();

Object o = expression.getValue(demo);

// demo.list will now be a real collection of 4 entries
// Each entry is a new empty String
```

#### 4.1.3\. SpEL Compilation

Spring 4.1 包含了一个基本的表达式编译器。表达式一般被解释，这可以在执行时提供许多动态的灵活性，但是不会提供合适的性能。如果偶尔使用表达式，这可以满足使用了，但是当通过其他组件例如Spring Integration使用时，性能就会变得非常重要，并且此时也不需要动态性。

SpEL编译器定位了这个需求。在执行时，编译器会产生表现表达式行为的真实Java类并且使用它来获取更快的表达式执行。由于表达式缺少类型，编译器编译时会根据表达式的解释执行过程的信息来推断类型。例如，它无法从表达式中知道一个属性引用的类型，但是在第一次被解释执行时，它会发现那个属性是什么类型。当然，如果表达式元素的类型在之后改变了，基于这种信息的编译就会出现问题。由于这个问题，编译最适合类型信息不会在重复执行时改变的表达式。

考虑下面的表达式：
```
someArray[0].someProperty.someOtherProperty < 0.1
```

因为这个例子包含了数组访问，属性引用以及数值操作，所以编译后性能会出现明显的增长。对它进行50000次遍历的压力测试，使用解释器需要执行75ms，而使用这个表达式的编译版本只需要执行3ms。

##### Compiler Configuration

编译器默认不会启用，但是你可以以两种不同的方式启用它。你可以使用前面提到的解析器配置类或者当SpEL被植入到其他组件时使用系统属性来启用它，这两种选择都将在本节介绍。

编译器可以以三种模式操作，它们定义在`org.springframework.expression.spel.SpelCompilerMode`枚举中。如下所示：

*   `OFF` (default): 选择关闭编译器。

*   `IMMEDIATE`: 在 immediate 模式中，会尽可能的编译表达式，这一般发生在第一次解释执行之后。如果被编译的表达式运行失败了（一般由于类型改变），执行表达式的调用者将会接收到一个异常。

*   `MIXED`: 在mixed模式下，表达式会随时间在解释以及编译模式间选择。在经过一定数量的解释运行后，会对它进行编译，并且如果编译的代码运行失败了，表达式会自动切换回解释执行。经过一段时间后，会再产生另外一段编译形式并且使用它。总的来说，在`IMMEDIATE`模式下产生的异常会在内部处理而不是提交给用户。

因为`MIXED`模式运行有副作用的表达式时会产生问题，所以出现了`IMMEDIATE`模式。如果一个被编译的表达式在部分执行后出现问题，它可能已经对系统中的某些状态产生了影响。如果发生了这种错误，调用者不会想要在解释模式下再运行一次，因为这部分的表达式会运行两次。

在选择模式后，可以使用`SpelParserConfiguration`来配置解析器。如下所示：

```
SpelParserConfiguration config = new SpelParserConfiguration(SpelCompilerMode.IMMEDIATE,
    this.getClass().getClassLoader());

SpelExpressionParser parser = new SpelExpressionParser(config);

Expression expr = parser.parseExpression("payload");

MyMessage message = new MyMessage();

Object payload = expr.getValue(message);
```

当你指定编译器模式时，你可以指定一个类加载器（也可以传入null），被编译的表达式会在使用这个类加载器创建的子类加载器中定义。如果指定了一个类加载器，那么要确保它可以发现所有在表达式执行过程中参与的类型。如果你没有指定类加载器，那么将会使用一个默认的类加载器（一般是thread context classloader）。

第二种配置编译器的方法在SpEL被植入到其他组件内部，并且不可能通过配置对象来配置它时使用。在这种情况下，可以使用系统属性。你可以使用`SpelCompilerMode`枚举中任意一个值(`off`, `immediate` 或 `mixed`)来设置`spring.expression.compiler.mode`属性。

##### Compiler Limitations

从Spring 4.1开始，加入了一个基础的编译器框架。但是，这个框架还不能支持任意种类的表达式，它的初始目标是在性能关键的上下文中使用的常见表达式。下面种类的表达式现在都还不能被编译：

- 包含赋值的表达式
- 依赖conversion service的表达式
- 使用自定义解析器或访问器的表达式
- 使用选择或投影[projection]的表达式

更多类型的表达式将在未来能够被编译。

### 4.2\. Expressions in Bean Definitions

你可以在基于XML或者基于注解的配置元数据中使用SpEL表达式。在这种情况下，定义表达式的语法是`#{ <expression string> }`。

#### 4.2.1\. XML Configuration

可以使用表达式来设置属性或构造方法的参数，如下所示：

```
<bean id="numberGuess" class="org.spring.samples.NumberGuess">
    <property name="randomNumber" value="#{ T(java.lang.Math).random() * 100.0 }"/>

    <!-- other properties -->
</bean>
```

`systemProperties`变量是预先定义的，所以你可以在你的表达式中使用它，如下所示：

```
<bean id="taxCalculator" class="org.spring.samples.TaxCalculator">
    <property name="defaultLocale" value="#{ systemProperties['user.region'] }"/>

    <!-- other properties -->
</bean>
```

注意你不需要在这个上下文中使用`#`符号作为这个预先定义的变量的前缀。

你可以通过名字引用其他的bean，如下所示：

```
<bean id="numberGuess" class="org.spring.samples.NumberGuess">
    <property name="randomNumber" value="#{ T(java.lang.Math).random() * 100.0 }"/>

    <!-- other properties -->
</bean>

<bean id="shapeGuess" class="org.spring.samples.ShapeGuess">
    <property name="initialShapeSeed" value="#{ numberGuess.randomNumber }"/>

    <!-- other properties -->
</bean>
```

#### 4.2.2\. Annotation Configuration

为了指定一个默认值，你可以在字段，方法以及方法或构造方法的参数上使用`@Value`注解。

下面的例子设置了字段变量的默认值：

```
public static class FieldValueTestBean

    @Value("#{ systemProperties['user.region'] }")
    private String defaultLocale;

    public void setDefaultLocale(String defaultLocale) {
        this.defaultLocale = defaultLocale;
    }

    public String getDefaultLocale() {
        return this.defaultLocale;
    }

}
```

下面的例子与前面的相同，但是在setter方法上指定：

```
public static class PropertyValueTestBean

    private String defaultLocale;

    @Value("#{ systemProperties['user.region'] }")
    public void setDefaultLocale(String defaultLocale) {
        this.defaultLocale = defaultLocale;
    }

    public String getDefaultLocale() {
        return this.defaultLocale;
    }

}
```

自动注入的方法以及构造方法也可以使用`@Value`注解，如下所示：

```
public class SimpleMovieLister {

    private MovieFinder movieFinder;
    private String defaultLocale;

    @Autowired
    public void configure(MovieFinder movieFinder,
            @Value("#{ systemProperties['user.region'] }") String defaultLocale) {
        this.movieFinder = movieFinder;
        this.defaultLocale = defaultLocale;
    }

    // ...
}
```

```
public class MovieRecommender {

    private String defaultLocale;

    private CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public MovieRecommender(CustomerPreferenceDao customerPreferenceDao,
            @Value("#{systemProperties['user.country']}") String defaultLocale) {
        this.customerPreferenceDao = customerPreferenceDao;
        this.defaultLocale = defaultLocale;
    }

    // ...
}
```

### 4.3\. Language Reference

本节介绍了Spring Expression Language如何工作，它包含了下面的话题：

*   [Literal Expressions](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-ref-literal)
*   [Properties, Arrays, Lists, Maps, and Indexers](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-properties-arrays)
*   [Inline Lists](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-inline-lists)
*   [Inline Maps](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-inline-maps)
*   [Array Construction](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-array-construction)
*   [Methods](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-methods)
*   [Operators](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-operators)
*   [Types](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-types)
*   [Constructors](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-constructors)
*   [Variables](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-ref-variables)
*   [Functions](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-ref-functions)
*   [Bean References](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-bean-references)
*   [Ternary Operator (If-Then-Else)](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-operator-ternary)
*   [The Elvis Operator](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-operator-elvis)
*   [Safe Navigation Operator](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-operator-safe-navigation)

#### 4.3.1\. Literal Expressions

支持的字面值表达式的类型是字符串，数值(int, real, hex)，boolean以及null。字符串可以通过单引号标识，为了在字符串中使用单引号来标记自身是一个字符串，可以使用两个单引号来标识字符。

下面的例子展示了字面量的简单使用。一般来说，它们不会像例子中那样隔离使用，而是作为一个更复杂的表达式的一部分 —— 例如，使用字面量作为逻辑比较操作符的一边。

```
ExpressionParser parser = new SpelExpressionParser();

// evals to "Hello World"
String helloWorld = (String) parser.parseExpression("'Hello World'").getValue();

double avogadrosNumber = (Double) parser.parseExpression("6.0221415E+23").getValue();

// evals to 2147483647
int maxValue = (Integer) parser.parseExpression("0x7FFFFFFF").getValue();

boolean trueValue = (Boolean) parser.parseExpression("true").getValue();

Object nullValue = parser.parseExpression("null").getValue();
```

数字支持使用负号，指数表达式以及小数点。默认情况下，浮点数使用`Double.parseDouble()`方法解析。

#### 4.3.2\. Properties, Arrays, Lists, Maps, and Indexers

使用属性引用是简单的，可以使用一个点来指示嵌套的属性值。`Inventor`类的实例`pupin` 和` tesla`使用在 [Classes used in the examples](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-example-classes) 中的数据注入。为了获取Tesla的出生年以及Pupin的的出生城市，我们使用下面的表达式：

```
// evals to 1856
int year = (Integer) parser.parseExpression("Birthdate.Year + 1900").getValue(context);

String city = (String) parser.parseExpression("placeOfBirth.City").getValue(context);
```

属性名称的第一个字符大小写不敏感。数组和列表的内容可以使用方括号来获取，如下所示：

```
ExpressionParser parser = new SpelExpressionParser();
EvaluationContext context = SimpleEvaluationContext.forReadOnlyDataBinding().build();

// Inventions Array

// evaluates to "Induction motor"
String invention = parser.parseExpression("inventions[3]").getValue(
        context, tesla, String.class);

// Members List

// evaluates to "Nikola Tesla"
String name = parser.parseExpression("Members[0].Name").getValue(
        context, ieee, String.class);

// List and Array navigation
// evaluates to "Wireless communication"
String invention = parser.parseExpression("Members[0].Inventions[6]").getValue(
        context, ieee, String.class);
```

map中的内容可以在方括号中指定字面值key来获取。在下面的例子中，因为`Officers` map的key是字符串，所有我们可以指定字符串字面值：

```
// Officer's Dictionary

Inventor pupin = parser.parseExpression("Officers['president']").getValue(
        societyContext, Inventor.class);

// evaluates to "Idvor"
String city = parser.parseExpression("Officers['president'].PlaceOfBirth.City").getValue(
        societyContext, String.class);

// setting values
parser.parseExpression("Officers['advisors'][0].PlaceOfBirth.Country").setValue(
        societyContext, "Croatia");
```

#### 4.3.3\. Inline Lists

你可以直接使用`{}`符号来在表达式中表示列表。

```
// evaluates to a Java list containing the four numbers
List numbers = (List) parser.parseExpression("{1,2,3,4}").getValue(context);

List listOfLists = (List) parser.parseExpression("{{'a','b'},{'x','y'}}").getValue(context);
```

`{}`本身意味着一个空列表。由于性能原因，如果列表本身完全由混合的字面量组成，那么将会创建一个常量列表来表示表达式（而不是在每次执行时都建立一个新的列表）。

#### 4.3.4\. Inline Maps

你也可以直接使用`{key:value}`符号在表达式中表示map。如下所示：

```
// evaluates to a Java map containing the two entries
Map inventorInfo = (Map) parser.parseExpression("{name:'Nikola',dob:'10-July-1856'}").getValue(context);

Map mapOfMaps = (Map) parser.parseExpression("{name:{first:'Nikola',last:'Tesla'},dob:{day:10,month:'July',year:1856}}").getValue(context);
```

`{:}`本身意味着一个空map。由于性能原因，如果map本身由复杂的字面量或者其他嵌套的常量结构（list或者map）组成，那么将会创建一个常量map来表示表达式（而不是在每次执行时都建立一个新的map）。map的键的引用是可选的，上面的例子没有使用任何键引用。

#### 4.3.5\. Array Construction

你可以通过与Java相同的语法来建立数组，可以选择提供一个初始化器来让数组在构建时被填充。如下所示：

```
int[] numbers1 = (int[]) parser.parseExpression("new int[4]").getValue(context);

// Array with initializer
int[] numbers2 = (int[]) parser.parseExpression("new int[]{1,2,3}").getValue(context);

// Multi dimensional array
int[][] numbers3 = (int[][]) parser.parseExpression("new int[4][5]").getValue(context);
```

现在你还不能在构造多维数组的时候提供初始化器。

#### 4.3.6\. Methods

你可以使用标准的Java编程语法调用方法，你也可以在字面量上调用方法，还支持变量参数。下面展示如何调用方法：

```
// string literal, evaluates to "bc"
String bc = parser.parseExpression("'abc'.substring(1, 3)").getValue(String.class);

// evaluates to true
boolean isMember = parser.parseExpression("isMember('Mihajlo Pupin')").getValue(
        societyContext, Boolean.class);
```

#### 4.3.7\. Operators

Spring Expression Language支持下面种类的操作符：

*   [关系运算符](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-operators-relational)

*   [逻辑运算符](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-operators-logical)

*   [数学运算发](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-operators-mathematical)

*   [赋值运算符](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions-assignment)

##### Relational Operators

关系运算符(==, !=, <, <=, >, >=)使用标准的操作符符号支持。下面展示了运算符的几个例子：

```
// evaluates to true
boolean trueValue = parser.parseExpression("2 == 2").getValue(Boolean.class);

// evaluates to false
boolean falseValue = parser.parseExpression("2 < -5.0").getValue(Boolean.class);

// evaluates to true
boolean trueValue = parser.parseExpression("'black' < 'block'").getValue(Boolean.class);
```

> 大于和小于比较遇到`null`时遵循一个简单的规则：`null`被视作没有（**不是**0）。因此，任何其他值总是会大于`null` (`X > null` 永远是 `true`) 并且没有别的值比`null`更小 (`X < null` 永远是 `false`)。<br/>
如果你更喜欢使用数值比较，注意避免使用`null`，而是通过0进行比较（例如，`X > 0` 或 `X < 0`）。

除了标准的关系运算符，SpEL还支持`instanceof`和基于正则表达式的`matches`操作符。如下所示：

```
// evaluates to false
boolean falseValue = parser.parseExpression(
        "'xyz' instanceof T(Integer)").getValue(Boolean.class);

// evaluates to true
boolean trueValue = parser.parseExpression(
        "'5.00' matches '^-?\\d+(\\.\\d{2})?$'").getValue(Boolean.class);

//evaluates to false
boolean falseValue = parser.parseExpression(
        "'5.0067' matches '^-?\\d+(\\.\\d{2})?$'").getValue(Boolean.class);
```

> 小心对待基本类型，因为它们立即被装箱成为包装类型，因此`1 instanceof T(int)`执行的结果是`false`，而`1 instanceof T(Integer)`执行结果是`true`。

每一个符号运算符与字母形式表达的运算符是完全相同的。这避免了当表达式被植入到其他组件（例如XML文件）中时符号拥有特殊意义的问题。文本形式等同如下：

*   `lt` (`<`)
*   `gt` (`>`)
*   `le` (`<=`)
*   `ge` (`>=`)
*   `eq` (`==`)
*   `ne` (`!=`)
*   `div` (`/`)
*   `mod` (`%`)
*   `not` (`!`).

所有的文本表达式都是大小不敏感的。

##### Logical Operators

SpEL支持下面的逻辑运算符：

*   `and`
*   `or`
*   `not`

下面展示了如何使用逻辑运算符：

```
// -- AND --

// evaluates to false
boolean falseValue = parser.parseExpression("true and false").getValue(Boolean.class);

// evaluates to true
String expression = "isMember('Nikola Tesla') and isMember('Mihajlo Pupin')";
boolean trueValue = parser.parseExpression(expression).getValue(societyContext, Boolean.class);

// -- OR --

// evaluates to true
boolean trueValue = parser.parseExpression("true or false").getValue(Boolean.class);

// evaluates to true
String expression = "isMember('Nikola Tesla') or isMember('Albert Einstein')";
boolean trueValue = parser.parseExpression(expression).getValue(societyContext, Boolean.class);

// -- NOT --

// evaluates to false
boolean falseValue = parser.parseExpression("!true").getValue(Boolean.class);

// -- AND and NOT --
String expression = "isMember('Nikola Tesla') and !isMember('Mihajlo Pupin')";
boolean falseValue = parser.parseExpression(expression).getValue(societyContext, Boolean.class);
```

##### Mathematical Operators

你可以在数字和字符串中使用加法运算符，同时你只可以使用在数字上使用减法，乘法，除法，取模(%)以及指数幂(^)运算符。强制使用标准的运算符优先级。下面展示了数学运算符的使用：

```
// Addition
int two = parser.parseExpression("1 + 1").getValue(Integer.class);  // 2

String testString = parser.parseExpression(
        "'test' + ' ' + 'string'").getValue(String.class);  // 'test string'

// Subtraction
int four = parser.parseExpression("1 - -3").getValue(Integer.class);  // 4

double d = parser.parseExpression("1000.00 - 1e4").getValue(Double.class);  // -9000

// Multiplication
int six = parser.parseExpression("-2 * -3").getValue(Integer.class);  // 6

double twentyFour = parser.parseExpression("2.0 * 3e0 * 4").getValue(Double.class);  // 24.0

// Division
int minusTwo = parser.parseExpression("6 / -3").getValue(Integer.class);  // -2

double one = parser.parseExpression("8.0 / 4e0 / 2").getValue(Double.class);  // 1.0

// Modulus
int three = parser.parseExpression("7 % 4").getValue(Integer.class);  // 3

int one = parser.parseExpression("8 / 5 % 2").getValue(Integer.class);  // 1

// Operator precedence
int minusTwentyOne = parser.parseExpression("1+2-3*8").getValue(Integer.class);  // -21
```

##### The Assignment Operator

为了设置属性，你可以使用赋值运算符(`=`)。这一般可以调用`setValue`方法完成，但也可以调用`getValue`方法完成，如下所示：

```
Inventor inventor = new Inventor();
EvaluationContext context = SimpleEvaluationContext.forReadWriteDataBinding().build();

parser.parseExpression("Name").setValue(context, inventor, "Aleksandar Seovic");

// alternatively
String aleks = parser.parseExpression(
        "Name = 'Aleksandar Seovic'").getValue(context, inventor, String.class);
```

#### 4.3.8\. Types

你可以使用特殊的 `T` 操作符来指定`java.lang.Class`（类型）的实例，也可以使用这个运算符调用静态方法。`StandardEvaluationContext`使用`TypeLocator`来发现类型，标准的`StandardTypeLocator`（可以被替换）类在`java.lang`包中内建。这意味着使用`T()`引用`java.lang`包中的类型时无需提供全限定名，但是所有其他的类型都必须提供全限定名。下面展示了如何使用`T`操作符：

```
Class dateClass = parser.parseExpression("T(java.util.Date)").getValue(Class.class);

Class stringClass = parser.parseExpression("T(String)").getValue(Class.class);

boolean trueValue = parser.parseExpression(
        "T(java.math.RoundingMode).CEILING < T(java.math.RoundingMode).FLOOR")
        .getValue(Boolean.class);
```

#### 4.3.9\. Constructors

你可以通过`new`操作符调用构造方法。你应当使用全限定类名，除了基本类型(`int`, `float` 等等)和String。下面展示了如何使用`new`操作符来调用构造方法：

```
Inventor einstein = p.parseExpression(
        "new org.spring.samples.spel.inventor.Inventor('Albert Einstein', 'German')")
        .getValue(Inventor.class);

//create new inventor instance within add method of List
p.parseExpression(
        "Members.add(new org.spring.samples.spel.inventor.Inventor(
            'Albert Einstein', 'German'))").getValue(societyContext);
```

#### 4.3.10\. Variables

你可以使用`#variableName`来引用表达式中的变量。变量可以通过`EvaluationContext`实现类的`setVariable`方法设置，如下所示：

```
Inventor tesla = new Inventor("Nikola Tesla", "Serbian");

EvaluationContext context = SimpleEvaluationContext.forReadWriteDataBinding().build();
context.setVariable("newName", "Mike Tesla");

parser.parseExpression("Name = #newName").getValue(context, tesla);
System.out.println(tesla.getName());  // "Mike Tesla"
```

##### The `#this` and `#root` Variables

`#this`变量是预先被定义好的，它引用了当前的evaluation对象。`#this`变量也是预先定义好的，它引用了root context对象。虽然`this`会随着表达式组件的执行改变，但是`root`变量永远引用root。下面的例子展示了如何使用它们：

```
// create an array of integers
List<Integer> primes = new ArrayList<Integer>();
primes.addAll(Arrays.asList(2,3,5,7,11,13,17));

// create parser and set variable 'primes' as the array of integers
ExpressionParser parser = new SpelExpressionParser();
EvaluationContext context = SimpleEvaluationContext.forReadOnlyDataAccess();
context.setVariable("primes", primes);

// all prime numbers > 10 from the list (using selection ?{...})
// evaluates to [11, 13, 17]
List<Integer> primesGreaterThanTen = (List<Integer>) parser.parseExpression(
        "#primes.?[#this>10]").getValue(context);
```

#### 4.3.11\. Functions

你可以注册在表达式中可以被调用的用户自定义的函数来拓展SpEL。函数通过`EvaluationContext`来注册，下面展示如何注册自定义函数：

```
Method method = ...;

EvaluationContext context = SimpleEvaluationContext.forReadOnlyDataBinding().build();
context.setVariable("myFunction", method);
```

例如，考虑下面的反转字符串的方法：

```
public abstract class StringUtils {

    public static String reverseString(String input) {
        StringBuilder backwards = new StringBuilder(input.length());
        for (int i = 0; i < input.length(); i++)
            backwards.append(input.charAt(input.length() - 1 - i));
        }
        return backwards.toString();
    }
}
```

你可以注册并使用它，如下所示：

```
ExpressionParser parser = new SpelExpressionParser();

EvaluationContext context = SimpleEvaluationContext.forReadOnlyDataBinding().build();
context.setVariable("reverseString",
        StringUtils.class.getDeclaredMethod("reverseString", String.class));

String helloWorldReversed = parser.parseExpression(
        "#reverseString('hello')").getValue(context, String.class);
```

#### 4.3.12\. Bean References

如果evaluation context中使用了一个bean解析器配置，那么你可以使用`@`符号在表达式中寻找bean，如下所示：

```
ExpressionParser parser = new SpelExpressionParser();
StandardEvaluationContext context = new StandardEvaluationContext();
context.setBeanResolver(new MyBeanResolver());

// This will end up calling resolve(context,"something") on MyBeanResolver during evaluation
Object bean = parser.parseExpression("@something").getValue(context);
```

如果要获取FactoryBean本身，你可以将bean name的前缀换成`&`符号，如下所示：

```
ExpressionParser parser = new SpelExpressionParser();
StandardEvaluationContext context = new StandardEvaluationContext();
context.setBeanResolver(new MyBeanResolver());

// This will end up calling resolve(context,"&foo") on MyBeanResolver during evaluation
Object bean = parser.parseExpression("&foo").getValue(context);
```

#### 4.3.13\. Ternary Operator (If-Then-Else)

你可以使用三元运算符在表达式中执行if-then-else条件逻辑，下面展示一个小例子：

```
String falseString = parser.parseExpression(
        "false ? 'trueExp' : 'falseExp'").getValue(String.class);
```

在这种情况下，boolean `false`会导致返回字符串`'falseExp'`。一个更加现实的例子如下：

```
parser.parseExpression("Name").setValue(societyContext, "IEEE");
societyContext.setVariable("queryName", "Nikola Tesla");

expression = "isMember(#queryName)? #queryName + ' is a member of the ' " +
        "+ Name + ' Society' : #queryName + ' is not a member of the ' + Name + ' Society'";

String queryResultString = parser.parseExpression(expression)
        .getValue(societyContext, String.class);
// queryResultString = "Nikola Tesla is a member of the IEEE Society"
```

查看下一节的Elvis Operator获取三元运算符的更短语法。

#### 4.3.14\. The Elvis Operator

Elvis operator是三元运算符的更短语法表示，它用于[Groovy](http://www.groovy-lang.org/operators.html#_elvis_operator)语言中。使用三元运算符时，你一般需要重复变量两次，如下所示：

```
String name = "Elvis Presley";
String displayName = (name != null ? name : "Unknown");
```

作为代替，你可以使用Elvis operator，下面展示了如何使用它：

```
ExpressionParser parser = new SpelExpressionParser();

String name = parser.parseExpression("name?:'Unknown'").getValue(String.class);
System.out.println(name);  // 'Unknown'
```

下面展示一个更复杂的例子：

```
ExpressionParser parser = new SpelExpressionParser();
EvaluationContext context = SimpleEvaluationContext.forReadOnlyDataBinding().build();

Inventor tesla = new Inventor("Nikola Tesla", "Serbian");
String name = parser.parseExpression("Name?:'Elvis Presley'").getValue(context, tesla, String.class);
System.out.println(name);  // Nikola Tesla

tesla.setName(null);
name = parser.parseExpression("Name?:'Elvis Presley'").getValue(context, tesla, String.class);
System.out.println(name);  // Elvis Presley
```

你可以使用Elvis operator在表达式中应用默认值。下面展示了如何在`@Value`中使用Elvis operator：

```
@Value("#{systemProperties['pop3.port'] ?: 25}")
```

如果系统属性`pop3.port`定义了，那么注入它，否则注入25。

#### 4.3.15\. Safe Navigation Operator

安全导引运算符（Safe navigation operator）被用来避免`NullPointerException`异常，它来自于[Groovy](http://www.groovy-lang.org/operators.html#_safe_navigation_operator)语言。一般来说，当你有一个对象的引用时，你可能需要在使用它之前确认它是非空的。为了避免检验，安全导引运算符返回null而不是抛出一个异常。下面展示了如何使用它：

```
ExpressionParser parser = new SpelExpressionParser();
EvaluationContext context = SimpleEvaluationContext.forReadOnlyDataBinding().build();

Inventor tesla = new Inventor("Nikola Tesla", "Serbian");
tesla.setPlaceOfBirth(new PlaceOfBirth("Smiljan"));

String city = parser.parseExpression("PlaceOfBirth?.City").getValue(context, tesla, String.class);
System.out.println(city);  // Smiljan

tesla.setPlaceOfBirth(null);
city = parser.parseExpression("PlaceOfBirth?.City").getValue(context, tesla, String.class);
System.out.println(city);  // null - does not throw NullPointerException!!!
```

#### 4.3.16\. Collection Selection

选择是一个强有力的表达式语言的特性，它可以让你将源集合在选择后转换为另一个集合。

选择的语法是`.?[selectionExpression]`。它过滤了源集合并且返回一个包含原始集合元素子集的新集合。如下所示：

```
List<Inventor> list = (List<Inventor>) parser.parseExpression(
        "Members.?[Nationality == 'Serbian']").getValue(societyContext);
```

选择在list和map中都能使用。对于list，选择规则在每一个独立的列表元素上执行，而对于map，选择规则会在每一个map条目（Java type `Map.Entry`）上执行。每一个map条目在选择时都将它的键和值作为属性使用。

下面的表达式返回了条目值小于27的子集：

```
Map newMap = parser.parseExpression("map.?[value<27]").getValue();
```

除了返回所有被选择的元素外，你还可以只提取第一个和最后一个值。如果想要获取第一个匹配的条目，语法是`.^[selectionExpression]`，获取最后一个匹配条目的语法是`.$[selectionExpression]`。

#### 4.3.17\. Collection Projection

映射让一个集合进行子表达式的执行，并且返回值是一个新的集合，映射的语法是`.![projectionExpression]`。例如，假设我们拥有一个inventor列表，但我们想要获取它们出生城市的列表。最有效的方法是，对于inventor列表中的每一个元素执行 'placeOfBirth.city'，如下所示：

```
// returns ['Smiljan', 'Idvor' ]
List placesOfBirth = (List)parser.parseExpression("Members.![placeOfBirth.city]");
```

你也可以在map中进行映射，在这种情况下，映射表达式在map上的每一个条目执行，遍历map后的映射结果是一个列表。

#### 4.3.18\. Expression templating

表达式模板允许使用一个或多个执行块来融合字面值，每一个执行块通过你定义的前缀和后缀符号来分割。一个常用的选择是使用`#{ }`作为分隔符，如下所示：

```
String randomPhrase = parser.parseExpression(
        "random number is #{T(java.lang.Math).random()}",
        new TemplateParserContext()).getValue(String.class);

// evaluates to "random number is 0.7038186818312008"
```

通过连结`'random number is '`字面量和`#{ }`分隔符中的表达式的执行结果，获取最终的执行结果。`parseExpression()`方法的第二个参数是`ParserContext`类型。`ParserContext`接口用来影响表达式如何被解析，以此来支持模板表达式特性。`TemplateParserContext`的定义如下所示：

```
public class TemplateParserContext implements ParserContext {

    public String getExpressionPrefix() {
        return "#{";
    }

    public String getExpressionSuffix() {
        return "}";
    }

    public boolean isTemplate() {
        return true;
    }
}
```

### 4.4\. Classes Used in the Examples

本节列出了在整个章节中使用的示例类。

Example 1\. Inventor.java

```
package org.spring.samples.spel.inventor;

import java.util.Date;
import java.util.GregorianCalendar;

public class Inventor {

    private String name;
    private String nationality;
    private String[] inventions;
    private Date birthdate;
    private PlaceOfBirth placeOfBirth;

    public Inventor(String name, String nationality) {
        GregorianCalendar c= new GregorianCalendar();
        this.name = name;
        this.nationality = nationality;
        this.birthdate = c.getTime();
    }

    public Inventor(String name, Date birthdate, String nationality) {
        this.name = name;
        this.nationality = nationality;
        this.birthdate = birthdate;
    }

    public Inventor() {
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getNationality() {
        return nationality;
    }

    public void setNationality(String nationality) {
        this.nationality = nationality;
    }

    public Date getBirthdate() {
        return birthdate;
    }

    public void setBirthdate(Date birthdate) {
        this.birthdate = birthdate;
    }

    public PlaceOfBirth getPlaceOfBirth() {
        return placeOfBirth;
    }

    public void setPlaceOfBirth(PlaceOfBirth placeOfBirth) {
        this.placeOfBirth = placeOfBirth;
    }

    public void setInventions(String[] inventions) {
        this.inventions = inventions;
    }

    public String[] getInventions() {
        return inventions;
    }
}
```

Example 2\. PlaceOfBirth.java

```
package org.spring.samples.spel.inventor;

public class PlaceOfBirth {

    private String city;
    private String country;

    public PlaceOfBirth(String city) {
        this.city=city;
    }

    public PlaceOfBirth(String city, String country) {
        this(city);
        this.country = country;
    }

    public String getCity() {
        return city;
    }

    public void setCity(String s) {
        this.city = s;
    }

    public String getCountry() {
        return country;
    }

    public void setCountry(String country) {
        this.country = country;
    }

}
```

Example 3\. Society.java

```
package org.spring.samples.spel.inventor;

import java.util.*;

public class Society {

    private String name;

    public static String Advisors = "advisors";
    public static String President = "president";

    private List<Inventor> members = new ArrayList<Inventor>();
    private Map officers = new HashMap();

    public List getMembers() {
        return members;
    }

    public Map getOfficers() {
        return officers;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public boolean isMember(String name) {
        for (Inventor inventor : members) {
            if (inventor.getName().equals(name)) {
                return true;
            }
        }
        return false;
    }

}
```

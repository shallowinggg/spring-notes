Spring框架实现了控制反转(IoC)原则，而IoC也被称作是依赖注入(DI)。DI是一个过程：对象只通过构造函数的参数、工厂方法的参数、
在构造或从工厂方法返回后之后设置成员变量这些方式定义它们的依赖（一起工作的其他对象），然后Spring容器在创建bean时会将注入它们的依赖。
这个过程本质上是反转bean自己通过使用直接的类构造方法等机制控制自己实例化或者定位它的依赖。

``` Java
package ioc.entity;

public class People {
    private String name;
    private int age;

    public People() {}

    public People(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "People[name=" + name + ", age=" + age + "]";
    }
}
```

```
package ioc;

import ioc.entity.People;
import org.junit.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class IoCTest {
    @Test
    public void test() {
        People people1 = new People("aaa", 5);

        People people2 = PeopleFactory.newPeople("bbb", 10);

        People people3 = new People();
        people3.setName("ccc");
        people3.setAge(15);

        System.out.println(people1);
        System.out.println(people2);
        System.out.println(people3);
    }

    @Test
    public void testIoC() {
        ApplicationContext context = new ClassPathXmlApplicationContext("spring-config.xml");
        People people1 = context.getBean("people1", People.class);
        People people2 = context.getBean("people2", People.class);
        People people3 = context.getBean("people3", People.class);

        System.out.println(people1);
        System.out.println(people2);
        System.out.println(people3);
    }

    static class PeopleFactory {
        public static People newPeople(String name, int age) {
            return new People(name, age);
        }
    }
}
```

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans 
          http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="people1" class="ioc.entity.People">
        <constructor-arg name="name" value="aaa" />
        <constructor-arg name="age" value="5" />
    </bean>


    <bean id="people2" class="ioc.IoCTest.PeopleFactory" factory-method="newPeople">
        <constructor-arg name="name" value="bbb" />
        <constructor-arg name="age" value="10" />
    </bean>

    <bean id="people3" class="ioc.entity.People">
        <property name="name" value="ccc" />
        <property name="age" value="5" />
    </bean>
</beans>
```

test()方法即是传统的new对象，然后手动注入依赖的例子。
而testIoC()方法使用Spring容器来注入依赖，将创建对象以及依赖注入的职责由程序员自己交给Spring容器。




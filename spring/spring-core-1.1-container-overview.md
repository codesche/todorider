# Spring 공식문서 읽기: Core Technologies

## 원본 링크
https://codesche.oopy.io/28ade3f7-e3a8-8025-8636-e901d44f428a

## 참고 링크
[Container Overview](https://docs.spring.io/spring-framework/reference/core/beans/basics.html)

## 원문

The interface `org.springframework.context.ApplicationContext`
represents the Spring IoC container and is responsible for
instantiating, configuring, and assembling the beans.

The container gets its instructions on what objects to instantiate,
configure, and assemble by reading configuration metadata.

The configuration metadata is represented in XML, Java annotations, or
Java code.

It lets you express the objects that compose your application and the
rich interdependencies between such objects.

Several implementations of the `ApplicationContext` interface are
supplied with Spring.

In standalone applications, it is common to create an instance of
`ClassPathXmlApplicationContext` or `FileSystemXmlApplicationContext`.

While XML has been the traditional format for configuration metadata,
you can instruct the container to use Java annotations or Java code as
metadata instead, by using the `AnnotationConfigApplicationContext` or
by registering `@Configuration` classes.

The following example shows a simple way to configure, instantiate, and
use the container in Java:

``` java
package com.example.core;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import com.example.core.services.HelloWorldService;

public class HelloWorldApp {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
        HelloWorldService service = context.getBean("helloWorldService", HelloWorldService.class);
        service.sayHello();
    }
}
```

And here is the same example written in Kotlin:

``` kotlin
package com.example.core

import org.springframework.context.support.ClassPathXmlApplicationContext
import com.example.core.services.HelloWorldService

fun main() {
    val context = ClassPathXmlApplicationContext("beans.xml")
    val service = context.getBean("helloWorldService", HelloWorldService::class.java)
    service.sayHello()
}
```

The `beans.xml` file contains the configuration metadata that tells the
container how to instantiate and configure the beans:

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans
           https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="helloWorldService" class="com.example.core.services.HelloWorldService"/>
</beans>
```

The `ApplicationContext` is usually loaded at application startup and
remains alive throughout the lifecycle of the application.

It manages the complete lifecycle of beans --- from creation to
destruction --- and resolves dependencies between them.

------------------------------------------------------------------------

### 번역

`org.springframework.context.ApplicationContext` 인터페이스는 **Spring
IoC 컨테이너**를 나타내며,\
**빈(bean)의 생성, 설정, 조립**을 담당한다.

컨테이너는 어떤 객체를 생성하고 설정하며 연결할지를 **설정
메타데이터(configuration metadata)** 를 읽어 파악한다.\
이 설정 메타데이터는 **XML**, **Java 애노테이션**, 또는 **Java 코드**
형태로 표현될 수 있다.

즉, 애플리케이션을 구성하는 객체들과 그들 간의 복잡한 의존 관계를
선언적으로 표현할 수 있게 해준다.

Spring은 `ApplicationContext` 인터페이스의 여러 구현체를 제공한다.\
독립 실행형(standalone) 애플리케이션에서는 일반적으로\
`ClassPathXmlApplicationContext`나 `FileSystemXmlApplicationContext`
인스턴스를 생성한다.

XML이 전통적인 설정 방식이지만, `AnnotationConfigApplicationContext`를
사용하거나\
`@Configuration` 클래스를 등록함으로써 **Java 애노테이션 또는 Java 코드
기반 설정**을 사용할 수도 있다.

------------------------------------------------------------------------

다음은 Java로 작성된 간단한 Spring 컨테이너 구성 및 사용 예시이다.

``` java
package com.example.core;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import com.example.core.services.HelloWorldService;

public class HelloWorldApp {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
        HelloWorldService service = context.getBean("helloWorldService", HelloWorldService.class);
        service.sayHello();
    }
}
```

이 예제에서는 `beans.xml` 파일을 통해 컨테이너를 초기화하고,\
`HelloWorldService` 빈을 가져와(`getBean`) 메서드를 호출한다.\
즉, Spring 컨테이너가 객체의 생성과 의존성 관리를 모두 담당한다.

------------------------------------------------------------------------

같은 예제를 Kotlin으로 표현하면 다음과 같다.

``` kotlin
package com.example.core

import org.springframework.context.support.ClassPathXmlApplicationContext
import com.example.core.services.HelloWorldService

fun main() {
    val context = ClassPathXmlApplicationContext("beans.xml")
    val service = context.getBean("helloWorldService", HelloWorldService::class.java)
    service.sayHello()
}
```

Kotlin 버전에서도 동작 원리는 동일하다.\
`ApplicationContext`를 생성하고, 등록된 빈을 조회한 뒤(`getBean`),\
메서드를 호출함으로써 의존성 주입이 작동한다.

------------------------------------------------------------------------

다음은 컨테이너 구성을 정의한 **`beans.xml`** 예시이다.

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans
           https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="helloWorldService" class="com.example.core.services.HelloWorldService"/>
</beans>
```

이 설정 파일은 `helloWorldService` 빈이
`com.example.core.services.HelloWorldService` 클래스로부터 생성되어야
함을 지정한다.\
컨테이너는 이를 읽고 해당 클래스를 인스턴스화하여 관리한다.

일반적으로 `ApplicationContext`는 **애플리케이션 시작 시 로드**되며,\
애플리케이션의 전체 생명주기 동안 유지된다.\
컨테이너는 **빈의 생성부터 소멸까지의 전 과정을 관리**하며,\
빈 간의 **의존 관계를 자동으로 해결(autowire)** 한다.

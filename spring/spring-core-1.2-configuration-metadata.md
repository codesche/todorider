# 🌱 Spring 공식문서 읽기: Core Technologies

## 1.2. Configuration Metadata

## 원본 링크
https://codesche.oopy.io/28ede3f7-e3a8-80ab-b60a-fdbadfbbbc24

## 참고 링크
https://docs.spring.io/spring-framework/reference/core/beans/basics.html


As the preceding diagram shows, the Spring IoC container consumes a form
of configuration metadata.
This configuration metadata represents how you, as an application
developer, tell the Spring container to instantiate, configure, and
assemble the components in your application.

The Spring IoC container itself is totally decoupled from the format in
which this configuration metadata is actually written.
These days, many developers choose Java-based configuration for their
Spring applications:

-   **Annotation-based configuration**: define beans using
    annotation-based configuration metadata on your application's
    component classes.
-   **Java-based configuration**: define beans external to your
    application classes by using Java-based configuration classes.
    To use these features, see the `@Configuration`, `@Bean`, `@Import`,
    and `@DependsOn` annotations.

Spring configuration consists of at least one and typically more than
one bean definition that the container must manage.
Java configuration typically uses `@Bean`‑annotated methods within a
`@Configuration` class, each corresponding to one bean definition.

These bean definitions correspond to the actual objects that make up
your application.
Typically, you define service layer objects, persistence layer objects
such as repositories or data access objects (DAOs), presentation objects
such as Web controllers, infrastructure objects such as a JPA
`EntityManagerFactory`, JMS queues, and so forth.
Typically, one does not configure fine‑grained domain objects in the
container, because it is usually the responsibility of repositories and
business logic to create and load domain objects.

------------------------------------------------------------------------

### 🧩 XML as an External Configuration DSL

XML-based configuration metadata configures these beans as `<bean/>`
elements inside a top-level `<beans/>` element.\
The following example shows the basic structure of XML-based
configuration metadata:

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="..." class="..."> (1) (2)
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <bean id="..." class="...">
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions go here -->

</beans>
```

1.  The `id` attribute is a string that identifies the individual bean
    definition.
2.  The `class` attribute defines the type of the bean and uses the
    fully qualified class name.

The value of the `id` attribute can be used to refer to collaborating
objects.
The XML for referring to collaborating objects is not shown in this
example. See *Dependencies* for more information.

For instantiating a container, the location path or paths to the XML
resource files need to be supplied to a `ClassPathXmlApplicationContext`
constructor that let the container load configuration metadata from a
variety of external resources, such as the local file system, the Java
`CLASSPATH`, and so on.

**Java Example**:

``` java
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");
```

**Kotlin Example**:

``` kotlin
val context = ClassPathXmlApplicationContext("services.xml", "daos.xml")
```

After you learn about Spring's IoC container, you may want to know more
about Spring's `Resource` abstraction (as described in *Resources*)
which provides a convenient mechanism for reading an `InputStream` from
locations defined in a URI syntax.\
In particular, `Resource` paths are used to construct application
contexts, as described in *Application Contexts and Resource Paths.*

------------------------------------------------------------------------

### 🇰🇷 번역 + 설명

앞선 도표가 보여주듯, Spring의 IoC 컨테이너는 **설정
메타데이터(configuration metadata)** 형태의 정보를 소비한다.
이 설정 메타데이터는 애플리케이션 개발자인 당신이 Spring 컨테이너에
**어떻게 구성 요소들을 생성하고, 설정하고, 조립할지를 지시하는
방식**이다.

Spring IoC 컨테이너는 이 메타데이터의 **표현 방식(format)** 에 전혀
종속되지 않는다.
즉, XML이든 Java 코드든 애노테이션이든 자유롭게 바꿔 사용할 수 있다.
요즘엔 많은 개발자가 **Java 기반 설정(Java-based configuration)** 을
선호한다:

-   **어노테이션 기반 설정**: 애플리케이션의 구성 클래스나 컴포넌트
    클래스에 빈(bean) 정의를 위한 어노테이션을 붙이는 방식
-   **Java 기반 설정**: 애플리케이션 클래스 밖에 별도의 설정 클래스를
    정의하여 빈을 만드는 방식 (`@Configuration`, `@Bean`, `@Import`,
    `@DependsOn` 등 사용)

Spring 설정은 적어도 하나 이상의 **bean 정의(bean definition)** 를
포함해야 하고,
실제로는 여러 개의 bean 정의를 갖는 것이 일반적이다.
Java 기반 설정에서는 `@Configuration` 클래스 내부의 `@Bean` 메서드들이
각각 하나의 bean 정의와 매핑된다.

이 bean 정의들은 애플리케이션을 구성하는 실제 객체들에 대응한다.
일반적으로 서비스 계층(service), 영속 계층(persistence, 예: DAO),
프레젠테이션 계층(Web 컨트롤러), 인프라 계층(JPA `EntityManagerFactory`,
JMS 큐 등)이 그 대상이다.
세밀한 도메인 객체까지 빈으로 등록하는 것은 일반적이지 않으며,\
이는 레포지토리나 비즈니스 로직에서 관리하는 것이 자연스럽다.

------------------------------------------------------------------------

#### 🧾 외부 DSL로서의 XML 설정

XML 기반 설정 메타데이터는 `<beans>` 요소 내에 여러 `<bean/>` 요소를
포함한다.
다음은 XML 설정 메타데이터의 기본 구조 예시이다:

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="..." class="..."> (1) (2)
        <!-- 이 빈의 연관 객체(collaborators) 및 설정은 여기 -->
    </bean>

    <bean id="..." class="...">
        <!-- 이 빈의 연관 객체 및 설정은 여기 -->
    </bean>

    <!-- 더 많은 빈 정의가 여기에 올 수 있음 -->

</beans>
```

1.  `id` 속성은 개별 빈 정의를 식별하는 문자열이다.
2.  `class` 속성은 빈의 타입을 정의하며, **완전한 패키지 경로가 포함된
    클래스 이름(FQCN)** 을 사용한다.

`id` 속성은 다른 빈을 참조(reference)할 때 사용된다.
XML에서의 참조 방식은 *Dependencies(의존성)* 절에서 자세히 다룬다.

컨테이너를 인스턴스화할 때는, XML 자원 파일 경로들을
`ClassPathXmlApplicationContext` 생성자에 전달해야 한다.
이 생성자는 로컬 파일 시스템, 클래스패스(Classpath) 등 다양한 위치에서
설정 메타데이터를 로드할 수 있다.

-   **Java 예시**:

    ``` java
    ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");
    ```

-   **Kotlin 예시**:

    ``` kotlin
    val context = ClassPathXmlApplicationContext("services.xml", "daos.xml")
    ```

Spring의 IoC 컨테이너를 이해하게 되면,
Spring의 `Resource` 추상화를 함께 살펴보는 것이 좋다.
이 추상화는 URI 문법으로 정의된 위치에서 `InputStream`을 읽는 편리한
메커니즘을 제공하며,
특히 `Resource` 경로는 애플리케이션 컨텍스트를 구성할 때 사용된다.
자세한 내용은 *Application Contexts and Resource Paths* 절에서 다룬다.

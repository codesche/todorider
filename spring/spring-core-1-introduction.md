# 🌱 Spring Framework: Core Technologies

## 1. Introduction to the Spring IoC Container and Beans

------------------------------------------------------------------------

### 🧩 원문

This chapter covers the Spring Framework implementation of the Inversion
of Control (IoC) principle.\
IoC is also known as *dependency injection (DI)*. It is a process
whereby objects define their dependencies (that is, the other objects
they work with) only through constructor arguments, arguments to a
factory method, or properties that are set on the object instance after
it is constructed or returned from a factory method. The container then
injects those dependencies when it creates the bean. This process is
fundamentally the inverse (hence the name, *Inversion of Control*) of
the bean itself controlling the instantiation or location of its
dependencies by using direct construction of classes or a Service
Locator pattern.

The `org.springframework.beans` and `org.springframework.context`
packages are the basis for Spring Framework's IoC container.\
The `BeanFactory` interface provides an advanced configuration mechanism
capable of managing any type of object.\
The `ApplicationContext` interface builds on top of the `BeanFactory`
(it is a sub-interface) and adds other functionality such as easier
integration with Spring's AOP features, message resource handling (for
use in internationalization), event publication, and application-layer
specific context support such as the `WebApplicationContext` for use in
web applications.

In short, the `BeanFactory` provides the configuration framework and
basic functionality, while the `ApplicationContext` adds more
enterprise-specific functionality. The `ApplicationContext` is a
superset of the `BeanFactory` and is used in most Spring-based
applications.

------------------------------------------------------------------------

### 🇰🇷 번역

이 장에서는 **Spring Framework에서 구현된 IoC(Inversion of Control,
제어의 역전)** 원리를 다룬다.\
IoC는 흔히 **의존성 주입(Dependency Injection, DI)** 이라고도 불린다.\
이 개념은 객체가 자신이 의존하는 다른 객체(즉, 함께 동작하는 객체들)를
직접 생성하거나 찾는 대신,\
**생성자 인자(constructor arguments)**, **팩토리 메서드의 인자(arguments
to a factory method)**,\
또는 **객체 인스턴스 생성 이후 설정되는 프로퍼티(properties)** 를
통해서만 정의하는 과정을 말한다.\
컨테이너는 이러한 의존성을 관리하고, **빈(bean)을 생성할 때 필요한
의존성을 주입(inject)** 한다.

이 과정은 이름 그대로 *제어의 역전(Inversion of Control)* 이다.\
즉, 객체가 직접 필요한 의존성을 생성하거나 찾는 대신(Spring 이전의
일반적인 방식처럼),\
컨테이너가 그 의존성을 대신 관리하고 주입하는 방식으로 제어의 주체가
반전되는 것이다.\
이것은 클래스의 직접 생성(new 연산)이나 **Service Locator 패턴**을
사용하는 것과는 근본적으로 반대되는 접근이다.

------------------------------------------------------------------------

`org.springframework.beans` 와 `org.springframework.context` 패키지는
Spring Framework의 **IoC 컨테이너의 핵심 기반**이다.\
`BeanFactory` 인터페이스는 모든 종류의 객체를 관리할 수 있는 **고급 구성
메커니즘(configuration mechanism)** 을 제공한다.\
`ApplicationContext` 인터페이스는 `BeanFactory`를 확장(하위
인터페이스)하여 다음과 같은 기능을 추가로 제공한다:

-   Spring AOP 기능과의 손쉬운 통합\
-   국제화(i18n)를 위한 **메시지 리소스 처리(message resource
    handling)**\
-   이벤트 발행(event publication) 기능\
-   웹 애플리케이션 전용 컨텍스트(`WebApplicationContext`) 등,
    **애플리케이션 계층별 컨텍스트 지원**

------------------------------------------------------------------------

요약하자면,\
`BeanFactory`는 **구성 및 기본 기능(configuration & core
functionality)** 을 담당하고,\
`ApplicationContext`는 **기업용(enterprise-level) 기능**을 추가로
제공한다.\
즉, `ApplicationContext`는 `BeanFactory`의 상위 개념이자 확장된 형태로,\
대부분의 Spring 기반 애플리케이션에서는 `ApplicationContext`를 사용하는
것이 일반적이다.

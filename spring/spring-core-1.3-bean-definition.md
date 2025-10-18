# 🌱 Spring 공식문서 읽기: Core Technologies

## 1.3. Bean Definition

## 원본 링크
https://codesche.oopy.io/290de3f7-e3a8-800f-ae17-e7de1824d74b

## 참고 링크
https://docs.spring.io/spring-framework/reference/core/beans/basics.html
https://docs.spring.io/spring-framework/docs/5.0.x/spring-framework-reference/core.html#beans-definition

## 원문

A bean definition is essentially a recipe for creating one or more
objects. The container looks at the recipe for a named bean when asked
and uses the configuration metadata encapsulated by that bean definition
to create (or acquire) an actual object.
([docs.spring.io](https://docs.spring.io/spring-framework/reference/core/beans/definition.html?utm_source=chatgpt.com))

If you use XML‑based configuration metadata, you specify the type (or
class) of object that is to be instantiated in the `class` attribute of
the `<bean/>` element. This `class` attribute (which, internally, is a
`Class` property on a `BeanDefinition` instance) is usually mandatory.
(For exceptions, see Instantiation by Using an Instance Factory Method
and Bean Definition Inheritance.)
([docs.spring.io](https://docs.spring.io/spring-framework/docs/5.0.x/spring-framework-reference/core.html?utm_source=chatgpt.com))

You can use the `Class` property in one of two ways:
- Typically, to specify the bean class to be constructed in the case
where the container itself directly creates the bean by calling its
constructor reflectively, somewhat equivalent to Java code with the
`new` operator.
([docs.spring.io](https://docs.spring.io/spring-framework/docs/5.0.x/spring-framework-reference/core.html?utm_source=chatgpt.com))\
- To specify the actual class containing the `static` factory method
that is invoked to create the object, in the less common case where the
container invokes a `static` factory method on a class to create the
bean. The object type returned from the invocation of the `static`
factory method may be the same class or another class entirely.
([docs.spring.io](https://docs.spring.io/spring-framework/docs/5.0.x/spring-framework-reference/core.html?utm_source=chatgpt.com))

**Inner class names**\
If you want to configure a bean definition for a nested class, you may
use either the binary name or the source name of the nested class.\
For example, if you have a class called `SomeThing` in the `com.example`
package, and this `SomeThing` class has a `static` nested class called
`OtherThing`, they can be separated by a dollar sign (`$`) or a dot
(`.`).\
So the value of the `class` attribute in a bean definition would be
`com.example.SomeThing$OtherThing` or
`com.example.SomeThing.OtherThing`.
([docs.spring.io](https://docs.spring.io/spring-framework/docs/5.0.x/spring-framework-reference/core.html?utm_source=chatgpt.com))

------------------------------------------------------------------------

### 🇰🇷 번역

빈 정의(bean definition)는 본질적으로 하나 이상의 객체를 생성하기 위한
**조리법(recipe)** 이다.\
컨테이너는 요청이 들어온 이름 있는 빈(named bean)의 조리법을 살펴보고,\
해당 빈 정의가 캡슐화한 **설정 메타데이터(configuration metadata)** 를
사용하여 실제 객체를 생성하거나 획득(acquire)한다.

XML 기반의 설정 메타데이터를 사용하는 경우, `<bean/>` 요소의 `class`
속성(attribute)에 인스턴스화될 객체의 타입(혹은 클래스)을 지정한다.\
이 `class` 속성은 내부적으로 `BeanDefinition` 인스턴스의 `Class`
속성이며, 보통은 필수로 요구된다.\
(예외 케이스로는 **인스턴스 팩토리 메서드(instance factory method)** 를
사용한 인스턴스화 혹은 **빈 정의 상속(bean definition inheritance)** 을
참고하라.)

`Class` 속성은 다음 두 가지 방식 중 하나로 사용할 수 있다:
- 일반적으로, 컨테이너가 자체적으로 **반사(reflectively)** 생성자를
호출하여 빈을 직접 생성하는 경우,\
`new` 연산자를 사용하는 Java 코드와 유사하게 빈 클래스를 지정한다.
- 덜 일반적인 경우, 컨테이너가 클래스의 **`static` 팩토리
메서드(static factory method)** 를 호출하여 객체를 생성하는 경우에는,\
그 메서드를 포함한 실제 클래스를 지정한다.\
이 팩토리 호출로 반환되는 객체 타입은 동일한 클래스이거나 전혀 다른
클래스일 수 있다.

**내부 클래스(inner class) 이름 처리**\
만약 **중첩 클래스(nested class)** 에 대한 빈 정의를 구성하고자
한다면,\
중첩 클래스의 **바이너리 이름(binary name)** 혹은 **소스 이름(source
name)** 중 하나를 사용할 수 있다.\
예를 들어 `com.example` 패키지에 `SomeThing`라는 클래스가 있고,\
이 클래스 안에 `static` 중첩 클래스 `OtherThing`가 있다면,\
빈 정의의 `class` 속성에는 `com.example.SomeThing$OtherThing` 또는
`com.example.SomeThing.OtherThing` 두 가지 형태 중 하나를 사용할 수
있다.


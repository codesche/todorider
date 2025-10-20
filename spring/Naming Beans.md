# [Spring 공식문서 읽기] Naming Beans

## 원본링크
https://codesche.oopy.io/292de3f7-e3a8-80b6-82ea-d0784dfae5f8

## 참고링크

- [https://docs.spring.io/spring-framework/docs/5.0.x/spring-framework-reference/core.html#beans-definition](https://docs.spring.io/spring-framework/docs/5.0.x/spring-framework-reference/core.html#beans-definition)
- [https://docs.spring.io/spring-framework/docs/5.0.x/spring-framework-reference/core.html#beans-beanname](https://docs.spring.io/spring-framework/docs/5.0.x/spring-framework-reference/core.html#beans-beanname)

## 원문 1

Every bean has one or more identifiers. These identifiers must be unique within the container that hosts the bean. A bean usually has only one identifier, but if it requires more than one, the extra ones can be considered aliases.

### 번역

모든 빈(bean)은 하나 이상의 식별자(identifier)를 가진다. 이 식별자들은 빈을 담고 있는 컨테이너 내에셔 고유해야 한다. 일반적으로 빈은 하나의 식별자만 가지지만, 여러 개의 이름이 필요한 경우 추가적인 이름들은 **별칭(alias)** 으로 간주된다.

## 원문 2

In XML-based configuration metadata, you use the id and/or name attributes to specify the bean identifier(s). The id attribute allows you to specify exactly one id. Conventionally these names are alphanumeric ('myBean', 'fooService', etc.), but may contain special characters as well. If you want to introduce other aliases to the bean, you can also specify them in the name attribute, separated by a comma (,), semicolon (;), or white space. As a historical note, in versions prior to Spring 3.1, the id attribute was defined as an xsd:ID type, which constrained possible characters. As of 3.1, it is defined as an xsd:string type. Note that bean id uniqueness is still enforced by the container, though no longer by XML parsers.

### 번역

XML 기반 설정 메타데이터에서 빈의 식별자를 지정할 때는 `id` 속성과 또는 `name` 속성을 사용한다. `id` 속성은 정확히 하나의 ID만 지정할 수 있다. 일반적으로 이러한 이름은 알파벳과 숫자를 조합한 형태(`'myBean'`, `'fooService'` 등)를 따르지만, 특수 문자를 포함할 수도 있다.

참고로, Spring 3.1 이전 버전에서는 `id` 속성이 `xsd:ID` 타입으로 정의되어 있어 사용할 수 있는 문자에 제약이 있었다.

Spring 3.1부터는 `xsd:string` 타입으로 변경되어 이러한 제약이 사라졌다. 다만 **빈 ID의 고유성은 여전히 컨테이너 수준에서 보장되며**, 이제는 XML 파서가 아닌 컨테이너가 이를 관리한다.

## 원문 3

You are not required to supply a name or id for a bean. If no name or id is supplied explicitly, the container generates a unique name for that bean. However, if you want to refer to that bean by name, through the use of the ref element or Service Locator style lookup, you must provide a name. Motivations for not supplying a name are related to using inner beans and autowiring collaborators.

### 번역

빈에 `name` 이나 `id` 를 반드시 제공해야 하는 것은 아니다. 명시적으로 이름이나 ID를 지정하지 않으면, 컨테이너가 자동으로 고유한 이름을 생성한다.

하지만 해당 빈을 `ref` 요소나 **Service Locator 패턴**을 통해 이름으로 참조하려면 반드시 이름을 지정해야 한다. 이름을 생략하는 주요 이유는 주로 **내부 빈(inner bean)** 이나 **자동 주입(autowiring)** 시나리오에서 볼 수 있다.

## 원문 4 (Bean Naming Conventions)

The convention is to use the standard Java convention for instance field names when naming beans. That is, bean names start with a lowercase letter, and are camel-cased from then on. Examples of such names would be (without quotes) 'accountManager', 'accountService', 'userDao', 'loginController', and so forth.

Naming beans consistently makes your configuration easier to read and understand, and if you are using Spring AOP it helps a lot when applying advice to a set of beans related by name.

With component scanning in the classpath, Spring generates bean names for unnamed components, following the rules above: essentially, taking the simple class name and turning its initial character to lower-case. However, in the (unusual) special case when there is more than one character and both the first and second characters are upper case, the original casing gets preserved. These are the same rules as defined by java.beans.Introspector.decapitalize (which Spring is using here).

### 번역

빈을 이름 짓는 규칙은 **일반적인 자바 인스턴스 필드 명명 규칙**을 따른다. 즉, 빈 이름은 **소문자로 시작**하며, 이후 단어마다 **CamelCase(낙타표기법)**를 사용한다.

예를 들어 `'accountManager'`, `'accountService'`, `'userDao'`, `'loginController'` 등이 일반적인 이름 예시이다.

이러한 규칙을 일관성 있게 사용하면 설정이 더 읽기 쉬울 뿐만 아니라 이해하기 쉬워진다. 또한 Spring AOP를 사용할 때 이름과 관련된 여러 빈에 공통의 advice를 적용하기도 훨씬 수월해진다.

classpath 에서 컴포넌트 스캐닝(component scanning) 을 사용할 경우, Spring은 이름이 지정되지 않은 컴포넌트에 대해 위의 규칙에 따라 자동으로 빈 이름을 생성한다.

즉, 클래스의 간단한 이름(simple class name)의 첫 글자를 소문자로 바꾼 이름이 생성된다.

다만 예외적으로, 클래스 이름의 첫 번째와 두 번째 문자가 모두 대문자인 경우(예: `URLParser`), 원래의 대소문자 형태를 그대로 유지한다.

이 규칙은 Java의 `java.beans.Introspector.decapitalize()` 메서드에서 정의된 규칙과 동일하며, Spring은 내부적으로 이 메서드를 사용한다.

## 원문 (Aliasing a bean outside the bean definition)

In a bean definition itself, you can supply more than one name for the bean, by using a combination of up to one name specified by the id attribute, and any number of other names in the name attribute. These names can be equivalent aliases to the same bean, and are useful for some situations, such as allowing each component in an application to refer to a common dependency by using a bean name that is specific to that component itself.

Specifying all aliases where the bean is actually defined is not always adequate, however. It is sometimes desirable to introduce an alias for a bean that is defined elsewhere. This is commonly the case in large systems where configuration is split amongst each subsystem, each subsystem having its own set of object definitions. In XML-based configuration metadata, you can use the `<alias/>` element to accomplish this.

```xml
<alias name="fromName" alias="toName"/>
```

In this case, a bean (in the same container) named `fromName` may also, after the use of this alias definition, be referred to as `toName`.

For example, the configuration metadata for subsystem A may refer to a DataSource by the name of `subsystemA-dataSource`. The configuration metadata for subsystem B may refer to a DataSource by the name of `subsystemB-dataSource`. When composing the main application that uses both these subsystems, the main application refers to the DataSource by the name of `myApp-dataSource`. To have all three names refer to the same object, you can add the following alias definitions to the configuration metadata:

```xml
<alias name="myApp-dataSource" alias="subsystemA-dataSource"/>
<alias name="myApp-dataSource" alias="subsystemB-dataSource"/>
```

Now each component and the main application can refer to the `dataSource` through a name that is unique and guaranteed not to clash with any other definition (effectively creating a namespace), yet they refer to the same bean.

### 번역

빈 정의 자체 내에서 `id` 속성으로 지정한 하나의 이름과, `name` 속성에 정의된 여러 이름을 조합하여 하나의 빈에 대해 여러 이름을 지정할 수 있다.

이 이름들은 모두 동일한 빈에 대한 **동등한 별칭(alias)** 으로 취급된다. 이러한 방식은 예를 들어, 애플리케이션의 여러 컴포넌트가 각자 고유한 이름으로 동일한 공통 의존성을 참조해야 할 때 유용하다.

그러나 모든 별칭을 빈이 정의된 곳에서만 지정하는 것이 항상 충분하지는 않다. 때로는 **다른 곳에 정의된 빈에 대해 새로운 별칭을 추가하고 싶을 때**가 있다. 이런 경우는 보통 대규모 시스템에서 서브시스템별로 설정 파일이 분리되어 있고, 각 서브시스템이 자체 객체 정의를 가지고 있는 경우에 발생한다.

이럴 때 XML 기반 설정 메타데이터에서는 `<alias/>` 요소를 사용하여 별칭을 지정할 수 있다.

```xml
<alias name="fromName" alias="toName"/>
```

이 경우, 동일한 컨테이너 내에서 `fromName` 이라는 이름을 가진 빈은 이제 `toName` 이라는 이름으로도 참조할 수 있게 된다.

예를 들어, 서브시스템 A의 설정 메타데이터에서는 `DataSource`를 `subsystemA-dataSource` 라는 이름으로 정의하고, 서브시스템 B에서는 `subsystemB-dataSource` 로 정의했다고 가정해보자.

메인 애플리케이션에서는 이 두 서브시스템을 함께 사용하면서 `DataSource`를 `myApp-dataSource` 라는 이름으로 참조하고자 할 수 있다.

이 세 이름이 모두 같은 객체를 가리키도록 하려면 다음과 같이 별칭을 정의할 수 있다.

```xml
<alias name="myApp-dataSource" alias="subsystemA-dataSource"/>
<alias name="myApp-dataSource" alias="subsystemB-dataSource"/>
```

이제 각 서브시스템과 메인 애플리케이션은 충돌 없는 고유한 이름(일종의 네임스페이스처럼 동작)을 사용하면서도 모두 동일한 `dataSource` 빈을 참조하게 된다.
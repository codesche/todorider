
# [Spring 공식문서 읽기] Spring Framework Overview 2

**참고 링크:** [Spring Framework Overview 공식 문서](https://docs.spring.io/spring-framework/reference/overview.html)

---

## 원문 1
Spring came into being in 2003 as a response to the complexity of the early J2EE specifications. While some consider Java EE and its modern-day successor Jakarta EE to be in competition with Spring, they are in fact complementary. The Spring programming model does not embrace the Jakarta EE platform specification; rather, it integrates with carefully selected individual specifications from the traditional EE umbrella:

### 번역
Spring은 2003년에 등장했으며, 이는 당시 J2EE(Java 2 Enterprise Edition)가 지나치게 복잡했던 것에 대한 해결책으로 만들어졌다.  
일부 사람들은 Java EE(현재의 Jakarta EE)가 Spring과 경쟁 관계라고 생각하지만, 실제로는 상호 보완적인 관계에 가깝다.  
Spring의 프로그래밍 모델은 Jakarta EE 전체 플랫폼을 포괄적으로 수용하지 않는다. 대신, 그중에서 필요하고 유용한 일부 명세(specification)만을 선택적으로 통합한다.

---

## 원문 2
The Spring Framework also supports the Dependency Injection (JSR 330) and Common Annotations (JSR 250) specifications, which application developers may choose to use instead of the Spring-specific mechanisms provided by the Spring Framework. Originally, those were based on common javax packages.

### 번역
Spring Framework는 또한 **의존성 주입(Dependency Injection, JSR 330)** 및 **공통 어노테이션(Common Annotations, JSR 250)** 명세도 지원한다.  
개발자는 Spring이 제공하는 고유 기능 대신, 이런 표준 명세 기반의 어노테이션을 사용할 수도 있다.  
(예: `@Inject`, `@Resource` 같은 표준 javax 기반 어노테이션들)  
과거에는 이러한 명세들이 javax 패키지를 기반으로 제공되었다.

---

## 원문 3
As of Spring Framework 6.0, Spring has been upgraded to the Jakarta EE 9 level (for example, Servlet 5.0+, JPA 3.0+), based on the jakarta namespace instead of the traditional javax packages. With EE 9 as the minimum and EE 10 supported already, Spring is prepared to provide out-of-the-box support for the further evolution of the Jakarta EE APIs. Spring Framework 6.0 is fully compatible with Tomcat 10.1, Jetty 11 and Undertow 2.3 as web servers, and also with Hibernate ORM 6.1.

### 번역
Spring Framework 6.0부터는 **Jakarta EE 9 수준**으로 업그레이드되었다.  
즉, 기존의 `javax.*` 패키지 대신 `jakarta.*` 네임스페이스를 사용한다.  
예를 들어, Servlet 5.0+, JPA 3.0+ 이상이 기본 적용된다.  
또한 Spring은 Jakarta EE 10까지 이미 지원하고 있으며, 앞으로의 EE 발전에 맞춰 즉시 대응 가능한 구조(out-of-the-box support)를 갖추고 있다.  
현재 Spring 6.0은 **Tomcat 10.1**, **Jetty 11**, **Undertow 2.3**, **Hibernate ORM 6.1**과 완벽히 호환된다.

---

## 원문 4
Spring continues to innovate and to evolve. Beyond the Spring Framework, there are other projects, such as Spring Boot, Spring Security, Spring Data, Spring Cloud, Spring Batch, among others. It’s important to remember that each project has its own source code repository, issue tracker, and release cadence. See spring.io/projects for the complete list of Spring projects.

### 번역
Spring은 지속적으로 **혁신하고 진화**하고 있다.  
이제는 단순히 “Spring Framework”만이 아니라, 그 위에 다양한 독립 프로젝트들이 함께 발전하고 있다.  
대표적인 예로는 다음과 같다.  
- Spring Boot → 설정 자동화 및 실행 환경 단순화  
- Spring Security → 인증 및 인가 보안 처리  
- Spring Data → 데이터 접근 통합 (JPA, MongoDB 등)  
- Spring Cloud → 마이크로서비스 및 분산 환경 지원  
- Spring Batch → 대규모 배치 처리  

각 프로젝트는 독자적인 **저장소, 이슈 트래커, 릴리스 주기**를 가진다.  
전체 목록은 [spring.io/projects](https://spring.io/projects)에서 확인할 수 있다.

---

## 원문 5 - Design Philosophy
When you learn about a framework, it’s important to know not only what it does but what principles it follows. Here are the guiding principles of the Spring Framework:

### 번역
프레임워크를 배울 때는, **무엇을 할 수 있는지뿐만 아니라 어떤 원칙으로 만들어졌는지를 이해하는 것**이 중요하다.  
아래는 Spring Framework의 설계를 이끄는 **핵심 철학(Design Philosophy)**이다.

1. **Provide choice at every level.**  
   Spring은 가능한 한 늦은 시점까지 설계 결정을 미룰 수 있도록 하여, 개발자가 설정만으로 기술을 교체할 수 있게 한다.  

2. **Accommodate diverse perspectives.**  
   Spring은 유연성을 존중하며, 특정한 방식으로 개발을 강요하지 않는다.  

3. **Maintain strong backward compatibility.**  
   버전 간 하위 호환성을 유지하여, 장기간 안정적인 애플리케이션 유지보수를 지원한다.  

4. **Care about API design.**  
   API를 직관적이고 일관성 있게 설계하여 오랜 시간 동안 유지될 수 있게 만든다.  

5. **Set high standards for code quality.**  
   높은 코드 품질과 최신 문서화를 유지하며, 순환 의존성이 없는 깨끗한 구조를 유지한다.

---

## 원문 6 - Feedback and Contributions
For how-to questions or diagnosing or debugging issues, we suggest using Stack Overflow. Click here for a list of the suggested tags to use on Stack Overflow. If you’re fairly certain that there is a problem in the Spring Framework or would like to suggest a feature, please use the GitHub Issues. If you have a solution in mind or a suggested fix, you can submit a pull request on Github. However, please keep in mind that, for all but the most trivial issues, we expect a ticket to be filed in the issue tracker, where discussions take place and leave a record for future reference. For more details see the guidelines at the CONTRIBUTING, top-level project page.

### 번역
Spring 사용 중 **질문, 문제 진단, 디버깅**이 필요하다면 **Stack Overflow**를 이용하는 것이 좋다.  
버그 제보나 기능 제안은 **GitHub Issues**를 통해 등록할 수 있으며,  
이미 해결 방안을 제시할 수 있다면 **Pull Request**를 제출할 수도 있다.  
단, 단순한 수정이 아닌 경우에는 반드시 **Issue Tracker에 티켓을 먼저 등록해야 하며**,  
그곳에서 이루어진 논의는 향후 참고 기록으로 남는다.  
자세한 내용은 **CONTRIBUTING 가이드라인**을 참고한다.

---

## 원문 7 - Getting Started
If you are just getting started with Spring, you may want to begin using the Spring Framework by creating a Spring Boot-based application. Spring Boot provides a quick (and opinionated) way to create a production-ready Spring-based application. It is based on the Spring Framework, favors convention over configuration, and is designed to get you up and running as quickly as possible.

You can use start.spring.io to generate a basic project or follow one of the "Getting Started" guides, such as Getting Started Building a RESTful Web Service. As well as being easier to digest, these guides are very task focused, and most of them are based on Spring Boot. They also cover other projects from the Spring portfolio that you might want to consider when solving a particular problem.

### 번역
Spring을 처음 시작하는 사람이라면, **Spring Boot 기반 애플리케이션을 만드는 것부터 시작**하는 것이 좋다.  
Spring Boot는 “설정보다 관례(Convention over Configuration)” 원칙을 따르며,  
**빠르고 실무 중심적인 애플리케이션 개발 환경**을 제공한다.  
[`start.spring.io`](https://start.spring.io)에서 기본 프로젝트를 생성하거나,  
“Getting Started Building a RESTful Web Service” 같은 가이드를 따라가며 학습할 수 있다.  
이 가이드들은 대부분 **Spring Boot**를 기반으로 작성되어 있으며,  
Spring Data, Spring Security, Spring Cloud 등 **다른 프로젝트들과의 통합 예제**도 포함되어 있다.

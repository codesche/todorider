
# 📘 [Spring 공식문서 읽기] Spring Framework Overview 1

## 원본 링크
https://codesche.oopy.io/287de3f7-e3a8-80d0-923c-e67afa6a654c

## 참고 링크
[Spring Framework Overview 공식 문서](https://docs.spring.io/spring-framework/reference/overview.html)

## 🧭 목차
- [📘 \[Spring 공식문서 읽기\] Spring Framework Overview 1](#-spring-공식문서-읽기-spring-framework-overview-1)
  - [원본 링크](#원본-링크)
  - [참고 링크](#참고-링크)
  - [🧭 목차](#-목차)
  - [🌱 Spring이 제공하는 것 (원문 1)](#-spring이-제공하는-것-원문-1)
  - [⚙️ 다양한 실행 시나리오 (원문 2)](#️-다양한-실행-시나리오-원문-2)
  - [🤝 오픈소스와 커뮤니티 (원문 3)](#-오픈소스와-커뮤니티-원문-3)
  - [🌼 “Spring”의 의미 (원문 4)](#-spring의-의미-원문-4)
  - [🧩 모듈 구조 개요 (원문 5)](#-모듈-구조-개요-원문-5)
  - [🧠 JPMS와 모듈 시스템 (원문 6)](#-jpms와-모듈-시스템-원문-6)
  - [📊 참고 다이어그램](#-참고-다이어그램)

---

## 🌱 Spring이 제공하는 것 (원문 1)

> **원문:**  
> Spring makes it easy to create Java enterprise applications... As of Spring Framework 6.0, Spring requires Java 17+.

**번역**  
Spring은 Java 엔터프라이즈 애플리케이션을 쉽게 개발할 수 있도록 도와주는 프레임워크이다.  
Java 언어 기반의 모든 엔터프라이즈 기능을 제공하며, Groovy 및 Kotlin도 지원한다.  
Spring Framework 6.0부터는 **Java 17 이상**이 필수이다.

**요약**
- Spring은 Java 기반 엔터프라이즈 애플리케이션 개발의 복잡성을 단순화한다.  
- JVM 위에서 Groovy, Kotlin 등 다언어 환경을 지원한다.  
- Spring 6.0부터 Java 17 이상이 필요하다.

**예제 코드**
```java
// src/main/java/com/example/demo/DemoApplication.java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args); // 엔터프라이즈 앱 실행
    }
}
```
> @SpringBootApplication 하나로 설정, DI, 서버 실행까지 모두 처리된다.  
> 과거의 복잡한 XML 설정이 필요 없으며, 한 줄로 서버 기반 애플리케이션 실행 가능.

---

## ⚙️ 다양한 실행 시나리오 (원문 2)

> **원문:**  
> Spring supports a wide range of application scenarios... standalone applications (such as batch or integration workloads).

**번역**  
Spring은 대규모 엔터프라이즈 환경부터 클라우드 네이티브, 배치 처리 등 다양한 시나리오를 지원한다.

**요약**
- 레거시 서버 환경부터 클라우드 네이티브까지 대응.  
- 내장 서버 기반 단일 실행 파일(.jar) 구조 지원.  
- 서버 없는 배치/통합형 애플리케이션도 구현 가능.

**예시 1 – 내장 서버 실행 (Spring Boot Web App)**
```java
@SpringBootApplication
public class WebApp {
    public static void main(String[] args) {
        SpringApplication.run(WebApp.class, args); // 내장 Tomcat 서버 실행
    }

    @RestController
    static class HelloController {
        @GetMapping("/")
        public String hello() {
            return "Hello from Embedded Tomcat!";
        }
    }
}
```

**예시 2 – 서버 없는 배치 실행 (Spring Batch Application)**
```java
@SpringBootApplication
public class BatchApp implements CommandLineRunner {
    public static void main(String[] args) {
        SpringApplication.run(BatchApp.class, args);
    }
    @Override
    public void run(String... args) {
        System.out.println("Running batch job... ✅");
    }
}
```

---

## 🤝 오픈소스와 커뮤니티 (원문 3)

> **원문:**  
> Spring is open source... This has helped Spring to successfully evolve.

**요약**
- Spring은 오픈소스이며 커뮤니티 중심으로 발전해왔다.  
- 수많은 실제 기업 및 개발자 피드백이 지속적 개선의 원동력이다.  
- 개방-폐쇄 원칙(OCP)에 따라 기능 확장과 대체 구현이 용이하다.

**예제 코드 – 확장하기 쉬운 구조**
```java
interface GreetingService { String greet(); }

@Service
class EnglishGreetingService implements GreetingService {
    public String greet() { return "Hello, Spring Community!"; }
}

@Service
class KoreanGreetingService implements GreetingService {
    public String greet() { return "안녕하세요, 스프링 커뮤니티!"; }
}

@RestController
class GreetingController {
    private final GreetingService greetingService;
    public GreetingController(GreetingService greetingService) {
        this.greetingService = greetingService;
    }
    @GetMapping("/greet")
    public String greet() { return greetingService.greet(); }
}
```

> `GreetingService` 인터페이스 중심으로 구현되어 있어 OCP 원칙을 실현.

---

## 🌼 “Spring”의 의미 (원문 4)

> **원문:**  
> The term "Spring" means different things in different contexts... focuses on the foundation: the Spring Framework itself.

**요약**
- “Spring”은 문맥에 따라 Framework 자체 또는 Spring 전체 생태계를 의미.  
- Spring Framework는 모든 Spring 프로젝트의 기반이다.  
- 본 문서는 Spring Framework 자체에 초점을 맞춘다.

**Spring Ecosystem 구조**
```
Spring Ecosystem
 ├─ Spring Framework (Core & Foundation)
 │   ├─ Core Container (Beans, Context)
 │   ├─ AOP, JDBC, Tx
 │   ├─ MVC / WebFlux
 │
 ├─ Spring Boot       (실행 환경 자동화)
 ├─ Spring Data       (DB, JPA)
 ├─ Spring Security   (인증/인가)
 ├─ Spring Cloud      (분산/마이크로서비스)
 ├─ Spring Batch      (대용량 배치)
 └─ Spring Integration(시스템 연동)
```

---

## 🧩 모듈 구조 개요 (원문 5)

> **원문:**  
> The Spring Framework is divided into modules... reactive web framework.

**요약**
- Spring은 모듈형 구조로, 필요한 기능만 선택 사용 가능.  
- 중심에는 **Core Container**가 존재하며, DI, Bean 관리 등을 담당.  
- Web 계층은 MVC(동기)와 WebFlux(비동기/리액티브)로 구성.

**Spring Framework Modules**
```
├── Core Container
│    ├─ spring-core
│    ├─ spring-beans
│    ├─ spring-context
│    └─ spring-expression
│
├── Data Access / Integration
│    ├─ spring-jdbc
│    ├─ spring-tx
│    ├─ spring-orm
│    ├─ spring-oxm
│    └─ spring-jms
│
├── Web
│    ├─ spring-web
│    ├─ spring-webmvc
│    └─ spring-webflux
│
└── Others
     ├─ spring-test
     └─ spring-aop
```

**DI 예제**
```java
@Service
class GreetingService {
    public String getGreeting() {
        return "Hello from Spring Core Container!";
    }
}

@RestController
class GreetingController {
    private final GreetingService greetingService;
    @Autowired
    public GreetingController(GreetingService greetingService) {
        this.greetingService = greetingService;
    }
    @GetMapping("/")
    public String greet() { return greetingService.getGreeting(); }
}
```
> `@Autowired`는 Spring의 의존성 주입 메커니즘을 담당.

---

## 🧠 JPMS와 모듈 시스템 (원문 6)

> **원문:**  
> Spring Framework’s jars allow for deployment to the module path (Java Module System)...

**요약**
- Spring은 Java 9+의 **JPMS**와 완벽히 호환된다.  
- 각 JAR에는 `Automatic-Module-Name`이 명시되어 있음.  
- `module path`와 `classpath` 모두에서 동작 가능.

**예제 – module-info.java**
```java
module com.example.app {
    requires spring.context;
    requires spring.web;
    requires spring.beans;
}
```
> 모듈 이름 기반으로 의존성을 관리하여 충돌을 방지.

---

## 📊 참고 다이어그램

```
Spring Ecosystem (전체)
 └── Spring Framework (핵심)
      ├─ Core Container
      ├─ Data / Integration
      ├─ Web (MVC, WebFlux)
      └─ Others (AOP, Test)
```
> Spring Boot, Data, Security 등은 Framework 위에서 동작하는 확장 프로젝트.

---

✅ **정리 요약**
- Spring은 Java 엔터프라이즈 개발의 복잡성을 제거한 통합 프레임워크.  
- 다양한 실행 시나리오(서버, 클라우드, 배치)에 대응.  
- 모듈화, 확장성, 오픈 커뮤니티 기반 발전 구조를 갖춤.  
- Java 17 이상, JPMS 완벽 호환.

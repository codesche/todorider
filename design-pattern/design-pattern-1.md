# 디자인 패턴 - 1


```
출처: [gyoogle.dev](https://gyoogle.dev/blog/)  
```

## 개요

일종의 설계 기법이며, 설계 방법이다.

<br/>

## 목적

- 소프트웨어 재사용성, 호환성, 유지 보수성을 보장하기 위해 사용한다.

<br/>

## 특징

- 디자인 패턴은 아이디어임, 특정한 구현이 아님.  
- 프로젝트에 항상 적용을 해야 하는 것은 아니지만, 추후 재사용, 호환, 유지 보수시 발생하는 문제 해결을 예방하기 위해 패턴을 만들어 둔 것임.

<br/>

## 원칙 - SOLID (객체 설계 원칙)

1. **SRP**: 하나의 클래스는 하나의 역할만 해야 함.  
2. **OCP**: 확장(상속)에는 열려있고, 수정에는 닫혀 있어야 함.  
3. **LSP**: 자식이 부모의 자리에 항상 교체될 수 있어야 함.  
4. **ISP**: 인터페이스가 잘 분리되어서, 클래스가 꼭 필요한 인터페이스만 구현하도록 해야 함.  
5. **DIP**: 상위 모듈이 하위 모듈에 의존하면 안 됨. 둘 다 추상화에 의존해야 하며, 추상화는 세부 사항에 의존하면 안 됨.

<br/>

## 분류 (중요)

1. **생성 패턴 (Creational)**: 객체의 생성 방식을 결정  
   a. Class-creational patterns  
   b. Object-creational patterns  
   c. DBConnection을 관리하는 Instance를 하나만 만들 수 있도록 제한하여, 불필요한 연결을 막음.

2. **구조 패턴 (Structural)**: 객체간의 관계를 조직  
   a. 2개의 인터페이스가 서로 호환이 되지 않을 때, 둘을 연결해주기 위해 새로운 클래스를 만들어서 연결시킬 수 있도록 함.

3. **행위 패턴 (Behavioral)**: 객체의 행위를 조직, 관리, 연합  
   a. 하위 클래스에서 구현해야 하는 함수 및 알고리즘들을 미리 선언하여, 상속 시 이를 필수로 구현하도록 함.

<br/>

## 어댑터 패턴 (Adapter Pattern)

클래스를 바로 사용할 수 없는 경우가 있음 (다른 곳에서 개발했다거나, 수정할 수 없을 때). 중간에서 변환 역할을 해주는 클래스가 필요 → 어댑터 패턴

- **사용 방법**: 상속  
- 호환되지 않은 인터페이스를 사용하는 클라이언트 그대로 활용 가능  
- 향후 인터페이스가 바뀌더라도, 변경 내역은 어댑터에 캡슐화 되므로 클라이언트 바뀔 필요 없음  

예: 아이폰 이어폰  
- 이어폰 잭이 아이폰 단자와 맞지 않음  
- 따라서 어댑터를 사용하여 연결  

기존 시스템과 업체에서 제공한 클래스가 맞지 않다면, 시스템을 수정하기보다는 어댑터를 활용해 유연하게 해결하는 것이 좋음.

<br/>

## 코드로 어댑터 패턴 이해하기

오리(Duck)와 칠면조(Turkey) 인터페이스 생성.  
만약 오리 객체가 부족해서 칠면조 객체를 대신 사용해야 한다면? 두 객체는 인터페이스가 다르므로, 바로 칠면조 객체를 사용하는 것은 불가능함.  
따라서 **칠면조 어댑터(TurkeyAdapter)** 를 생성해서 활용해야 함.

```java
// Duck.java
public interface Duck {
    public void quack();
    public void fly();
}

// Turkey.java
public interface Turkey {
    public void gobble();
    public void fly();
}

// WildTurkey.java
public class WildTurkey implements Turkey {
    @Override
    public void gobble() {
        System.out.println("Gobble gobble");
    }
    @Override
    public void fly() {
        System.out.println("I'm flying a short distance");
    }
}

// TurkeyAdapter.java
public class TurkeyAdapter implements Duck {
    Turkey turkey;
    public TurkeyAdapter(Turkey turkey) {
        this.turkey = turkey;
    }
    @Override
    public void quack() {
        turkey.gobble();
    }
    @Override
    public void fly() {
        turkey.fly();
    }
}

// DuckTest.java
public class DuckTest {
    public static void main(String[] args) {
        MallardDuck duck = new MallardDuck();
        WildTurkey turkey = new WildTurkey();
        Duck turkeyAdapter = new TurkeyAdapter(turkey);

        System.out.println("The turkey says...");
        turkey.gobble();
        turkey.fly();

        System.out.println("The Duck says...");
        testDuck(duck);

        System.out.println("The TurkeyAdapter says...");
        testDuck(turkeyAdapter);
    }
    public static void testDuck(Duck duck) {
        duck.quack();
        duck.fly();
    }
}
```

<br/>

## 템플릿 메소드 패턴 (Template Method Pattern)

로직을 단계별로 나눠야 하는 상황에서 적용한다. 단계별로 나눈 로직들이 앞으로 수정될 가능성이 있을 경우 더 효율적이다.

### 조건

- 클래스는 추상(abstract)으로 만든다.  
- 단계를 진행하는 메소드는 수정이 불가능하도록 `final` 키워드를 추가한다.  
- 각 단계들은 외부는 막고, 자식들만 활용할 수 있도록 `protected`로 선언한다.

예시: 피자를 만들 때는 크게 반죽 → 토핑 → 굽기 로 3단계로 이루어져 있다.  
이 단계는 항상 유지되며, 순서가 바뀔 일은 없다. 물론 실제로는 도우에 따라 반죽이 달라질 수 있지만, 일단 모든 피자의 반죽과 굽기는 동일하다고 가정하자.  
그러면 피자 종류에 따라 토핑만 바꾸면 된다.

```java
abstract class Pizza {
    protected void 반죽() {
        System.out.println("반죽!");
    }
    abstract void 토핑();
    protected void 굽기() {
        System.out.println("굽기!");
    }
    final void makePizza() {
        // 상속 받은 클래스에서 수정 불가
        this.반죽();
        this.토핑();
        this.굽기();
    }
}

class PotatoPizza extends Pizza {
    @Override
    void 토핑() {
        System.out.println("고구마 넣기!");
    }
}

class TomatoPizza extends Pizza {
    @Override
    void 토핑() {
        System.out.println("토마토 넣기!");
    }
}
```

<br/>

## abstract과 Interface의 차이는

- **abstract**: 부모의 기능을 자식에서 확장시켜나가고 싶을 때  
- **interface**: 해당 클래스가 가진 함수의 기능을 활용하고 싶을 때  

<br/>

## 팩토리 메서드 패턴 (Factory Method Pattern)

객체를 만드는 부분을 Sub class에 맡기는 패턴  

- `Robot` (추상 클래스) → `SuperRobot`, `PowerRobot`  
- `RobotFactory` (추상 클래스) → `SuperRobotFactory` / `ModifiedSuperRobotFactory`

예:

```java
public abstract class RobotFactory {
    abstract Robot createRobot(String name);
}

public class SuperRobotFactory extends RobotFactory {
    @Override
    Robot createRobot(String name) {
        switch (name) {
            case "super" : return new SuperRobot();
            case "power" : return new PowerRobot();
        }
        return null;
    }
}
```

- 생성하는 클래스를 따로 만들었음.  
- 클래스는 factory 클래스를 상속하고 있기 때문에, 반드시 `createRobot`을 선언해야 함.  
- name으로 건너오는 값에 따라서, 생성되는 `Robot`이 다르게 설계됨.  

정리하면, 생성하는 객체를 별도로 둠. 그리고 그 객체에 넘어오는 값에 따라서, 다른 로봇을 만들어냄.

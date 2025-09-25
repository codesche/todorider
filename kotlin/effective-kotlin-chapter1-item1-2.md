# 이펙티브 코틀린 완전 정복기 #1 - 1장 안전성 시작하기

코틀린을 제대로 배우고 싶다면 마르친 모스칼라의 "이펙티브 코틀린"은 필독서다. 단순한 문법서를 넘어서 실무에서 정말 중요한 베스트 프랙티스들을 다루고 있기 때문이다. 오늘부터 이 책을 차근차근 정복해보려고 한다.

## 왜 이펙티브 코틀린인가?

이 책은 조슈아 블로크의 "이펙티브 자바"처럼, 언어의 기능을 단순히 나열하는 게 아니라 **어떻게 써야 하는지**를 알려준다. 총 50개의 아이템으로 구성되어 있으며, 각각이 하나의 핵심 원칙을 담고 있다.

## 1장: 안전성 - 버그 없는 코드를 위한 첫걸음

1장은 코틀린의 가장 큰 강점인 **안전성**에 대해 다룬다. 런타임에서 터질 수 있는 오류들을 컴파일 타임에 미리 잡아내는 것이 핵심이다.

## Item 1: 가변성을 제한하라

첫 번째 원칙부터 강력하다. **상태가 변할 수 있는 지점을 최대한 줄여라**는 것이다.

### 왜 가변성을 제한해야 할까?

- 프로그램을 추론하기 어려워진다
- 멀티스레드 환경에서 동기화 문제가 발생한다
- 테스트하기 복잡해진다
- 상태 변경을 추적하기 어려워진다

### 실무 예제: 장바구니 구현

```kotlin
// ❌ 나쁜 예 - 외부에서 마음대로 변경 가능
class BadShoppingCart {
    val items = mutableListOf<Item>() // 위험!
}

val cart = BadShoppingCart()
cart.items.clear() // 외부에서 장바구니를 비워버릴 수 있다
```

```kotlin
// ✅ 좋은 예 - 변경을 통제
class GoodShoppingCart {
    private val _items = mutableListOf<Item>()
    val items: List<Item> get() = _items.toList() // 읽기 전용으로 노출
    
    fun addItem(item: Item) {
        _items.add(item)
    }
    
    fun removeItem(item: Item) {
        _items.remove(item)
    }
    
    fun clearAll() {
        _items.clear()
    }
}
```

### `val` vs `var` 올바른 사용법

```kotlin
// ✅ 읽기 전용 프로퍼티 활용
class BankAccount(private val accountNumber: String) {
    private var _balance = 0L
    val balance: Long get() = _balance  // 외부에서는 읽기만 가능
    
    fun deposit(amount: Long) {
        require(amount > 0) { "입금액은 0보다 커야 한다" }
        _balance += amount
    }
    
    fun withdraw(amount: Long) {
        require(amount > 0) { "출금액은 0보다 커야 한다" }
        require(_balance >= amount) { "잔액이 부족하다" }
        _balance -= amount
    }
}
```

### 불변 컬렉션 활용

```kotlin
// ✅ 불변 컬렉션 선호
val numbers = listOf(1, 2, 3)  // List<Int> - 불변
val mutableNumbers = mutableListOf(1, 2, 3)  // MutableList<Int> - 가변

// ✅ data class로 불변 객체 생성
data class Point(val x: Int, val y: Int) {
    fun moveBy(dx: Int, dy: Int): Point = Point(x + dx, y + dy)  // 새 인스턴스 반환
}

val point = Point(0, 0)
val movedPoint = point.moveBy(10, 5) // 기존 point는 변경되지 않음
```

## Item 2: 변수의 스코프를 최소화하라

변수가 사용되는 범위를 최대한 좁게 만들어야 한다. 이는 다음과 같은 이점을 제공한다:

- 프로그램을 추적하고 관리하기 쉬워진다
- 변수의 초기화를 잊을 가능성이 줄어든다
- 변수가 잘못 사용될 가능성이 줄어든다

### 실무 예제: 사용자 이메일 검증

```kotlin
// ❌ 나쁜 예 - 변수 스코프가 불필요하게 넓음
fun findUserEmail(userId: Long): String? {
    var user: User? = null  // 너무 일찍 선언
    var email: String? = null
    
    user = userRepository.findById(userId)
    if (user != null) {
        email = user.email
        if (email.isNotEmpty()) {
            return email
        }
    }
    return null
}
```

```kotlin
// ✅ 좋은 예 - 스코프 최소화
fun findUserEmail(userId: Long): String? {
    val user = userRepository.findById(userId) ?: return null
    val email = user.email
    return if (email.isNotEmpty()) email else null
}
```

```kotlin
// 🚀 더 좋은 예 - 코틀린다운 방식
fun findUserEmail(userId: Long): String? {
    return userRepository.findById(userId)
        ?.email
        ?.takeIf { it.isNotEmpty() }
}
```

### 반복문에서의 스코프 최소화

```kotlin
// ❌ 나쁜 예
fun processUsers() {
    var user: User? = null  // 스코프가 너무 넓음
    var isValid = false
    
    for (id in userIds) {
        user = findUser(id)
        if (user != null) {
            isValid = validateUser(user)
            if (isValid) {
                processValidUser(user)
            }
        }
    }
}
```

```kotlin
// ✅ 좋은 예
fun processUsers() {
    for (id in userIds) {
        val user = findUser(id) ?: continue
        if (validateUser(user)) {
            processValidUser(user)
        }
    }
}
```

### 구조 분해 선언 활용

```kotlin
// ✅ 구조 분해로 스코프 최소화
fun processCoordinates(points: List<Pair<Int, Int>>) {
    for ((x, y) in points) {  // 필요한 곳에서 바로 분해
        if (x > 0 && y > 0) {
            drawPoint(x, y)
        }
    }
}
```

## 핵심 정리

### Item 1: 가변성을 제한하라
- **불변성을 기본으로** - `val`을 우선적으로 사용하고, 꼭 필요할 때만 `var`를 쓴다
- **컬렉션도 불변을 선호** - `List<T>` vs `MutableList<T>`
- **상태 변경을 통제** - private으로 숨기고 함수로만 접근 허용

### Item 2: 변수의 스코프를 최소화하라
- **사용하는 곳 가까이에서 선언** - 변수를 너무 일찍 선언하지 말자
- **Elvis 연산자 적극 활용** - `?.` 와 `?:` 로 코드를 간결하게
- **구조 분해 선언 활용** - 필요한 값만 추출해서 사용

## 다음 예고

다음 글에서는 Item 3부터 계속해서 1장의 나머지 내용들을 다뤄볼 예정이다. 특히 플랫폼 타입 처리와 inferred 타입 활용법이 흥미로울 것 같다.
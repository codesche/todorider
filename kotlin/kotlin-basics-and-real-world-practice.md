# Kotlin 기초 문법과 실무 적용 감각 키우기

Kotlin은 JVM 위에서 동작하면서 Java보다 훨씬 간결하고 안전한 코드를 쓸 수 있게 해준다. Android 앱 개발의 공식 언어이기도 하고, 백엔드(Spring)에서도 점점 많이 쓰이고 있다. 이 글에서는 Kotlin의 핵심 문법을 실무 맥락으로 정리하고, 실제로 어떻게 쓰이는지 감각을 익히는 데 집중한다.

---

## 변수 선언: val과 var

Kotlin에서 변수는 `val`(불변)과 `var`(가변) 두 가지로 나뉜다. 기본적으로 `val`을 쓰고, 값이 바뀌어야 할 때만 `var`를 쓰는 게 원칙이다.

```kotlin
val name = "kotlin"       // 재할당 불가
var count = 0             // 재할당 가능
count = 1                 // OK

// 타입을 명시할 수도 있다
val userId: Long = 1001L
var isLoggedIn: Boolean = false
```

실무에서는 `val`을 최대한 쓰는 게 좋다. 값이 어디서 바뀌는지 추적할 필요가 없어서 코드를 읽기 쉬워진다.

---

## 널 안전성(Null Safety)

Kotlin의 가장 강력한 특징 중 하나다. 타입 자체에 null 허용 여부를 명시해서 NullPointerException을 컴파일 단계에서 막는다.

```kotlin
// null을 허용하지 않는 타입
val name: String = "kotlin"
// name = null  // 컴파일 에러

// null을 허용하는 타입 (? 붙이기)
val nickname: String? = null

// 안전 호출 연산자 ?.
val length = nickname?.length  // null이면 null 반환, 에러 없음

// Elvis 연산자 ?: (null일 때 기본값 제공)
val displayName = nickname ?: "익명 사용자"

// null이 아님을 단언 !! (확실할 때만 쓴다, 남용 금지)
val forcedLength = nickname!!.length  // null이면 NPE 발생
```

실무에서 Elvis 연산자는 굉장히 자주 쓰인다.

```kotlin
// 실무 예시: 사용자 조회 후 없으면 예외 던지기
fun getUser(id: Long): User {
    return userRepository.findById(id)
        ?: throw IllegalArgumentException("사용자를 찾을 수 없다. id=$id")
}
```

---

## 함수 선언

Kotlin의 함수는 `fun` 키워드로 선언한다. 반환 타입은 `:` 뒤에 명시한다.

```kotlin
// 기본 함수
fun add(a: Int, b: Int): Int {
    return a + b
}

// 단일 표현식 함수 (간결하게)
fun add(a: Int, b: Int): Int = a + b

// 반환값이 없으면 Unit (생략 가능)
fun printHello(): Unit {
    println("Hello, Kotlin!")
}

// 기본값 파라미터
fun greet(name: String, greeting: String = "안녕") {
    println("$greeting, $name!")
}

greet("철수")           // 안녕, 철수!
greet("영희", "반가워")  // 반가워, 영희!

// 이름 있는 인수 (Named Arguments)
greet(name = "민수", greeting = "좋은 아침")
```

실무에서 기본값 파라미터와 이름 있는 인수는 메서드 오버로딩을 줄여주는 데 유용하다.

---

## 클래스와 데이터 클래스

```kotlin
// 일반 클래스
class Person(val name: String, var age: Int) {
    fun introduce() = "저는 $name이고 $age살입니다."
}

val person = Person("김철수", 25)
println(person.introduce())

// data class: equals, hashCode, toString, copy를 자동 생성
data class User(
    val id: Long,
    val email: String,
    val name: String,
    val isActive: Boolean = true
)

val user = User(id = 1L, email = "test@email.com", name = "홍길동")
val updatedUser = user.copy(name = "김길동")  // 일부만 바꾼 복사본

println(user)         // User(id=1, email=test@email.com, name=홍길동, isActive=true)
println(user == updatedUser)  // false (name이 다름)
```

실무에서 `data class`는 DTO, 요청/응답 객체, 도메인 모델 등에 광범위하게 쓰인다.

---

## 확장 함수(Extension Function)

기존 클래스를 수정하지 않고 새로운 함수를 추가할 수 있다. Kotlin의 핵심 특징 중 하나다.

```kotlin
// String에 이메일 검증 함수 추가
fun String.isValidEmail(): Boolean {
    return contains("@") && contains(".")
}

// Long에 원화 포맷 함수 추가
fun Long.toKoreanWon(): String = "${this}원"

// 사용
println("test@email.com".isValidEmail())  // true
println("invalid-email".isValidEmail())   // false
println(50000L.toKoreanWon())              // 50000원
```

실무에서는 유틸리티성 로직을 확장 함수로 만들어두면 코드가 훨씬 읽기 쉬워진다.

```kotlin
// 실무 예시: 응답 객체 변환
fun User.toResponse() = UserResponse(
    id = this.id,
    email = this.email,
    name = this.name
)

// 사용
val response = user.toResponse()
```

---

## when 표현식

Java의 switch보다 훨씬 강력하다. 표현식으로도 쓸 수 있어서 값을 반환하는 것도 가능하다.

```kotlin
// 값 반환
fun getStatusMessage(code: Int): String = when (code) {
    200 -> "성공"
    400 -> "잘못된 요청"
    401 -> "인증 필요"
    403 -> "권한 없음"
    404 -> "찾을 수 없음"
    in 500..599 -> "서버 오류"
    else -> "알 수 없는 오류"
}

// 타입 체크와 결합
fun describe(obj: Any): String = when (obj) {
    is String -> "문자열: $obj"
    is Int -> "정수: $obj"
    is List<*> -> "리스트, 크기: ${obj.size}"
    else -> "알 수 없음"
}

// 조건 없는 when (if-else 대체)
fun classifyScore(score: Int): String = when {
    score >= 90 -> "A"
    score >= 80 -> "B"
    score >= 70 -> "C"
    else -> "F"
}
```

---

## 컬렉션과 람다

Kotlin의 컬렉션 API는 함수형 스타일로 데이터를 처리할 수 있어 실무에서 매우 유용하다.

```kotlin
val users = listOf(
    User(1L, "a@test.com", "김철수"),
    User(2L, "b@test.com", "이영희"),
    User(3L, "c@test.com", "박민수")
)

// filter: 조건에 맞는 요소만 추출
val activeUsers = users.filter { it.isActive }

// map: 변환
val emails = users.map { it.email }

// find: 조건에 맞는 첫 번째 요소
val user = users.find { it.id == 2L }

// any / all / none
val hasAdmin = users.any { it.email.contains("admin") }
val allActive = users.all { it.isActive }

// groupBy: 그룹핑
val groupedByActive = users.groupBy { it.isActive }

// 체이닝
val result = users
    .filter { it.isActive }
    .map { it.toResponse() }
    .sortedBy { it.name }
```

---

## 스코프 함수: let, apply, run, also, with

스코프 함수는 객체를 다룰 때 코드 블록을 만들어주는 함수다. 실무에서 굉장히 자주 쓰인다.

```kotlin
// let: null 체크 후 처리할 때 주로 사용
val email: String? = getUserEmail()
email?.let {
    println("이메일: $it")
    sendVerificationMail(it)
}

// apply: 객체 초기화/설정에 사용 (this 반환)
val user = User(1L, "test@email.com", "홍길동").apply {
    // 빌더 패턴처럼 사용
}

// also: 부수 작업(로깅 등)에 사용 (it 사용, 원본 객체 반환)
val savedUser = userRepository.save(user).also {
    log.info("사용자 저장 완료: ${it.id}")
}

// run: 객체 설정 + 결과값 반환
val result = user.run {
    validateEmail(email)
    validateName(name)
    "검증 완료"
}

// with: 이미 있는 객체에 여러 작업을 묶을 때
with(user) {
    println("id: $id")
    println("email: $email")
    println("name: $name")
}
```

---

## 코루틴(Coroutine) 기초

비동기 처리를 위한 Kotlin의 핵심 기능이다. Android 앱 개발에서 네트워크 요청, DB 조회 등 비동기 작업에 필수적으로 쓰인다.

```kotlin
import kotlinx.coroutines.*

// suspend 함수: 코루틴 안에서만 호출 가능
suspend fun fetchUserFromApi(id: Long): User {
    delay(1000)  // 네트워크 요청을 흉내냄 (스레드 블로킹 없음)
    return User(id, "test@email.com", "홍길동")
}

// 코루틴 실행
fun main() = runBlocking {
    // launch: 반환값 없는 코루틴
    launch {
        println("코루틴 시작")
        delay(500)
        println("코루틴 완료")
    }

    // async/await: 반환값이 있는 비동기 작업
    val user = async { fetchUserFromApi(1L) }
    println("사용자: ${user.await().name}")

    // 여러 작업 병렬 처리
    val (user1, user2) = awaitAll(
        async { fetchUserFromApi(1L) },
        async { fetchUserFromApi(2L) }
    )
    println("${user1.name}, ${user2.name}")
}
```

Android 앱에서는 보통 ViewModel에서 `viewModelScope.launch`로 코루틴을 실행한다.

```kotlin
class UserViewModel(private val repository: UserRepository) : ViewModel() {

    val userState = MutableStateFlow<User?>(null)

    fun loadUser(id: Long) {
        viewModelScope.launch {
            try {
                val user = repository.getUser(id)  // suspend 함수
                userState.value = user
            } catch (e: Exception) {
                // 에러 처리
            }
        }
    }
}
```

---

## 실무에서 바로 쓰는 패턴 모음

```kotlin
// 1. 싱글톤: object 키워드
object AppConfig {
    const val BASE_URL = "https://api.example.com"
    const val TIMEOUT = 30_000L
}

// 2. 동반 객체(Companion Object): 팩토리 메서드 패턴
class ApiResponse<T>(val data: T?, val message: String) {
    companion object {
        fun <T> success(data: T) = ApiResponse(data, "성공")
        fun <T> failure(message: String) = ApiResponse<T>(null, message)
    }
}

val response = ApiResponse.success(user)

// 3. sealed class: 상태 표현에 강력
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

fun handleResult(result: Result<User>) = when (result) {
    is Result.Success -> println("성공: ${result.data.name}")
    is Result.Error -> println("실패: ${result.message}")
    Result.Loading -> println("로딩 중...")
}

// 4. 지연 초기화: lazy
val heavyObject: SomeHeavyClass by lazy {
    SomeHeavyClass()  // 처음 접근할 때 한 번만 초기화
}
```

---

## 마치며

Kotlin은 문법이 간결하고 널 안전성, 확장 함수, 코루틴 같은 강력한 기능 덕분에 Android 앱 개발에서 Java를 완전히 대체했다. 처음엔 `val/var`, 널 안전성, data class, 컬렉션 API에 익숙해지는 것부터 시작한다. 그다음 확장 함수와 스코프 함수를 자연스럽게 쓸 수 있게 되면 코드 품질이 눈에 띄게 달라진다. 코루틴은 비동기 처리가 필요한 시점에 집중적으로 파고들면 된다.

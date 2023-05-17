# item 5. 예외를 활용해 코드에 제한을 걸어라

확실하게 어떤 형태로 동작해야 하는 코드가 있다면 예외를 활용해 제한을 걸어주는 것이 좋음. 아래는 코틀린에서의 동작에 제한을 걸 때 사용할 수 있는 방법들임.

- require : 아규먼트를 제한
- check : 상태와 관련된 동작을 제한할 수 있음.
- assert : 어떤 것이 true인지 확인할 수 있음. 테스트 모드에서만 작동.
- return or throw와 함께 활용하는 Elvis 연산자.

제한을 걸었을 때의 장점은 다음과 같음.

- 문서을 읽지 않은 개발자도 문제를 확인할 수 있음.
- 문제가 있을 경우 함수가 예상하지 못한 동작을 하지 않고 예외를 throw함. 예상하지 못한 동작을 하는 것은 상태를 관리하는 데 꼬이게 함.
- 코드가 어느정도 자체적으로 검사됨. 따라서 이와 관련된 단위테스트를 줄일 수 있음.
- 스마트 캐스트 기능을 활용할 수 있으므로 캐스트(타입 변환)를 적게 할 수 있음.

<br/>

## 아규먼트

함수를 정의할 때 타입 시스템을 활용해서 argument에 제한을 거는 코드를 많이 사용함. 일반적으로 이러한 제한을 걸 때는 require 함수를 많이 사용함. require는 제한을 확인하고 만족하지 못할 경우 throw함. → IllegalArgument Exception을 발생시키므로 제한을 무시할 수 없음. 

```kotlin
fun factorial(n: Int): Long {
    require(n >= 0) // 제한!
    return if (n <= 1) 1 else factorial(n - 1) * n
}
```

<br/>

## 상태

구체적인 조건을 만족할 때만 함수를 사용할 수 있게 해야될 수 있음.

- 어떤 객체가 미리 초기화 되어야만 작동하는 함수
- 사용자가 로그인했을 때만 처리하는 함수
- 객체를 사용할 수 있는 시점에 사용하고 싶은 함수

이럴 때 check()를 사용함. 

```kotlin
fun speak(text: String) {
    check(isInitialized)
    // ...
}
```

check 함수는 require와 비슷하지만 지정된 예측을 만족하지 못할 때 IllegalStateException을 발생시킴. 일반적으로 require 뒤에 배치함. 

<br/>

## Assert 계열 함수 사용

참을 낼 수 있는 코드에 대해 점검하는 것. 단, 참이 아닐 때 프로덕션 환경에서는 오류를 throw하지 않음. 테스트를 할 때만 활성화가 됨. 

```kotlin
fun pop(num: Int = 1): List<T> {
    assert(ret.size == num)
    return ret
}
```

<br/>

## nullability와 스마트 캐스팅

require, check 블록으로 어떤 조건을 확인해서 true로 나오면 해당 조건 이후로도 true일 것이라고 가정해 스마트 캐스트가 작동함. null check 시 유용하게 사용할 수 있음. 이런 경우에는 requireNotNull, checkNotNull을 사용해서 변수를 언팩하는 용도로 사용할 수 있음. 

```kotlin
fun changeDress(person: Person) {
   require(person.email != null)
   val email: String = person.email
   //...
}
```

그리고 elvis 연산자도 우측에 throw, return을 두고 활용하면 Null check가 가능함. 만약 null일 때 여러 처리를 해주어야 하면 return/throw + run 함수를 조합해서 사용하면 됨. 

```kotlin
fun sendEmail(person: Person, text: String) {
    val email: String = person.email ?: return
}

fun sendEmail(person: Person, text: String) {
    val email: String = person.email ?: run {
			// ...
			return
		}
}
```

# item 2. 변수의 스코프를 최소화하라

프로퍼티보다는 지역 변수를 사용하는 것이 좋고, 최대한 좁은 스코프를 갖게 변수를 사용해야 함. 코틀린의 스코프는 기본적으로 중괄호에 만들어지며 내부 스코프에서 외부 스코프에 있는 요소에만 접근할 수 있음. 

<br/>

아래 코드에서 user는 스코프 내에서 선언 후 초기화해주면 되는데 굳이 스코프 외부에서 사용하고 있는 좋지 않은 방법의 예시임.

```kotlin
val users = listOf<User>()
var user: User

// 좋지 않은 방법
for (i in users.indices) {
    user = users[i]
    println("User at $i is $user")
}
```

아래 코드의 첫 번째 for문은 for문 내에서 변수를 선언해 변수의 스코프를 제한하고 있음. 두 번째 for문은 구조분해선언으로 변수 자체를 사용하지 않고 있음.  

```kotlin
for (i in users.indices) {
    val user = users[i]
    println("User at $i is $user")
}

for ((i, user) in users.withIndex()){
    println("User at $i is $user")
}
```

스코프 내부에 스코프가 있을 수도 있음. ex) 람다 안의 람다식. 

**스코프를 좁게 만들 때의 장점** → 프로그램을 추적하고 관리하기 쉬워짐. 왜냐하면 코드를 분석할 때 어떤 시점에 어떤 요소가 있는지 알아야 하는데 모든 요소의 스코프가 다 넓으면 모든 시점에서 모든 요소를 고려해야 함. 아주 어려움. 협업 시 어렵다는 점도 있음. 

**변수는 정의할 때 초기화되는 것이 좋음**

if, when, try-catch, elvis 표현식 등을 활용하면최대한 변수를 정의할 때 초기화할 수 있음. 또한 여러 프로퍼티를 한꺼번에 선언해야 되는 경우 구조분해 선언을 사용하면 좋음.

```kotlin
// if로 정의할 때 초기화
val user: User = if (hasValue) {
    getValue()
} else {
    User()
}

// 구조분해 선언
fun updateWeather(degrees: Int) {
    val (description, color) = when {
        degrees < 5 -> "color" to Color.BLUE
        degrees < 23 -> "mild" to Color.YELLOW
        else -> "hot" to Color.RED
    }
}
```


<br/>

## 캡처링

시퀀스에 대한 이해가 필요함. → [https://iosroid.tistory.com/79](https://iosroid.tistory.com/79)

**캡처링에 대한 이해**

[https://javabom.tistory.com/121](https://javabom.tistory.com/121)

[https://bottom-to-top.tistory.com/64](https://bottom-to-top.tistory.com/64)

근데 prime이 2일 때 왜 4가 filter 되는지 모르겠음.

# item 17. 이름 있는 아규먼트를 사용하라

### 의미가 명확하지 않은 아규먼트

```kotlin
// 1
val text = (1..10).joinToString("|")

// 2
val text = (1..10).joinToString(separator="|")

// 3
val separator= "|"
val text = (1..10).joinToString(separator)

// 4
val separator = "|"
val text = (1..10).joinToString(separator = separator)
```

1번의 코드는 `|`의 의미가 명확하지 않음. 2번에서는 이름 있는 아규먼트를 사용하고 있고, 3번에서는 변수를 사용해서 의미를 명확하게 하고 있음. 

변수 이름을 사용하는 방법은 개발자의 의도를 쉽게 알 수 있지만, 실제로 코드에서 제대로 사용되고 있는지 알 수 있는 방법이 없음. 변수를 잘 못 만들 수도 있고 함수 호출 때 잘못된 위치에 배치할 수도 있음. 이름 있는 아규먼트는 이런 문제가 발생하지 않음. 따라서 변수를 사용할 때도 4번과 같이 이름 있는 아규먼트를 사용하는 것이 best임. 

<br/>

### 이름 있는 아규먼트는 언제 사용?

코드가 길어지지만 다음과 같은 장점이 생김.

- 이름을 기반으로 값이 무엇을 나타내는지 알 수 있음.
- 파라미터 입력 순서와 상관 없으므로 안전함.

아래는 그 예시임. 

```kotlin
// worst
sleep(100)

// good
sleep(timeMillis = 100)
sleep(Millis(100))
sleep(100.ms) // 확장 프로퍼티로 DSL과 유사한 문법을 활용
```

성능을 고려한다면 inline class를 활용하면 됨. 

> inline class는 deprecated 됨. kotlin의 value class를 활용하는 것이 좋음.
> 

<br/>

### 디폴트 아규먼트의 경우

프로퍼티가 디폴트 아규먼트를 가질 경우 항상 이름을 붙여서 사용하는 것이 좋음. 

<br/>

### 같은 타입의 파라미터가 많은 경우

파라미터가 모두 다른 타입이라면 오류가 발생해 실수할 확률이 적음. 하지만 파라미터에 같은 타입이 있다면 잘못 입력했을 때 문제를 찾아내기 어려울 수 있음. 

<br/>

### 함수 타입 파라미터

일반적으로 함수 타입 파라미터는 마지막 위치에 배치하는 것이 좋음. 또한 이름 있는 아규먼트를 사용하게 된다면 아래 코드와 같이 의미를 더 명확하게 할 수 있음. 

```kotlin
checkedApiResult(
    apiResult = it,
    success = { data -> _gameList.value = data },
)
```

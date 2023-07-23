# item 13. Unit?을 리턴하지 말라

`Unit?`은 `Unit` 또는 `null`이라는 값을 가질 수 있음. 따라서 `Boolean`과 `Unit?` 타입은 서로 바꿔서 사용할 수 있음. 

```kotlin
// 1 - Boolean
fun keyIsCorrect(key: String): Boolean = //...
if(!keyIsCorrect(key)) return
```

```kotlin
// 2 - Unit?
fun verifyKey(key: String): Unit? = //...
verifyKey(key) ?: return
```

1번과 같이 구현했던 걸 Unit?을 사용해서 2번과 같이 구현할 수 있다. 하지만 각 코드들의 두 번째 줄(사용 예시)을 보면 Unit?을 사용하는게 이해하는데 리소스 소모가 더 크다는 걸 알 수 있음. if문으로 작성한 1번 예시가 더 이해하기 쉬움. 

####

📌 결론 : Unit?을 리턴하거나 이를 기반으로 연산하는 것은 좋지 않음.

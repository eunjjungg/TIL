# item 12. 연산자 오버로드를 할 때는 의미에 맞게 사용하라

### 오버로딩 의미에 맞게 써야 함

kotlin의 연산자들의 원래 의미

![image](https://github.com/eunjjungg/TIL/assets/100047095/3abec346-38ea-474e-b2db-c9be47bd719b)

예를 들어 + 연산자가 일반적인 의미로 사용되지 않고 있다면 연산자를 볼 때마다 연산자를 개별적으로 이해해야 하기 때문에 코드를 이해하기 어려울 것임. 

```kotlin
x + y == z

// plus의 리턴 타입이 널러블이 아닐 경우 아래와 같이 변환되어 실행됨.
x.plus(y).equal(z)

// plus의 리턴 타입이 널러블일 경우 아래와 같이 변환되어 실행됨.
(x.plus(y))?.equal(z) ?: (z == null) 
```

<br/>

### 오버로드를 했는데 의미가 명확하지 않다면 infix 활용

```kotlin
operator fun Int.times(operation:()->Unit) {
		repeat(this) { operation() }
}

3 * { print("Hello") } // Prints: HelloHelloHello
```

```kotlin
infix fun Int.timesRepeated(operation: ()->Unit) = { 
		repeat(this) { operation() }
}

val tripledHello = 3 timesRepeated { print("Hello") }
```

두 코드가 같은 기능을 수행하지만, times(*)를 오버로딩하는 것은 의미가 불분명하게 다가올 수 있기 때문에 저런 형식으로 코드를 짜고 싶다면 아래와 같이 infix를 사용해서 구현하는 걸 추천하고 있음. 

<br/>

### 연산자 오버로딩 규칙 무시해도 되는 경우

kotlin DSL을 설계할 때는 무시해도 됨.

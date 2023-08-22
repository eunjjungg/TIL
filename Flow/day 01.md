## 비동기 Flow

### ****복수의 값들 표현하기****

코틀린의 복수의 값들은 collections로 나타낼 수 있음. 

<br/>

### ****Sequences****

만약 CPU 리소스를 사용하면서 블로킹을 하는 코드로 숫자에 대한 연산을 한다면 Sequence로 나타낼 수 있음. 

```kotlin
fun simple(): Sequence<Int> = sequence { // sequence builder
    for (i in 1..3) {
        Thread.sleep(100) // pretend we are computing it
        yield(i) // yield next value
    }
}

fun main() { 
    simple().forEach { value -> println(value) } 
}
```

<br/>

### ****일시중단 함수들****

위 코드의 연산은 코드를 실행하는 메인 스레드를 블로킹함. 따라서 일시 중단 함수로 바꾸려면 아래와 같이 바꿔주면 됨.

- suspend fun으로 바꿔줌
- Thread.sleep 대신 일시 중단 함수인 delay 사용

```kotlin
suspend fun simple(): List<Int> {
    delay(1000) 
    return listOf(1, 2, 3)
}

fun main() = runBlocking<Unit> {
    simple().forEach { value -> println(value) } 
}
```

<br/>

### ****Flows****

위의 코드와 같이 결과 타입으로 `List<Int>`를 사용하면 한 번에 모든 값을 반환해야 함. 동기적으로 계산된 값을 `Sequence`에 담았던 것처럼 비동기적으로 계산된 값들을 스트림으로 나타내기 위해서는 `Flow`를 사용하면 됨.

```kotlin
fun simple(): Flow<Int> = flow { // flow builder
    for (i in 1..3) {
        delay(100) 
        emit(i) 
    }
}

fun main() = runBlocking<Unit> {
    launch {
        for (k in 1..3) {
            println("I'm not blocked $k")
            delay(100)
        }
    }
    // Collect the flow
    simple().collect { value -> println(value) }
}

// I'm not blocked 1
// 1
// I'm not blocked 2
// 2
// I'm not blocked 3
// 3
```

위 코드에서 flow는 메인 스레드를 블로킹하지 않고 100ms 기다린 후 값을 방출함. 메인스레드를 블로킹하지 않는다는 것은 I’m not blocked가 출력되는 것으로 알 수 있음. 

- Flow의 빌더 함수는 flow이다.
- flow { ... } 블록 내부의 코드들은 일시 중단 될 수 있다.
- **simple 함수는 더이상 suspend 수정자로 표시되어 있지 않다.**
- emit 함수를 사용해 flow에서 값들이 방출된다.
- collect 함수를 사용해 flow로부터 값들을 수집한다.

<br/>

### ****Flows는 차갑다****

Flow는 시퀀스와 마찬가지로 차가운 스트림임. flow { } 내부는 collect가 되기 전까지 실행되지 않음. 

```kotlin
fun simple(): Flow<Int> = flow {
    println("Flow started")
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    println("Calling simple function...")
    val flow = simple()
    println("Calling collect...")
    flow.collect { value -> println(value) }
    println("Calling collect again...")
    flow.collect { value -> println(value) }
}

// Calling simple function...
// Calling collect...
// Flow started
// 1
// 2
// 3
// Calling collect again...
// Flow started
// 1
// 2
// 3
```

이 코드를 보면 simple 함수를 호출했을 때는 아무런 작동을 하지 않다가 collect를 하고 나서야 플로우 빌더 내부가 실행된 것을 알 수 있다. 이것이 flow를 반환하는 simple 함수에 suspend가 붙지 않은 이유다. simple 함수의 호출 그 자체는 곧바로 반환되어 아무것도 기다리지 않음. flow는 collect가 될 때마다 **새로** 시작됨. 

---

**📚 차가운 스트림?**

flow는 reactive programming의 publisher 중 cold stram에 속함. 콜드 스트림은 구독이 일어나기 전까지 내부 코드가 실행되지 않고 구독이 일어날 경우에 블록 내부의 코드가 실행되는 스트림임. 반대로 Hot stream은 명시적으로 구독을 해주지 않아도 코드가 실행됨. i.e. Hot Stream, Cold Stream을 나누는 기준은 코드의 실행이지 발행이 아님. 

---

<br/>

### ****Flow 취소 기초****

Flow는 코루틴의 기본적인 협력적인 취소를 따름. 취소 가능한 일시 중단 함수에서 Flow가 일시중단될 때 Flow로부터 값을 수집하는 것이 취소될 수 있음. 

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    withTimeoutOrNull(250) { // Timeout after 250ms
        simple().collect { value -> println(value) }
    }
    println("Done")
}

// Emitting 1
// 1
// Emitting 2
// 2
// Done
```

<br/>

### ****Flow 빌더****

- `flowOf { }`는 정해진 값의 세트를 방출하는 Flow를 정의함.
- 다양한 Collection들과 Sequence들은 `.asFlow()` 확장 함수를 사용해 Flow로 변환할 수 있음.

<br/>

### ****Flow 중간 연산자****

Flow들은 연산자를 이용해 반환될 수 있음. 이러한 연산자들은 일시중단 함수가 아니며 빠르게 작동해 새로운 플로우를 반환함. 

기본 연산자들은 map, filter등이 있음. 이러한 연산자들과 Sequence의 차이점은 이 연산자들 내부의 코드 블록에서는 suspend fun을 호출할 수 있음. 

```kotlin
suspend fun performRequest(request: Int): String {
    delay(1000) // imitate long-running asynchronous work
    return "response $request"
}

fun main() = runBlocking<Unit> {
    (1..3).asFlow() // a flow of requests
        .map { request -> performRequest(request) } // request: Int
        .collect { response -> println(response) } // response: String
}

// response 1
// response 2
// response 3
```

위의 코드에서 request의 타입은 정수형이고 response의 타입은 문자열이다. 

<br/>

### Ref.

- [https://myungpyo.medium.com/코루틴-공식-가이드-자세히-읽기-part-9-a-d0082d9f3b89](https://myungpyo.medium.com/%EC%BD%94%EB%A3%A8%ED%8B%B4-%EA%B3%B5%EC%8B%9D-%EA%B0%80%EC%9D%B4%EB%93%9C-%EC%9E%90%EC%84%B8%ED%9E%88-%EC%9D%BD%EA%B8%B0-part-9-a-d0082d9f3b89)
- https://seyoungcho2.github.io/CoroutinesKoreanTranslation/flow.html
- https://simplecode.kr/37

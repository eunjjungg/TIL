## 비동기 Flow

### ****Transform 연산자****

map, filter와 같은 간단한 변환을 할 때 사용되며 transform 연산자를 사용하면 임의의 횟수만큼 값을 emit 할 수 있음.

```kotlin
suspend fun performRequest(request: Int): String {
    delay(1000) 
    return "response $request"
}

fun main() = runBlocking<Unit> {
    (1..3).asFlow() 
        .transform { request ->
            emit("Making request $request")
            emit(performRequest(request))
        }
        .collect { response -> println(response) }
}

// Making request 1
// response 1
// Making request 2
// response 2
// Making request 3
// response 3
```

위의 예시와 같이 `emit`을 원하는 횟수만큼 넣을 수 있음. 따라서 비동기 작업이 오래 걸리면 응답 값을 받기 전까지 특정 값을 `emit`하고 그 작업이 완료되면 응답 값을 `emit`해주는 방식으로 응용할 수 있음. 

<br/>

### ****크기 한정 연산자****

take과 같은 크기 한정 중간 연산자들은 임계치에 도달하면 flow의 실행을 취소함. coroutine의 취소는 CancellationException을 throw하여 실행되므로 try - finally 구문으로 처리해주면 됨. 

```kotlin
fun numbers(): Flow<Int> = flow {
    try {
        emit(1)
        emit(2)
        println("This line will not execute")
        emit(3)
    } catch (e: CancellationException) {
        println("cancel")
    } finally {
        println("Finally in numbers")
    }
}

fun main() = runBlocking<Unit> {
    numbers()
        .take(2) // take only the first two
        .collect { value -> println(value) }
}

// 1
// 2
// cancel
// Finally in numbers
```

위의 코드에서는 임계치가 2이므로 1, 2에 대해서만 emit이 되고 3번째 값이 emit 되었을 때 플로우가 캔슬됨. CancellationException은 main에서는 잡히지 않으므로 flow 안에서 try - catch를 수행해주어야 함. 만약 try - catch - finally 없이 실행하면 아래와 같이 출력 후 정상종료가 됨. (애초에 take은 익셉션이 발생하지 않음.)

```kotlin
fun numbers(): Flow<Int> = flow {
        emit(1)
        emit(2)
        emit(3)
}

fun main() = runBlocking<Unit> {
    numbers()
        .take(2) // take only the first two
        .collect { value -> println(value) }
}

// 1
// 2
```

<br/>

### ****Flow 터미널 연산자****

flow의 터미널 연산자는 flow를 collect를 시작하는 suspend fun임. 플로우는 차가운 스트림이므로 터미널 연산자가 실행되어야 방출되어야 하는 값들의 수집을 시작함. 

- collect: flow를 collect하는 가장 기본적인 연산자
- toList: list로 변환
- toSet: set으로 변환
- first: 첫 번째 값만 가져옴
- single: 하나의 값만 가져오고 하나의 값만 방출된 것을 확인하는연산자
- flow를 값으로 만드는 reduce, fold 연산자.

<br/>

📚 **toList**

```kotlin
fun numbers(): Flow<Int> = flow {
        emit(1)
        emit(2)
        emit(3)
}

fun main() = runBlocking<Unit> {
    println( numbers().toList())
}

// [1, 2, 3]
```

![image](https://github.com/eunjjungg/TIL/assets/100047095/bbe8e74f-5d7a-4877-9f3a-abc24d380ccd)

Flow의 확장 함수로 되어있는 toList임. collect 연산으로 종단연산임. 

<br/>

**📚 first, single**

```kotlin
fun numbers(): Flow<Int> = flow {
        emit(1)
        emit(2)
        emit(3)
}

fun main() = runBlocking<Unit> {
    println(numbers().first()) // 1
    println(numbers().single()) // ... IllegalArgumentException: Flow has more than one element
}
```

만약 위 코드에서 emit(1)만 하고 2, 3에 대한 방출을 지워준다면 오류 없이 둘 다 1이 출력됨. 

<br/>

**📚 reduce, fold**

reduce, fold는 원래는 컬렉션 내의 데이터를 모두 모으는 함수임. 근데 이걸 플로우에도 적용할 수 있음. reduce는 초기값 없이 첫 번째 요소로 시작하고, fold는 지정해준 초기 값으로 시작함. 

```kotlin
fun main() = runBlocking<Unit> {
    val sum = (1..5).asFlow()
        .map { it * it } // squares of numbers from 1 to 5
        .reduce { a, b -> a + b } // sum them (terminal operator)
    println(sum) 

    val sum2 = (1..5).asFlow()
        .map { it * it } // squares of numbers from 1 to 5
        .fold(10) { a, b -> a + b } // sum them (terminal operator)
    println(sum2) 
}

// 55
// 65
```

<br/>

### ****Flow는 순차적이다****

- 기본적으로 개별 플로우 컬렉션은 **순차적**으로 동작함.
- collect는 종단 연산자를 호출한 코루틴에서 직접 수행되며 기본적으로 새로운 코루틴을 생성해서 collect하는 것이 아님.
- 각각 emit 된 값들은 업스트림의 모든 중간 연산자들에 의해 처리되어 다운스트림으로 전달되며 최종적으로 종단 연산자(collect…)로 전달됨.

```kotlin
fun main() = runBlocking<Unit> {
    (1..5).asFlow()
        .filter {
            println("Filter $it") // 1, 2, 3, 4, 5
            it % 2 == 0
        }
        .map {
            println("Map $it") // 2, 4
            "string $it"
        }.collect {
            println("Collect $it") 
        }
}

// Filter 1
// Filter 2
// Map 2
// Collect string 2
// Filter 3
// Filter 4
// Map 4
// Collect string 4
// Filter 5
```

<br/>

### ****Flow Context****

flow의 수집은 항상 호출한 코루틴의 context 안에서 수행됨. ⇒ Context Preservation (컨텍스트 보존)이라는 성질로 불림. 

따라서 simple이라는 플로우가 있을 때 사용자가 지정한 context 상에서 collect가 시작됨. 

```kotlin
fun simple(): Flow<Int> = flow {
    log("Started simple flow")
    for (i in 1..3) {
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    simple().collect { value -> log("Collected $value") }
}

// [main @coroutine#1] Started simple flow
// [main @coroutine#1] Collected 1
// [main @coroutine#1] Collected 2
// [main @coroutine#1] Collected 3
```

⇒ flow는 실행 컨텍스트에 관계 없이 + 호출자를 블록하지 않는 비동기 작업을 수행함. 

<br/>

### ****withContext를 사용할 때 일반적으로 겪을 수 있는 함정****

오래도록 실행되는 CPU 소모적인 작업들은 Dispatchers.Default와 같은 별도의 스레드에서 실행되어야 함. 반면 UI update 코드들은 Dispatchers.Main 스렏에서 실행되어야 함. 보통 withContext는 코루틴 코드에서 컨텍스트를 전환하기 위해 사용됨. 하지만 flow 빌더 내부의 코드는 컨텍스트 보존 특성을 지켜야 하기 때문에 다른 컨텍스트에서 값을 방출하는 것이 허용되지 않음. 

<br/>

### Ref.

- https://simplecode.kr/40
- https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/to-set.html
- [https://myungpyo.medium.com/코루틴-공식-가이드-자세히-읽기-part-9-a-d0082d9f3b89](https://myungpyo.medium.com/%EC%BD%94%EB%A3%A8%ED%8B%B4-%EA%B3%B5%EC%8B%9D-%EA%B0%80%EC%9D%B4%EB%93%9C-%EC%9E%90%EC%84%B8%ED%9E%88-%EC%9D%BD%EA%B8%B0-part-9-a-d0082d9f3b89)
- https://seyoungcho2.github.io/CoroutinesKoreanTranslation/flow.html
- https://velog.io/@devoks/Kotlin-Flow

## 비동기 Flow

### ****flowOn 연산자****

Flow에서 값 방출을 위한 Context를 변경하는데 사용.

```kotlin
fun simple(): Flow<Int> = flow {
        for (i in 1..3) {
                Thread.sleep(100)
                log("Emitting $i")
                emit(i) 
        }
}.flowOn(Dispatchers.Default)

fun main() = runBlocking<Unit> {
        simple().collect { value ->
                log("Collected $value")
        }
}

// [DefaultDispatcher-worker-1 @coroutine#2] Emitting 1
// [main @coroutine#1] Collected 1
// [DefaultDispatcher-worker-1 @coroutine#2] Emitting 2
// [main @coroutine#1] Collected 2
// [DefaultDispatcher-worker-1 @coroutine#2] Emitting 3
// [main @coroutine#1] Collected 3
```

위의 코드는 `flowOn` 연산자를 사용해 Context를 main → Default Dispatcher로 변경하고 있음. 

`flowOn` 연산자가 flow의 기본적인 특성인 순차성을 일부 포기함. 플로우의 값이 수집되는 코루틴과 값을 방출하는 코루틴은 각기 다른 스레드에서 실행됨. 

`flowOn` 연산자는 컨텍스트 내에서 `CoroutineDispatcher`를 변경해야 할 경우 업스트림 플로우(여기서는 값을 방출하는 코루틴)를 위해 다른 코루틴을 생성함. 

- [x]  근데 순차성을 왜 포기하는거지 이게? 컨텍스트 바꾸는게?!?! 컨텍스트 보존을 포기하는거 아닌가 ⇒ 순차성이라는 건 각 값이 발행 되고 소비 되었을 때 다음 발행을 시작하는 것이니까요…

<br/>

### ****Buffering****

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        log("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        simple()
            .buffer()
            .collect { value ->
                println("$value 진입")
                delay(300)
                println(value)
            }
    }
    println("Collected in $time ms")
}
```

코드가 위와 같이 있을 때 buffer를 주석처리 했을 때와 안 했을 때의 실행 결과는 다음과 같음.

| buffer 연산자 없을 때 | buffer 연산자 있을 때  |
| --- | --- |
| [main @coroutine#1] Emitting 1 | [main @coroutine#2] Emitting 1|
1 진입 |1 진입|
1 |[main @coroutine#2] Emitting 2|
[main @coroutine#1] Emitting 2 |[main @coroutine#2] Emitting 3|
2 진입 |1|
2 |2 진입|
[main @coroutine#1] Emitting 3 |2|
3 진입 |3 진입|
3 |3|
Collected in 1246 ms | Collected in 1058 ms 
| 값 1 발행 → 값 1 소비 → 값 1 소비 완료 → 값 2 발행 → 값 2 소비 → 값 2 소비 완료 → 값 3 발행 → 값 3 소비 → 값 소비 완료 | 값 1 발행 → 값 1 소비 & 값 2 발행 → 값 3 발행 → 값 1 소비 완료 → 값 2 소비 → 값 2 소비 완료 → 값 3 소비 → 값 3 소비 완료 |
| 각 값의 emit이 완료된 이후에 collect 내부를 실행하고 다음 값의 발행을 하는 순차적인 구조임. 발행과 소비가 순차적이므로 즉 발행에 지연이 생기면 소비에도 지연이 생기게 됨.  | 각 값의 소비 완료 여부에 관계 없이 발행할 값이 있다면 발행을 함 → 요건 buffer가 발행하는 쪽과 소비하는 쪽의 코루틴을 분리하는 역할을 했기에 가능함. |

<br/>

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        log("Emitting $i")
        emit(i)
    }
}.flowOn(Dispatchers.Default)

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        simple()
            .collect { value ->
                println("$value 진입")
                delay(300)
                println(value)
            }
    }
    println("Collected in $time ms")
}
```

위 코드는 `buffer`를 지우고 `flowOn`을 플로우에 사용했음. 이랬을 때의 결과는 `buffer`를 사용했을 때의 결과와 같음. 

⇒ `flowOn` 연산자는 `CoroutineDispatcher`를 변경해야 할 때 동일한 buffering 매커니즘을 사용한다고 함. 

<br/>

### ****Conflation****

flow의 중간 값들은 필요 없는데 collect의 연산 속도가 너무 느려 collect 순간에 여러 값들이 있을 때 최종 값에 대해서만 collect를 해주고 싶은 경우 `conflate` 연산자를 사용해주면 됨. 

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        log("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        simple()
            .conflate()
            .collect { value ->
                println("$value 진입")
                delay(300)
                println(value)
            }
    }
    println("Collected in $time ms")
}

// [main @coroutine#2] Emitting 1
// 1 진입
// [main @coroutine#2] Emitting 2
// [main @coroutine#2] Emitting 3
// 1
// 3 진입
// 3
// Collected in 760 ms
```

1을 수집하는건 당연함. 수집할게 없다가 생겼으니까. 근데 1 소비가 끝나고 다음거 소비할거 찾는데 방출된 값인 2, 3이 있음. 그러나 `conflate` 연산자를 사용했기 때문에 가장 최신 값인 3에 대해서만 소비를 해주면 됨. 

`conflate` 연산자도 내부적으로 `buffer`를 사용함.

<br/>

### ****최신 값 처리하기****

새로운 값이 방출될 때마다 collect의 작업이 이미 진행중이었다면 이를 취소하고 새로운 방출된 값으로 수집을 다시 시작하게 하는 연산자가 `collectLatest`임. 

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        log("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        simple()
            .collectLatest { value -> // cancel & restart on the latest value
                println("진입 $value")
                delay(300) // pretend we are processing it for 300 ms
                println("완료 $value")
            }
    }
    println("Collected in $time ms")
}

// [main @coroutine#2] Emitting 1
// 진입 1
// [main @coroutine#2] Emitting 2
// 진입 2
// [main @coroutine#2] Emitting 3
// 진입 3
// 완료 3
// Collected in 692 ms
```

`collectLatest` 연산자를 사용했으므로 1을 제외한 값들이 방출될 시점에는 이전 값의 수집이 이루어지고 있기 때문에 취소하고 자신의 값에 대해 처리를 하게 됨. 

내부적으로 얘도 buffer를 사용하고 있는듯함. 아래 페이지 참고.

- https://stackoverflow.com/questions/76349848/what-is-the-effect-of-buffer0-in-the-collectlatest-implementation-in-kotlin

<br/>

### Ref.

- [https://myungpyo.medium.com/코루틴-공식-가이드-자세히-읽기-part-9-a-d0082d9f3b89](https://myungpyo.medium.com/%EC%BD%94%EB%A3%A8%ED%8B%B4-%EA%B3%B5%EC%8B%9D-%EA%B0%80%EC%9D%B4%EB%93%9C-%EC%9E%90%EC%84%B8%ED%9E%88-%EC%9D%BD%EA%B8%B0-part-9-a-d0082d9f3b89)
- https://kotlinworld.com/253
- https://simplecode.kr/40
- https://seyoungcho2.github.io/CoroutinesKoreanTranslation/flow.html
- https://hik-coding.tistory.com/124

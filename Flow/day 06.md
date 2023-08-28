## ****Flow 수집 완료 처리하기****

### ****선언적인 처리****

- flow의 수집이 완료되었을 때 실행되는 onCompletion 중간 연산자가 있음.
- onCompletion은 수집 작업이 정상적으로 or 예외적으로 완료되었는지 확잏나는데 사용할 수 있는 람디식의 nullable한 Throwable 파라미터임.
- catch와는 다르게 onCompletion 연산자는 예외를 처리하지 않음. 예외는 여전히 다운스트림으로 전달됨.

```kotlin
fun main() = runBlocking<Unit> {
    simple()
        .onCompletion { cause -> if (cause != null) println("Flow completed exceptionally") }
        .catch { cause -> println("Caught exception") }
        .collect { value -> println(value) }
}
```

<br/>

### ****성공적인 완료****

`catch` 연산자와 또 다른 점은 `onCompletion`은 모든 예외를 확인할 수 있음. 업스트림 플로우가 취소나 실패 없이 성공적으로 완료되면 `null`을 수신함. 

```kotlin
fun simple(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
    simple()
        .onCompletion { cause -> println("Flow completed with $cause") }
        .collect { value ->
            // check(value <= 1) { "Collected $value" }
            println(value)
        }
}
```

`check`가 없다면 아래와 같이 출력됨.

```kotlin
1
2
3
Flow completed with null
```

그러나 `check`가 있다면 2에서 오류가 발생하므로 아래와 같이 출력됨. 주목할 부분은 Flow completed 사유가 2가 아니라는 점임. 

```kotlin
1
Flow completed with java.lang.IllegalStateException: Collected 2
Exception in thread "main" java.lang.IllegalStateException: Collected 2
```

<br/>

## ****명령적으로 다루기 vs 선언적으로 다루기****

- 두 방법 다 상관 없고 선호도와 코드 스타일에 따라 선택하면 됨.

<br/>

### ****Flow 실행하기****

플로우를 수집하기 위해서는 터미널 연산자가 필요함. 따라서 들어오는 이벤트에 대한 반응을 `onEach`로만 수행시킬 때는 의미가 없음. 

만약 `onEach` 이후에 collect 터미널 연산자를 사용하면 이후의 코드는 Flow가 수집될 때까지 기다림. ⇒ 비동기적인 수행이 아니라 동기적인 수행임.

이럴 때 launchIn 터미널 연산자를 사용하면 됨. Flow의 수집을 별도의 코루틴에서 실행할 수 있으므로 이후의 코드들이 즉시 계속해서 실행되는 비동기 구조임.

```kotlin
fun main() = runBlocking<Unit> {
    simple()
        .onEach { event -> println("Event: $event") }
        .launchIn(this) // <--- Launching the flow in a separate coroutine
    println("Done")
}

// Done
// Event: 1
// Event: 2
// Event: 3
```

`launchIn`에서 필요로 하는 CoroutineScope 파라미터는 코루틴 스코프를 특정해서 플로우가 실행되면 어떤 코루틴이 수집할지 결정하도록 함. 

위 코드에서는 runBlocking 코루틴 빌더로부터 시작 ⇒ flow가 실행되는 동안 runBlocking Scope가 자식 코루틴이 완료될 때까지 기다리도록 하고, 메인 함수 반환을 막음.

실제 어플리케이션에서는 한정된 라이프사이클을 가진 것으로부터 스코프를 가져옴. → 스코프 종료 시 해당 플로우의 수집도 중단됨. 따라서 자식 코루틴(플로우)에 대한 종료를 일일이 체크해줄 필요가 없음. (because of 구조화된 동시성!)

⇒ e.g. `viewModelScope`: lifecycle이 종료되면 해당 스코프는 취소되며 플로우의 수집은 종료됨. 

![image](https://github.com/eunjjungg/TIL/assets/100047095/5c4fdf70-f01d-4e5a-b25b-c2e58bd4a8c7)

`launchIn`의 반환값이 `Job`인데 이게 전체 Scope에 대한 cancel, join을 지원하는 것이 아닌 해당 플로우를 수집하는 코루틴만 cancel 가능한 `Job`을 반환함. 

```kotlin
public fun <T> Flow<T>.launchIn(scope: CoroutineScope): Job = scope.launch {
    collect()
}
```

launchIn 내부에서는 설정한 스코프로 코루틴을 launch해주고 있고 수신객체인 플로우에 대해 collect해주고 있음. 

<br/>

### ****Flow 취소 체크****

flow 빌더는 추가적으로 방출된 값에 대한 취소 동작을 하기 위한 `ensureActive` 체크를 수행함. ⇒ emit을 하는 것이 취소가 가능하다는 의미. 

다른 플로우 연산자들은 성능상의 이유로 추가적인 취소 체크를 하지 않음. 

```kotlin
fun main() = runBlocking<Unit> {
    (1..5).asFlow().collect { value -> 
        if (value == 3) cancel()  
        println(value)
    } 
}

// 1
// 2
// 3
// 4
// 5
// Exception in thread "main" kotlinx.coroutines.JobCancellationException:
```

따라서 위의 예시처럼 1~5까지의 모든 숫자들이 수집되고 runBlocking이 반환되기 전에 취소가 감지됨.

이런 <바쁜 플로우>를 취소 가능하게 만드려면 ? → 명시적으로 취소를 체크해야 됨. 그 방법에는 두 가지가 있음.

1. `.onEach { currentCoroutineContext().ensureActive() }` 연산자 추가
2. `.cancellable()` 연산자 추가

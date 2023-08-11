## ****일시중단 함수 구성하기****

### ****기본적인 순차 처리****

```kotlin
fun main(args: Array<String>) = runBlocking {
    val time = measureTimeMillis {
        val one = one()
        val two = two()
        println("The answer is ${one + two}")
    }
    println("Completed in $time ms")
}

suspend fun one(): Int {
    delay(1000L)
    return 13
}

suspend fun two(): Int {
    delay(1000L)
    return 29
}

// The answer is 42
// Completed in 2018 ms
```

원격 서비스 호출이나 계산 같은 두 suspend fun 함수들이 서로 다른 위치에 정의되어 있을 때, `one`을 호출하고 `two`를 호출한 뒤 두 결과의 조합이 필요한 경우 어떻게 구성하면 좋을까? 위와 같이 순차적으로 호출하게 되면 Coroutines는 일반적인 코드와 같이 기본적으로 순차적이기 때문에 일반적인 순차 호출을 사용함. 따라서 2초가까이 걸리게 됨. 

<br/>

### ****async를 사용한 동시성****

one 함수의 계산 결과를 바탕으로 two 함수가 실행된다면 위의 코드도 괜찮지만 one, two 사이의 종속성이 없어 같이 시작되어도 무방할 때 `async`를 사용하면 됨. 

개념적으로 `async`는 `launch`와 같음. `async`는 다른 스레드들과 동시에 동작하는 별도의 경량 스레드인 코루틴을 시작함. 그러나 `launch`는 **결과값을 전달하지 않는** `Job`을 리턴하지만 async는 **나중에 결과값을 반환할 것을 약속**하는 `Future`인 `Deffered`를 반환함. `Deferred`에 대해 `.await()` 함수로 결과값을 얻을 수 있지만 `Deferred` 또한 `Job`이라 필요할 때 취소할 수 있음. 따라서 아래와 같이 사용하면 됨. 

```kotlin
fun main(args: Array<String>) = runBlocking {
    val time = measureTimeMillis {
        val one = async { one() }
        val two = async { two() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}

suspend fun one(): Int {
    delay(1000L)
    return 13
}

suspend fun two(): Int {
    delay(1000L)
    return 29
}

// The answer is 42
// Completed in 1016 ms
```

내부적으로 `launch` 빌더는 StandaloneCoroutine을 생성하고 `async` 빌더는 DefferedCoroutine을 생성하며 DefferedCoroutine은 Deffered<T> 인터페이스를 통해 await 함수를 제공함. 

코루틴 프레임워크는 순차 실행이 기본이기 때문에 동시 수행은 항상 명시적이어야 함. 

<br/>

### ****async lazy하게 시작하기****

선택적으로 첫 파라미터 값을 `CoroutineStart.LAZY`로 설정하면 `async`를 lazy하게 만들 수 있음. 이렇게 하면 Coroutine의 결과값이 `await`에 의해 필요해지거나 Job의 `start` 함수가 실행될 때 시작됨. 

```kotlin
fun main() = runBlocking {
    val time = measureTimeMillis {
        val one = async(start = CoroutineStart.LAZY) { one() }
        val two = async(start = CoroutineStart.LAZY) { two() }
				
				// one.start()
        // two.start()

        runBlocking {
            println("The answer is ${one.await() + two.await()}")
        }
    }
    println("Completed in $time ms")
}

// The answer is 42
// Completed in 2023 ms
```

따라서 다음과 같이 코드를 작성하면 2초가 지난 후에 연산이 완료되는데 그 이유는 one.await()이 실행될 때 첫 번째 작업이 시작되고 종료된 후 two.await()이 실행되는 lazy하게 실행되는 구조이기 때문. 따라서 이를 동시에 시작하게 하려면 각각의 Job을 시작하는 주석처리된 start 함수를 주석해제 하면 됨. 

lazy하게 사용하는 건 일반적인 `async`와 동작이 다르기에 주의해서 사용하면 좋을듯. 

<br/>

### ****비동기 스타일 함수****

구조적인 동시성에서 벗어나기 위해 `GlobalScope`를 참조하는 `async` 코루틴 빌더를 사용하여 비동기 스타일의 함수를 실행할 수 있음.

- `async` 코루틴 빌더를 통해 만든 비동기 스타일의 함수에는 접미사로 Async 키워드를 갖는 것이 좋음. 그러면 사용할 때 이 함수가 비동기 계산을 실행하기만 하고 결과값을 얻기 위해 `Deferred` 값을 사용해야 한다는 것을 알 수 있음.
- 명시적으로 `GlobalScope`를 `@OptIn(DelicateCoroutinesApi::class)`과 함께 사용해야 함.
- [ ]  왜?

```kotlin
@OptIn(DelicateCoroutinesApi::class)
fun oneAsync() = GlobalScope.async {
    one()
}

@OptIn(DelicateCoroutinesApi::class)
fun twoAsync() = GlobalScope.async {
    two()
}

suspend fun one(): Int {
    delay(1000L)
    return 13
}

suspend fun two(): Int {
    delay(1000L)
    return 29
}
```

이러한 `xxxAsync` 함수는 일시중단 함수가 아님. 이 함수들은 어디에서든지 사용될 수 있음. 하지만 이 함수를 호출할 때 이 동작은 언제나 비동기적 실행을 포함함. 따라서 위의 코드를 실제로 사용하는 쪽을 보면 다음과 같음.

```kotlin
fun main() {
    val time = measureTimeMillis {
        val one = oneAsync()
        val two = twoAsync()
        runBlocking {
            println("The answer is ${one.await() + two.await()}")
        }
    }
    println("Completed in $time ms")
}

// The answer is 42
// Completed in 1060 ms
```

여기서 주목해서 보아야 하는 점은 다음과 같음.

- `xxxAsync` 함수는 코루틴 바깥에서 시작될 수 있음. 그러나 결과를 기다리는 `await`을 사용하기 위해서는 일시중단 함수 내부에서 사용하거나 블로킹을 사용해야 함.
- 따라서 우리는 결과를 얻기 위해 `runBlocking` 내부에서 값을 기다려주고 있음.

만약 위 코드에서 val one = oneAsync() 부분과 one.await() 부분 사이에 프로그램이 예외를 발생시켜 프로그램에 의해 수행되던 작업이 중단되면 어떻게 될까? 일반적으로 전역 오류 처리기는 이 예외를 잡아 로깅하고 보고할 수 있고 그렇지 않으면 프로그램은 다른 작업을 계속 실행할수 있음. 그러나 시작한 작업이 종료되었음에도 불구하고 oneAsync가 계속해서 실행중 상태이게 됨. 이런 문제는 구조적인 동시성을 적용한 후에는 발생하지 않음. 

→ 이해가 잘 가지 않아 찾아보니 그냥 `xxxAsync` 함수 호출과 await() 사이에 오류가 발생되면 `xxxAsync` 함수는 오류와 관계없이 수행되는 문제가 있다는 식으로 받아들이면 될듯. 

<br/>

### ****구조화된 동시성과 async****

```kotlin
suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { one() }
    val two = async { two() }
    one.await() + two.await()
}

suspend fun one(): Int {
    delay(1000L)
    return 13
}

suspend fun two(): Int {
    delay(1000L)
    return 29
}
```

`async` Coroutine Builder가 `CoroutineScope`의 확장 함수로 정의되어 있기 때문에 이를 Scope내에 포함해야 하며, 이것이 `coroutineScope` 함수가 제공하는 기능이다. 이렇게 만들면 `concurrentSum` 함수 내부에서 문제가 생겨서 예외 발생 시 Scope 내부에서 실행된 모든 코루틴들이 취소됨. 즉 바로 위 주제에서 발생했던 문제인 `xxxAsync` 함수 호출과 await() 사이에 오류가 발생되면 `xxxAsync` 함수는 오류와 관계없이 수행되는 문제가 해결됨. 구조화된 동시성 모델을 유지하면 일관된 에러 처리를 할 수 있으며 에러의 전파도 정확하게 동작함. Cancellation 역시 코루틴 계층을 따라 전파됨. 

```kotlin
fun main(args: Array<String>) = runBlocking {
    try {
        val time = measureTimeMillis {
            println("The answer is ${failedConcurrentSum()}")
        }
        println("Completed in $time ms")
    } catch (throwable: Throwable) {
        println("Computation failed with $throwable")
    }
}

suspend fun failedConcurrentSum(): Int = coroutineScope {
    val one = async {

        try {
            delay(Long.MAX_VALUE) // emulates very long computation
            one()
        } finally {
            println("First child was canceled.")
        }
    }
    val two = async<Int> {
        println("Second child throw an exception.")
        two()
        throw ArithmeticException("Exception on purpose.")
    }
    one.await() + two.await()
}

// Second child throw an exception.
// First child was canceled.
// Computation failed with java.lang.ArithmeticException: Exception on purpose.
```

위 코드의 동작 순서는 아래와 같음.

1. `failedConcurrentSum` 실행
2. `one.await()`으로 `one` 먼저 실행 → 그러나 delay를 어마무시하게 걸어놨기 때문에 작업은 진행중이지만 결과가 나오지는 않음. 
3. `two.await()`으로 `two` 실행 → Second child throw an exception. 출력 → `doSomethingUsefulTwo` 내부 작업 다 실행 → throw `ArithmeticException("Exception on purpose.")`
4. error main으로 전파 → one 비동기 코루틴 취소 → First child was canceled. 출력 
5. Computation failed .. 출력
- [ ]  one의 cancellation은 왜 안 잡히지? → 잡힌다! 근데 왜 two에서 발생한 익셉션이 원으로 감? : 그런거 아님. two에서 발생시킨 ArithmeticException이 main으로 전파되고 main이 내부 코루틴들을 종료시키기 위해 one에 코루틴 취소를 발생시키는 거라고 이해하면 될까요?!

<br/>

### Ref.

- [https://myungpyo.medium.com/코루틴-공식-가이드-자세히-읽기-part-4-70b0c0fb492](https://myungpyo.medium.com/%EC%BD%94%EB%A3%A8%ED%8B%B4-%EA%B3%B5%EC%8B%9D-%EA%B0%80%EC%9D%B4%EB%93%9C-%EC%9E%90%EC%84%B8%ED%9E%88-%EC%9D%BD%EA%B8%B0-part-4-70b0c0fb492)
- https://seyoungcho2.github.io/CoroutinesKoreanTranslation/coroutine-context-dispatcher.html

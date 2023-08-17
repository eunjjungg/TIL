## ****Coroutine 예외 처리****

### ****Exception 전파****

코루틴 빌더는 자동으로 예외를 전파하거나 사용자에게 예외를 노출함. 

- 전파: `launch`, `actor`
- 노출: `async`, `produce`

`launch`와 `actor`로 root coroutine을 생성할 때 예외를 uncaught exception으로 다룸. 그러나 `async`, `produce`로 root coroutine을 생성하면 `await`, `recieve`를 통해 사용자가 마지막 예외를 소비하는지에 의존함. 

```kotlin
@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    val job = GlobalScope.launch {
        println("Throwing exception from launch")
        throw IndexOutOfBoundsException()
        println("Unreached") // 도달 x
    }
    job.join()
    println("Joined failed job")
    val deferred = GlobalScope.async {
        println("Throwing exception from async")
        throw ArithmeticException()
        println("Unreached") // 도달 x
    }
    try {
				delay(1500L)
        deferred.await()
        println("Unreached") // 도달 x
    } catch (e: ArithmeticException) {
        println("Caught ArithmeticException")
    }
}

// Throwing exception from launch
// Exception in thread "DefaultDispatcher-worker-1 @coroutine#2"
// Exception while trying to handle coroutine exception ... 중략
// Joined failed job
// Throwing exception from async
// Caught ArithmeticException
```

`job`의 경우에 `launch`로 루트 코루틴을 생성하였으므로 예외가 전파됨. 따라서 예외가 바로 발생함. 그러나 `async`로 생성한 루트 코루틴의 경우에는 `await`이 되는 시점에 예외가 발생함. 따라서 만약 `await`(`produce`의 경우에 `recieve`)을 사용하지 않는다면 예외가 처리되지 않음. 

<br/>

### ****CoroutineExceptionHandler 사용해 전파된 예외 처리하기****

root coroutine(부모가 없는 최상위 코루틴)의 Context의 요소인 `CoroutineExceptionHandler`는 루트 코루틴과 모든 자식 코루틴들에 대해 커스텀한 예외 처리가 필요한 경우 일반 catch 블록으로 사용될 수 있음. 

`CoroutineExceptionHandler`를 사용해서는 예외가 발생한 지점의 다음 블럭을 실행하지 못함. 왜냐하면 코루틴은 핸들러가 호출되었을 때 이미 그 예외에 대한 처리를 완료했기 때문. 따라서 `CoroutineExceptionHandler`의 목적은 오류를 로깅하거나, 에러 메시지를 보여주거나, 어플리케이션을 종료하는 등의 행위를 하기 위해 사용됨. 

`CoroutineExceptionHandler`는 uncaught exception에 대해서만 실행됨. 모든 자식 코루틴(다른 Job의 컨텍스트를 이용해 생성된 코루틴)들은 그들의 예외를 부모 코루틴에서 처리하도록 위임함. 부모는 자신의 부모에게 처리하도록 위임해서 결국 반복하다 보면 root coroutine까지 올라감. 따라서 자식의 context에 추가된 `CoroutineExceptionHandler`는 절대 사용되지 않음. 

`async` 빌더는 모든 예외를 잡아 `Deferred` 객체에 나타내므로 `CoroutineExceptionHandler`가 효과가 없음. 

Supervision Scope 상에서 실행되는 코루틴은 예외를 그들의 부모로 전파하지 않음. 

```kotlin
@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }
    val job = GlobalScope.launch(handler) {
        throw AssertionError()
    }
    val job2 = GlobalScope.launch(handler) {
        throw IndexOutOfBoundsException()
    }
    val deferred = GlobalScope.async(handler) {
        throw ArithmeticException()
    }
    joinAll(job, job2, deferred)
}

// CoroutineExceptionHandler got java.lang.AssertionError
// CoroutineExceptionHandler got java.lang.IndexOutOfBoundsException
```

launch로 만든 root coroutine에 대해서만 `CoroutineExceptionHandler`가 동작하는 것을 확인할 수 있음. 괜히 루트 코루틴과 자식 코루틴의 핸들러를 따로주는 실험도 해보았지만 이렇게 되면 루트 코루틴이 아니므로 제대로 동작하지 않음. 

```kotlin
@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }

    val handler2 = CoroutineExceptionHandler { _, exception ->
        println("handler 2 got $exception")
    }

    val job = GlobalScope.launch(handler) {
        launch(handler2) {
            throw IndexOutOfBoundsException()
        }
        throw AssertionError()
    }

    joinAll(job)
}

// CoroutineExceptionHandler got java.lang.AssertionError
```

`CoroutineExceptionHandler`를 사용할 때는 루트 컨트롤러에만 핸들러를 context에 추가해주고 자식들은 전파 개념을 사용해서 잡히게 하게끔 구현하는 것이 맞을 것 같음. 

<br/>

### ****Cancellation과 Exceptions****

코루틴은 취소를 위해 `CancellationException`을 사용함. 이 익셉션은 모든 핸들러에서 무시됨. 따라서 이들은 catch 블록으로부터 얻을 수 있는 추가적인 디버그 정보를 확인하기 위해서만 사용되어야 함. 

```kotlin
@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }

    val job = GlobalScope.launch(handler) {
        val t = launch(handler) {
            println("test")
            delay(1000L)
        }
        t.cancel()
        println("job end reached")
    }

    joinAll(job)
}

// test
// job end reached
```

위 코드에서 확인할 수 있듯 핸들러에서 `CancellationException`을 잡지 않음. 자식 코루틴이 `CancellationException`이 아닌 다른 예외를 던진다면 그 예외로 부모 코루틴까지 취소함. 이 동작은 구조화된 동시성을 위해 안정적인 코루틴 계층구조를 제공하는데 사용됨. 

- [x]  `CoroutineExceptionHandler`의 구현은 자식 Coroutine들을 위해 사용되지 않는다. ? → 자식 코루틴이 모두 종료된 다음에야 `CoroutineExceptionHandler`가 동작한다는 의미 같음.

```kotlin
fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }
    val job = GlobalScope.launch(handler) {
        launch { // the first child
            try {
                delay(Long.MAX_VALUE)
            } finally {
                withContext(NonCancellable) {
                    println("Children are cancelled, but exception is not handled until all children terminate")
                    delay(100)
                    println("The first child finished its non cancellable block")
                }
            }
        }
        launch { // the second child
            delay(10)
            println("Second child throws an exception")
            throw ArithmeticException()
        }
    }
    job.join()
}

// Second child throws an exception
// Children are cancelled, but exception is not handled until all children terminate
// The first child finished its non cancellable block
// CoroutineExceptionHandler got java.lang.ArithmeticException
```

따라서 이렇게 실행했을 때 `ArithmeticException`으로 인해 부모가 취소되면서 첫 번째 자식도 취소되기 때문에 main이 delay(MAX_VALUE)를 모두 기다려주지 않고, `finally`에서 `NonCancellable`로 만들어진 코루틴만 실행하고 종료되는 것임. 

만약 루트 코루틴에 핸들러를 추가하지 않고 자식에만 추가하면 자식에서 예외가 발생하더라도 핸들러가 동작하지 않음. 


<br/>


### ****Exceptions 합치기****

만약 루트 코루틴이 복수의 자식들이 있는데, 자식들 중 여러명이 예외를 발생시킨다면 일반적인 규칙은 첫 번째로 발생시킨 예외만 처리됨. 첫 예외 이후에 발생한 모든 예외는 suppressed로 붙여지게 됨. 

```kotlin
@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("handler got $exception with suppressed ${exception.suppressed.contentToString()}")
    }
    val job = GlobalScope.launch(handler) {
        launch {
            try {
                delay(Long.MAX_VALUE) 
            } finally {
                throw ArithmeticException() 
        }
        launch {
            delay(100)
            throw IOException() 
        }
        delay(Long.MAX_VALUE)
    }
    job.join()
}

// handler got java.io.IOException with suppressed [java.lang.ArithmeticException]
```

- [x]  취소 예외는 투명하고 기본적으로 감싸진다. ? → CancellationException은 예외 전파에 있어서 투명하고 기본적으로 언래핑됨. 아래 예제처럼 취소 이외의 예외가 발생하여 코루틴이 취소될 경우 CoroutineExceptionHandler가 예외 처리를 할 때에는 캔슬 익셉션은 바로 언래핑되고 최초 발생 예외가 핸들러의 익셉션 파라미터로 전달된다는 의미.

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    println("CoroutineExceptionHandler got $exception")
}
val job = GlobalScope.launch(handler) {
    val inner = launch { // all this stack of coroutines will get cancelled
        launch {
            launch {
                throw IOException() // the original exception
            }
        }
    }
    try {
        inner.join()
    } catch (e: CancellationException) {
        println("Rethrowing CancellationException with original cause")
        throw e // cancellation exception is rethrown, yet the original IOException gets to the handler  
    }
}
job.join()

// Rethrowing CancellationException with original cause
// CoroutineExceptionHandler got java.io.IOException
```

<br/>

### ****Supervision****

취소는 코루틴의 전체 계층을 통해 전파되는 양방향 전파임. 즉 자식 코루틴에서 취소 발생 시 부모 코루틴으로 전파되고 부모 코루틴이 다른 자식 코루틴이 있다면 자식 코루틴들도 취소되는 구조임. 

그러나 단방향으로 전파되는 취소가 필요할 수 있음.

- e.g. 여러 자식 Job을 생성하고 이들 중 실패 된 것만 재시작하는 프로세스

<br/>

### ****Supervision job****

취소가 아래 방향으로만 전파되는 것을 제외하면 일반적인 Job과 비슷함. 

```kotlin
fun main() = runBlocking {
    val supervisor = SupervisorJob()
    with(CoroutineScope(coroutineContext + supervisor)) {
        // launch the first child -- its exception is ignored for this example (don't do this in practice!)
        val firstChild = launch(CoroutineExceptionHandler { _, _ ->  }) {
            println("The first child is failing")
            throw AssertionError("The first child is cancelled")
        }
        // launch the second child
        val secondChild = launch {
            firstChild.join()
            // Cancellation of the first child is not propagated to the second child
            println("The first child is cancelled: ${firstChild.isCancelled}, but the second one is still active")
            try {
                delay(Long.MAX_VALUE)
            } finally {
                // But cancellation of the supervisor is propagated
                println("The second child is cancelled because the supervisor was cancelled")
            }
        }
        // wait until the first child fails & completes
        firstChild.join()
        println("Cancelling the supervisor")
        supervisor.cancel()
        secondChild.join()
    }
}

// The first child is failing
// The first child is cancelled: true, but the second one is still active
// Cancelling the supervisor
// The second child is cancelled because the supervisor was cancelled
```

예시와 같이 자식에서 예외가 발생해도 부모와 그 부모의 또다른 자식을 취소시키는게 아니고 자식과 자식의 자식들만 취소됨. 그러나 부모가 취소되면 모두 취소됨. 

<br/>

### ****Supervision Scope****

취소를 한 방향으로만 전파하여 자신 실패 시 자신과 자신의 자식을 취소하게끔 할 수 있음. coroutineScope 대신 supervisorScope를 사용하면 됨. supervisorScope도 coroutineScope처럼 자식들이 완료되는 것을 모두 기다린 다음에야 자신도 완료됨. 

```kotlin
fun main() = runBlocking {
    try {
        supervisorScope {
            val child = launch {
                try {
                    println("The child is sleeping")
                    delay(Long.MAX_VALUE)
                } finally {
                    println("The child is cancelled")
                }
            }
            yield()
            println("Throwing an exception from the scope")
            throw AssertionError()
        }
    } catch(e: AssertionError) {
        println("Caught an assertion error")
    }
}

// The child is sleeping
// Throwing an exception from the scope
// The child is cancelled
// Caught an assertion error
```

- [x]  근데 양방향 전파 안 한다면서 아래 코드에서는 왜 child2는 실행이 되지 않을까요? →이렇게 하면 자식 중 하나가 예외를 발생시켜도 다른 자식이 취소되지 않음. 그러면서 구조화된 동시성을 지킴.

```kotlin
fun main() = runBlocking {
    try {
        supervisorScope {
            val child = launch {
                try {
                    println("The child is sleeping")
                    throw AssertionError()
                } finally {
                    println("The child is cancelled")
                }
            }

            val child2 = launch {
                delay(1500L)
                println("child2 end")
            }
        }
    } catch(e: AssertionError) {
        println("Caught an assertion error")
    }
}

// The child is sleeping
// The child is cancelled
// Exception in thread "main @coroutine#2" java.lang.RuntimeException ... 중략
// child2 end
```

<br/>

### ****Supervise가 사용된 Coroutine에서의 예외****

SupervisorJob의 모든 자식은 자신의 예외를 예외 처리 매커니즘에 따라 직접 처리해야 함. 왜냐 자식의 실패가 부모에 전파되지 않기 때문. 따라서 그들의 스코프 내부에 핸들러를 설치해주어야 함. 따라서 위와 비슷한 코드인데 핸들러를 설치해주면 다음과 같이 로깅이 가능함. 

```kotlin
fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }

    supervisorScope {
        val child = launch(handler) {
            try {
                println("The child is sleeping")
                throw AssertionError()
            } finally {
                println("The child is cancelled")
            }
        }
    }

    val child2 = launch {
        delay(1500L)
        println("child2 end")
    }

}

// The child is sleeping
// The child is cancelled
// CoroutineExceptionHandler got java.lang.AssertionError
// child2 end
```

<br/>

### Ref.

- [https://myungpyo.medium.com/코루틴-공식-가이드-자세히-읽기-part-6-dd7796150ff3](https://myungpyo.medium.com/%EC%BD%94%EB%A3%A8%ED%8B%B4-%EA%B3%B5%EC%8B%9D-%EA%B0%80%EC%9D%B4%EB%93%9C-%EC%9E%90%EC%84%B8%ED%9E%88-%EC%9D%BD%EA%B8%B0-part-6-dd7796150ff3)
- https://simplecode.kr/50
- https://seyoungcho2.github.io/CoroutinesKoreanTranslation/coroutine.html


<br/>

### Q

- [x]  uncaughtExceptionPreHandler: 코틀린 내부 동작임. 사용되는 곳은 다음 링크를 확인, https://github.com/search?q=repo%3AKotlin%2Fkotlinx.coroutines%20uncaughtExceptionPreHandler&type=code
- [x]  async에 join 걸면 뭐가 어떻게 되는데?: 쓰지말자 어싱크 쓸건데 왜 조인을 써 그냥 어웨잇 써

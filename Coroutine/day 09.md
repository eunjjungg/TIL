## ****Coroutine 공유 상태와 동시성****

### ****Coroutine을 여러개 실행했을 때의 문제점****

```kotlin
fun main() = runBlocking {
    var counter = 0

    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}

suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        coroutineScope { // scope for coroutines
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")
}

// Completed 100000 actions in 17 ms
// Counter = 69985
```

여러 코루틴이 하나의 공유 자원에 대한 작업을 수행할 때 동기화 없이 사용했기 때문에 원하는 방식으로 동작하지 않음. 

<br/>

### ****Volatile은 동시성 문제를 해결하지 못한다****

변수를 volatile로 만드는게 동시성 문제를 해결할 수 있다는 인식이 있음. 

코틀린의 volatile은 `@Volatile` 어노테이션을 변수 선언 시 지정해서 사용할 수 있음. 변수 선언 시 volatile을 지정하면 메인 메모리에만 적재하게 됨. volatile을 사용하지 않는 변수는 일반적으로 성능 향상을 위해 메인 메모리로부터 읽어온 값을 CPU 캐시에 저장함. 그러나 이 값은 멀티스레드 애플리케이션에서는 각 스레드 값에서 캐싱한 값이 상이함. → 따라서 동기화를 보장한다고는 할 수 없음. 

<br/>

### ****Thread-safe한 데이터 구조****

스레드, 코루틴 모두에 작동하는 일반적인 해결 방법은 공유 상태에 수행되어야 하는 모든 동작에 대한 필수적인 동기화를 제공하는 Thread Safe한 데이터 구조를 사용하는 것임. 

`AtomicInteger` 클래스의 `incrementAndGet`이라 불리는 원자적인 동작을 제공하는 클래스를 사용하는 것도 방법임. 그러나 간단한 것에만 사용될 수 있음.

## ****Coroutine 공유 상태와 동시성****

### ****세밀하게 Thread 제한하기****

스레드 제한은 공유 자원에 단일 스레드로만 제한할 때 접근 방식임. 세밀하게 스레드를 제한한다는 말이 단일 스레드로 공유 자원을 사용할 때를 말하는 것 같음. 

```kotlin
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

val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            // confine each increment to a single-threaded context
            withContext(counterContext) {
                counter++
            }
        }
    }
    println("Counter = $counter")
}

// Completed 100000 actions in 767 ms
// Counter = 100000
```

위의 코드를 보면 counter++를 하는 action을 Dispatchers.Default 컨텍스트에서 withContext(counterContext)로 단일 스레드 컨텍스트로 전환하고 있음. 그러나 느리게 작동한다는 단점이 있음.

> 이는 일반적으로 모든 UI 상태가 단일 이벤트 디스패처/어플리케이션 스레드로 제한된 UI가 있는 어플리케이션에 사용된다.
> 
- [x] Q. 이게 무슨 뜻인지 이해가 잘 안 갔는데 이해한 바로는.. → 어쨌든 위의 세밀하게 스레드를 제한하는 방식을 UI가 있는 어플리케이션에서 사용한다는 뜻인데, 안드로이드의 경우에는 메인 스레드가 된다. 따라서 Dispatchers.Main을 사용하게 되면 메인스레드를 사용하고, 메인 스레드 하나만 공유 자원에 대해 접근을 하게 됨. 따라서 데드락이 발생하지 않음. 

<br/>

### ****굵게 Thread 제한하기****

실제 스레드 제한은 더 큰 코드 블록 단위로 이루어짐. 

```kotlin
fun main() = runBlocking {
    // confine everything to a single-threaded context
    withContext(counterContext) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}

// Completed 100000 actions in 16 ms
// Counter = 100000
```

위의 코드와 massiveRun 함수는 똑같고 Dispatcher.Default에서 실행하지 않는다는 점만 다름. withContext로 Dispatcher.Default로의 스레드 교체가 이루어지지 않음. 

- [x] Q. 근데 이게 왜 세밀하거나 굵은 스레드의 차이인지 잘 모르겠음. → 세밀하게 스레드 제한을 했을 때 Dispatchers.Default → (massive run 1번째)CounterContext → Dispatchers.Default → (massive run 2번째)CounterContext → Dispatchers.Default → (massive run 3번째)CounterContext → Dispatchers.Default … → (massive run 10_000번째)CounterContext → Dispatchers.Default 로 교환이 일어난다. 근데 이때 Dispatchers.Default가 어쨌든 한 번 백그라운드 풀 중에서 하나의 스레드로 정해지면 그 매시브 런 내부를 돌고 나와서도 스레드로 복귀하는 줄 알았는데 그게 아니라 그 풀에서 또 다른 스레드를 찾아서 걸로 복귀시킴. 그러므로 시간이 당연히 오래 걸릴 수밖에 없다. (아래의 굵게 스레드를 제한하는 방식보다) 아래의 굵게 스레드 제한하는 방식은 복귀를 메인 스레드로 하기 때문에 시간이 훨씬 적게 소모된다고 이해할 수 있을 것 같음.

<br/>

### Mutex

Mutex 솔루션은 모든 공유 상태에 대한 변경을 절대로 동시에 실행되지 않는 critical section으로 만드는 것. 코루틴에서는 Mutex를 사용하는데 이는 critical section을 구분하기 위한 `lock`, `unlock` 함수를 가짐. 여기서 lock은 일시 중단 함수임. 즉 스레드를 블록하지 않음. 

`mutex.lock(); try { ... } finally { mutex.unlock() }`를 나타내는 `withLock` 확장함수도 존재함. 

<br/>

### ****Actors****

액터는 코루틴 + 이 코루틴 내부로 캡슐화된 상태 값 + 다른 코루틴과 통신할 수 있는 채널의 조합으로 이루어진 엔티티임. 간단한 액터는 함수로 작성될 수 있지만 복잡한 상태를 가진 액터는 클래스로 작성되어야 함. 

액터를 사용하기 위해서는 우선 액터가 처리할 메세지들의 클래스를 정의하는 것임. sealed class가 이러한 목적에 적합함. 그리고 응답을 보내야 함. 이러한 목적으로 사용되는 CompletableDeferred communication primitive 타입을 사용함. 

`actor`는 다음과 같은 특징을 가짐.

1. **메시지 패싱**: 액터는 메시지를 통해 상호작용함. 액터에게 작업을 지시하려면 메시지를 보내야 함.
2. **상태 캡슐화**: 액터 내부의 상태는 액터 외부에서 직접 접근할 수 없음. 이는 동시성 문제를 방지하는 데 중요함.
3. **순차적 처리**: 액터는 자신의 메일박스에 있는 메시지를 순차적으로 하나씩 처리함.

액터는 어떤 context에서 자신을 수행하는지와 상관 없이 동기화 문제에서 자유로울 수 있음. 

```kotlin
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

// Message types for counterActor
sealed class CounterMsg
object IncCounter : CounterMsg() // one-way message to increment counter
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg() // a request with reply

// This function launches a new counter actor
fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0 // actor state
    for (msg in channel) { // iterate over incoming messages
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}

fun main() = runBlocking<Unit> {
    val counter = counterActor() // create the actor
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.send(IncCounter)
        }
    }
    // send a message to get a counter value from an actor
    val response = CompletableDeferred<Int>()
    counter.send(GetCounter(response))
    println("Counter = ${response.await()}")
    counter.close() // shutdown the actor
}
```

위의 코드를 분석해보면 다음과 같음.

1. 10_000번 increase counter를 보내고 
2. 리스폰스를 CompletableDeferred의 Primitive type인 Int로 선언
3. massive run 액션이 끝난 뒤 counter에 GetCounter class를 보냄. 
4. response에 공유 자원인 counter 값이 들어감. 값을 받아오려면 await()을 사용해주어야 하는데 그 이유는 Deferred를 상속한 CompletableDeferred이기 때문~
5. 액터를 close함. 

<br/>

**📚 추가적으로 공부한 Actor**

액터는 coroutine에서 스레드간 동기화를 지원하기 위한 도구임. 

[액터 = 상태 액세스를 단일 스레드로 한정 + 다른 스레드는 채널을 통해서 상태 수정을 요청]이라고 생각하면 액터는 공유 자원에 대한 동기화를 위한 도구라고 이해할 수 있음. 

액터를 생성하면 Received Channel을 가짐. 요건 채널에서 나왔던 개념. 클라이언트 입장에서는 액터는 채널과 같기에 send()로 메세지를 보냄. 

```kotlin
var counter = 0

fun main() = runBlocking {
    val actorCounter = actor<Void?> {
        for (msg in channel) {
            counter++
        }
    }

    val workA = async {

        repeat(2000) {
            actorCounter.send(null)
        }
    }

    val workB = async {

        repeat(300) {
            actorCounter.send(null)
        }
    }

    workA.await()
    workB.await()

    println(counter)
}

// 2300
```

위 액터는 값을 증가시키는 기능만 하므로 굉장히 단순한데, 액터가 수신 받은 메세지를 확장하면 기능을 확장시킬 수 있음.

```kotlin
var counter = 0

fun main() = runBlocking {
    val actorCounter = actor<Action> {
        for (msg in channel) {
            when(msg) {
                Action.Minus -> counter--
                Action.Plus -> counter++
            }
        }
    }

    val workA = async {

        repeat(2000) {
            actorCounter.send(Action.Plus)
        }
    }

    val workB = async {

        repeat(300) {
            actorCounter.send(Action.Minus)
        }
    }

    workA.await()
    workB.await()

    println(counter)
}

sealed class Action {
    object Plus: Action()
    object Minus: Action()
}

// 1700
```

<br/>

### Ref.

- https://seyoungcho2.github.io/CoroutinesKoreanTranslation/coroutine-1.html
- [https://myungpyo.medium.com/코루틴-공식-가이드-자세히-읽기-part-8-1b434772a100](https://myungpyo.medium.com/%EC%BD%94%EB%A3%A8%ED%8B%B4-%EA%B3%B5%EC%8B%9D-%EA%B0%80%EC%9D%B4%EB%93%9C-%EC%9E%90%EC%84%B8%ED%9E%88-%EC%9D%BD%EA%B8%B0-part-8-1b434772a100)
- [https://origogi.github.io/coroutine/액터/](https://origogi.github.io/coroutine/%EC%95%A1%ED%84%B0/)
- https://yk-coding-letter.tistory.com/16
- https://onlyfor-me-blog.tistory.com/724

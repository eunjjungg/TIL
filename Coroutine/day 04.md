## ****Coroutine Context와 Dispatcher****

### ****Dispatchers와 Threads****

코루틴은 언제나 CoroutineContext 타입 값으로 표현되는 일부 Context에서 실행됨. Coroutine Context에는 해당 코루틴의 실행에 사용되는 단일 스레드나 복수의 스레드를 결정하는 Coroutine Dispatcher가 포함됨. Coroutine Dispatcher는 코루틴의 실행에 사용될 스레드를 특정 스레드로 제한하거나 스레드풀에 분배하거나 제한 없이 실행되도록 할 수 있다. 

launch나 async 같은 모든 코루틴 빌더들은 새로운 코루틴 생성을 위해 Dispatcher나 다른 Context 요소들을 명시적으로 지정하는데 사용할 수 있는 `CoroutineContext` 파라미터를 선택적으로 받을 수 있음.

📚 정리해보자

1. 코루틴은 항상 `CoroutineContext` 값으로 표현되는 일부 context에서 실행됨.
2. 이 `CoroutineContext`에는 해당 코루틴의 실행에 사용될 스레드를 결정하는 Coroutine Dispatcher가 포함됨.
3. `launch`나 `async`와 같은 코루틴 빌더들은 새로운 코루틴을 위해 `Dispatcher`를 직접적으로 받거나 그를 포함하고 있는 `CoroutineContext`를 파라미터로 받음.

```kotlin
launch { // context of the parent, main runBlocking coroutine
    println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
}
launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
    println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
}
launch(Dispatchers.Default) { // will get dispatched to DefaultDispatcher 
    println("Default               : I'm working in thread ${Thread.currentThread().name}")
}
launch(newSingleThreadContext("MyOwnThread")) { // will get its own new thread
    println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
}
```

위 코드를 실행해보면 아래와 같이 나옴.

```kotlin
Unconfined            : I'm working in thread main
Default               : I'm working in thread DefaultDispatcher-worker-1
main runBlocking      : I'm working in thread main
newSingleThreadContext: I'm working in thread MyOwnThread
```

`launch`가 파라미터 없이 실행된다면 실행되는 `CoroutineScope`로부터 Context를 상속 받음. Context 내부에는 Dispatcher도 있으므로 같이 상속 받음. 따라서 `main` 함수의 `runBlocking` 코루틴으로부터 context를 상속 받아 메인 스레드에서 실행됨. 여기서의 코루틴 스코프는 interface coroutine scope임. 

https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/

`Dispatchers.Unconfined` 또한 메인 스레드에서 실행되는 특별한 Dispatcher이지만 실제로는 다른 동작방식임.

`Dispatchers.Default`는 Scope내에서 다른 디스패처 사용이 명시적으로 지정되지 않았을 때 사용됨. 스레드들이 공유하는 background pool을 사용함. `launch(Dispatchers.Default){}`와 `GlobalScope.launch{}`는 동일한 디스패처를 사용.

`newSingleThreadContext`는 코루틴이 실행되기 위한 새로운 스레드를 생성함. 이러한 전용 스레드는 매우 비싼 스레드임. 따라서 더 이상 필요하지 않을 때 close 함수를 사용해 해제되어야 하며 최상위 레벨의 변수에 저장하여 애플리케이션이 실행되는 동안 재사용 될 수 있도록 해야 함.

<br/>

### ****Unconfined vs confined dispatcher****

`Dispatchers.Unconfined`는 처음 일시 중단 되기 전까지만 호출 스레드에서 코루틴을 시작함. 일시 중단 이후에는 실행된 일시 중단 함수에 의해 완전히 결정된 스레드 상에서 코루틴을 재개함. Unconfined Dispatcher는 CPU 시간을 소모하지 않거나 공유되는 데이터(UI)를 업데이트하지 않는 경우처럼 특정 스레드에 국한된 작업이 아닌 경우 적절함. 

반면 Dispatcher는 기본적으로 바깥 coroutineScope에 의해 상속됨. 특히 runBlocking 코루틴에 대한 기본 디스패처는 호출 스레드로 제한되므로, 이를 상속하면 예측 가능한 FIFO 스케줄링으로 이 스레드의 실행을 제한할 수 있음. 

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    launch(Dispatchers.Unconfined) {
        // not confined -- will work with main thread
        println("Unconfined      : I'm working in thread ${Thread.currentThread().name}")
        delay(500)
        println("Unconfined      : After delay in thread ${Thread.currentThread().name}")
    }
    launch {
        // context of the parent, main runBlocking coroutine
        println("main runBlocking: I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("main runBlocking: After delay in thread ${Thread.currentThread().name}")
    }
}

// Unconfined : I’m working in thread main
// main runBlocking: I’m working in thread main
// Unconfined : After delay in thread kotlinx.coroutines.DefaultExecutor
// main runBlocking: After delay in thread main
```

다음과 같이 코루틴을 `runBlocking` 안에서 시작하는데 이를 상속한 코루틴은 메인 스레드에서 작업을 계속 해감. 반면 `Unconfined` 코루틴은 delay 함수(일시 중단 함수)가 사용하는 default executor에서 일시중단점 이후에 스레드가 변경됨. 

참고하면 좋은 문서: https://stackoverflow.com/questions/61844659/how-delay-function-is-working-in-kotlin-without-blocking-the-current-thread

<br/>

## ****Coroutines와 Threads 디버깅 하기****

### ****IDEA를 사용해 디버깅 하기****

코루틴 디버거를 통해 얻어낼 수 있는 것

- 각 코루틴에 대한 상태를 확인할 수 있음.
- 실행 중이거나 일시 중단된 코루틴에 대한 지역변수나 캡처된 변수의 값들을 확인할 수 있음.
- 코루틴 생성 스택과 코루틴 내부의 콜스택에 대해 확인할 수 있음. 스택은 일반적인 디버깅에서 손실되는 프레임을 포함한 전체 프레임들에 대한 변수의 값들을 포함함.
- 각 코루틴의 상태와 그 스택들을 포함하는 전체 리포트를 가져올 수 있음. 코루틴 탭을 우클릭 후 get coroutine dump를 실행.

<br/>

### ****로깅을 통해 디버깅 하기****

먼저 JVM option에 `-Dkotlinx.coroutines.debug`를 추가해주어야 함. 추가하는 방법은 main 함수의 ▶️ 버튼을 우클릭하고 modify run configuration을 선택한 후 VM options에 넣어주면 됨. 

Coroutine Debugger 없이 스레드를 사용하는 어플리케이션을 디버깅하는 방법은 각 로그 구문에 스레드 이름을 넣어, 로그 파일에 스레드 이름을 인쇄하는 것임. 

```kotlin
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")
```

<br/>

### ****Thread 전환 하기****

```kotlin
newSingleThreadContext("Ctx1").use { ctx1 ->
    newSingleThreadContext("Ctx2").use { ctx2 ->
        runBlocking(ctx1) {
            log("Started in ctx1")
            withContext(ctx2) {
                log("Working in ctx2")
            }
            log("Back to ctx1")
        }
    }
}

// [Ctx1 @coroutine#1] Started in ctx1
// [Ctx2 @coroutine#1] Working in ctx2
// [Ctx1 @coroutine#1] Back to ctx1
```

- runBlocking을 명시적으로 구체화된 context와 함께 사용함.
- withContext를 같은 코루틴에 있으면서 context를 변경하는데 사용함.
- use로 resource가 많이 드는 코루틴인 `newSingleThreadContext`의 close 처리를 해주고 있음.

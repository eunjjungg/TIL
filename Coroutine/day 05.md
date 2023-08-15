## ****Coroutine Context와 Dispatcher****

### ****Context 내부의 Job****

coroutine의 Job은 Context의 구성요소이고, coroutineContext[Job] 표현식으로 가져올 수 있음. 

```kotlin
fun main() = runBlocking<Unit> {
    println("My job is ${coroutineContext[Job]}")
}

// My job is BlockingCoroutine{Active}@5c8da962
```

<br/>

### ****Coroutine의 자식들****

코루틴이 다른 코루틴의 CoroutineScope 내에서 실행되면 Context를 CoroutineScope.coroutineContext를 통해 상속 받고 새로운 코루틴의 Job은 부모 코루틴의 Job의 자식이 됨. 부모 코루틴이 취소되면, 자식 코루틴들 또한 재귀적으로 취소됨. 

📚 정리

- 코루틴이 다른 코루틴 내에서 실행되면 부모의 context를 상속 받음.
- 새로운 코루틴의 job은 부모 코루틴의 job의 자식이 됨.
- 부모 코루틴이 취소되면 자식도 취소됨.

그러나 이렇게 만들어진 부모-자식 관계는 아래의 경우에 바뀔 수 있음. 이 두 경우 모두 실행된 코루틴은 실행된 scope에 묶이지 않고 독립적으로 동작함. → 부모 코루틴이 취소되어도 자식은 취소되지 않는다는 의미. 또한 부모가 독립적으로 동작하는 코루틴의 종료를 기다려주지도 않음. 

- 코루틴을 실행할 때 두 개의 다른 스코프가 명시적으로 설정하는 경우 부모 스코프로부터 Job을 상속 받지 않음.
    - e.g. GloabalScope.launch
- 새로운 코루틴을 위해 다른 Job이 Context로 전달되는 경우 Job을 재정의함.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    val request = launch {
        // it spawns two other jobs
        launch(Job()) {
            println("job1: 독립적으로 동작!")
            delay(1000)
            println("job1: 부모 취소되어도 동작!")
        }

        // and the other inherits the parent context
        launch {
            delay(100)
            println("job2: 부모한테 종속~")
            delay(1000)
            println("job2: 부모 취소되면 실행 안 되므로 이 문구는 출력 X")
        }
    }
    delay(500)
    request.cancel()
    println("main: Who has survived request cancellation?")
    delay(1000)
}

// job1: 독립적으로 동작!
// job2: 부모한테 종속~
// main: Who has survived request cancellation?
// job1: 부모 취소되어도 동작!
```

위 코드에서 `launch(Job()) { }` 대신 `GlobalScope.launch { }`로 실행해도 결과는 같게 나옴. `CoroutineScope(Dispatchers.IO).launch { }`로 실행한 결과도 같게 나옴.

그렇다면 왜 `CoroutineScope(Dispatchers.IO).launch { }` 이런 식으로 다른 CoroutineContext를 자식에 재정의되면 부모 코루틴과는 다른 Job을 갖게 되는 걸까? → 코루틴스코프를 통해 CoroutineContext가 재정의 되면 CoroutineContext는 부모로부터 상속 받지 않게 되어 Job == null이 됨. 그러면 내부적으로 context에 Job을 추가하여 코루틴 스코프가 만들어지게 됨. + CoroutineContext는 구성 요소를 Key-Value로 관리하는데 Key인 Job에 대응하는 Value는 하나뿐임. 그런데 새로운 Job이 들어오게 되면 이에 대응하는 새로운 Job의 구성요소를 담는 Value가 생성됨. 

<br/>

### ****부모의 책임****

- 부모 코루틴은 언제나 자식이 완료될 때까지 기다림.
- 부모는 모든 자식들의 실행을 명시적으로 추적하지 못함.
- 자식들이 종료될 때까지 기다리기 위해 `Job.join`을 해줄 필요가 없음.

따라서 부모에만 join을 걸어주면 자식이 종료되기 전까지는 종료되지 않음. 또한 부모 → 자식 → 조손 식으로 코루틴을 만들어주고 부모에만 join을 걸어도 조손이 종료될 때까지 부모는 종료되지 않음. 

단 위에서 작성한 부분과 같이 부모 코루틴과는 다른 Job을 갖도록 자식 코루틴을 만들면 부모는 부모의 책임에서 벗어나게 됨. 

<br/>

### ****디버깅을 위해 Coroutines에 이름 짓기****

자동 설정된 ID도 있지만 코루틴이 특정 용도로만 사용된다면 이름을 설정하는 것도 방법임. Context의 요소인 coroutineName은 스레드의 이름과 같은 용도로 사용됨. 이는 디버깅 모드가 켜져 있을 때 Coroutine을 실행하는 스레드 이름에 포함됨. 

```kotlin
fun main() = runBlocking<Unit> {
    log("Started main coroutine")
    // run two background value computations
    val v1 = async(CoroutineName("v1coroutine")) {
        delay(500)
        log("Computing v1")
        252
    }
    val v2 = async(CoroutineName("v2coroutine")) {
        delay(1000)
        log("Computing v2")
        6
    }
    log("The answer for v1 / v2 = ${v1.await() / v2.await()}")
}

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

// [main @coroutine#1] Started main coroutine
// [main @v1coroutine#2] Computing v1
// [main @v2coroutine#3] Computing v2
// [main @coroutine#1] The answer for v1 / v2 = 42
```

### ****Context 요소들 결합하기****

Coroutine Context에 복수의 요소들을 정의해야 할 수 있음. `+` 연산자를 사용할 수 있음. 

```kotlin
launch(Dispatchers.Default + CoroutineName("test")) {
    println("I'm working in thread ${Thread.currentThread().name}")
}

// I'm working in thread DefaultDispatcher-worker-1 @test#2
```

이 코드는 명시적으로 디스패처를 디폴트로 정하고 있고 코루틴의 이름 또한 정해주고 있음. `+` 연산자로 Coroutine Context들을 결합할 수 있는 이유는 CoroutineContext에 operator fun plus를 확인해보면 됨. 

![image](https://github.com/eunjjungg/TIL/assets/100047095/a792d201-84b5-4110-bd1d-54697e3bfb12)

<br/>

### ****Coroutine Scope****

Activity의 생명주기에 묶은 CoroutineScope의 인스턴스를 생성해 코루틴의 생명주기를 관리함. CoroutineScope instance는 CoroutineScope(), MainScope() 같은 팩토리 함수로 생성될 수 있음. CoroutineScope()로 생성된 인스턴스는 일반적인 목적으로 사용되고 MainScope()으로 생성된 인스턴스는 Dispatchers.Main을 기본 디스패처로 사용함. 

<br/>

### ****Thread-local 데이터****

ThreadLocal을 위한 `asContextElement` 확장 함수는 쓰레드 로컬 데이터를 코루틴으로 전달하거나 코루틴들간에 전달하는 기능임. 이는 부가적인 컨텍스트 요소를 생성함. 주어진 스레드 로컬의 값을 저장했다가 코루틴이 속한 컨텍스트를 변경할 때마다 복원함. 뒤는..잘 모르겠음..~ 쓸 일이 있을 때 다시 공부해야될듯 ㅎㅎ; 

<br/>

### Ref.

- https://seyoungcho2.github.io/CoroutinesKoreanTranslation/coroutine-context-dispatcher.html
- https://simplecode.kr/34
- https://simplecode.kr/35
- https://simplecode.kr/36
- [https://myungpyo.medium.com/코루틴-공식-가이드-자세히-읽기-part-5-62e886f7862d](https://myungpyo.medium.com/%EC%BD%94%EB%A3%A8%ED%8B%B4-%EA%B3%B5%EC%8B%9D-%EA%B0%80%EC%9D%B4%EB%93%9C-%EC%9E%90%EC%84%B8%ED%9E%88-%EC%9D%BD%EA%B8%B0-part-5-62e886f7862d)

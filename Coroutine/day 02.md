## 취소와 타임아웃

### ****Coroutine 실행 취소하기****

백그라운드에서 실행되는 코루틴에 대한 세밀한 제어가 필요할 수 있음. e.g. 유저가 코루틴을 실행시킨 페이지를 닫아 결과가 더 이상 필요하지 않은 경우. `launch` 함수는 실행 중인 코루틴을 취소하는 데 사용할 수 있는 `Job` 객체를 반환함. `Job.cancel()`을 호출하면 Job이 취소됨. Job의 확장 함수로 `cancel`과 `join`을 결합한 `cancelAndJoin` 함수도 있음.

<br/>

### ****Coroutines 취소는 협력적이다****

`kotlinx.coroutines` 패키지의 모든 일시 중단 함수들은 취소 가능함. → 코루틴은 실행 중 취소 가능 지점마다 취소 요청이 있다면 즉시 취소되어야 함. 만약 취소되었다면 `CancellationException`을 발생시키며 종료함. 

```kotlin
// 1
fun main(args: Array<String>) = runBlocking {
    val job = launch(Dispatchers.Default) {
        for (i in 1..10) {
            println("I'm sleeping $i ...")
            Thread.sleep(500L)
						// delay(500L)
        }
    }

    delay(1300L)
    println("main : I'm tired of waiting!")
    job.cancelAndJoin()
    println("main : Now I can quit.")
}
```

따라서 위 코드에서는 job.cancelAndJoin()을 했음에도 불구하고 작업을 멈추지 않음. why? 따로 취소 요청을 확인하지 않기 때문. 이 코루틴을 취소 요청에 응답할 수 있는 코드로 변경하는 건 아래에서 이어서 작성. 

<br/>

### ****Coroutine의 Computation 코드를 취소 가능하게 만들기****

이 코루틴을 취소 요청에 응답할 수 있는 코드로 변경하려면 다음과 같은 선택지를 사용해야 함. 

- 다른 Continuation에 실행 시간을 양보하는 yield() 함수를 호출 (주기적으로 일시중단 함수를 실행시켜서 취소되었는지 확인하는 목적)

```kotlin
// 2
for (i in 1..10) {
		yield()
		println("I'm sleeping $i ...")
		Thread.sleep(500L)
}
```

- CoroutineScope에 정의 된 isActive 속성을 참조하여 코루틴이 비활성 상태인 경우 작업을 중단하도록 함.

```kotlin
// 3
for (i in 1..10) {
		if (!isActive) {
				break
		}
		println("I'm sleeping $i ...")
		Thread.sleep(500L)
}
...
```

이런 식으로 바꿔주면 `CancellationException`을 발생시키며 중단할 수 있음. 그리고 이 예외는 `try-finally` 구문 혹은 `use()`로 처리할 수 있음.

<br/>

### ****finally 사용해 리소스 닫기****

```kotlin
// 
fun main(args: Array<String>) = runBlocking {
    val job = launch(Dispatchers.Default) {
        try {
            repeat(100) {
                println("I'm sleeping $it ...")
                delay(500L)
            }
        } catch (e: Exception) {
            if (e is CancellationException) {
                println("main : catch cancellationException arrived!")
            }
        }
        finally {
            println("main : finally arrived!")
        }
    }

    delay(1300L)
    println("main : I'm tired of waiting!")
    job.cancelAndJoin()
    println("main : Now I can quit.")
}

// I'm sleeping 0 ...
// I'm sleeping 1 ...
// I'm sleeping 2 ...
// main : I'm tired of waiting!
// main : catch cancellationException arrived!
// main : finally arrived!
// main : Now I can quit.
```

try-finally를 사용해 예외처리를 하는 모습임. 그러나 이 try-finally를 repeat 안에서 해주면 어떻게 될까?

```kotlin
fun main(args: Array<String>) = runBlocking {
    val job = launch(Dispatchers.Default) {
        repeat(10) {
            try {
                println("I'm sleeping $it ...")
                delay(500L)
            } catch (e: CancellationException) {
                println("main : CancellationException caught!")
            }
        }
    }

    delay(1300L)
    println("main : I'm tired of waiting!")
    job.cancelAndJoin()
    println("main : Now I can quit.")
}

// I'm sleeping 0 ...
// I'm sleeping 1 ...
// I'm sleeping 2 ...
// main : I'm tired of waiting!
// main : CancellationException caught!
// I'm sleeping 3 ...
// main : CancellationException caught!
// I'm sleeping 4 ...
// main : CancellationException caught!
// 9까지 반복
```

catch에서 CancellationException을 전파시키지 않고 catch로 처리하기 때문에 코루틴의 끝까지 반복됨. 그러나 catch에서 throw를 해주면 3부터는 출력되지 않게 됨. 

그렇다면 use()는 뭘까? IO Stream, Socket, IO Reader 등 close 처리를 해야하는 것들이 있는데 요런 것을 close()해주는걸 보통 아래와 같이 해줌

```kotlin
try {
		socket = Socket(...)
} catch (e: Exception) {
		// error handling 
} finally {
		socket.close()
} 
```

나도 처음에 학교에서 자바 배울 때 이렇게 해줬던 것 같음. 하지만 finally 내부에서 실행하는 close 역시 Exception이 발생할 수 있기 때문에 안전하게 처리해주어야 함. 그러면 아래와 같이 사용하게 됨. → 코드 매우 복잡 + 널러블 처리까지 필요 

```kotlin
try { 
		socket = Socket(...)
} catch (e: Exception) {
		// error handling 
} finally {
		try { socket?.close() } catch (e: Excpetion) { /* handling */ }
} 
```

하지만 use Extension을 사용하면 간편하게 처리할 수 있다. use는 try-catch-finally를 모두 포함하고 있음. 내부 코드는 다음과 같음.

![image](https://github.com/eunjjungg/TIL/assets/100047095/59882e9f-b1c5-4a8e-9b92-823aaf4f04ab)

catch에서 throw를 해주는 부분이 있기 때문에 use를 사용할 때 try-catch 안에서 사용하는 것이 가장 이상적임. 그래서 아래와 같이 사용하게 됨. 

```kotlin
try {
		Socket(...).use {
				// ...
		}
} catch (e: Exception) {
		// error handling 
} 
```

그래서 다시 본론에서의 coroutine에서의 예외 처리를 use로도 할 수 있다는 건데 예시는 아래와 같음. 

```kotlin
fun main(args: Array<String>) = runBlocking {
    val job = launch(Dispatchers.Default) {
        try {
            CloseableSleep().use {
                it.sleep()
            }
        } catch (e: Exception) {
            println("main : excpetion ${e.message}")
        }
    }

    delay(1300L)
    println("main : I'm tired of waiting!")
    job.cancelAndJoin()
    println("main : Now I can quit.")
}

class CloseableSleep : Closeable {

    suspend fun sleep() {
        repeat(100) {
            println("I'm sleeping $it ...")
            delay(500L)
        }
    }

    override fun close() {
        println("CloseableSleep close()")
    }
}

// I'm sleeping 0 ...
// I'm sleeping 1 ...
// I'm sleeping 2 ...
// main : I'm tired of waiting!
// CloseableSleep close()
// main : exception StandaloneCoroutine was cancelled
// main : Now I can quit.
```

<br/>

### ****실행 취소가 불가능한 블록 실행하기****

CancellableException이 발생했다는 것은 취소된 코루틴이라는 의미. 이 코루틴을 finally 구문 안에서 호출하게 되면 이미 취소된 상태가 되었기 때문에 다시 `CancellableException`이 발생하게 됨. 

```kotlin
fun main(args: Array<String>) = runBlocking {
    val job = launch(Dispatchers.Default) {
        try {
            coroutineTest()
        } finally {
            println("main : finally arrived!")
            coroutineTest()
        }
    }

    delay(1300L)
    println("main : I'm tired of waiting!")
    job.cancelAndJoin()
    delay(2000L)
    println("main : Now I can quit.")
}

suspend fun coroutineTest() {
    repeat(10) {
        println("I'm sleeping $it ...")
        delay(500L)
    }
}

// I'm sleeping 0 ...
// I'm sleeping 1 ...
// I'm sleeping 2 ...
// main : I'm tired of waiting!
// main : finally arrived!
// I'm sleeping 0 ...
// main : Now I can quit.
```

그래서 위와 같이 실행되는 것 같음. delay는 cancel 요청에 대한 친화적인 함수임을 생각해서 이해해보면 아래와 같음.

1. 1300L 동안 `coroutineTest` 실행
2. main : I'm tired of waiting! 출력
3. cancelAndJoin 실행
4. `CancellationException`이 발생하여 finally 도달, main : finally arrived! 출력
5. 다시 `coroutineTest` 실행. 그러나 첫 번째 반복 시작하고 delay 만났을 때 현재 코루틴이 cancel 상태이므로 실행 중단 
6. 2000L 중 남은 시간 다시 기다리고 main : Now I can quit. 출력

하지만 동기적으로 캔슬되었을 때 어떤 중단 함수를 끝까지 다 처리해야 된다면 `withContext{ … }` 빌더에 `NonCancellabe` 컨텍스트를 전달하여 이를 처리할 수 있음.

```kotlin
fun main(args: Array<String>) = runBlocking {
    val job = launch(Dispatchers.Default) {
        try {
            coroutineTest()
        } finally {
            withContext(NonCancellable) {
                println("main : finally arrived!")
                coroutineTest()
            }
        }
    }

    delay(1300L)
    println("main : I'm tired of waiting!")
    job.cancelAndJoin()
    delay(1300L)
    println("main : Now I can quit.")
}

suspend fun coroutineTest() {
    repeat(10) {
        println("I'm sleeping $it ...")
        delay(500L)
    }
}

// I'm sleeping 0 ...
// I'm sleeping 1 ...
// I'm sleeping 2 ...
// main : I'm tired of waiting!
// main : finally arrived!
// I'm sleeping 0 ... ~ 9까지 종료될 때까지 다 출력됨.
// main : Now I can quit.
```

<br/>

### ****Timeout****

일반적으로 어떤 코루틴의 실행을 중간에 취소해야 하는 경우가 있음. 주로 시간이 너무 오래 걸리면 인위적으로 취소를 해주어야 함. 이것을 timeout을 몰랐을 때라면 난 따로 세어주는 타이머를 넣어서 해주어야 하나..? 싶었는데 코루틴에 Timeout을 지정할 수가 있었음. `withTimeout()` 함수를 사용하면 됨. 이 함수를 사용하면 `TimeoutCancellationException`이 발생하게 됨. 

![image](https://github.com/eunjjungg/TIL/assets/100047095/3a5a824a-e47f-4879-9036-1237d23a8745)

`withTimeout`의 내부 구현체인데 다른 건 어려워서 못 알아보겠고 throw를 한다는 건 알겠음 ㅎㅎ; 🫠 이 익셉션은 아래에서 볼 수 있듯 코루틴이 종료되면 발생하는 `CancellationException`의 하위 클래스임. 

![image](https://github.com/eunjjungg/TIL/assets/100047095/e09f7d9a-465b-4982-baea-10f055bd47ac)

```kotlin
fun main(args: Array<String>) = runBlocking {
    withTimeout(1300L) {
        try {
            coroutineTest()
        } catch (e: Exception) {
            println("main: exception ${e.message}")
        } finally {
            println("main : finally arrived!")
        }
    }

    println("main : Now I can quit.")
}

suspend fun coroutineTest() {
    repeat(10) {
        println("I'm sleeping $it ...")
        delay(500L)
    }
}

// I'm sleeping 0 ...
// I'm sleeping 1 ...
// I'm sleeping 2 ...
// main: exception Timed out waiting for 1300 ms
// main : finally arrived!
// main exception 발생 후 종료
```

따라서 withTimeout에 대한 예외 처리를 해주지 않았다면 다음과 같이 비정상 종료하게 됨. 하지만 예외 처리를 하게 된다면 다음과 같이 비정상 종료가 발생하지 않음.

```kotlin
fun main(args: Array<String>) = runBlocking {
    try {
        withTimeout(1300L) {
            coroutineTest()
        }
    } catch (e: Exception) {
        println("main: exception ${e.message}")
    } finally {
        println("main : finally arrived!")
    }

    println("main : Now I can quit.")
}

suspend fun coroutineTest() {
    repeat(10) {
        println("I'm sleeping $it ...")
        delay(500L)
    }
}

// I'm sleeping 0 ...
// I'm sleeping 1 ...
// I'm sleeping 2 ...
// main: exception Timed out waiting for 1300 ms
// main : finally arrived!
// main : Now I can quit.
```

또다른 방법으로는 아예 throw를 하지 않는 `withTimeoutOrNull` 함수를 사용하는 것임. 내부 구현체를 보면 다음과 같음. 이 코루틴에서 발생한 익셉션이라면 timeout으로 발생한 것이므로 널처리해주면 되고 아니라면 전파하는 형식으로 이해하면 될 거 같음.

![image](https://github.com/eunjjungg/TIL/assets/100047095/23ffe344-dad2-498c-b55c-35454e64f8d1)

```kotlin
fun main(args: Array<String>) = runBlocking {
    try {
        withTimeoutOrNull(1300L) {
            coroutineTest()
        }
    } finally {
        println("main : finally arrived!")
    }
    
    println("main : Now I can quit.")
}

suspend fun coroutineTest() {
    repeat(10) {
        println("I'm sleeping $it ...")
        delay(500L)
    }
}

// I'm sleeping 0 ...
// I'm sleeping 1 ...
// I'm sleeping 2 ...
// main : finally arrived!
// main : Now I can quit.
```

그리고 실제로도 정상 종료되는 것을 알 수 있음. 

<br/>

### ****비동기 Timeout과 리소스****

withTimeout으로 발생되는 Timeout 이벤트는 현재 실행중인 블록의 코드와 비동기적으로 일어남! → 언제든지 이 이벤트가 발생할 수 있음. 문서에 따르면 Timeout 블록에서 return이 일어나기 직전에도 발생할 수 있다고 함.. 따라서 withTimeout으로 리소스를 해제할 생각이라면 섬세한 접근이 필요함. 

<br/>

### Ref.

- https://seyoungcho2.github.io/CoroutinesKoreanTranslation/undefined.html
- https://thdev.tech/kotlin/2020/11/03/kotlin_effective_09/
- [https://myungpyo.medium.com/코루틴-공식-가이드-자세히-읽기-part-2-ccd47699b520](https://myungpyo.medium.com/%EC%BD%94%EB%A3%A8%ED%8B%B4-%EA%B3%B5%EC%8B%9D-%EA%B0%80%EC%9D%B4%EB%93%9C-%EC%9E%90%EC%84%B8%ED%9E%88-%EC%9D%BD%EA%B8%B0-part-2-ccd47699b520)

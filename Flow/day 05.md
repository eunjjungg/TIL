## ****Flow 예외 처리****

### ****수집기에서의 try와 catch****

flow의 예외를 처리하기 위해 `try - catch` 블록을 사용할 수 있음. 

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
    try {
        simple().collect { value ->
            println(value)
            check(value <= 1) { "Collected $value" }
        }
    } catch (e: Throwable) {
        println("Caught $e")
    }
}

// Emitting 1
// 1
// Emitting 2
// 2
// Caught java.lang.IllegalStateException: Collected 2
```

<br/>

### ****모든 것이 잡힌다****

```kotlin
fun simple(): Flow<String> =
    flow {
        for (i in 1..3) {
            println("Emitting $i")
            emit(i) // emit next value
        }
    }.map { value ->
        check(value <= 1) { "Crashed on $value" }
        "string $value"
    }

fun main() = runBlocking<Unit> {
    try {
        simple().collect { value -> println(value) }
    } catch (e: Throwable) {
        println("Caught $e")
    }
}

// Emitting 1
// string 1
// Emitting 2
// Caught java.lang.IllegalStateException: Crashed on 2
```

위의 예시는 방출기, 중간 혹은 터미널 연산자 안에서의 예외들을 모두 잡아냄. 

<br/>

### ****예외 투명성(Exception Transparency)****

플로우는 예외에 있어서 반드시 투명해야 함. flow 빌더 코드 블록 내부에서 try catch 블록으로 예외를 처리한 후 방출하는 것은 예외 투명성을 위반하는 것임. 

방출 로직은 이러한 예외 투명성을 보존하기 위해 catch 연산자를 사용할 수 있음. catch 연산자 블록은 예외를 분석하고 발생한 예외의 타입에 따라 각기 다른 대응이 가능.

- throw 연산자를 통한 예외 다시 던지기
- catch 로직에서 emit을 사용해서 값 타입으로 방출
- 다른 코드를 통한 예외 무시, 로깅..

<br/>

### ****투명한 catch****

catch 중간 연산자는 오직 업스트림에서 발생하는 예외들에 대해서만 동작함. 즉 catch 연산자 위의 모든 연산자들에 대해서만 동작함. 다운스트림에서 발생한 예외에 대해서는 처리하지 않음. 

```kotlin
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
				// check(i <= 1)
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    foo()
        .catch { e -> println("Caught $e") } // does not catch downstream exceptions
        .collect { value ->
            check(value <= 1) { "Collected $value" }
            println(value)
        }
}

// Emitting 1
// 1
// Emitting 2
// Exception in thread "main" java.lang.IllegalStateException: Collected 2 ...
```

따라서 catch의 다운스트림인 collect에서 발생한 에러는 핸들링 되지 않음. 

주석 처리 된 것을 지우면 결과는 아래와 같이 정상 종료가 됨. 체크를 메인 함수에서만 수행한 코드는 에러를 핸들링해주지 않아 비정상 종료가 됐음. 

```kotlin
Emitting 1
1
Emitting 2
Caught java.lang.IllegalStateException: Check failed.
```

<br/>

### ****선언적으로 잡기****

```kotlin
fun simple(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    simple()
        .onEach { value ->
            check(value <= 1) { "Collected $value" }
            println(value)
        }
        .catch { e -> println("Caught $e") }
        .collect()
}
```

collect에 파라미터 없이 사용된 것을 보고, onEach에서 수행하는 로직을 보자. onEach가 마치 최종 연산해서 해야되는 일들을 해줌으로써 catch에서 최종적인 예외를 모두 처리할 수 있음. collect는 파라미터 없이 사용해서 flow의 수집을 발생시키기만 함. 

⇒ 명시적으로 try-catchㄹ르 사용하지 않고도 모든 예외들을 잡아낼 수 있음. 

<br/>

## ****Flow 수집 완료 처리하기****

flow의 수집이 완료되면 완료에 따른 동작을 실행해야 할 수 있음. 

<br/>

### ****명령적인 finally 블록****

collect 동작이 끝난 후 무언가를 수행하고 싶을 때 try - finally 블록을 사용하면 됨. 

```kotlin
fun simple(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
    try {
        simple().collect { value -> println(value) }
    } finally {
        println("Done")
    }
}

// 1
// 2
// 3
// Done
```

<br/>

### Ref.

- https://seyoungcho2.github.io/CoroutinesKoreanTranslation/flow.html
- [https://myungpyo.medium.com/코루틴-공식-가이드-자세히-읽기-part-9-a-d0082d9f3b89](https://myungpyo.medium.com/%EC%BD%94%EB%A3%A8%ED%8B%B4-%EA%B3%B5%EC%8B%9D-%EA%B0%80%EC%9D%B4%EB%93%9C-%EC%9E%90%EC%84%B8%ED%9E%88-%EC%9D%BD%EA%B8%B0-part-9-a-d0082d9f3b89)

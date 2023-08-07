## coroutine 기초

<br/>

### ****당신의 첫 번째 coroutine, Job 명시적으로 사용하기****

**코루틴**?

- 일시정지 연산을 위한 인스턴스
- 코드의 나머지 부분들과 동시에 실행되는 코드 블럭을 가진다는 점에서 스레드와 개념적으로 비슷하지만, 코루틴은 특정 스레드에 종속되어 실행되지 않음.
- 코루틴은 하나의 스레드에서 일시정지(suspend) 되었다가 다른 스레드에서 재개(resume) 될 수 있음.

```kotlin
// 1
fun main() = runBlocking { 
    launch { 
        delay(1000L) 
        println("World!") 
    }
    println("Hello") 
}
```

- `launch`: 코루틴 빌더로 독립적으로 동작하는 새로운 코루틴을 나머지 코드와 동시에 실행되도록 함.
- `delay`: 특별한 일시중단 함수로 특정한 시간 동안 코루틴을 일시중단함. 코루틴을 일시중단하는 것은 코루틴이 실행 중인 스레드를 블록하지 않으며 다른 코루틴이 해당 스레드에서 자신의 코드를 실행할 수 있도록 함.
- `runBlocking`: 코루틴 빌더. `launch`는 코루틴 스코프에서만 실행될 수 있는 반면, `runBlocking`은 그렇지 않음. `runBlocking`은 `{ … }` 내부의 모든 코루틴의 실행이 완료될 때까지 이를 실행하는 스레드가 호출 시간 동안 블록된다는 뜻임. → 비효율적

만약 위 예시에서 `runBlocking`으로 코루틴을 시작하지 않고 `GlobalScope.launch`로 실행하게 되면 코루틴의 특성으로 호출 스레드를 블록하지 않게 되어 메인 함수가 바로 종료되고, 메인 함수를 실행한 메인 스레드 역시 종료됨. 만약 GlobalScope.launch를 그대로 사용하면서 내부 작업들이 종료될 때까지 메인 스레드를 종료시키지 않게 하려면 어떻게 해야할까? → Job 인스턴스를 이용하여 해결 가능.

```kotlin
// 2 이렇게 하면 World가 실행되지 않음.
fun main() = runBlocking {
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    println("Hello")
}
```

```kotlin
// 3 이렇게 하면 World가 실행 됨. 
fun main() = runBlocking {
    val job = GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    println("Hello")
    job.join()
}
```

그러나 이 방식도 여전히 단점이 존재함. 여러 자식 코루틴이 있을 때 그것들이 다 실행될 수 있도록 부모 코루틴이 종료되는 시점에 `join()`을 사용해줘야 하기 때문. → 이렇기 때문에 코루틴 스코프가 중요함. 모든 코루틴들은 각자의 스코프를 가짐. 그래서 맨 위의 1번 예시에서 runBlocking으로 실행한 코루틴 블록 내부에서 launch로 생성된 새로운 코루틴이 종료될 때까지 대기할 수 있음. 

<br/>

### ****구조화된 동시성****

코루틴은 새로운 코루틴의 수명을 제한하는 특정한 코루틴 스코프 내에서만 실행될 수 있다는 원칙임. 위 코드에서는 runBlocking이 해당 스코프를 만듦. 구조화된 동시성으로 코루틴들이 손실되거나 누수를 일으키지 않도록 함. 바깥 스코프는 자식 코루틴들이 모두 완료될 때까지 완료되지 못함. 위 코루틴 스코프와 빌더 부분도 함께 보면 좋을듯. 

#### 💬 스터디 중간 나온 내용
globalScope는 싱글톤임. 그 어떤 Job에도 연결되어 있지 않음. 따라서 최상위 레벨에서 코루틴을 시작하므로 구조화된 동시성을 따르지 않음. 또한 Job에 연결되어 있지 않기 때문에 예외가 발생해도 다른 코루틴에 영향을 미치지 않음. 그리고 내부 코드의 종료를 기다리지 않음. 그리고 구조화된 동시성이라는 원칙 때문에 코루틴은 부모 - 자식의 계층 관계를 형성할 수 있음. 따라서 바깥 Scope인 부모는 자식 코루틴들이 완료될 때까지 완료되지 못함. 

<br/>

### ****함수 추출해 리펙토링하기****

```kotlin
fun main() = runBlocking { 
    launch { doWorld() }
    println("Hello")
}

suspend fun doWorld() {
    delay(1000L)
    println("World!")
}
```

1번의 코드에서 launch 내부를 함수로 추출해 리팩토링한 것임. `suspend` modifier를 붙여주어야 함. 일시중단 함수는 다른 보통의 함수들과 마찬가지로 Coroutines 내부에서 사용될 수 있지만 추가 기능은 다른 일시 중단 함수(e.g. `delay`)를 사용해 코루틴 실행을 일시 중단할 수 있음. ← why? suspend 키워드를 붙인 함수는 중단 함수임. 따라서 특정 코루틴 컨텍스트 안에서 수행되고 있고, 코루틴 컨텍스트 안에서는 모든 중단 함수를 호출할 수 있기 때문. 

> 🤪 이해 안 됨
> 
> 
> 하지만 만약 중단 함수가 현재 스코프에서 수행 될 코루틴 빌더를 포함한다면 어떨까요? 이 경우에 suspend 키워드 만으로는 충분하지 않습니다. 위 예제에서 doWorld() 함수를 CoroutineScope 의 확장함수로 만드는 방법을 생각해 볼 수있겠지만 이것은 API 를 불명확하게 만듭니다.
> 
> 좀 더 나은 방법은 명시적으로 CoroutineScope 을 필드로 갖는 클래스를 만들고 그 클래스가 해당 suspend 함수를 갖게하는 것, 혹은 외부(outer) 클래스의 구현을 암시적으로 사용하는 방법이 있습니다.
> 
> 또 다른 수단으로는 CoroutineScope(coroutineContext) 자체를 생성하여 사용할 수 있지만 이러한 접근 방식은 구조적으로 안전하지 않습니다. 왜냐하면 이러한 방식을 사용하는 순간부터 우리에겐 더 이상 이 메서드가 실행되는 스코프를 컨트롤 할 방법이 없어지기 때문입니다. private API 만이 이 빌더를 사용할 수 있습니다.
> 

> **From GPT**
> 
> 
> 위의 글은 Kotlin의 코루틴에 관한 내용을 다루고 있습니다. 주로 코루틴의 함수를 어떻게 분리하고 구조화하여 사용하는지에 대한 내용을 다루고 있습니다. 아래에서 각 부분을 자세히 설명하겠습니다.
> 
> **메인 함수와 코루틴**
> 
> 코드의 시작은 `main` 함수입니다. 이 함수는 프로그램의 진입점이며, Kotlin의 코루틴을 사용하여 비동기 작업을 실행하고 관리하는 방법을 보여줍니다. `runBlocking` 블록 내에서 `launch` 함수를 사용하여 `doWorld` 함수를 실행합니다.
> 
> **suspend 함수와 CoroutineScope**
> 
> `doWorld` 함수는 `suspend` 키워드를 사용하여 선언되었습니다. 이것은 코루틴 내에서 호출될 수 있는 중단 함수임을 나타냅니다. 중단 함수는 일시적으로 실행을 일시정지하고 나중에 재개할 수 있는 함수입니다.
> 
> 그리고 이어서 설명된 것처럼, 중단 함수가 현재 스코프의 코루틴 빌더를 포함한다면, `suspend` 키워드만으로는 충분하지 않습니다. 그래서 `doWorld` 함수를 별도의 클래스나 스코프 안에 정의하는 것이 좋습니다. 이렇게 하면 적절한 스코프 내에서 중단 함수를 사용하고 관리할 수 있습니다.
> 
> 또한, `CoroutineScope`를 사용하여 새로운 코루틴 스코프를 생성할 수 있습니다. 하지만 이 접근 방식은 일반적으로 안전하지 않으며, 스코프를 컨트롤하기 어렵게 만들 수 있습니다. 따라서 일반적으로는 함수나 클래스 단위로 스코프를 구성하는 것이 더 바람직합니다.
> 
> **결론**
> 
> 이 글은 코루틴을 사용하는 방식을 설명하면서 중단 함수와 코루틴의 구조적인 사용에 대해 언급하고 있습니다. 중단 함수를 선언하고, `suspend` 키워드를 이용하여 코루틴 내에서 호출할 수 있게 만드는 것이 핵심 포인트입니다. 또한 중단 함수를 어떻게 구조화하고 관리할지에 대한 고민도 함께 나타내고 있습니다.
> 
> **Q.**
> 그러니까 중단 함수 안에 코루틴 스코프 빌더 사용을 지양하자는 의미로 이해하는게 맞나요?
> 
> **A.**
> 
> 네, 맞습니다. 글에서 설명하고자 하는 주요 내용 중 하나는 중단 함수 내에서 코루틴 스코프 빌더를 사용하는 것을 지양하는 것입니다. 중단 함수 내에서 또 다른 코루틴 스코프를 생성하면 코드의 가독성과 관리가 어려워질 수 있습니다.
> 
> 코루틴 스코프는 주로 코드의 구조와 비동기 작업의 범위를 제어하기 위해 사용됩니다. 하지만 중단 함수 내부에서 새로운 코루틴 스코프를 생성하는 것은 코드의 복잡성을 증가시킬 수 있으며, 코루틴이 어떤 스코프 내에서 실행되는지를 명확하게 이해하기 어렵게 만들 수 있습니다.
> 
> 따라서 일반적으로 중단 함수가 코루틴 스코프 빌더를 생성하는 것은 지양하고, 대신 중단 함수를 호출하는 곳에서 적절한 코루틴 스코프를 설정하는 것이 더 바람직한 방법입니다. 이렇게 하면 코드의 구조가 더 명확해지며 코루틴이 어떤 범위에서 동작하는지 예측하기 쉬워집니다.
> 

<br/>

### ****Scope Builder, Scope Builder와 동시성****

- 스코프: 코루틴이 실행되는 범위
- 빌더: 코루틴 스코프의 **확장 함수**로, 개별적인 코루틴을 만드는 방법 (e.g. `launch`, `async`, `withContext`, `runBlocking`, `actor`, `produce`)

만일 어떤 코루틴들을 위한 사용자 정의 스코프가 필요한 경우가 있다면 `coroutineScope{…}` 빌더를 이용하면 됨. 이 빌더를 통해 생성된 코루틴은 모든 자식 코루틴이 끝날 때까지 종료되지 않는 스코프를 정의하는 코루틴임. `runBlocking`과 달리 `coroutineScope`는 자식들의 종료를 기다리는 동안 현재 스레드를 블록하지 않음. 

```kotlin
// 4 World까지 모두 출력 됨. 
fun main() = runBlocking {
    coroutineScope {
        launch {
            delay(1000L)
            println("World!")
        }
        println("Hello")
    }
}
```

```kotlin
// 5
fun main(args: Array<String>) = runBlocking {
    launch {
        delay(200L)
        println("Task from runBlocking")
    }

    coroutineScope {
        launch {
            delay(500L)
            println("Task from nested launch")
        }
        delay(100L)
        println("Task from coroutine scope")
    }

    launch {
        delay(50L)
        println("Task from last coroutine in runBlocking")
    }
    
    println("Coroutine scope is over")
}

// Task from coroutine scope
// Task from runBlocking
// Task from nested launch
// Coroutine scope is over
// Task from last coroutine in runBlocking
```

5번 예시를 보면 어찌 됐든 코루틴 스코프 빌더로 생성된 코루틴은 끝나기 전까지 다른게 실행되지는 않음. 그러나 이미 실행된 코루틴은 계속 실행 됨. 왜냐면 `runBlocking`과 다르게 스레드를 블록하지는 않기 때문. 

<br/>

### ****Coroutines는 경량(light-weight) 이다****

> Coroutines는 JVM의 Thread들보다 덜 리소스 집약적이다. Thread를 사용할 때 JVM의 가용 메모리를 소진시키지는 코드는 Coroutine을 사용하여 리소스의 제한치에 도달하지 않도록 표현될 수 있다.
> 

<br/>

### Ref.

- https://seyoungcho2.github.io/CoroutinesKoreanTranslation/coroutines-1.html
- https://jaejong.tistory.com/64
- https://myungpyo.medium.com/reading-coroutine-official-guide-thoroughly-part-1-98f6e792bd5b

## ****여러 Flow 하나로 합치기****

### ****Zip****

- 두 개의 flow 값을 결합하는 연산자
- 하나의 플로우가 완료되면 결과 플로우는 즉시 완료되고 나머지 남은 플로우에서는 취소가 호출됨.

```kotlin
fun main() = runBlocking {
    val nums = (1..3).asFlow() // numbers 1..3
    val strs = flowOf("one", "two", "three") // strings
    nums.zip(strs) { a, b -> "$a -> $b" } // compose a single string
        .collect { println(it) } // collect and print
}

// 1 -> one
// 2 -> two
// 3 -> three
```

<br/>

### ****Combine****

다음과 같은 경우에 combine을 사용함.

- Flow가 가장 최신의 값 혹은 연산을 표시할 때 ← 해당 Flow의 가장 최신 값에 의존적인 연산이 필요할 때
- 별도의 업스트림이 새로운 값을 방출할 때 다시 연산이 필요할 때

또한 가장 느린 플로우가 모두 종료될 때가지 값이 방출되므로 값의 손실이 일어나지 않음. 

```kotlin
fun main() = runBlocking {
    val nums = (1..3).asFlow().onEach { delay(300) }
    val strs = flowOf("one", "two", "three").onEach { delay(400) }
    val startTime = System.currentTimeMillis()
    nums.연산자(strs) { a, b -> "$a -> $b" }
        .collect { value ->
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
}
```

위 코드에서 연산자라고 쓰여진 부분에 combine or zip을 사용하면 그 차이를 알 수 있음. 두 플로우의 속도가 다를 때 결과가 달라짐.

| combine | zip |
| --- | --- |
| 1 -> one at 435 ms from start | 1 -> one at 428 ms from start
| 2 -> one at 635 ms from start | 2 -> two at 824 ms from start
| 2 -> two at 845 ms from start | 3 -> three at 1233 ms from start
| 3 -> two at 940 ms from start | | 
| 3 -> three at 1252 ms from start | | 
| 각 플로우에서 방출이 일어날 때마다 값을 만들어냄 | 느린 플로우 기준으로 값을 만들어냄 |

![image](https://github.com/eunjjungg/TIL/assets/100047095/2098fcc8-be56-46d7-8ec6-74c139414f36)

<br/>

## ****Flow를 Flatten하기****

플로우는 비동기로 수신되는 값들의 시퀀스를 나타냄. 따라서 어떤 플로우에서 수신되는 값들이 다른 값들의 시퀀스를 요청하는 플로우가 되는 경우가 종종 발생함. (그러니까 플로우의 각 수신 값에서 새로운 플로우를 만드는 경우)

![image](https://github.com/eunjjungg/TIL/assets/100047095/0ce00f24-3e87-4b93-97a6-ce7ce66050f5)

만약 위 코드의 `requestFlow` 함수를 다음과 같이 요청한다고 생각해보면 `tmp` 변수의 타입은 `Flow<Flow<String>>`이 됨. 이는 하나의 단일 플로우로 flatten해줄 필요가 있음. 

<br/>

### ****flatMapConcat****

여러 flow를 연결해 새로운 플로우를 만드는 연산자임. 다음 플로우의 수집을 시작하기 전에 현재 플로우가 완료될 때까지 기다림. 

<br/>

**📚 내부 로직**

```kotlin
@ExperimentalCoroutinesApi
public fun <T, R> Flow<T>.flatMapConcat(transform: suspend (value: T) -> Flow<R>): Flow<R> =
    map(transform).flattenConcat()
```

`flatMapConcat`은 내부적으로는 두 개의 과정을 수행함.

1. `transform`을 받아 `transform`변수에 대해 `map`을 수행 → 발행된 데이터를 각각 다른 flow로 변환함. 
2. 1에서 생성된 flow들에 `flattenConcat()`을 수행해 합쳐져 하나의 flow가 만들어짐. 

```kotlin
@ExperimentalCoroutinesApi
public fun <T> Flow<Flow<T>>.flattenConcat(): Flow<T> = flow {
    collect { value -> emitAll(value) }
}
```

<br/>

**📚 예제** 

```kotlin
fun main() = runBlocking {
    val startTime = System.currentTimeMillis() // remember the start time
    val flow: Flow<String> = (1..3).asFlow().onEach { delay(100) } // a number every 100 ms
        .flatMapConcat { requestFlow(it) }
    
    flow
        .collect { value -> // collect and print
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
}

// 1: First at 124 ms from start
// 1: Second at 635 ms from start
// 2: First at 741 ms from start
// 2: Second at 1243 ms from start
// 3: First at 1347 ms from start
// 3: Second at 1853 ms from start
```

First, Second의 발행 시각 차이가 500ms 정도인걸 확인할 수 있음. 그리고 1의 First, Second가 완료되어야 2의 First가 시작하는 걸 알 수 있음.

순전히 내 생각인데 flatMapConcat은 1..3을 방출하는 플로우의 확장함수로 사용되고 있고 flatMapConcat의 람다식인 transform은 또다른 플로우를 생성하는 requestFlow()임. 여기서 맵을 하게 되면 1..3 플로우의 각 값에 대해 transform을 붙여줌. ⇒ 여기서 만들어진게 Flow<Flow<T>>임. 그러면 이 플로우+플로우에 대해 collect를 해줘서 각 각 값들을 모두 방출한게 결과가 됨..?????????????????

- [ ]  사실 잘 모르겠다. 내부코드를 이해하기엔 난 멀었다..

<br/>

**📚 한계**

transform에 해당하는 연산이 오래 수행되는 연산일 경우 데이터 실제 발행 시점과 변환되기 전 발행 시점 사이에 많은 차이가 생길 수 있음. ⇒ 첫 데이터 파이프라인의 발행 시점과 최종 데이터 파이프라인의 소비 시점에 많은 차이가 생길 수 있음.

<br/>

### ****flatMapMerge****

들어오는 모든 플로우들을 동시에 수집하고 그 값들을 단일 플로우로 합쳐서 값들이 가능한 빨리 방출되도록 함. 

```kotlin
fun main() = runBlocking {
    val startTime = System.currentTimeMillis() // remember the start time
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms
        .flatMapMerge { requestFlow(it) }
        .collect { value -> // collect and print
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
}

// 1: First at 144 ms from start
// 2: First at 242 ms from start
// 3: First at 348 ms from start
// 1: Second at 650 ms from start
// 2: Second at 748 ms from start
// 3: Second at 854 ms from start
```

flatMapMerge는 내부의 코드 블록(requestFlow)을 순차적으로 호출하지만 결과 flow는 동시적으로 수집함. 이는 map { requestFlow }를 먼저 호출하고 flattenMerge를 호출하는 것과 같음.

<br/>

**📚 더 알아보기**

순차적으로 처리되지 않아도 되는 데이터라면 **데이터들을 동시에 수집한 후** 값들이 가능한 빨리 방출될 수 있도록 병렬로 처리하는 것이 맞음. 따라서 앞선 값에 대한 변환의 완료를 기다리지 않고 자신의 값의 변환을 시작함.

⇒ 순서를 지키지 않아도 되는 데이터를 병렬적으로 처리하고 싶을 때 사용. 최종 플로우의 데이터들이 원본 플로우의 순서를 보장하지 않음.

<br/>

### ****flatMapLatest****

새로운 flow의 collection이 방출되면 이전 플로우의 컬렉션 처리는 취소되는 방식임. 

```kotlin
fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First")
    delay(500) // wait 500 ms
    emit("$i: Second")
}

fun main() = runBlocking {
    val startTime = System.currentTimeMillis() // remember the start time 
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms 
        .flatMapLatest { requestFlow(it) }
        .collect { value -> // collect and print 
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
}

// 1: First at 142 ms from start
// 2: First at 279 ms from start
// 3: First at 386 ms from start
// 3: Second at 888 ms from start
```

<br/>

**📚 더 알아보기**

`flatMapLatest`는 플로우의 최신 데이터만을 이용해 새로운 플로우로 변환할 수 있도록 도와주는 함수. 발행된 데이터를 **변!환!**하는 도중 새로운 데이터가 발행되면 **변환 로직을 취소**하고 새로운 데이터를 사용해 변환을 수행함. 

그러면 위의 예시도 이해가 되는데 1을 방출하고 마저 변환 도중에 2가 방출되므로 2를 또 변환함. 2 변환 도중에 3이 방출되니 3은 방해 없이 변환이 완료됨.

```kotlin
fun requestFlow(i: Int): Flow<String> = flow {
    delay(1000)
    emit("$i: First")
    delay(500) // wait 500 ms
    emit("$i: Second")
}

fun main() = runBlocking {
    val startTime = System.currentTimeMillis() // remember the start time
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms
        .flatMapLatest { requestFlow(it) }
        .collect { value -> // collect and print
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
}

// 3: First at 1393 ms from start
// 3: Second at 1901 ms from start
```

이 코드를 보고 나는 좀 더 `flatMapLatest`에 대해 이해가 됨. 강제로 변환 코드의 딜레이를 늘리니 2가 방출될 시점에 1의 변환 로직 중 아무것도 emit 된게 없어서 (캔슬되므로) print 되지 않고 3이 방출된 시점에도 마찬가지다. 따라서 3에 대한 변환 로직만 성공적으로 수행하게 됨.

- [ ]  https://kotlinworld.com/263?category=973477 여기 블로그에서 `flatMapLatest`는 flow에서 발행된 데이터를 변환할 때 발행된 순서대로 순차적으로 변환한다고 되어있는데 `flatMapLatest`도 병렬 처리 아닌가? → 병렬처리 맞는듯. 단 블로그 작성자분께서는 그런 의도로 쓰신게 아니고 flatMapMerge가 발행된 데이터를 순차적으로 처리하지 않는다는 의미에서 작성하신 것 같음. 

<br/>

📚 더.

```kotlin
fun requestFlow(i: Int): Flow<String> = flow {
    delay(1000)
    emit("$i: First")
    delay(500) // wait 500 ms
    emit("$i: Second")
}

fun flowFrom(ele: Int) = flowOf(1, 2, 3)
    .onEach { delay(1000L) }
    .map { "${it}_${ele}" }

fun main() = runBlocking {
    val startTime = System.currentTimeMillis() // remember the start time
    (1..3).asFlow().onEach { delay(100) } // a number every 100 ms
        .flatMapLatest { 변환함수(it) }
        .collect { value -> // collect and print
            println("$value at ${System.currentTimeMillis() - startTime} ms from start")
        }
}
```


<br/>

### Ref.

- https://kotlinworld.com/261
- https://simplecode.kr/41
- [https://velog.io/@sana/KotlinFlow-Flow-결합-연산자-zip-combine](https://velog.io/@sana/KotlinFlow-Flow-%EA%B2%B0%ED%95%A9-%EC%97%B0%EC%82%B0%EC%9E%90-zip-combine)
- [https://myungpyo.medium.com/코루틴-공식-가이드-자세히-읽기-part-9-a-d0082d9f3b89](https://myungpyo.medium.com/%EC%BD%94%EB%A3%A8%ED%8B%B4-%EA%B3%B5%EC%8B%9D-%EA%B0%80%EC%9D%B4%EB%93%9C-%EC%9E%90%EC%84%B8%ED%9E%88-%EC%9D%BD%EA%B8%B0-part-9-a-d0082d9f3b89)
- https://seyoungcho2.github.io/CoroutinesKoreanTranslation/flow.html
- https://kotlinworld.com/261
- https://kotlinworld.com/262?category=973477
- [https://roomedia.tistory.com/entry/flatMapConcat-flatMapMerge-flatMapLatest-차이](https://roomedia.tistory.com/entry/flatMapConcat-flatMapMerge-flatMapLatest-%EC%B0%A8%EC%9D%B4)

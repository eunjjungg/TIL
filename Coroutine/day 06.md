## ****Channels (1) 🙆‍♀️****

### ****Channel 기초****

Channel은 값의 스트림을 전달하는 방법을 제공함. Channel은 개념적으로 `BlockingQueue`와 유사함. 주요한 다른 점은 blocking 연산인 `put` 대신 suspend 연산인 `send`를 가지고, blocking 연산인 `take` 대신 suspend 연산인 `receive`를 가짐.

---

**📚 BlockingQueue?** 

queue + blocking 개념. 모든 연산이 스레드를 블록하는 것이 기본임! producer, consumer model을 구성하기에 적절한 데이터 타입으로 아래와 같은 특성을 가짐. 

1. 큐의 capacity가 가득 찬 상태에서 enqueue 발생 시 기존 큐라면 exception throw. 그러나 blocking queue는 이 queue에 insert 가능할 때까지 block하고 대기함. 다른 스레드에서 dequeue가 발생하면 block 풀리고 enqueue가 됨.
2. queue가 empty 상황일 때 dequeue 발생 시 exception throw 대신 block 해두다가, 다른 스레드에서 enqueue가 발생 시 dequeue가 이뤄짐. 

---

```kotlin
suspend fun main() = runBlocking {
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) {
            delay(1500L)
            channel.send(x * x)
        }
    }
    repeat(5) { println(channel.receive()) }
    println("Done!")
}

// 1
// 4
// 9
// 16
// 25
// Done!
```

따라서 1.5초마다 1, 4, 9, 16, 25 순으로 출력이 됨. 여기서 receive 쪽 반복문을 repeat(6)로 바꿔주면 Done도 출력되지 않고 종료가 되지 않음. 

<br/>

### ****Channel 닫기와 반복적으로 수신하기****

channel은 queue와 다르게 더 이상 다른 원소들이 오지 않는 것을 알리기 위해 닫힐 수 있음. `channel.close()`를 사용하면 채널로 특별한 닫기 토큰을 삽입하는 거와 같다고 이해하면 됨. 

또한 channel로부터 값을 받는 쪽에서는 일반적인 for loop를 사용해서 원소를 편하게 받을 수 있음. 

예제와는 다르게 내가 실험해보려고 아래와 같이 작성했음.

```kotlin
suspend fun main() = runBlocking {
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) {
            delay(100L)
            channel.send(x * x)
        }
        channel.close()
    }
    repeat(6) { println(channel.receive()) }
    println("Done!")
}
```

내 생각에는 이렇게 했을 때 자동으로 close가 되니까 마지막 receive()를 무시할 줄 알았는데 Exception이 발생했음. 

![image](https://github.com/eunjjungg/TIL/assets/100047095/4cc11644-2c93-423d-afd5-d550263cd2d9)

채널이 close 되었을 때 recieve를 사용해주면 안 됨. 예제와 같이 지속적 요청이 있게끔 구현했을 때 close를 사용하는게 안전하게 사용하는 방법인 것 같음. 

```kotlin
suspend fun main() = runBlocking {
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) {
            delay(100L)
            channel.send(x * x)
        }
        channel.close()
    }
    for (y in channel) println(y)
    println("Done!")
}
```

위와 같이 구현했을 때는 문제 없이 1, …, 25, Done!까지 Exception 없이 잘 출력이 됨.

<br/>

### ****Producer로 Channel 만들기****

코루틴이 원소들의 시퀀스를 생성하는 패턴이 일반적인데 ⇒ 생산자 소비자 패턴의 일종임. 이걸 채널로도 구현할 수 있음. 생산자는 produce, 소비자는 consumeEach로 사용할 수 있음. 

<br/>

### ****파이프라인****

파이프라인은 하나의 코루틴이 값의 스트림을 생성하는 것을 뜻함. 값의 스트림은 무한할 수 있음. 

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) send(x++) // infinite stream of integers starting from 1
}
```

위와 같이 만든 스트림은 다른 코루틴들이 그 스트림을 소비하여 다른 결과를 수행할 수 있음. 

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) send(x++) // infinite stream of integers starting from 1
}

fun CoroutineScope.square(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (x in numbers) send(x * x)
}

suspend fun main() = runBlocking {
    val numbers = produceNumbers() // produces integers from 1 and on
    val squares = square(numbers) // squares integers
    repeat(5) {
        println(squares.receive()) // print first five
    }
    println("Done!") // we are done
    coroutineContext.cancelChildren() // cancel children coroutines
}
```

![image](https://github.com/eunjjungg/TIL/assets/100047095/5773509e-ebb7-4d44-99d2-d8267a842282)

produce는 CoroutineScope의 확장 함수로 ReceiveChannel을 리턴함. 코루틴 스코프의 확장함수이므로 글로벌하게 남은 코루틴이 없게끔 부모 코루틴이 자식이 종료되어야만 종료되도록 만드는 구조화된 동시성의 원칙을 지키게 할 수 있음. 

<br/>

### ****파이프라인으로 소수 만들기****

```kotlin
fun main() = runBlocking {
    var cur = numbersFrom(2)
    repeat(10) {
        val prime = cur.receive()
        println(prime)
        cur = filter(cur, prime)
    }
    coroutineContext.cancelChildren() // cancel all children to let main finish
}

fun CoroutineScope.numbersFrom(start: Int) = produce<Int> {
    var x = start
    while (true) send(x++) // infinite stream of integers from start
}

fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int) = produce<Int> {
    for (x in numbers) if (x % prime != 0) send(x)
}
```

- [ ]  솔직히 이해 안 됨.

위의 예시에서 시퀀스로 바꾸려면 다음과 같이 해주면 됨.

- produce → Iterator
- send → yield
- receive → next
- ReceiveChannel → Iterator

<br/>

### Ref.

- blocking queue : https://huzz.tistory.com/28
- https://simplecode.kr/46
- https://seyoungcho2.github.io/CoroutinesKoreanTranslation/channels.html

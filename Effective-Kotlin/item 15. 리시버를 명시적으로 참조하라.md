# item 15. 리시버를 명시적으로 참조하라

함수와 프로퍼티를 local or top level 변수가 아닌 다른 리시버로부터 가져온다는 것을 나타내는 경우가 있음. 예를 들어 클래스의 메서드라는 것을 나타내기 위한 this가 있음.

비슷하게 확장 리시버(확장 함수에서의 this)를 명시적으로 참조하게 할 수도 있음. 

```kotlin
// 명시 참조하지 않은 경우
fun <T: Comparable<T>> List<T>.quickSort(): List<T> { 
		if (size < 2) return this
		// ...
}

// 명시 참조한 경우
fun <T: Comparable<T>> List<T>.quickSort(): List<T> { 
		if (this.size < 2) return this
		// ...
}

```

<br/>

### 여러 개의 리시버

스코프 내부에 둘 이상의 리시버가 있는 경우 리시버를 명시적으로 나타내면 좋음. `apply`, `with`, `run` 함수를 사용할 때가 대표적인 예임. 

왜 리시버를 명시적으로 나타내면 좋을지 예시를 보면 확인할 수 있음. 

```kotlin
// 1
class Node(val name: String) {
    fun makeChild(childName: String) = create("$name.$childName")
        .apply { print("Created ${name}") }

    fun create(name: String): Node? = Node(name)
}

fun main() {
    val node = Node("parent") 
    node.makeChild ("child")
}
```

<br/>


1번 코드의 경우에는 Created parent.child가 출력될거라고 예상하지만 실제로 출력되는 것은 Created parent가 출력됨. 왜냐하면 makeChild 함수의 create까지는 parent.child라는 네임을 가진 노드를 만들기는 하지만~ print가 되는 것은 따로 리시버를 지정해주지 않았기 때문에 parent의 name이 출력되는 것임. 더 자세한 설명은 아래 인용임.

> The intuitive answer is "Created child", but the actual answer is "Created parent". Why? Notice that the `create` function declares a nullable result type, so the receiver inside `apply` is `Node?`. Can you call `name` on `Node?` type? No, you need to unpack it first. However, Kotlin will automatically (without any warning) use the outer scope, and that is why "Created parent" will be printed. We fooled ourselves. The solution is to not create unnecessary receivers. This is not a case in which we should use `apply`: it is a clear case for `also`, for which Kotlin would force us to use the argument value safely if we used it.
> 
> 출처 : https://kt.academy/article/fk-scope-functions


<br/>

```kotlin
//2
class Node(val name: String) {
    fun makeChild(childName: String) = create("$name.$childName")
        .apply { print("Created ${this.name}") }

    // Compilation error
    fun create(name: String): Node? = Node(name)
}
```


하지만 2번과 같이 바꿨다고 해서 바로 실행되는 것은 아님. 왜냐하면 apply 내부의 this의 타입이 Node?라서 이를 직접 사용할 수 없고 언팩하여 사용해야 함. 

```kotlin
// 3
class Node(val name: String) {
    fun makeChild(childName: String) = create("$name.$childName")
        .apply { print("Created ${this?.name}") }

    fun create(name: String): Node? = Node(name)
}

fun main(){
    val node = Node("parent") 
    node.makeChild("child") // Prints: Created parent.child 
}
```

이렇게 3번과 같이 바꿔주어야 됨. 하지만 3번처럼 코드를 작성하는 것은 `apply`를 잘못 사용하고 있는 것임. 만약 `also` 함수와 `name`을 사용했다면 이런 문제 자체가 일어나지 않음. `also`를 사용하면 이전과 마찬가지로 명시적으로 리시버를 `it`으로 지정하게 됨. `also`, `let`을 사용하는 것이 `nullable` 값을 처리할 때 좋은 선택지임. 레이블 없이 리시버를 사용하면 가장 가까운 리시버를 의미하게 됨. 외부에 있는 리시버를 사용하려면 레이블을 사용해야 함. 

<br/>

### DSL 마커

코틀린 DSL을 사용할 때는 여러 리시버를 가진 요소들이 중첩되더라도 리시버를 명시적으로 붙이지 않음. 왜냐 DSL 자체가 그렇게 하도록 설계되었기 때문임.

# item 1. 가변성을 제한하라

코틀린은 모듈로 설계하고 모듈은 클래스, 객체, 함수, type alias 등 다양한 요소로 구성됨. 이 요소들 중 일부는 상태(state)를 가질 수 있음. 그 예로 read-write property인 var를 사용하거나 mutable 객체를 사용하면 state를 가지게 됨. 요소가 상태를 가지게 되면 사용 방법 뿐 아니라 history에도 의존하게 됨. 

<br/>

**상태가 있을 때의 문제점**

- 프로그램 이해 & 디버깅에 어려움. 상태를 갖는 부분들의 관계를 이해해야 하고 상태 변경이 많아지면 추적하는 것이 힘들어짐.
- 가변성이 있으면 코드의 실행을 추론하기 어려워짐. 시점에 따라서 값이 달라질 수 있기 때문임. 또한 같은 시점이라고 항상 같은 값이 있을거라는 보장도 없음.
- 멀티스레드 프로그램일 때의 동기화 문제.
- 모든 상태를 테스트해야 하므로 테스트가 어려움.
- 상태 변경이 일어났을 때 이런 변경을 다른 부분에 알려야 하는 경우가 있음. 정렬된 리스트에 가변 요소를 추가하면 요소에 변경이 일어날 때마다 전체 리스트를 다시 정렬해야 함.

→ 따라서 변할 수 있는 지점을 줄이는 것이 좋음. 가변성을 완벽히 제한한 건 함수형 언어임. 

<br/>

## 코틀린에서 가변성 제한하기

코틀린은 가변성을 제한할 수 있는 언어임. 그 방법들 중 중요한 것은 아래 세 가지.

- 읽기 전용 프로퍼티 val
- 가변 컬렉션과 읽기 전용 컬렉션 구분하기
- 데이터 클래스의 copy

<br/>

### 읽기 전용 프로퍼티 val

value처럼 동작하며 일반적인 방법으로는 값이 변하지 않음. val은 읽기 전용 프로퍼티지만 immutable 프로퍼티는 아님. 따라서 완전히 변경할 필요가 없다면 final property를 사용해야 함. 

- 값이 변하는 경우 1 : val이 mutable 객체를 담고 있다면 내부적으로 변할 수 있음.
    
    ```kotlin
    // list는 재할당만 막음. 
    val list = mutableListOf(1, 2, 3)
    list.add(4)
    ```
    
- 값이 변하는 경우 2 : val 프로퍼티는 다른 프로퍼티를 활용하는 사용자 정의 게터로도 정의할 수 있음. var property를 사용하는 val property의 경우 var property가 변할 때 같이 변할 수 있음.
    
    ```kotlin
    // name, surname이 변화하면 fullName도 변화하게 됨.
    
    var name: String = "ej"
    var surname: String = "jjj"
    val fullName 
        get() = "$name $surname"
    ```
    

코틀린의 프로퍼티는 기본적으로 캡슐화 되어있고 추가적으로 getter, setter를 가질 수 있음. var은 getter, setter 모두 제공하고 val은 변경이 불가능하므로 getter만 지원함. → val을 var로 오버라이드할 수 있음.

```kotlin
interface Element {
    var active: Boolean
}

class ActualElement: Element {
    override var active: Boolean = false
}
```

아래는 getter로 정의한 val property인 fullName가 스마트 캐스트가 안 되는 것을 보여줌. 

```kotlin
val name: String? = "Marton"
val surname: String = "Braun"

val fullName: String? 
    get() = name?.let { "$it $surname" }
val fullName2: String? = name?.let { "$it $surname" }

fun main() {
    if (fullName != null) {
        println(fullName.length) // 오류
    }
    
    if (fullName2 != null) {
        println(fullName2.length)
    }
}
```

<br/>

### 가변 컬렉션과 읽기 전용 컬렉션 구분하기

![image](https://github.com/eunjjungg/TIL/assets/100047095/c96bce72-6d61-429e-8cba-cd890d81ab9a)

- 코틀린은 읽고 쓸 수 있는 컬렉션과 읽기 전용 컬렉션으로 나뉨. 위 사진에서 좌측은 읽기 전용으로 변경을 위한 메소드가 없지만 우측과 같이 mutable이 붙은 collection들은 읽고 쓸 수 있는 컬렉션임.
    - mutable이 붙은 인터페이스는 대응되는 읽기 전용 인터페이스를 상속 받아서 변경을 위한 메소드를 추가한 것임.
- 다운캐스팅은 절대 하면 안 됨. 리스트를 읽기 전용으로 리턴하면 읽기 전용으로만 사용해야 함. 대신 copy를 통해서 새로운 mutable 컬렉션을 만드는 방식을 사용해야 함.
    
    ```kotlin
    val list = listOf(1, 2, 3)
    // 다운캐스팅 하면 안 됨.
    if (list is MutableList) {
    	list.add(4)
    }
    
    // 권장 방법
    val list = listOf(1,2,3)
    val mutableList = list.toMutableList()
    ```
    

<br/>

### 데이터 클래스의 copy

immutable 객체를 사용하면 아래와 같은 장점이 있음. 

- 한 번 정의된 상태가 유지되므로 코드를 이해하기 쉬움.
- immutable 객체는 공유해도 충돌이 일어나지 않아 병렬 처리를 안전하게 할 수 있음.
- immutable 객체에 대한 참조는 변경되지 않아 쉽게 캐시할 수 있음.
- immutable 객체는 방어적 복사본(아래 나옴)을 만들 필요가 없음.
- immutable 객체는 set, map의 키로 사용할 수 있음. mutable은 사용할 수 없음. 왜냐하면 요소에 수정이 일어나면 해시 테이블 내부에서 요소를 찾을 수 없게 되기 때문.
    
    ```kotlin
    val names: SortedSet<FullName> = TreeSet()
    val person = FullName("AAA", "AAA")
    names.add(person)
    names.add(FullName("Jordan", "Hansen"))
    names.add(FullName("David", "Blanc"))
    println(names)
    println(person in names) // true
    
    person.name = "ZZZ"
    println(names)
    println(person in names) // false
    ```
    

mutable 객체는 예측하기 어려우며 위험하다는 단점이 있지만 immutable 객체는 변경할 수 없다는 단점이 있음. → immutable 객체는 자신의 일부를 수정한 새로운 객체를 만들어 내는 메소드를 가져야 함. 다만 모든 프로퍼티를 대상으로 메소드를 만드는 것은 번거로움. → 이럴 때는 data 한정자를 사용하면 됨. data는 copy라는 이름의 메소드를 자동으로 만들어 줌. copy 메소드는 모든 기본 생성자 프로퍼티가 같은 새로운 객체를 만들어낼 수 있음. 

```kotlin
class User(
    val name: String,
    val surname: String,
) {
    fun withSurname(surname: String) = User(name, surname)
}
var user = User("Nathan", "Hong")
user = user.withSurname("Jo")
print(user)

// data 한정자!
data class(
    val name: String,
    val surname: String,
)
var user = User("Nathan", "Hong")
user = user.copy(usrname = "Jo")
print(user)
```

<br/>

### 다른 종류의 변경 가능 지점

변경할 수 있는 리스트를 만든다고 할 때 선택지는 두 가지가 있음.

- mutable collection을 만드는 것 
`val list1: MutableList<Int> = mutableListOf()`
- var로 읽고 쓸 수 있는 프로퍼티를 만드는 것 
`val list2: List<Int> = listOf()`

둘 다 변경이 가능하지만 변경 가능 지점의 위치가 다름. 후자는 사용자 정의 세터를 활용해서 그 변경을 추적할 수 있음. (e.g. `Delegates.observable`)

```kotlin
var names by Delegates.observable(listOf<String>()) {_, old, new ->
    println("Names changed from $old to $new")
}

names += "Fabio"
names += "Bill"
```

<br/>

🥲 **최악의 방법은?**

`var list3 = mutableListOf<Int>()` → 다음과 같이 프로퍼티와 컬렉션을 모두 변경 가능한 지점으로 만드는 것. 

<br/>

### 변경 가능 지점 노출하지 말기

상태를 나타내는 mutable 객체를 외부에 노출하는 것은 위험함. 왜냐하면 외부에서 수정할 수 있기 때문. 이를 막는 방법으로는 두 가지가 있음.

- 리턴되는 mutable 객체를 복제하는 방어적 복제 방식 사용. → `.copy()`로 새로운 객체 리턴
- 리턴할 일이 있을 때 읽기 전용 슈퍼 타입으로 업캐스트하여 가변성을 제한. → `MutableMap`을 리턴하지 말고 `Map`을 리턴

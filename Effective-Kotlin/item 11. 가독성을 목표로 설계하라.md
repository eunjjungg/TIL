# item 11. 가독성을 목표로 설계하라

### 인식 부하 감소

```kotlin
// 코드 1
if (person != null && person.isAdult) {
		view.showPerson(person)
} else {
		view.showError()
}

// 코드 2
person?.takeIf { it.isAdult }
		?.let(view::showPerson)
		?: view.showError()
```

가독성이란 코드를 읽고 얼마나 빠르게 이해할 수 있는지를 의미함. 이는 우리의 뇌가 얼마나 많은 관용구에 익숙해져 있는지에 따라서 다름. 코드 2는 숙련자에게는 쉽게 읽을 수 있는 코드지만 숙련된 개발자만을 위한 코드는 좋은 코드가 아님. 또한 코드 1은 코드 2에 비해 수정하기 쉬움. 

또한 코드 2에서 let 뒤를 람다식으로 바꾸었을 때 let은 람다식의 결과를 리턴하게 됨. 이때 람다식의 결과가 null이라면 ?: 뒤의 코드도 실행하게 되어 실행 결과가 원하던 바와 달라질 수 있음. 

⇒ 결론적으로 인지 부하를 줄이는 방향으로 코드를 작성해야 함. 

<br/>

### 극단적이게 되지 않기

위에 말한 것처럼 let이 원하던 결과를 내지 않는다고 해서 let을 절대로 쓰면 안 된다는 의미는 아님. let은 nullable 가변 프로퍼티가 있고 null이 아닐 때 특정 작업을 수행해야 하는 경우가 있을 때 → **가변 프로퍼티는 쓰레드와 관련된 문제를 발생시킬 수 있으므로 스마트 캐스팅이 불가능**함. 이때 `?.let`을 사용하면 됨.

그 외에도 let을 사용하는 경우

- 연산을 아규먼트 처리 후로 이동시킬 때 (함수 레퍼런스를 사용중)
    - `print(students.filter{}.joinToString{})` → `students.filter{}.joinToString{}.let(::print)`
- 데코레이터를 사용해서 객체를 랩할 때

```kotlin
// 위의 두 가지 예시를 사용 중인 코드 
students
		.filter { it.pointsInSemester > 15 && it.result>=50}
		.sortedWith(compareBy({ it.surname }, { it.name}))
		.joinToString(separator = "\n") {
        "${it.name} ${it.surname}, ${it.result}"
		}
		.let(::print)

var obj = FileInputStream("/file.gz") 
		.let(::BufferedInputStream) 
		.let(::ZipInputStream) 
		.let(::ObjectInputStream) 
		.readObject() as SomeObject
```

나는 개인적으로 위 코드를 알아보기 힘들다고 생각했는데 책에서는 경험이 적은 코틀린 개발자는 이해하기 어려운 것이 맞아 비용이 발생하는 것까지는 인정해주지만.. 이 비용은 지불할 만한 가치가 있으므로 사용해도 괜찮다고 한다. 

<br/>

### 컨벤션

뒷장부터 프로그래머들이 지키는 컨벤션에 대해 알아보게 됨.

# item 4. inffered type으로 리턴하지 말라

### kotlin의 타입 추론

코틀린의 타입 추론은 할당 때 inferred 타입은 정확하게 오른쪽에 있는 피연산자에 맞게 설정됨. 슈퍼클래스나 인터페이스의 타입으로 설정되지 않음. 

```kotlin
open class Animal
class Zebra: Animal()

fun main() {
    var animal = Zebra()
    animal = Animal() //  타입 미스매치 error 
		// 위의 animal 변수 선언 시 타입을 Animal로 명시해주면 에러 발생하지 않음. 
}
```

<br/>

### 결론

타입을 확실하게 지정해야 하는 경우에는 명시적으로 타입을 지정하자.

외부 API를 만들 때는 더더욱 명시적으로 타입을 지정하는 것이 좋음.

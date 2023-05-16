# item 3. 최대한 플랫폼 타입을 사용하지 말라

null-safety는 코틀린의 주요 기능 중 하나.

자바에서 String 타입을 리턴하는 메서드가 있을 때 @Nullable이 붙어있다면 String?으로 타입을 정하고 @NotNull 어노테이션이 붙어있다면 String으로 변경하면 됨. 그러나 어노테이션이 붙어있지 않을 때 처리가 곤란해짐. → 모든 타입을 nullable로 다루는 것도 방법임.

하지만 자바에서 List<User>를 리턴하는데 어노테이션이 붙어있지 않은 경우 모든 타입을 Nullable로 다룬다고 하면 아래 두 가지를 체크해주어야 함.

- List 자체가 null?
- List 내부의 User 객체들이 null?

**platform type**? → 코틀린은 다른 프로그래밍 언어에서 넘어온 nullable인지 알 수 없는 타입들을 platform type이라고 부름.

  <br/>
  
### 플랫폼 타입이 안전하지 않은 이유

첫 번째는 자바에서 값을 가져오는 시점에 NPE가 발생하고 두 번째는 값을 활용할 때 NPE가 발생함. 두 번째는 플랫폼 타입으로 변수를 지정했기에 nullable일 수도 아닐 수도 있음. 따라서 다른 개발자가 해당 변수를 사용할 때 오류가 발생한다면 찾는 데 오래 걸릴 수 있음.

```kotlin
public class JavaClass {
    public String getValue() {
        return null;
    }
}

fun staredType() {
    val value: String = JavaClass().value // NPE
    println(value.length)
}

fun platformType() {
    val value = JavaClass().value
    println(value.length) // NPE
}
```

  <br/>
  
### 결론

플랫폼 타입을 사용하는 코드는 이를 활용하는 곳까지 영향을 줄 수 있는 위험한 코드임. 따라서 제거하거나 연결된 자바 코드에 nullable 여부를 어노테이션으로 지정하는 것도 좋은 방법임.

# item 8. 적절하게 null을 처리하라

### nullable 타입 처리

- ?., 스마트캐스팅, Elvis 연산자 등을 활용해서 안전하게 처리함.
    - 엘비스 연산자는 오른쪽에 return or throw를 포함한 모든 표현식이 가능함.
- 오류를 throw함.
    - 오류를 강제로 발생시킬 때는 throw, !!, requireNotNull, checkNotNull 등을 활용하면 됨.
- 원래 코드를 리팩토링하여 nullable 타입이 나오지 않도록 함.

<br/>

### 방어적 프로그래밍 vs 공격적 프로그래밍

- 방어적 프로그래밍
    - 모든 가능성을 올바른 방식으로 처리하는 것. 코드가 프로덕션 환경으로 들어갔을 때 발생할 수 있는 수많은 것들로부터 프로그램을 방어해서 안정성을 높이는 방법을 나타내는 포괄적인 용어.
- 공격적 프로그래밍
    - 예상하지 못한 상황이 발생했을 때 이러한 문제를 개발자에게 알려서 수정하게 만드는 것. require, check, assert가 이런 공격적 프로그래밍을 위한 도구.
- 방어적 프로그래밍과 공격적 프로그래밍은 충돌하는 것이 아니고 둘 모두 코드의 안전을 위해 필요함.

<br/>

### not null assertion(!!)과 관련한 문제

- nullable을 처리하는 가장 간단한 방법이지만 좋은 해결 방법은 아님.
- !!은 타입이 nullable이지만 null이 나오지 않는다는 것이 거의 확실한 상황에서 많이 사용됨.
- 그냥 !! 쓰지마라!

<br/>

### 의미 없는 nullability 피하기

nullability는 어떻게든 적절하게 처리해야 하므로, 추가 비용이 발생함. 따라서 필요한 경우가 아니라면 nullability 자체를 피하는 것이 좋음. nullability를 피할 수 있는 방법은 아래와 같음. 

- 어떤 값이 클래스 생성 이후에 확실하게 설정된다는 보장이 있다면 lateinit property와 notNull delegate를 사용하라.
- 빈 컬렉션 대신 null을 리턴하지 말아야 함. 요소가 부족하다는 것을 나타내려면 빈 컬렉션을 사용해야 함.

<br/>

### lateinit property, notNull delegate

`private var dao: UserDao? = null` 과 같이 변수를 선언했다가 이 프로퍼티를 사용할 때마다 nullable에서 null이 아닌 것으로 타입 변환을 해주는 것은 바람직하지 않음. 왜냐하면 이 값이 만약 클래스 초기에 무조건 초기화 된다고 보장할수 있다면 의미없는 코드의 반복이기 때문. 이럴 때 `lateinit` 한정자를 사용하면 좋음. 

#### lateinit

- 프로퍼티가 이후에 설정될 것임을 명시하는 한정자.
- 초기화 전에 값을 사용하려고 하면 예외가 발생함.
- !! 연산자로 unpack하지 않아도 됨.
- 이후에 어떤 의미를 나타내기 위해 null을 사용하고 싶을 때 nullable로 만들수 있음.
- 프로퍼티가 초기화된 이후에는 초기화되지 않은 상태로 돌아갈 수 없음.
- primitive type으로 초기화하고 싶은 경우에는 사용 불가능함.

#### notNull delegate

- JVM에서 Int, Long, Double, Boolean과 같은 primitive type과 연결된 타입으로 프로퍼티를 초기화해야 하는 경우 Delegates.notNull을 사용함.

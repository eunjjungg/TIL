## Hilt

<br/>

### Dependency

하나의 클래스가 다른 하나의 클래스에 의존하는 것을 의미함. 
A 클래스가 객체를 만들기 위해 B 클래스를 필요로 할 때 B는 A의 의존 대상이 됨. 예를 들어 샐러드를 만들기 위해 드레싱이 필요함. 샐러드는 드레싱에 의존하고 있음. 따라서 샐러드 ← 드레싱 관계가 됨. 여기서 드레싱이 **의존의 대상, Dependency**가 됨. 

<br/>

### Injection

위의 예를 계속 사용해서 생각해보면 샐러드 클래스, 드레싱 클래스가 있을 때 샐러드 객체를 만들기 위해서는 드레싱의 객체가 필요함. 하지만 이 드레싱 객체를 샐러드 클래스 내부에서 만드는 것이 아니라 외부에서 만들어서 건네주는 방법도 있음. 이렇게 외부에서 의존관계가 있는 대상을 가져오는 것을 의존성(Dependency)을 외부에서 주입(Inject)해준다고 함. 따라서 드레싱의 종류가 발사믹 드레싱이었는데 요거트 드레싱으로 바뀐다고 샐러드 내부의 코드를 수정할 필요 없이 주입만 받으면 됨. → Test에 편해지기도 함. 

<br/>

### DI : Dependency Injection

중요한 요지는 의존성이 있는 클래스 내부에서 의존의 대상이 되는 객체를 직접 생성하는 것이 아니라 외부에서 넘겨받아야 함. 그렇다면 그 방법으로는 constructor에서 넘겨주거나, setter 메소드를 사용해서 넘겨주는 방식이 있음. 안드로이드에서 사용하는 대표적인 DI 라이브러리로는 Koin, Dagger, Hilt가 있음. 이중 Hilt가 학습 곡선이 상대적으로 낮은 편이라고 해서 Hilt에 대해 공부할 예정임. 

의존성 주입을 했을 때 장점으로는 다음과 같음. 

- 샐러드를 만드는 쪽과 드레싱을 만드는 쪽, 즉 의존성이 있는 클래스와 의존 대상이 분리되어 있으므로 의존성이 있는 클래스를 테스트할 때 모의객체를 사용하면 더 쉽게 테스트가 가능함.
- 의존 대상과 의존성이 있는 쪽이 각각 독립되어 있기 때문에 결합도를 낮추고 유지보수에 용이함.
- 코드의 재사용성이 증가함.

<br/>

### Hilt

구글의 Dagger를 기반으로 만든 DI 라이브러리. 

> Hilt는 Android 애플리케이션의 구조를 단순화하고, 유지 관리 및 확장성을 향상시키며, 종속성 주입 프레임워크를 사용하여 런타임 오버헤드를 줄일 수 있도록 도와줍니다.
> 

<br/>

### 시작하기

build.gradle에 종속성 추가

<br/>

### HiltAndroidApp

Hilt 사용하는 모든 앱은 @HiltAndroidApp Annotation으로 지정된 Application Class를 포함해야만 함. @HiltAndroidApp은 Hilt의 코드 생성을 트리거함. annotation으로 생성된 Hilt 구성요소는 Application 객체의 수명주기에 연결됨. ApplicationContext에 Hilt가 접근할 일이 많기 때문에 이 과정이 필요함. Application Class를 사용하지 않다가 추가했다면 AndroidManifest.xml에 application 태그에 추가해주어야 함. 

```kotlin
@HiltAndroidApp
class MainApplication : Application() {
}
```

<br/>

### Hilt가 지원하는 Android Class

- Application (@HiltAndroidApp 사용)
- ViewModel (@HiltViewModel 사용)
- Activity (@AndroidEntrypoint 사용)
- Fragment (@AndroidEntrypoint 사용)
- View (@AndroidEntrypoint 사용)
- Service (@AndroidEntrypoint 사용)
- BroadcastReceiver (@AndroidEntrypoint 사용)

<br/>

### Hilt가 생성하는 Component의 계층 구조

![hilt-hierarchy](https://github.com/eunjjungg/TIL/assets/100047095/8589f41c-5b7a-42b3-a748-de9b41fb3331)

최상위는 ApplicationClass를 사용하는 SingletonComponent가 됨. 각 Component들은 그 부모로부터 Dependency를 받을 수 있음. 

<br/>

### AndroidEntryPoint

안드로이드 클래스에 @AndroidEntryPoint Annotation을 사용하면 이 클래스에 종속된 다른 클래스에도 해당 Annotation을 사용해야 함. @AndroidEntryPoint는 프로젝트의 각 Android 클래스에 대한 개별 Hilt 구성요소를 생성함

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

    }
}
```

<br/>

### Inject

@Inject 주석을 사용하여 필드 삽입을 함. Hilt가 삽입한 필드는 private이 불가능함. @Inject를 사용해서 Constructor Injection도 가능함. 

```kotlin
@AndroidEntryPoint
class ExampleActivity : AppCompatActivity() {

  @Inject lateinit var analytics: AnalyticsAdapter
  ...
}
```

<br/>

### Hilt Inject 더 자세하게 - 1

필드에 외부 객체를 주입해주려면 Hilt가 어떻게 인스턴스를 제공하는지 알아야 함. binding에는 특정 유형의 인스턴스를 dependency로 제공하는데 필요한 info가 들어있음. 클래스의 constructor에 @Inject 주석을 사용해서 Hilt에 알려주어야 함. 

```kotlin
class AnalyticsAdapter @Inject constructor(
  private val service: AnalyticsService
) { ... }
```

@Inject 주석이 지정된 클래스의 생성자 매개변수는 그 클래스의 종속 항목임. 위의 코드에서는 AnalyticsAdapter 클래스의 생성자에 @Inject annotation이 있고 이 생성자는 AnalyticsService가 종속 항목으로 있음. 그래서 Hilt에게 AnalyticsAdapter 객체를 생성할 때 AnalyticsService 객체도 생성하여 주입해주어야 함을 알려줌. 위와 같이 Constructor Injection을 사용할 경우에는 다음과 같은 장점이 있음. Constructor Injection은 생성 시 어떤 클래스의 컴포넌트가 필요한지 명확하게 알 수 있음. → 가독성에 좋음. 

또한 위의 AnalyticsAdapter 클래스는 Component들에서 사용이 가능함. 예시로는 @AndroidEntryPoint annotation을 붙인 Activity에서 사용이 가능함. 아래 코드는 Constructor Injection이 아닌 Field Injection을 사용한 방법임. 다시 한 번 Field Injection은 접근 제한자를 **private**으로 선언하면 안 됨! 

```kotlin
@AndroidEntryPoint
class ActivitySample : AppCompatActivity() {
		@Inject
		lateinit var analyticsAdapter: AnalyticsAdapter
		
		...
}
```

<br/>

❓ 내가 헷갈린 부분 : @Inject Annotation은 그럼 주입을 받을 대상에 붙이는게 맞나? → No 🙅‍♀️

> @Inject 어노테이션은 주입을 받을 대상과 주입을 제공하는 대상 모두에게 붙여줘야 합니다.
> 
> 
> 보통 클래스의 생성자에서 필요한 의존성을 주입받을 때 @Inject 어노테이션을 사용합니다. 이때 생성자 매개변수에 @Inject 어노테이션이 붙은 경우 해당 클래스를 사용하는 다른 클래스에서 인스턴스를 생성할 때, 의존성을 주입할 수 있도록 Dagger나 Hilt와 같은 프레임워크가 자동으로 의존성을 주입합니다.
> 
> 따라서, 주입을 받을 대상의 생성자 매개변수에 @Inject 어노테이션을 붙여줘야 하고, 의존성을 제공하는 대상(예를 들어, 다른 클래스나 모듈)의 메서드 또는 필드에도 @Inject 어노테이션을 붙여줘야 합니다. 이를 통해 프레임워크가 의존성을 주입할 대상을 알아내고, 자동으로 의존성을 주입할 수 있도록 합니다.
> 

또한 @Inject는 Dependency Graph를 이어준다고 함. Hilt가 inject annotation이 붙어있는 :: Dependency를 주입 받을 객체와 Dependency를 제공해서 생성할 객체의 클래스에 Dependency Graph를 이어붙여줌. 

<br/>

### Hilt 모듈

constructor로 주입을 제공할 수 없는 경우 Hilt 모듈을 사용해서 Hilt에 binding info를 제공함. constructor로 주입을 제공할 수 없는 경우는 다음과 같음.

- 인터페이스
    - A interface를 B Class, C Class에서 implement해서 구현했다고 가정했을 때 각 클래스의 생성자에 A interface를 넣어주면 안됨. 에러 발생.
- 외부 라이브러리 클래스

따라서 위와 같은 경우 Hilt 모듈을 사용해야 함. Hilt 모듈은 @Module로 Annotation이 지정된 클래스임. 또한 **@InstallIn 주석을 지정하여 각 모듈을 사용할 안드로이드 클래스를 Hilt에 알려주어야 함**. Hilt 모듈에 제공한 dependency들은 Hilt 모듈을 설치한 Android Class의 모든 구성요소에서 사용 가능. 즉 **@Module annotation은 Hilt에게 이 클래스가 Module이 있는 곳이라고 알려주는 것이고 @InstallIn annotation은 해당 모듈이 activity와 같은 Hilt 모듈을 설치한 클래스에서 사용 가능하다고 선언하는 의미**임.

Module을 사용하는 방식으로는 Provides Annotation과 Binds Annotation이 있음. Provides 방식이 더 편하다고 하고 임의의 module class를 생성하고 여기에 클래스들 작성하는 식으로 사용한다고 함. 

<br/>

### @Provides

일단 Porvides Annotation 방식으로 Module을 생성한다고 해도 @Module, @InstallIn annotation을 붙여주어야 함. 그리고 모듈 클래스 내부에 @Provides annotation을 사용해서 Provides 함수들을 넣어주어야 함. 

[https://developer88.tistory.com/349](https://developer88.tistory.com/349) 이 블로그를 보고 따라한 예시는 다음과 같음. 

첫 번째 코드에 대한 설명 : ClassBForInterfaceTest 클래스는 InterfaceA를 Implements한 클래스이고 따라서 showString() method를 override함. 이 메소드에서는 stringDependency라는 변수를 포함하는 스트링을 리턴하는데 이 stringDependency 변수는 ClassBForInterfaceTest 클래스가 생성시 주입되도록 생성자에서 @Inject 키워드를 사용해 선언해놓았음. 그리고 ClassAForInterfaceTest 클래스는 생성 시 InterfaceA를 주입받도록 클래스 생성자에 @Inject 키워드를 사용해 선언해놓음. 

두 번째 코드에 대한 설명 : ClassBForInterfaceTest는 인터페이스를 주입받아야 함. 따라서 Module을 생성해주어야 하므로 @Module, @InstallIn annotation을 사용함. @InstallIn annotation을 사용해 이 모듈이 사용되는 컴포넌트를 ActivityComponent로 지정해줬음. → 이 말은 즉 Activity에서 이 의존성을 사용하겠다고 말하는 것임. 따라서 Activity class에서 @Inject를 사용해서 의존성을 주입해줄 수 있게 됨. 

```kotlin
interface InterfaceA {
    fun showString(): String
}

class ClassAForInterfaceTest @Inject constructor(private val bClass: InterfaceA) {
    fun doTestA(): String {
        return bClass.showString()
    }
}

class ClassBForInterfaceTest
@Inject constructor(private val stringDependency: String) : InterfaceA {
    override fun showString(): String {
        return "stringDependency : ${stringDependency}"
    }
}
```

```kotlin
@Module
@InstallIn(ActivityComponent::class)
class ModuleForInterface {
    @Provides
    fun provideString(): String {
        return "dependency String by ModuleForInterface"
    }

    @Provides
    fun testProvides(cString: String): InterfaceA {
        return ClassBForInterfaceTest(cString)
    }
}
```

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var classAForInterfaceTest : ClassAForInterfaceTest

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val text = classAForInterfaceTest.doTestA()
        println(text)
    }
}
```

<br/>

❓궁금해서 해본 것 : @Module에서 string 타입을 주입해주고 있는데 String을 주입해주는 함수가 여러개라면..? → 뭘 provide 해줄지 모르므로 오류 발생

```kotlin
@Module
@InstallIn(ActivityComponent::class)
class ModuleForInterface {
    @Provides
    fun provideString(): String {
        return "dependency String by ModuleForInterface"
    }

    @Provides
    fun provideString2(): String {
        return "dependency String 2 by ModuleForInterface"
    }

    @Provides
    fun testProvides(cString: String): InterfaceA {
        return ClassBForInterfaceTest(cString)
    }
}
```

<br/>

❓궁금해서 해본 것 2 : @Module에서 string 타입을 주입해주고 있는데 String을 주입해주는 함수가 여러개인만큼 받는 것도 있다면..? → 동일하게 뭘 provide 해줄지 모르므로 오류 발생. 하지만 타입이 다른 것은 ok여요! (아래 코드에서 Int는 Ok)

```kotlin
@Module
@InstallIn(ActivityComponent::class)
class ModuleForInterface {
    @Provides
    fun provideString(): String {
        return "dependency String by ModuleForInterface"
    }

    @Provides
    fun provideString2(): String {
        return "dependency String 2 by ModuleForInterface"
    }

    @Provides
    fun provideInt(): Int {
        return 1
    }

    @Provides
    fun testProvides(cString: String, cString2: String, num: Int): InterfaceA {
        Log.d("num", "$num")
        Log.d("string", "$cString2")
        return ClassBForInterfaceTest(cString)
    }
}
```

하지만 위에 궁금해서 해봤다고 한 건 역시 방법이 있었음. 바로 @Qualifier, @Retention annotation을 사용하면 됨. 이 방법을 한정자 주입이라고 한다고 함. 

<br/>

### @Qualifier, @Retention

> **AI가 설명해준 코드**
아래 코드는 동작합니다. 그 이유는 `QualifierTest` 모듈이 `SingletonComponent`에 `@InstallIn` 되었기 때문에 `QualifierInterfaceType1`과 `QualifierInterfaceType2` 어노테이션을 가진 `Provides` 메서드가 `Singleton` 범위 내에서 유일하게 존재합니다. 이렇게 한정자를 사용하여 의존성을 구분하는 방식을 "한정자 주입"이라고 합니다.
> 
> 
> 따라서, `QualifierTestClassA` 생성자에서 `QualifierInterfaceType1`과 `QualifierInterfaceType2` 한정자를 이용해 각각의 구현체를 주입받을 수 있습니다. 이를 통해, `QualifierTestClassA`에서 `concatString()` 메서드를 호출하면 `QualifierTestClassB`와 `QualifierTestClassC`가 구현한 `showString()` 메서드의 결과를 합친 문자열이 반환됩니다.
> 

```kotlin
// Module

@Module
@InstallIn(SingletonComponent::class)
class QualifierTest {

    @Qualifier
    @Retention(AnnotationRetention.BINARY)
    annotation class QualifierInterfaceType1

    @Qualifier
    @Retention(AnnotationRetention.BINARY)
    annotation class QualifierInterfaceType2

    @QualifierInterfaceType1
    @Provides
    fun testProvides1(): QualifierInterface {
        return QualifierTestClassB()
    }

    @QualifierInterfaceType2
    @Provides
    fun testProvides2(): QualifierInterface {
        return QualifierTestClassC()
    }
}
```

```kotlin
// Class 

class QualifierTestClassA @Inject constructor(
    @QualifierTest.QualifierInterfaceType1 private val qInterfaceType1: QualifierInterface,
    @QualifierTest.QualifierInterfaceType2 private val qInterfaceType2: QualifierInterface
){
    fun concatString(): String{
        return qInterfaceType1.showString() + qInterfaceType2.showString()

    }
}

class QualifierTestClassB @Inject constructor(): QualifierInterface {
    override fun showString(): String {
        return "QualifierTestClassB"
    }
}

class QualifierTestClassC @Inject constructor(): QualifierInterface {
    override fun showString(): String {
        return "QualifierTestClassC"
    }

}

interface QualifierInterface {
    fun showString(): String
}
```

```kotlin
// Activity

@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    @Inject
    lateinit var qualifierTestClassA: QualifierTestClassA

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // qualifier test
        val text = qualifierTestClassA.concatString()
        println(text)
    }
}
```

<br/>

### @Binds

@Binds 주석은 인터페이스의 인스턴스를 제공해야할 때 사용할 구현을 Hilt에 알려줌. 그러나 @Binds 방식은 외부 라이브러리에는 사용할 수 없음. 

```kotlin
class BindTestClassA @Inject constructor(
    private val bindTestClassB: BindInterface
){
    fun testBindTestClassA(): String {
        return bindTestClassB.showString()
    }
}

class BindTestClassB @Inject constructor(): BindInterface{
    override fun showString(): String {
        return "BindTestClassB"
    }
}

interface BindInterface {
    fun showString(): String
}
```

```kotlin
@Module
@InstallIn(ActivityComponent::class)
abstract class ModuleForBind {
    @ActivityScoped
    @Binds
    abstract fun bindInterfaceDependency(bindTestClassB: BindTestClassB): BindInterface
}
```

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var bindTestClassA: BindTestClassA

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val text = bindTestClassA.testBindTestClassA()
        println(text)
    }
}
```

> **AI가 설명해준 @Module 부분 코드**
> 
> 
> @Module 어노테이션이 붙은 클래스는 Hilt에서 의존성 주입을 수행할 때 필요한 바인딩 정보를 제공하는 역할을 합니다. 바인딩 정보란 어떤 클래스를 어떤 인터페이스나 추상 클래스에 바인딩할 것인지를 정의한 것을 의미합니다.
> 
> @Module 어노테이션에는 @InstallIn 어노테이션을 함께 사용하여 이 모듈이 어떤 컴포넌트에 설치되어야 하는지를 지정합니다. 이번 예시에서는 ActivityComponent에 설치됩니다.
> 
> @Module 어노테이션이 붙은 클래스 내부에는 @Provides, @Binds, @Multibinds 등의 메서드를 사용하여 바인딩 정보를 제공합니다. 이 중에서 @Binds 어노테이션은 **추상 클래스**나 인터페이스를 구현하는 클래스를 바인딩할 때 사용합니다.
> 
> @Module 어노테이션이 붙은 클래스인 ModuleForBind 내부에는 다음과 같은 코드가 있습니다.
> 
> ```kotlin
> @Binds
> abstract fun bindInterfaceDependency(bindTestClassB: BindTestClassB): BindInterface
> ```
> 
> 이 코드는 BindTestClassB 클래스가 구현하는 BindInterface 인터페이스를 바인딩하는 코드입니다. 즉, BindTestClassB 객체가 BindInterface 타입으로 주입될 수 있도록 Hilt에게 바인딩 정보를 제공하는 것입니다.
> 
> 그리고 이제 MainActivity에서 BindTestClassA를 주입받아 사용할 때, Hilt는 BindTestClassB 객체를 BindInterface 타입으로 주입해줍니다. 이를 통해 BindTestClassA에서는 BindInterface에 정의된 showString() 메서드를 호출할 수 있습니다.
> 

<br/>

<br/>

### ViewModel

@HiltViewModel을 사용하면 됨. 이후 @AndroidEntryPoint 주석으로 지정된 액티비티나 프래그먼트에서 ****by viewModels() 키워드로 뷰모델을 가져올 수 있음. 

```kotlin
@HiltViewModel
class MainViewModel @Inject constructor(
    @ModuleForViewModel.ViewModelIntType1 private val a: Int,
    @ModuleForViewModel.ViewModelIntType2 private val b: Int
): ViewModel() {

    fun getNum(): Int {
        return a + b
    }
}
```

```kotlin
@Module
@InstallIn(ViewModelComponent::class)
class ModuleForViewModel {

    @Qualifier
    @Retention(AnnotationRetention.BINARY)
    annotation class ViewModelIntType1

    @Qualifier
    @Retention(AnnotationRetention.BINARY)
    annotation class ViewModelIntType2

    @ViewModelIntType1
    @Provides
    fun provideA(): Int = 1

    @ViewModelIntType2
    @Provides
    fun provideB(): Int = 3
}
```

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    private val mainViewModel: MainViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val text = mainViewModel.getNum().toShort()
        println(text)
    }
}
```

> **AI가 설명해준 코드**
> 
> 
> 이 코드는 안드로이드의 Hilt 라이브러리를 사용하여 생성된 뷰 모델의 값을 사용하는 예제입니다.
> 
> 우선 @HiltViewModel 어노테이션은 ViewModel에 대한 의존성 주입을 사용하기위한 Hilt-specific 어노테이션입니다.
> 
> MainViewModel 클래스는 @HiltViewModel 어노테이션을 사용하여 뷰 모델임을 표시합니다. 이 클래스는 두 개의 정수 a와 b를 인자로 받습니다. a와 b는 ModuleForViewModel 모듈의 두 개의 어노테이션에 의해 제공되는 @Provides 메서드로 주입됩니다.
> 
> @ModuleForViewModel 모듈에는 뷰 모델에서 필요한 a와 b 값에 대한 제공자가 포함되어 있습니다. 이 모듈은 ViewModelComponent 스코프에 설치되어 있으며 ViewModel에 대한 의존성을 주입하기 위해 사용됩니다.
> 
> 마지막으로 MainActivity 클래스는 Hilt를 사용하여 MainViewModel 객체를 생성합니다. 생성된 MainViewModel 객체의 getNum() 메서드를 호출하여 a와 b를 더하고 결과를 출력합니다.
> 

<br/>

### 미해결 의문

❓hilt를 사용해서 viewmodel에 applicationcontext를 안전하게 제공하는 방법

<br/>


### 레퍼런스

- [https://developer.android.com/training/dependency-injection/hilt-android?hl=ko](https://developer.android.com/training/dependency-injection/hilt-android?hl=ko)
- [https://developer88.tistory.com/172](https://developer88.tistory.com/172)
- [https://developer88.tistory.com/349](https://developer88.tistory.com/349) → 개인적으로 공식 문서와 이 분 블로그 게시글을 보는걸 가장 추천함.
- [https://velog.io/@haanbink/Android-Hilt-사용하기](https://velog.io/@haanbink/Android-Hilt-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0)
- [https://developer.android.com/training/dependency-injection/hilt-jetpack?hl=ko#viewmodelscoped](https://developer.android.com/training/dependency-injection/hilt-jetpack?hl=ko#viewmodelscoped) → Hilt, ViewModel

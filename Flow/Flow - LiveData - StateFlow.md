
## LiveData - Flow - StateFlow

고민되는게 어떤 예시에서는 Flow와 livedata를 같이 씀. 근데 이렇게만 사용할거면 코루틴이랑 다를게 뭔지 의문이 들음. 물론 Flow 나름의 장점이 있는 영역이 있음. 예를 들어 실시간으로 업데이트를 해줘야 할 때라든지… 그러나 그냥 서버로 api 한 번 호출하고 해당 값을 livedata에 넣어주기만 하면 되는 구조라면 뭐가 이득인지 잘 이해가 안 감. 그래서 찾아봤더니 아주 잘 설명한 블로그가 있어서 그 블로그를 보며 이해한 내용을 적어봄. 

🔗 링크 : https://onlyfor-me-blog.tistory.com/557

🔗 링크 : https://readystory.tistory.com/207

<br/>

### LiveData

LiveData는 Observable data holder class. 생명주기를 인식함. → (다른 앱 구성요소(Activity, Broadcast Receiver, Content Provider, Service)의 생명주기를 고려함.) 이를 observing하고 있는 객체에게만 업데이트 시 그 정보를 알림. 단 그 관찰자가 active 상태여야만 함. 

데이터 영역 클래스에서 LiveData를 사용해서 바로 데이터를 넣어주면 flow가 필요한 이유도 없어짐. 그러나 LiveData는 비동기 데이터 스트림을 처리하도록 설계되지 않음. 엄밀히 말하면 데이터 스트림 결합 기능을 수행할 수는 있지만 제한적이고 변환을 통해 만들어진 라이브데이터들은 main thread(기본 스레드)에서 관찰된다고 함. 따라서 앱의 다른 레이어에서 데이터 스트림의 값을 사용해야 한다면 Flow를 사용하고 asLiveData()를 사용하는 것이 좋다고 함. 

즉 데이터 관련 작업은 main 스레드가 아닌 worker 스레드에서 작동되어야 하는데 라이브 데이터로만 작업하면 이 방식이 불가능함. 

또한 클린 아키텍처 관점에서도 이 방식이 좋지 않음. 왜냐하면 Domain Layer는 Android Component에 dependency가 없어야 함. 그러나 멀티 모듈 사용시 LiveData로 구현하려고 한다면 해당 domain 모듈에 android component인 livedata의 Import가 필요함. 

<br/>

### LiveData - Flow

![image](https://github.com/eunjjungg/TIL/assets/100047095/1adb8d0e-26c2-4495-87eb-af7087144fae)

> LiveData와 Flow는 서로를 보완하는 역할을 한다. LiveData는 구성 변경 시에 안정성을 제공하고 최신 데이터를 뷰로 전달하는 역할을 한다. Flow는 UseCase, Repository, DataSources Layer와 긴밀하게 작동해 데이터를 수집 및 처리해서 서로 다른 코루틴 범위에서 작업을 실행한다. 그래서 ViewModel, View 사이의 상호작용은 LiveData, 더 깊은 레이어와 쓰레딩 같은 더 복잡한 처리는 Flow가 처리한다.
> 

왜 복잡한 처리만 Flow가 해야하는가?

- Flow는 스스로 안드로이드 생명주기를 알지 못함. 따라서 생명주기에 따른 정지와 restart가 어려움.
- Flow는 상태가 없어 값이 할당됐는지 현재 값은 무엇인지 알기 어려움. 그냥 받고 넘겨주기만 하는 역할.
- Flow는 콜드스트림 방식. **연속해서 계속 들어오는 데이터를 처리할 수 없고 collect 되었을 때만 생성되고 값을 반환한다.** 만약 하나의 Flow Builder에 대해 다수의 collector가 있다면 collector 하나마다 하나씩 데이터를 호출하기 때문에 비용이 비싼 DB 접근, 서버 통신 등을 수행한다면 여러 번 리소스 요청을 하게 될 수 있다. → 이 부분 그냥 복붙해왔는데 이해가 안됨. 그래서 왜 이게 단점인지 모르겠음.

<br/>

### StateFlow

StateFlow는 현재 상태와 새로운 상태 업데이트를 Collector에 내보내는 observable status holder flow임. value 속성을 통해 현재 상태 값을 읽을수도 있음. 상태를 업데이트하고 Flow에 emit하려면 `MutableStateFlow` 클래스의 `value` property에 새 값을 할당하면 됨. 

StateFlow는 관찰 가능한 변경 가능 상태를 유지해야 하는 클래스에 적합함. like ViewModel… 아래 코드가 그 예시인데 코멘트 하나하나 주옥같음. mutable한 건 숨기는 것부터 시작해서  `_uiState.value`에 collect한 값 넣어주는 것까지.. 

```kotlin
class LatestNewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {

    // Backing property to avoid state updates from other classes
    private val _uiState = MutableStateFlow(LatestNewsUiState.Success(emptyList()))
    // The UI collects from this StateFlow to get its state updates
    val uiState: StateFlow<LatestNewsUiState> = _uiState

    init {
        viewModelScope.launch {
            newsRepository.favoriteLatestNews
                // Update View with the latest favorite news
                // Writes to the value property of MutableStateFlow,
                // adding a new element to the flow and updating all
                // of its collectors
                .collect { favoriteNews ->
                    _uiState.value = LatestNewsUiState.Success(favoriteNews)
                }
        }
    }
}

// Represents different states for the LatestNews screen
sealed class LatestNewsUiState {
    data class Success(val news: List<ArticleHeadline>): LatestNewsUiState()
    data class Error(val exception: Throwable): LatestNewsUiState()
}
```

MutableStateFlow의 업데이트를 담당하는 클래스가 producer, stateFlow에서 collect하는 모든 클래스가 consumer임. stateFlow는 hot stream이라고 함. stream에서 수집해도 생산자 코드가 트리거 되지 않음. 

```kotlin
class LatestNewsActivity : AppCompatActivity() {
    private val latestNewsViewModel = // getViewModel()

    override fun onCreate(savedInstanceState: Bundle?) {
        ...
        // Start a coroutine in the lifecycle scope
        lifecycleScope.launch {
            // repeatOnLifecycle launches the block in a new coroutine every time the
            // lifecycle is in the STARTED state (or above) and cancels it when it's STOPPED.
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                // Trigger the flow and start listening for values.
                // Note that this happens when lifecycle is STARTED and stops
                // collecting when the lifecycle is STOPPED
                latestNewsViewModel.uiState.collect { uiState ->
                    // New value received
                    when (uiState) {
                        is LatestNewsUiState.Success -> showFavoriteNews(uiState.news)
                        is LatestNewsUiState.Error -> showError(uiState.exception)
                    }
                }
            }
        }
    }
}
```

StateFlow와 LiveData의 차이점

- StateFlow의 경우 초기 상태를 생성자에게 전달해야 하지만 LiveData는 전달하지 않음.
- View가 Stopped 상태가 되면 LiveData.observe()는 소비자를 자동으로 등록 취소하지만 StateFlow는 다른 Flow에서 수집하고 있다면 자동으로 중지하지 않음.

<br/>

### StateFlow build

```kotlin
class ExViewModel(): ViewModel() {
		val result: StateFlow<Result<Item>> = flow {
        emit(repository.getItem())
    }.stateIn(
        scope = viewModelScope, 
        started = WhileSubscribed(5000), // Or Lazily because it's a one-shot
        initialValue = Result.Loading
    )
}
```

🔗 링크 : https://readystory.tistory.com/207

위 코드는 stateIn을 사용해서 Flow를 stateFlow로 변환하는 코드임. stateIn 함수의 파라미터로 아래와 같은 항목이 필요함.

- scope : 공유가 시작되는 CoroutineScope
- started : 공유가 시작 및 중지되는 시기를 제어하는 전략을 설정하는 파라미터
    - Lazily : 첫 번째 subscriber가 나타나면 시작하고 scope가 취소되면 중지함.
    - Eagerly : 즉시 시작되며 scope가 취소되면 중지
    - WhileSubscribed : collector가 없을 때 upstream flow를 취소함. 앱이 백그라운드로 전환되면 취소하게 하는 등의 전략이 가능하다고 함…
    
    <br/>

### Livedata 대비 StateFlow의 이점

1. StateFlow는 순수 kotlin lib임. 따라서 Android component에 종속되면 안 되는 domain layer에서 사용이 가능함.
2. StateFlow는 zip, flatMapMerge 등 다양한 Flow API를 사용할 수 있어서 다양한 활용이 가능하다고 함.

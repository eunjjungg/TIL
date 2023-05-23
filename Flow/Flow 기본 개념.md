# Flow

Coroutine의 Flow는 데이터 스트림이며 코루틴 상에서 reactive programming을 지원하기 위한 구성요소임. 

- 리액티브 프로그래밍이란?
    - 데이터가 변경될 때 이벤트를 발생시켜서 데이터를 계속해서 전달하도록 하는 프로그래밍 방식.
    - 발행자가 데이터를 발행시키면 이를 구독하는 구독자가 처리를 하는 구조.
    - 기존 명령형 프로그래밍에서 소비자는 데이터를 요청한 후 받은 결과값을 일회성으로 수신함. 그러므로 데이터가 필요할 때마다 매번 요청해야 한다는 점에서 비효율적. 그러나 리액티브 프로그래밍에서는 발행자가 있고 발행하는 데이터를 구독하는 소비자가 있음. 따라서 새로운 데이터가 들어오면 데이터의 소비자에게 지속적으로 발행함. → 이를 데이터 스트림이라고 함.

<br/>

## Flow in Android Developers

필요하다고 생각하는 것 위주로 가져오기. 

<br/>

### 개념

flow는 단일 값만 반환하는 suspend fun과는 달리 여러 값을 순차적으로 내보낼 수 있음. flow는 코루틴을 기반으로 빌드되며 비동기식으로 계산할 수 있는 데이터 스트림의 개념임. 데이터 스트림에는 아래의 세 항목이 포함됨.

- 생산자 : 스트림에 추가되는 데이터를 생산. 코루틴으로 flow에서 비동기적으로 데이터를 생산할 수 있음.
- 중개자 [opt] : 스트림에 내보내는 각각의 값이나 스트림 자체를 수정할 수 있음.
- 소비자 : 스트림의 값을 사용함.

![image](https://github.com/eunjjungg/TIL/assets/100047095/2b98832d-5e5e-4a6f-b838-d72b087b7434)

<br/>

### flow 만들기

flow를 만드려면 flow builder API를 사용해야 함. 

```kotlin
fun <T> flow(block: suspend FlowCollector<T>.() -> Unit): Flow<T>
```

위 함수가 flow builder 함수인데 emit 함수를 사용해서 새 값을 수동으로 데이터 스트림에 내보낼 수 있는 새 flow를 만듦. 

```kotlin
abstract suspend fun emit(value: T)
// Collects the value emitted by the upstream. 
// This method is not thread-safe and should not be invoked concurrently.
```

위 함수는 emit 함수임. 

아래 코드는 데이터소스에서 `refreshIntervalMs` 간격 단위로 뉴스를 가져옴. 

```kotlin
class NewsRemoteDataSource(
    private val newsApi: NewsApi,
    private val refreshIntervalMs: Long = 5000
) {
    val latestNews: Flow<List<ArticleHeadline>> = flow {
        while(true) {
            val latestNews = newsApi.fetchLatestNews()
            emit(latestNews) // Emits the result of the request to the flow
            delay(refreshIntervalMs) // Suspends the coroutine for some time
        }
    }
}

// Interface that provides a way to make network requests with suspend functions
interface NewsApi {
    suspend fun fetchLatestNews(): List<ArticleHeadline>
}
```

**flow의 제한사항? (: 솔직히 이해 안 됨)**

- flow는 순차적임. 생산자가 코루틴에 있으므로 suspend 함수를 호출하면 생산자는 suspend 함수가 반환될 때까지 정지 상태로 유지됨. 위 예에서 생산자는 `fetchLatestNews`가 완료될 때까지 정지했다가 완료되면 그 결과를 반환함.
- flow 빌더에서 생산자가 다른 CoroutineContext의 값을 emit할 수 없음.

<br/>

### stream 수정

중개자(Intermediary)는 중간 연산자를 이용해서 값을 소비하지 않고도 데이터 스트림을 수정할 수 있음. 이 연산자는 데이터 스트림에 사용되면 값이 향후에 사용될 때까지 실행되지 않을 작업 체인을 설정하는 함수임. 아래 예에서는 중간 연산자 map을 사용해 데이터가 view에 표시되도록 변환함.

```kotlin
class NewsRepository(
    private val newsRemoteDataSource: NewsRemoteDataSource,
    private val userData: UserData
) {
    /**
     * Returns the favorite latest news applying transformations on the flow.
     * These operations are lazy and don't trigger the flow. They just transform
     * the current value emitted by the flow at that point in time.
     */
    val favoriteLatestNews: Flow<List<ArticleHeadline>> =
        newsRemoteDataSource.latestNews
            // Intermediate operation to filter the list of favorite topics
            .map { news -> news.filter { userData.isFavoriteTopic(it) } }
            // Intermediate operation to save the latest news in the cache
            .onEach { news -> saveInCache(news) }
}
```

<br/>

### Collect

스트림의 모든 값을 가져오려면 collect 함수를 사용함. 이는 터미널 연산자임. collect는 suspend fun이므로 코루틴 내부에서 실행해야 함. 아래 예시는 viewmodel에서 newsRepository를 사용해 데이터 스트림의 데이터를 collect하는 코드임. producer는 while(true) loop로 항상 active 상태가 유지되므로 viewmodel이 삭제되어 viewmodelscope가 취소되면 데이터 스트림이 종료됨. 

```kotlin
class LatestNewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {

    init {
        viewModelScope.launch {
            // Trigger the flow and consume its elements using collect
            newsRepository.favoriteLatestNews.collect { favoriteNews ->
                // Update View with the latest favorite news
            }
        }
    }
}
```

<br/>

### Callback 기반 API를 flow로 변환

callbackFlow는 콜백 기반 API를 flow로 변환할 수 있는 flow builder임. 

<br/>

## Reference

- [https://kotlinworld.com/175](https://kotlinworld.com/175)
- [https://developer.android.com/kotlin/flow?hl=ko](https://developer.android.com/kotlin/flow?hl=ko)

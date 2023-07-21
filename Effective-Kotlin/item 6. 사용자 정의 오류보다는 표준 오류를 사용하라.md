# item 6. 사용자 정의 오류보다는 표준 오류를 사용하라

예를 들어 JSON 형식을 파싱하는 라이브러리를 구현한다고 할 때 기본적으로 입력된 JSON 파일의 형식에 문제점이 있다면 사용자가 정의한 JSONParsingException을 발생시키는 것이 좋음. 그러나 사용자 정의 오류는 표준 라이브러리의 오류 중 마땅한 것이 없을 때나 사용하는 것이 좋음. 왜냐 이미 정의된 오류들을 재사용하면 사람들이 API를 더 쉽게 이해할 수 있기 때문. 

<br/>

### 대표적인 예외

- `IllegalArgumentException` : `require`를 사용할 때 `throw` 될 수 있음.
- `IllegalStateException` : `check`를 사용할 때 `throw` 될 수 있음.
- `IndexOutOfBoundsException`
- `ConcurrentModificationException` : 동시수정을 금지했는데 발생했을 때 오류 `throw`.
- `UnsupportedOperationException` : 사용자가 사용하려고 했던 메소드가 현재 객체에서는 사용할 수 없다는 것을 의미함.
- `NoSuchElementException`

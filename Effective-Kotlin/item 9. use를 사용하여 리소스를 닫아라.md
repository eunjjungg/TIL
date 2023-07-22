# item 9. use를 사용하여 리소스를 닫아라

더 이상 사용하지 않을 때 close 메서드를 이용해서 명시적으로 닫아야 하는 리소스가 있음. 아래는 그 예시임.

- InputStream, OutputStream
- java.sql.Connection
- java.io.Reader
- java.new.Socket, java.util.Scanner

이 리소스들은 AutoCloseable을 상속받는 closeable interface를 implementation하고 있음. 이러한 모든 리소스는 최종적으로 시소스에 대한 레퍼런스가 없어질 때 가비지 컬렉터가 처리함. 그러나 굉장히 느리며 그동안 리소스를 유지하는 비용이 많이 들어감. 따라서 필요하지 않을 때 명시적으로 close 메소드를 호출해주는 것이 좋음. 

자바에서 try-finally를 사용했던 것처럼 코틀린에서 use라는 이름의 함수로 사용하면 됨. 아래 부분은 이해가 잘 안 되는데 결론적으로 use를 사용하면 Closeable, AutoCloseable을 구현한 객체를 쉽고 안전하게 처리할 수 있다고 함.

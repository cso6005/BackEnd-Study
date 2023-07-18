# 아이템 9 - try-finally보다는try-with-resources를 사용하라

자원 닫기는 클라이언트가 놓치기 쉽기에 예측할 수 없는 성능 문제로 이어질 수 있다.

전통적인 방법으로 try-finally가 쓰이지만

이 방법보다 try-with-resources를 사용해야 하는 이유에 대해 알아볼 것이다.

- **자원 닫기가 필요한 이유**

아래 코드에서 BufferedReader는 IOException이 발생할 수 있다.

만약 br.readLine() 메소드에서 IOException이 발생하게 된다면, 바로 메서드가 종료되므로

**br.close가 호출되지 않고 스트림이 메모리에 남아**있게 된다.

그렇기 때문에 우린 close 닫기를 필수적으로 보장하게 코드를 짜야 한다.

```java
static String firstLineOfFile(String path) throws IOException {
		BufferedReader br = new BufferedReader(new FileReader((path)));
		String result = br.readLine(); // 여기서 예외 발생하면,
		br.close(); // 자원 닫기 실행 안됨.
		return result;
}
```

# 1. try-finally 방법

finally 블록은 try, catch 블록이 끝난 뒤, 반드시 실행할 로직을 정의해 주는 블록입니다.

즉, 만약 IOException이 발생해도 상위 메소드로 IOException 을 던져 준 뒤,

finally 블록 내 close 를 실행하게 되는 것입니다.

```java
static String firstLineOfFile(String path) throws IOException {
        BufferedReader br = new BufferedReader(new FileReader((path)));
        try {
            return br.readLine();
        } finally {
            br.close(); //br 자원 닫기
        }
    }
```

### try-finally 방법 문제점

### 1. 예외 체크 불가

위 코드 firstLineOfFile 메소드의

try 블록을 실행하던 중, 만약 기기 물리적인 문제가 발생한다고 하자. 

그렇다면, try 블록 내, readLine이 정상적으로 실행되지 못하고 예외를 던지게 되고,

finally 블록의 close 메소드도 예외를 던지며 실패하게 된다.

만약, 상위 메소드에서 해당 예외들을 catch해서 체크해도,

finally 블록의 예외가  try 블록에서 생긴 예외를 삼켜,  finally 블록의 예외만 체크하게 된다.

스택 추적 내역에 try 블록 내 예외 첫 번째 예외에 관한 정보는 남지 않게 되고,

이는 실제 시스템에서 디버깅을 몹시 어렵게 만든다.

즉, try 블록 실행 중 발생한 예외가 최초 원인임에도 예외를 체크할 수 없게 되는 것이다.

물론, 최초 원인 예외를 체크하는 로직을 짤 수 있겠지만, 코드가 더러워지기 때문에 추천하는 방법은 아니다.

### 2. 자원이 둘 이상이라면 코드가 지저분 해지고, 실수가 발생할 가능성이 커진다.

```java
static  void copy(String src, String dst) throws IOException {
        InputStream in = new FileInputStream(src);

        try {
            OutputStream out = new FileOutputStream(dst);
            try {
                byte[] buf = new byte[BUFFER_SIZE];
                int n;
                while ((n = in.read(buf)) >= 0) {
                    out.write(buf, 0, n);
                }
            } finally {
                out.close(); // out 자원 닫기
            }
        } finally {
            in.close();  // in 자원 닫기
        }
    }
```

 

# 2. try-with-rescources 방법

이러한 try-finally의 단점을 보완하기 위해 자바 7 버전부터는 try-with-resources가 도입되었다.

- try-with-resources를 사용하기 위해서는 해당 자원이 AutoCloseable 인터페이스를 구현해야 한다.

```
public interface AutoCloseable {

    void close() throws Exception;
}
```

AutoCloseable 인터페이스는 단순히 close 메서드 하나만을 정의해 놓은 간 인터페이스이며,

자바 라이브러리와 서드파티 라이브러리들의 수많은 클래스와 인터페이스는 이미 AutoCloseable을 구현하거나 확장해두었다.

따라서, 꼭 닫아야 할 자원 클래스를 작성한다면  AutoCloseable을 반드시 구현할 것을 권장한다.

### 장점

1. 짧고 읽기 수월하여, 정확하고 쉽게 자원을 회수할 수 있다.
2. 예외 정보도 훨씬 유용하여, 디버깅하기 좋다.
    
    

### 예시 코드

- **inputString 메서드를 try-with-resources 방식으로 바꾼 코드**

```java
static String firstLineOfFile(String path) throws IOException {
        try (BufferedReader br = new BufferedReader(new FileReader((path)))) {
            return br.readLine();
        }
    }
```

해당 코드를 보았을 때,

만약 readLine과 close 모두에서 예외가 발생할 때,

close 호출에서 발생하는 예외는 숨겨지고 readLine의 예외가 기록된다.

이때, 숨겨진 close 예외는 무시되는 것이 아니라, suppressed 상태가 되어 stackTrace 시 숨겨졌다는 메시지로 출력된다. 

ex. `Suppressed: 해당 예외~~` 과 같이 출력된다.

suppressed 상태가 된 예외를 자바 7부터 getSuppressed 메서드를 통해 가져올 수 있다.

- **inputString 메서드를 try-with-resources 방식으로 바꾼 코드, catch 추가**

```java
static String firstLineOfFile(String path) {
        try (BufferedReader br = new BufferedReader(new FileReader((path)))) {
            return br.readLine();
        } catch (IOException) {
						return "IOException 발생";
				}
    }
```

- **copy 메서드를 try-with-resources 방식으로 바꾼 코드**

```java
static  void copy(String src, String dst) throws IOException {
        try (InputStream in = new FileInputStream(src);
						OutputStream out = new FileOutputStream(dst);){
    
                byte[] buf = new byte[BUFFER_SIZE];
                int n;
                while ((n = in.read(buf)) >= 0) {
                    out.write(buf, 0, n);
								}
		    }
}
```
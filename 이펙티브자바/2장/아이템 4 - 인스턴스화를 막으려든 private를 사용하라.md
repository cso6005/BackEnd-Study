# [item 4] 인스턴스화를 막으려든 private를 사용하라

단순히 정적 메서드와 정적 필드만을 담은 클래스들은, 객체 지향적이지 않은 것처럼 보이지만 나름의 쓰임새가 존재한다. 
다음의 예시들을 봐보자.

정적 메서드와 정적 필드만을 담은 클래스 예시
`java.lang.Math`, `java.util.Arrays` 이나 `java.util.Collections` 는 
특정 인터페이스를 구현하는 객체를 생성해주는 정적 메서드를 모아놓고 있다. 
혹은, `final` 클래스와 관련 메서드들을 모아놓을 수도 있다.
```java


public class Arrays {
private Arrays() {}

    public static void sort(int[] a) { ... }
    public static void parallelSort(byte[] a)
    // ...
}
```

하지만 중요한 것은 해당 정적 클래스들은, 인스턴스로 만들어 쓰려고 설계한 것이 아니다. 따라서 만약 기본 생성자를 명시하지 않으면 컴파일러가 자동으로 매개변수를 받지 않는 public 생성자를 만들어주기 때문에 의도치 않게 인스턴스화가 가능하게 되는 문제가 발생한다.

그렇다면 어떻게 인스턴스화를 막을 수 있을까? 뭔가 추상 클래스를 만들어도 좋을 것 같다는 생각이 들 수 있다. 하지만 아래의 예제를 봐보자.

☁️ 추상 클래스 도입: 인스턴스화가 가능하다
추상 클래스로 만들면 물론 new UtilClass() 에서 인스턴스화가 불가능해져 컴파일 에러가 날 것이다. 하지만, 아래 처럼 AnotherClass 라는 추상 클래스를 상속받은 하위 클래스를 통해 인스턴스화가 가능해진다.

또한 오히려 상속해서 쓰라는 뜻으로 오해할 여지도 존재한다.
```java


public abstract class UtilClass {
public static String getName() {
return "ABC";
}
}

class AnotherClass extends UtilClass {
}

public class Test {
public static void main(String[] args) {
AnotherClass anotherClass = new AnotherClass();  // 인스턴스화 가능
    }
}
```
☁️ 인스턴스화를 막는 방법: `private` 생성자
결론은, private 생성자를 추가하면 클래스의 인스턴스화를 간단하게 막을 수 있다. 생성자를 제공하지만 쓸 수 없기 때문에 직관에 어긋나는 점이 있는데, 그 때문에 주석을 추가하는 것을 권장한다.

```java


public class UtlityClass {
//기본 생성자가 만들어지는 것을 막음(인스턴스화 방지용)
private UtilityClass(){
throw new AssertionError();
    }
...
}
```
또한, 이 방식은 상속을 불가능하게 하는 효과도 존재한다. 모든 생성자는 상위 클래스의 생성자를 호출하게 되는데, 
이를 private 으로 선언했으니 상위 클래스의 생성자에 접근할 방도가 없기 때문이다.
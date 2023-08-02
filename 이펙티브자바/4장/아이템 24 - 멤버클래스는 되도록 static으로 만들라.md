# 아이템 24: 멤버클래스는 되도록 static으로 만들라

자바에는 다양한 중첩 클래스(nested class)가 있다. 종류는 다음과 같다.

1. 정적 멤버 클래스
2. 멤버 클래스
3. 익명 클래스
4. 지역 클래스

이번 아이템은 각각의 중첩 클래스를 언제, 왜 사용해야 되는지 필요성에 대해 다뤄보고자 한다.

## 중첩클래스
중첩클래스는 다름 클래스 안에 정의된 클래스를 말한다.
중첩클래스는

(1) 자신을 감싼 바깥 클래스에서만 사용되어야 하며,
(2) 그외의 쓰임새가 있다면 톱레벨 클래스(파일명과 동일한 .java 클래스)로 만들어야 한다.

### 정적, 비정적 멤버 클래스
#### 선언
둘의 차이는 `static` 예약어가 클래스에 같이 작성되어 있는지로 판단된다.

```java
class A{
	private int a;
    
    static class B{
    	private int b;
    }
}
```
위의 코드에서 B는 정적 내부 클래스,

```java
class A{
	private int a;
    
    class B{
    	private int b;
    }
}
```
위의 코드에서 B는 비정적 내부 클래스이다.

#### 생성

static의 존재 여부에 따라 해당 내부 클래스를 생성하는 방식이 달라진다.

먼저 정적 내부클래스의 객체 생성은 다음과 같이 할 수 있다.
```java
void foo(){
	A.B b = new B();
}
```
`static` 예약어로 인해 독립적인 객체 생성이 가능하다. 하지만 비정적 내부 클래스인 경우 다음과 같이 해야한다.

```java
void foo(){
	A a = new A();
    A.B b= new B();
}
//or

void foo(){
	A.B b = new A().B();
}
```
비정적 클래스의 객체 선언의 경우 반드시 A 객체를 생성한 뒤 객체를 이용해 생성해야 한다. 즉, 비정적 내부 클래스는 바깥클래스에 대한 참조가 필요하다.

#### 참조
클래스를 컴파일해 `.class`로 변환한 뒤 `Diassembler`결과를 확인하여 참조를 확인해본다.
정적 내부 클래스가 멤버변수와 기본 생성자만을 참조로 가지고 있는 것과 달리, 비정적 내부 클래스는 추가로, <u>바깥클래스의 참조</u>를 가지고 있다.

이는 메모리 누수의 가능성이 있다. 바깥 클래스는 더 이상 사용되지 않지만 내부 클래스의 참조로 인해 GC가 수거하지 못해 바깥클래스이 메모리 해제를 못하는 경우가 발생한다.


#### 특징

정적 멤버 클래스의 주목할만한 특징은 다음과 같다.

1. 바깥 클래스의 `private`멤버애도 접근할 수 있다.
2. 또한 다른 정적 멤버와 같은 규칙을 적용받는다.

예를들어 `private`으로 선언하면 바깥클래스에서만 접근할 수 있다.

```java
class OuterClass [
	
    //private 정적 멤버 클래스
    private static class StaticNestedClass{
    	private static int staticNestedValue = 10;
        private int instanceNestedValue = 20;
    }
    public static int getStaticNestedValue(){
    	return StaticNestedClass.staticNestedValue;
    }
    
    public static void main(String[]args){
    	// 정접 멤버 변수에 접근  (바깥 클래스에서 직접 접근 가능)
    	int value = StaticNestedClass.staticNestedValue;
    
    	//정적 메서드에서 접근 (바깥클래스에서 직접 접근 가능)
    	int result = getStaticNestedValue;
    }
}
```

#### 필요성 및 사용시기

정적 멤버 클래스는 흔히 바깥 클래스와 함께 쓰일 때만 유용한 public 도우미 클래스로 쓰인다. 다음은 Operation 열거 타입이 Calculator클래스의 public 정적 멤버가 되어야하는 예시이다.

``` java
class Calculator{
	
    //Operation 열거 타입 선언
    public enum Operation{
    	PLUS, MINUS
    }
    // 메서드에서 operation 열거 타입 활용
    publilc static int calculate(Operation operation, int operand1, int operand2){
    	switch(operation){
        	case PLUS:
            	return operand1 + operand2;
            case MINUS:
            	return operand1 - operand2;
            default:
            	throw new IllegalArgumentException("Unsupported operation");
        }
    }
}
```

또한 메모리 누수가 발생할 수 있는 문제점이 있기에 만약 **내부 클래스가 독립적으로 사용된다면 정적클래스로 선언**하여 사용하는 것이 좋다. 바깥 클래스에 대한 참조를 가지고 있지 않아 메모리 누수가 발생하지 않기 때문이다.

---

비정적 멤버 클래스는 어댑터를 정의할 때 자주 사용된다. 즉, 어떤 클래스의 인스턴스를 감싸 마치 다른 클래스의 인턴스처럼 보이게 하는 뷰로 사용한다.

예를 들어 `Map` 인터페이스의 구현체들은 보통 (`keySet`, `entrySet`, `values` 메서드가 반환하는) 자신의 컬렉션 뷰를 구현할 때 비정적 멤버 클래스를 사용한다.
비슷하게, `Set`, `List` 같은 다른 컬렉션 인터페이스 구현들도 자신의 반복자를 구현할 때 비정적 멤버 클래스를 주로 활용한다.

```java
public class MySet<E> extends AbstractSet<E>{
	...// 생략
    
    @Override public Iterator<E> iterator(){
    	return new MyIterator();
    }
    private class MyIterator implements Iterator<E>{
    	...
    }
}
```
위 코드는 비정적 멤버 클래스의 흔한 쓰임으로서, `MyIterator`의 인스턴스를 반환하는 자신의 반복자를 구현하는 예시이다.

<u>**멤버 클래스에서 바깥 인스턴스에 접근할 일이 없다면 static을 붙여 정적 멤버 클래스로 만들자**</u>고 저자는 강조한다.

이유는 앞에서 언급했듯, GC가 바깥 클래스의 인스턴스를 수거하지 못해 메모리 누수가 발생한다늠 점에서 기인한다. 
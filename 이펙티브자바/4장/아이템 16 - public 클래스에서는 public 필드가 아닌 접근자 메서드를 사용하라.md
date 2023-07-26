# [item 16 public class 에서는 public 필드가 아닌 접근자 메서드를 사용하라.]

다음의 코드는 <u>**"지양"**</u> 해야될 코드이다.

```java
class Point{
	public double x;
    public double y;
}
```
문제점은 다음과 같다.

1. <u>데이터 필드에 직접 접근 가능</u>하여 캡슐화의 장점을 전혀 사용하지 못한다.
2. API를 수정하지 않고는 내부 표현을 바꿀 수 없다.
3. 불변식을 보장 할 수 없다.
4. 외부에서 필드에 접근함과 동시에 다른 작업이 불가능하다.

위 코드를 개선한다면, <u>**패키지 바깥에서 접근할 수 있는 클래스라면 접근자를 제공**</u> 하여 클래스 내부 표현 방식을 언제든 바꿀수 있는 유연성을 가지게 할 수 있다.

또한, 필드를 모두 `private`으로 바꾼다.

```java
// 접근자와 변경자 메서드를 활용해 데이터를 캡슐화!
class Point{
	private double x;
    private double y;
    public Point(double x, double y){
    this.x=x;
    this.y=y;
    }
    
    public double getX() { return x; }
    public double getY() { return y; }
    
    public void setX(double x) { this.x = x; }
    public void setY(double y) { this.y = y; }
}
```

위 코드는
- getter / setter 로 내부표현 변경이 가능하다.
- 클라이언트는 public 메서드를 통해서만 데이터 접근이 가능하다.
- 외부에서 부수작업을 수행시킬 수 있다.

### package-private클래스 혹은 private 중첩 클래스 데이터 캡슐화

`package-private` 클래스 혹은 `private` 중첩 클래스라면 데이터 필드를 노출한다 해도 클래스가 표현하려는 추상 개념만 올바르게 표현해주면 문제가 없다.

```java
public class ColorPoint {
    private static class Point{
        public double x;
        public double y;
    }

    public Point getPoint(){ // 클라이언트 코드가 클래스 내부에 묶인다.
        Point point = new Point(); //ColorPoint 외부에서는 
        point.x = 1.2; // Point 클래스 내부 조작이 불가능하다.
        point.y = 5.3; 

        return point; 
    }
}
```

- private(또는 package-private) 클래스를 중첩시키면 탑 클래스(제일 바깥 클래스) 외부에서는 Point 클래스의 필드에 접근하는 것이 불가능하다. 하지만, ColorPoint에서는 Point 클래스 필드 조작이 가능하다.

- 클래스를 중첩시키는 방식은 클래스 선언 면에서나 이를 사용하는 클라이언트 코드면에서나 접근자 방식보다 깔끔하다.

- 클래스를 중첩시키면 클라이언트 코드가 클래스 내부에 묶이기는 하지만 클라이언트도 어차피 이 클래스를 포함하는 패키지 안에서 동작하는 코드이므로 패키지 바깥 코드를 손대지 않고 데이터 표현방식을 바꿀 수 있다.

- private 중첩 클래스의 경우 수정 범위가 더 좁아져서 이 클래스를 포함하는 외부 클래스까지로 제한된다.
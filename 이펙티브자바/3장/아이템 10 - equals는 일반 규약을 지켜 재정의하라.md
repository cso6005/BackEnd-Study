# 아이템 10 - ****equals는 일반 규약을 지켜 재정의하라****

Object의 equals는 이미 비교를 정확히 수행해주도록 구현되어 있다.

equals 메서드는 아예 재정의해야 말아야 하는 상황과 재정의가 필요한 상황을 알아볼 것이다.

그리고 재정의한다면

그 클래스의 핵심 필드 모두가 빠짐없이 지켜야 할 다섯 가지 규약을 알아보자.

# 재정의하지 않아야 하는 상황

1. 각 인스턴스가 본질적으로 고유하다.
    
    ex. Thread
    
    값을 표현하는 게 아니라 동작하는 개체를 표현하는 클래스로 Object의 equals 메서드는 이러한 클래스에 딱 맞게 구현되었다.
    
2. 인스턴스의 논리적 동치성을 검사할 일이 없다.

1. 상위 클래스에서 재정의한 equals가 하위 클래스에도 딱 들어맞는다.
    
    예시로 
    
    대부분 Set 구현체는 AbstractSet이 구현한 equals를 상속 받아 쓰고,
    
    List 구현체는 AbstractList로부터
    
    Map 구현체들은 AbstractMap으로부터 상속받아 그대로 쓴다.
    
2. 클래스가 private 이거나 package-private이고 equals 메서드를 호출할 일이 없다.
- 만약 equals가 실수로라도 호출되는 걸 막고 싶다면 아래처럼 하자.

```
@Override
public boolean equals(Object o) {
    throw new AssertionError();  // 호출 금지
}
```

# 재정의해야 하는 상황

논리적인 동치성을 확인해야 하는데, 재정의되지 않았을 때, 재정의 해야 한다.

주로 값 클래스들(Integer, String …)이 여기에 해당하는데, 

보통 우린 둘의 객체가 같은지가 아니라 값 자체가 같은지 알고 싶기 때문에 이다.

**예외)**

- 값 클래스라도 값이 같은 인스턴스가 둘 이상 만들어지지 않음을 보장하는 인스턴스 통제 클래스(item1)라면 equals를 재정의하지 않아도 된다.
    
    논리적으로 같은 인스턴스가 2개 이상 만들어 지지 않으니, 논리적 동치성과 객체 식별성을 같은 의미로 보면 된다. 즉, Object의 equals로 같다는 것을 보고, 논리적으로 같다는 의미를 갖게 되는 것.
    
    ex) Enum
    

# equals 재정의 시, 지켜야 할 일반 규약

컬렉션 클래스들을 포함해 수많은 클래스는 전달받은 객체가 equals 규약을 지킨다고 가정하고 동작한다.

만약, equals 규약을 어긴다면, 프로그램이 이상하게 동작하거나 종료될 것이고, 디버깅도 굉장히 어려울 것이다. 

그렇기에 반드시 지켜야 한다.

### Object 명세에 적힌 규약(동치 관계를 만족시키기 위한 다섯 요건)

```java
null이 아닌 모든 참조 값 x,y,z에 대해

- 반사성
    - x.equals(x)는 true 이다.
- 대칭성
    - x.equals(y) == true라면,
    - y.equals(x) == true이다.
- 추이성
    - x.equals(y) == true && y.equals(z) == true 라면,
    - x.equals(z) == true 이다.
- 일관성
    - x.equals(y)를 반복해서 호출하면,
    - 항상 true를 반환하거나 항상 false를 반환한다. 값이 변하지 않는다.
- null-아님
    - x.equals(null) 는 false 이다.
```

- **Object명세에서 말하는 동치 관계란?**
    
    집합을 서로 같은 원소들로 이뤄진 부분집합으로 나누는 연산이다.
    
    이 부분 집합을 동치류(equivalence class; 동치 클래스) 라 한다.
    
    equals 메서드가 쓸모 있으려면 모든 원소가 같은 동치류에 속한 어떤 원소와도 교환할 수 있어야 한다.
    

## 1. 반사성

- null이 아닌 모든 참조 값 x에 대해 x.equals(x)는 true다.
- 자기 자신과 같아야 한다.

## 2. 대칭성

- nul이 아닌 모든 참조 값 x, y에 대해, x.equals(y)가 true면 y.equals(x)도 true다.
- 서로에 대한 동치 여부는 같아야 한다.

대소문자를 구별하지 않는 문자열 클래스를 만들었다고 생각해보자.

```
public final class CaseInsensitiveString {
    ...
    // 대칭성 위배
    @Override
    public boolean equals(Object o) {
        if (o instanceOf CaseInsensitiveString)
            return s.equalsIgnoreCase((CaseInsensitiveString) o).s);

        if (o instanceOf String) // 한 방향으로만 작동한다
            return s.equalsIgnoreCase((String) o);

        return false;
    }
}
```

```
CaseInsensitiveString cis = new CaseInsensitiveString("Polish");
String str = "polish";
System.out.println(cis.equals(str)); // true
System.out.println(str.equals(cis)); // false
```

일단 해당 클래스에서 equals는 대소문자를 무시한다.

그렇기에 cis.equals(s)는 true를 반환한다.

CaseInsenstivieString.equals()는 일반 String을 알고 있지만,

String.equals()는 CaseInsensitiveString의 존재를 모른다는 것이 문제다.

따라서 s.equals(cis)는 false를 반환하여 대칭성을 명백히 위반한다.

그렇기에 CaseInsensitiveString의 equals를 String과도 연동해선 안된다.

## 3. 추이성

- null이 아닌 모든 참조 값 x, y, z에 대해,
    
    x.equals(y)가 true이고 y.eqauls(z)도 true면,
    
    x.equals(z)도 true다.
    

- **구체 클래스를 확장해 새로운 값을 추가하면서 equals 규약을 만족시킬 방법은 존재하지 않는다.**

## 4. 일관성

- null 이 아닌 모든 참조 값 x, y에 대해, x.equals(y)를 반복해서 호출하면 항상 true를 반환하거나 항상 false를 반환한다

- 두 객체가 같다면(객체가 수정되지 않는 한), 앞으로도 영원히 같아야 한다는 뜻

- 가변 객체는 비교 시점에 따라 같을 수도 다를 수도 있는 반면, 불변 객체는 한번 다르면 끝까지 달라야 한다.

불변 객체라면 equals가 한번 같다고 한 객체와는 영원히 같다고 해야 하고,

다르다고 한 것은 영원히 다르다고 답하게 만들어야 한다.

⇒ **equals 판단에 신뢰할 수 없는 자원이 끼어들게 해서는 안된다. 이를 어기면 일관성 조건을 만족하기 어려워진다.**

```
@Test
void consistencyTest() throws MalformedURLException {
    URL url1 = new URL("www.xxx.com");
    URL url2 = new URL("www.xxx.com");

    System.out.println(url1.equals(url2)); // 항상 같지 않다는 것을 보장할 수 없다.
}
```

java.net.URL 의 equals는 주어진 URL과 매핑된 host의 IP주소를 이용해 비교한다.

이때, 같은 도메인 주소라도 나오는 IP정보가 달라질 수 있기 때문에 반복적으로 호출할 경우 결과가 달라질 수 있다.

따라서 이런 문제를 피하려면 equals는 항시 메모리에 존재하는 객체만을 사용한 결정적 계산만 수행해야 한다.

## 5. null-아님

- null이 아닌 모든 참조 값 x에 대해,x.equals(null)은 false다.
- 모든 객체는 null과 같지 않아야 한다.

- **instanceof 연산자를 사용하여, 입력이 null인지 확인해 자신을 보호해야 한다.**
    
    동치성을 검사하려면 equals는 건네받은 객체를 적절히 형변환한 후 필수 필드들의 값을 알아내야 한다. 
    
    이를 위해 형변환에 앞서 instanceof 연산자로 입력 매개변수가 올바른 타입인지 검사해야 한다.
    
    instanceof는 두 번째 피연산자와 무관하게 첫 번째 피연산자가 null이면 false를 반환한다.
    
    따라서 **입력이 null이면 타입 확인 단계에서 false를 반환하기 때문에 null 검사를 명시적으로 하지 않아도 되게 해주는 것이다.** 
    

```java
@Override 
public boolean equals(Object o) {

    // 흔히 우리가 생성하는 equals 방법
		// 명시적인 검사 o == null 하지말고, 아래 방법을 사용하자.
    if (o == null || getClass() != o.getClass()) {
        return false;
    }
    
    // instanceof 이용 방법 (묵시적 null 검사)
    if (!(o instanceof Point)) {
        return false;
    }
}
```

# 양질의 equals 메서드 구현 방법(단계별로)

### 1. == 연산자를 사용해 입력이 자기 자신의 참조인지 확인한다. 자기 자신이면 true를 반환한다.

### 2. instanceOf 연산자로 입력이 올바른 타입인지 확인한다.

### 3. 입력을 오바른 타입으로 형변환한다.

### 4. 입력 개체와 자기 자신의 대응되는 '핵심' 필드들이 모두 일치하는지 하나씩 검사한다.
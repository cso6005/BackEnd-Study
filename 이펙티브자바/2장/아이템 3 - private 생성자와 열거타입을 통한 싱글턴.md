# [아이템 3] private 생성자와 열거타입을 통한 싱글턴
[벨로그 정리글](https://velog.io/@trendsetter/Item-3-private-%EC%83%9D%EC%84%B1%EC%9E%90%EC%99%80-%EC%97%B4%EA%B1%B0%ED%83%80%EC%9E%85%EC%9D%84-%ED%86%B5%ED%95%9C-%EC%8B%B1%EA%B8%80%ED%84%B4)

![](https://velog.velcdn.com/images/trendsetter/post/93bf9146-a7de-4486-809a-6774c3903085/image.jpg)


아이템 3를 설명하기 앞서 우선 `singleton`에 대한 전반적인 이해가 필요하다.
### Singleton Pattern

`singleton pattern` 이란, `하나의 클래스에 오직 하나의 인스턴스`를 가지는 패턴이다.

하나의 클래스를 기반으로 여러 개의 개별적인 인스턴스를 만들 수 있지만, 그렇게 하지 않고 `하나의 클래스를 기반`으로 `단 하나의 인트턴스를 만들어` 이를 기반으로 로직을 만드는데 쓰이며, 보통 DB연결 모듈 (설계상 유일해야하는 시스템 컴포넌트)에 많이 사용한다. 함수와 같은 무상태(stateless) 객체도 예시이다.

>장점

하나의 인스턴스를 만들어 놓고 해당 인스턴스를 다른 모듈들이 공유하며 사용
=> 인스턴스 생성시 드는 비용이 줄어든다.

> 단점1. 높은 의존성

System상의 다른 클래스들이 싱글톤의 하나의 인스턴스에 의존함
=> `의존성이 높아진다`, 즉, 싱글톤으로 구현된 클래스가 변경등 `수정이 필요하면` 그것에 의존하는 `다른 시스템의 부분들이 영향`을 받는다.

이러한 `응집력 높은 커플링`은 `의존성을 관리`하는데 있어서 `복잡도`와 `어려움`을 증가시킨다.

`싱글턴의 단점`을 설명하기 위해 `Spring Framework`로 예시를 들어본다.

Spring에서는 Singleton `Bean`을 `configuration`을 활용해 기본으로 지정할 수 있다.
아래와 같이 UserService 클래스가 Singleton으로서 implement한다고 가정하자.

```java
@Service	// sterotype anotation : mark a class as a service component 
public class UserService {
    // ...
}
```
기본적으로, Spring은 UserService 클래스의 싱글톤 인스턴스를 생성한다.

자, 이제  **'UserService'** 클래스에 의존하는 **'OrderService'** 클래스를 생성한다.

```java
@Service
public class OrderService {
    private final UserService userService;

    public OrderService(UserService userService) { 
    //constructor injection
        this.userService = userService;
    }

    // ...
}
```
위 코드는 `생성자 주입`을 통하여 **'UserService'**의 인스턴스를 받고 있다. UserService 클래스가 바뀌면 OrderService 클래스도 변경이 일어나야 하는 것이다!

=> 시스템 전체에 파급효과(ripple effect)가 발생하고 코드베이스를 관리하고 발전시키는 것이 어렵다.

> 단점 2. 클라이언트 테스팅의 어려움

아이템 3를 읽다가 이해가 안되는 구절이 있었다.


**_"타입을 인터페이스로 정의한 다음 그 인터페이스를 구현해서 만든 싱글턴"_**

Spring으로 예시를 찾아본 결과 다음과 같은 뜻이었다.

Spring은 기본적으로 Bean을 지원하기 때문에 간단히 구현 가능하다.
**'Logger'** 인터페이스를 정의한다고 하자.
```java
public interface Logger{
	void log(String message);
}
```
이 인터페이스를 구현한 클래스를 생성한다.
```java

@Component
@Scope("singleton")
public class ConsoleLogger implements Logger{
	@Override
    public void log(String message){
    	System.out.println("Logging message" + message);
    }
}
```
Spring에서는 **'@Component'** 를 통해 빈을 등록할 수 있다.
**`@Scope("singleton")`** 어노테이션을 사용하여 싱글턴 스코프를 지정할 수 있다.

이제 다른 클래스에서 **'Logger'** 인터페이스를 주입받아 사용할 수 있다. 아래는 **'MyService'** 클래스에서 **'Logger'** 를 의존성으로 주입받아  사용하는 코드 예시이다.
```java
@Service
public class MyService{
	private final Logger logger;
    
    @Autowired
    public MyService(Logger logger){	// constructor injection
    	this.logger = logger;
    }
    
    public void doSomething(){
    	logger.log("Doing something");
    }
}
``` 

이렇게 하면 **'MyService'** 클래스는 'ConsoleLogger'싱글턴 빈을 사용하여 로깅기능을 수행할 수 있다. 여러 클래스에서 **'Logger'** 인터페이스를 주입받아 사용하더라도 항상 같은 **'ConsoleLogger'** 인스턴스를 공유하게 된다.

위의 케이스는
1. `의존성 주입`
2. `Mock 객체`(**'Logger'** Mock 객체를 생성하여 테스트 할때는 실제 로깅이 발생하지 않음)사용
   3.`단일 인스턴스`(테스트시에도 동일한 인스턴스가 사용되어 테스트간 상태 공유나 인스턴스 생성에 따른 부작용을 방지하여 테스트의 일관성+의존성 향상)

이 3가지의 이유로 테스트가 수월해진다.


## 싱글턴을 만드는 방식
이제 싱글턴을 만드는 2가지 방식에 대해 알아본다.

1. public static 멤버가 final 필드인 방식
2. 정적 팩터리 메서드를 public static 멤버로 제공

>공통점

두 방식 모두

1. 생성자는 `private`으로 감춰두고,
2. 유일한 인스턴스에 접근할 수 있는 수단으로 `public static` 멤버를 하나 마련해둔다.

### public static 멤버가 final 필드인 방식

```java 
public class Elvis{
	public static final Elvis INSTANCE = new Elvis(); // 방법 1
    private Elvis() {...}	// 생성자는 private
    
    public void leaveTheBuilding() {
    	
    }
}
```

위의 코드에서 `private` 생성자는 public static final 필드인 `Elvis.INSTANCE`를 초기화 할 때 **"딱 한번만 호출"**된다.

Elvis.INSTANCE필드는 클래스가 초기화 될 때 단 한번 생성되고, final키워드로 인해 다른 곳에서 변경할 수 없는 상수로 선언된다.

leaveTheBuilding() 메서드는 INSTANCE 필드를 반환하여 클래스의 인스턴스를 얻을 수 있도록 한다.

이를 통해 다음과 같이 클래스의 인스턴스가 하나임이 보장된다.

```java
Elvis obj1 = Elvis.INSTANCE;
Elvis obj2 = Elvis.INSTANCE;

System.out.println(obj1 == obj2);	// true
```

위의 예시에서 'obj1'과 'obj2'는 leaveTheBuilding() 메서드를 통해 동일한 Elvis 인스턴스를 얻으므로, obj1==obj2는 true를 출력한다.
이는 Elvis 클래스의 인스턴스가 전체 시스템에서 하나임이 보장되는 것을 보여준다. (클라인트가 수정 불가)

>장점
>> 1. 해당 클래스가 싱글턴임이 API에 명백히 드러난다.
>> - public static 필드가 final이니 절대로 다른 객체를 참조할 수 없다.
>> 2. 간결함

예외는 리플렉션 API(`아이템 65`)인 `AccessibleObject.setAccessible`을 사용해 private 생성자를 호출할 수 있는 것이다.

위 공격을 방어하려면 생성자를 수정해 두 번째 객체가 생성되려 할 때 예외를 던지면 된다. 

### 정적 팩터리 메서드를 public static 멤버로 제공

```java
public class Elvis {
	private static final Elvis INSTANCE = new Elvis();
    private Elvis() {...}
    public static Elvis getInstance() { return INSTANCE; }	
    //같은 객체의 참조 반환
    
    public void leaveTheBuilding() {...}
}
```

Elvis.getInstance()는 항상 같은 객체의 참조를 반환하므로 한개의 인스턴스만 생성된다.

>장점
>> 1. API를 바꾸지 않고도 싱클턴이 아니게 변경
      - 유일한 인스턴스를 반환하던 팩토리 메서드(`객체 생성 처리`)가 호출하는 스레드별로 다른 인스턴스를 넘겨주게 함
>> 2. 원한다면 정적 팩터리를 제너릭 싱글턴 팩터리(아이템 30)으로 만들 수 있다.
>> 3. 정적 팩터리의 메서드 참조를 공급자로 사용






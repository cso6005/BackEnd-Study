# [아이템 3] private 생성자와 열거타입을 통한 싱글턴

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




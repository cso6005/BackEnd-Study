# 아이템14. Comparable을 구현할지 고려하라

## ****Comparable?****

![스크린샷 2023-07-19 오후 1.14.17.png](https://github.com/cso6005/BackEnd-Study/assets/51026414/a5b8f8ff-0992-4a1d-8a39-6d94310d5a9c)

보통 객체를 정렬하기 위한 인터페이스로 이해하고 있지만, **객체를 비교할 수 있는 인터페이스**로 보는 것이 바람직 하다. ****Comparable****을 구현한 클래스는 compteTo 메서드에서 정렬 기준을 정의하여 객체 비교를 통해서 정렬 기준을 정의할 수 있고 이를 natural ordering라고 이갸이한다

자바에선 객체를 비교하는 인터페이스는 `Comparable` 와 `Comparator` 두가지가 존재 한다.

## compareTo 규약

[Compare javadoc](https://github.com/openjdk/jdk/blob/jdk-17%2B35/src/java.base/share/classes/java/lang/Comparable.java)을 살펴보면 객체 비교간 몇가지 규약이 정해져 있는걸 확인할 수 있다.

![Untitled](https://github.com/cso6005/BackEnd-Study/assets/51026414/56bfe22d-ee2e-4a4e-8052-0b356b426b4e)

- 추가로 아래 규약에서 BigDecimal 객체를 사용하고 있는데 BigDecimal객체에서 [compareTo](https://github.com/openjdk/jdk/blob/jdk-17%2B35/src/java.base/share/classes/java/math/BigDecimal.java#L3108) 메소드의

  연산 결과를

    - 객체간의 값이 같다면(넘겨받은 객체와 값이 동일) 0
    - 넘겨 받은 객체보다 값이 크다면 1
    - 넘겨 받은 객체보다 값이 작다면 0을 반환한다

1. 반사성
    - 거울을 보면 자기 자신을 비춰지는 것 처럼 자기 자신과 비교 했을때 같아야한다.

    ```java
    @Test
    void test() {
        BigDecimal bigDecimal1 = BigDecimal.valueOf(20);
        BigDecimal bigDecimal2 = BigDecimal.valueOf(10);
        BigDecimal bigDecimal3 = BigDecimal.valueOf(30);
        BigDecimal bigDecimal4 = BigDecimal.valueOf(10);
    
        // 반사성
        System.out.println(bigDecimal1.compareTo(bigDecimal1)); // 0
    }
    ```

2. 대칭성
    - 아래 사진 처럼 어디서 보더라도 좌우 대칭이 동일한 것 처럼, 객체 비교에서 기준이 바뀌더라도 결과는 동일해야한다.

    ![스크린샷 2023-07-19 오후 4.02.28.png](https://github.com/cso6005/BackEnd-Study/assets/51026414/df368852-60b2-45d6-af05-0bb32d37dd9a)

    ```java
    @Test
    void test() {
        BigDecimal bigDecimal1 = BigDecimal.valueOf(20);
        BigDecimal bigDecimal2 = BigDecimal.valueOf(10);
        BigDecimal bigDecimal3 = BigDecimal.valueOf(30);
        BigDecimal bigDecimal4 = BigDecimal.valueOf(10);
    
    		// bigDecimal1이 bigDecimal2보다 큰건 기준이 바뀌더라도 동일하다
        System.out.println(bigDecimal1.compareTo(bigDecimal2)); // 1
        System.out.println(bigDecimal2.compareTo(bigDecimal1)); // -1
    }
    ```

3. 추이성
    - bigDecimal3은 bigDecimal1 크며 bigDecimal1은 bigDecimal2다 크다.

      그럼 bigDecimal3은 bigDecimal2 보다 큼을 보장해야한다

    ```java
    @Test
    void test() {
        BigDecimal bigDecimal1 = BigDecimal.valueOf(20);
        BigDecimal bigDecimal2 = BigDecimal.valueOf(10);
        BigDecimal bigDecimal3 = BigDecimal.valueOf(30);
        BigDecimal bigDecimal4 = BigDecimal.valueOf(10);
    
        System.out.println(bigDecimal3.compareTo(bigDecimal1) > 0); // 1
        System.out.println(bigDecimal1.compareTo(bigDecimal2) > 0); // 1
        System.out.println(bigDecimal3.compareTo(bigDecimal2) > 0); // 1
    }
    ```

4. 일괄성
    - 크기가 같은 객체들끼리는 어떤 객체와 비교하더라도 항상 같아야 한다는 뜻이다.

    ```java
    @Test
    void test() {
        BigDecimal bigDecimal1 = BigDecimal.valueOf(20);
        BigDecimal bigDecimal2 = BigDecimal.valueOf(10);
        BigDecimal bigDecimal3 = BigDecimal.valueOf(30);
        BigDecimal bigDecimal4 = BigDecimal.valueOf(10);
    
        System.out.println(bigDecimal4.compareTo(bigDecimal2)); // 0
        System.out.println(bigDecimal2.compareTo(bigDecimal1)); // -1
        System.out.println(bigDecimal4.compareTo(bigDecimal1)); // -1
    }
    
    ```

5. compareTo로 객체를 비교해서 객체의 값이 동일하더라도 equals는 항상 true를 보장하지 않는다.
    1. BigDecimal compareTo 발췌, BigDecimal처럼 compareTo와 equals 결과가 동일하지 않다면

       javadoc으로 문서화 해야한다

    2. equals는 스케일(자릿수) 까지 비교하기때문에 compareTo와 equals가 동일한 결과를 보장하지 않는
    경우도 발생한다

    ![스크린샷 2023-07-19 오후 4.26.06.png](https://github.com/cso6005/BackEnd-Study/assets/51026414/68e6f98b-e88e-4a4c-accf-a23fa2a0d121)
    
    ```java
    @Test
    void test() {
        BigDecimal oneZero = new BigDecimal("1.0");
        BigDecimal oneZeroZero = new BigDecimal("1.00");
        System.out.println(oneZero.compareTo(oneZeroZero)); // 0
        System.out.println(oneZero.equals(oneZeroZero));   // false
    
    }
    ```
    
    c. compareTo와 equals의 동작이 동일한 결과를 보장되지 않는다면 순서를 보장하는 컬렉션(TreeSet 등 단일값 보장)과 순서를 보장하지 않는 컬렉션(HashSet 등 단일값 보장) 엘리먼트를 적재할때 다른 결과를 초래하게 된다.
    
    - 순서를 보장하는 컬렉션은 데이터를 적재할때 compareTo를
        
    ![스크린샷 2023-07-19 오후 5.00.00.png](https://github.com/cso6005/BackEnd-Study/assets/51026414/875394af-2ef8-4fd4-a1bf-b1e0e9b1708f)
        
    순서를 보장하지 않는 컬렉션은 데이터를 적재할때 equals를 사용한다

    ![스크린샷 2023-07-19 오후 5.01.50.png](https://github.com/cso6005/BackEnd-Study/assets/51026414/7ae42d9e-1407-4592-9048-ee3d54dbb721)

    ```java
    @Test
    void test() {
        BigDecimal oneZero = new BigDecimal("1.0");
        BigDecimal oneZeroZero = new BigDecimal("1.00");
    
        Set<BigDecimal> hashSet = new HashSet<>();
        hashSet.add(oneZero);
        hashSet.add(oneZeroZero);
    
        System.out.println(hashSet.size());  // 2
    
        Set<BigDecimal> treeSet = new TreeSet<>();
        treeSet.add(oneZero);
        treeSet.add(oneZeroZero);
    
        System.out.println(treeSet.size());  // 1
    }
    ```


## compareTo 사용하기

1. 원시타입(Primitive type) 비교하기
    - 원시타입간 비교가 필요하다면 Boxing Type의 compare 메서드를 활용한다
    - 당연하지만 비교하고자 하는 필수 필드들이 여러개 존재한다면  가장 핵심적인 필드부터 비교해야 하며 순서가 결정된다면 이후 필드는 고려하지 않아도된다.
    - Integer.compare 코드를 보면 단순히 부등호로 표현 하고 있는데 우리들이 직접 구현 하면 오류를 범할 수도 있어서 내장된 함수를 사용하는게 마음이 편하다.(책에서도 권장하지 않음)

    ```java
    @Getter
    @RequiredArgsConstructor
    static class Money implements Comparable<Money> {
        private final int unitOfThousand;
        private final int unitOfHundred;
        private final int unitOfTen;
    
        @Override
        public int compareTo(Money money) {
    
            int result = Integer.compare(unitOfThousand, money.unitOfThousand);
            if (result == 0) {
                result = Integer.compare(unitOfHundred, money.unitOfHundred);
                if (result == 0) {
                    result = Integer.compare(money.unitOfTen, money.unitOfTen);
                }
            }
            return result;
        }
    }
    ```

2. Java8 부터는 `Comparator` 인터페이스 에서 지원하는 comparing* **[static 메소드](https://docs.oracle.com/javase/tutorial/java/IandI/interfaceDef.html)** 를 사용하여

   좀더 편하게 구현할수 있다.

    ```java
    @Getter
    @RequiredArgsConstructor
    static class Money implements Comparable<Money> {
    
        private final int unitOfThousand;
        private final int unitOfHundred;
        private final int unitOfTen;
    
    			private static final Comparator<Money> COMPARATOR = Comparator.comparingInt(Money::getUnitOfThousand)
                .thenComparingInt(Money::getUnitOfHundred)
                .thenComparingInt(Money::getUnitOfTen);
    
        @Override
        public int compareTo(Money money) {
            returnCOMPARATOR.compare(this, money);
        }
    }
    
    ```

3. Comparable은 타입을 인수로 받는 제네릭 인터페이스이므로, compareTo 인수타입은 컴파일 시에 정해진다. 즉, 입력 인수 확인이나 형변환을 할 필요가 없다.
4.  null을 인수로 넣으면 NullPointerException을 던져야 한다.

    ![스크린샷 2023-07-19 오후 6.31.34.png](https://github.com/cso6005/BackEnd-Study/assets/51026414/0ba3f123-d9ac-49f5-85a0-5e7284f53100)


## compareTo 사용 주의점

1. 상위 타입(부모 클래스)에서 ****Comparable****을 구현해서 compateTo를 오버라이딩 했다면 하위(자식 클래스)에서 ****Comparable**** 인터페이스를 구현할 수 없다
    - 하위 타입에서 ****Comparable****을 구현하면 아래와 사진과 같이 컴파일 예외가 발생한다.

      이는 ****오버 라이딩****이 아닌 **오버로딩**이 되기 때문이다.

        - 된다 하더라도 오버로딩이기때문에 하위 타입에서 구현해봤자 호출되지 않는다.(다형성 적용안됨)
    - SubMoney에서도 ****Comparable를**** 구현하여 1 단위까지 비교를 하고 싶다면 조합(Composition)을

      사용하면 된다

        ```java
        // 훨씬 객체 지향적이다.
        
        @Getter
        @RequiredArgsConstructor
        static class Money implements Comparable<Money> {
        
            private final int unitOfThousand;
            private final int unitOfHundred;
            private final int unitOfTen;
        
            @Override
            public int compareTo(Money money) {
        
                int result = Integer.compare(unitOfThousand, money.unitOfThousand);
                if (result == 0) {
                    result = Integer.compare(unitOfHundred, money.unitOfHundred);
                    if (result == 0) {
                        result = Integer.compare(money.unitOfTen, money.unitOfTen);
                    }
                }
                return result;
            }
        }
        
        @Getter
        @RequiredArgsConstructor
        static class SubMoney implements Comparable<SubMoney> {
        
            private final Money money;
            private final int unitOfOne;
        
            @Override
            public int compareTo(SubMoney subMoney) {
                int result = this.money.compareTo(subMoney.money);
                if (result == 0) {
                    result = Integer.compare(unitOfOne, subMoney.unitOfOne);
                }
                return result;
            }
        }
        ```

    ![Untitled](https://github.com/cso6005/BackEnd-Study/assets/51026414/d255166a-5427-48a5-bead-b8afc7a1ffab)

2. 해시코드 값의 차 or 원시타입의 연산 or 부동 소수점(가수부 표현하다 double, float 이 표현하는 범위를 넘을 수 있음)으로 인해 overflow or underflow가 발생하여 추이성을 위반할 수 있다.

    - 보통 정수의 표현 범위는 2,147,483,648 ~ 2,147,483,647이다. 표현 범위 밖을 넘어가면 아래 예시처럼반대편의 값 으로 표현이 되는데 이는 객체를 비교하는데 있어 오류를 범할 수 있다.

    - 2,147,483,648 - 1 = -2,147,483,649 이 아닌 2,147,483,647로
    - 2,147,483,647 + 1 = 2,147,483,648이 아닌 2,147,483,648로 표현이 되는데
    
    ```java
    Comparator<Object> hashCodeOrder = new Comparator<>() {
        public int compare(Object o1, Object o2) {
            return o1.hashCode() - o2.hashCode();
        }
        };

    @Getter
    @RequiredArgsConstructor
    static class Money implements Comparable<Money> {

        private final int unitOfThousand;
        private final int unitOfHundred;
        private final int unitOfTen;

        @Override
        public int compareTo(Money money) {
            return money.unitOfThousand - money.unitOfHundred;
        }
    }
    ```
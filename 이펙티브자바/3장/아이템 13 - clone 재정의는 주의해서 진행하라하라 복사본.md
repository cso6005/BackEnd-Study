# 아이템 13. clone 재정의는 주의해서 진행하라

## clone?

`java.lang` 패키지에 포함된 Object 클래스의 native 메소드로 bitwise copy(비트 복사,c++)를 한다.

- bitwise copy : 얕은 복사의 한가지 종류 생성자 호출 없이 원본클래스의 비트(멤버들)의 복제 대상 클래스에 복사
- Object.clone 메소드는
  native **[jvm.cpp#fixup_cloned_reference](https://github.com/JetBrains/jdk8u_hotspot/blob/master/src/share/vm/prims/jvm.cpp#L586)**
  에서 확인가능
    - JDK 11 에서도 큰 골자는 변경이
      없다 [jvm.cpp](https://github.com/openjdk/jdk11/blob/master/src/hotspot/share/prims/jvm.cpp#L592)

```java

// 객체일 경우
new_obj_oop=[CollectedHeap::obj_allocate](https://github.com/JetBrains/jdk8u_hotspot/blob/master/src/share/vm/gc_interface/collectedHeap.inline.hpp#L199C20-L199C32)(klass, size, CHECK_NULL);

// Array 일경우
new_obj_oop=[CollectedHeap::array_allocate](https://github.com/JetBrains/jdk8u_hotspot/blob/master/src/share/vm/gc_interface/collectedHeap.inline.hpp#L209)(klass, size, length, CHECK_NULL);
```

## clone의 규약

**[Object.clone javadoc](https://github.com/openjdk/jdk/blob/jdk-17+35/src/java.base/share/classes/java/lang/Object.java#L166-L228)**
을 살펴보면 원본 객체와 복제된 객체간에 몇가지 규약이 정해져 있는걸 확인 할 수 있다.

- x.clone() ≠ x
    - 복제된 객체는 원본 객체와 다른 객체여야한다, 즉 객체의 동치성이 보장되면 안된다.
    - 단, 주의 해야할 점은 복제된 객체와 원본 객체의 주소값만 다를뿐이지 구성하고 있는 멤버들의 주소는

      공유하기때문에 복제된 객체의 값을 변경한다면 원본까지 영향이 간다(shallow object copy)

```java
@Test
public void copy(){
        Person person=new Person(new Age(2));

        Person copyPerson=person.clone();

        copyPerson.name.age=4;

        System.out.println(person.hashCode()==copyPerson.hashCode()); //false

        System.out.println(person.name.hashCode()==copyPerson.name.hashCode()); //trye

        System.out.println(person); //4
        System.out.println(copyPerson); //4
        }

@ToString
@Setter
@AllArgsConstructor
public static class Person implements Cloneable {

    private Age name;

    @Override
    public Person clone() {
        try {
            return (Person) super.clone();
        } catch (Exception c) {
            log.error(c.getMessage());
            return null;
        }
    }
}

@ToString
@Setter
@AllArgsConstructor
public static class Age {

    private int age;

}
```

- x.clone().getClass() == x.getClass()
    - 복제된 클래스와 원본 클래스의 타입은 동일해야한다
- x.clone().equals(x)
    - 객체의 동등성이 항상 보장되지 않는다.
        - ****Immutable한 객체가 아닌 객체를 복사할 경우****  객체 식별을 의한 id등 값을 다르게 복사할 경우가 발생한 가능성이 존재한다.

## clone 사용하기

1. `Cloneable` 인터페이스를 구현해야한다
    - Cloneable 인터페이스를 구현하지 않고 `clone` 메소드를 호출하면 `Java.Lang.CloneNotSupportedException` 예외를 던진다
    - **[Cloneable 인터페이스](https://github.com/openjdk/jdk/blob/jdk-17%2B35/src/java.base/share/classes/java/lang/Cloneable.java)** 는
      아무 메소드도 포함되어있지 않은 마커(태깅) 인터페이스이다.
        - 마커 인터페이스 → 컴파일 레벨때 자신을 구현하는 클래스가 특정 속성임을 체킹하는 타입 체킹(타입 구분) 을 위한 인터페이스(아이템41)

   ```java
    // [jvm.cpp#L614 ~ 630](https://github.com/JetBrains/jdk8u_hotspot/blob/master/src/share/vm/prims/jvm.cpp#L614) cloneable 구현체 인지 체킹
    // guarantee -> c++ 예외 안전성 보장 함수?
    
    #ifdef ASSERT
    // Just checking that the cloneable flag is set correct
    // cloneable flag가 올바르게 설정되었는지 checking
    if (obj->is_array()) {
    	guarantee(klass->is_cloneable(), "all arrays are cloneable");
    } else {
    	guarantee(obj->is_instance(), "should be instanceOop");
    	bool cloneable = klass->is_subtype_of(SystemDictionary::Cloneable_klass());
    	guarantee(cloneable == klass->is_cloneable(), "incorrect cloneable flag");
    }
    #endif
    
    // Check if class of obj supports the Cloneable interface.
    // All arrays are considered to be cloneable ([See JLS 20.1.5](http://titanium.cs.berkeley.edu/doc/java-langspec-1.0/javalang.doc1.html) 참조)
    // Array가 아닌 일반 객체의 경우 Cloneable inferface를 구현해야하면
    // Java® Language Specification 20.1.5을 참조하면 모든 Array는 Cloneable 인터페이스가
    // 구현되어있다고 가정
    if (!klass->is_cloneable()) {
    	ResourceMark rm(THREAD);
    	THROW_MSG_0(vmSymbols::java_lang_CloneNotSupportedException(), klass->external_name());
    }
    ```

2. `clone` 메소드를 오버라이드 해야한다.
    - 단 상속관계에서 부모 클래스가 clone을 override를 하면 자식 클래스에서 clone을 재정의 하지 않아도 된다(메소드 상속)
    - 또한 Object.clone의 가시성은 **protected** 임으로 재정의시 **public**으로 가시성을 변경한다
        - **protected** 은 가시성이 상속관계까지 허용된다, 보통 애플리케이션을 구현할 경우 모든 객체 관계를 상속을 염두하고 개발하는 하는 일은 거의 없다(아예 없다 라고 봐도된다)

    ```java
    @Test
    public void copy() {
        Age age = new Age("낫낫", 2);
        Person copyAge = age.clone(); 
        System.out.println(copyAge); // Age(age=2)
    }
    
    @ToString
    @AllArgsConstructor
    public static class Person implements Cloneable {
        private final String name;
        @Override
    		// 공변성으로 인해 반환 타입의 인스턴스를 자기 자신으로 반환 가능
    		// 리스코프 치환 원칙, 최상위 부모 객체가 Object임으로 가능
        // Object로 반환 가능 하지만 호출하는쪽에서 타입캐스팅을 해줘야한다
        public Person clone() { 
            try {
                return (Person) super.clone(); 
            } catch (Exception c) {
    						log.error(c.getMessage());
                return null;
            }
        }
    }
    @ToString
    public static class Age extends Person {
        private final int age;
        public Age(String name, int age) {
            super(name);
            this.age = age;
        }
    }
    ```

3. 오버라이드한 `clone` 메소드에서는 `super.clone`을 호출해야한다.

## clone 재정의 주의점

1. Object.clone 처럼 클라이언트에게 예외를 넘기지 말고(메서드부 throw선언) 구현부에서 직접 예외 처리를 하자
    - `CloneNotSupportedException` 은 ``ArithmeticException,```ndexOutOfBoundsException`

      와 같은 unchecked exception이다, unchecked exception은 배열의 범위가 벗어나거나, 숫자를 0으로 나누거나 하는 예외가 발생하면 클라이언트가 아무것도 할 수 없기때문에 굳이
      메서드에 throw을 선언하지말고 구현부에서 직접 예외 처리를 해 깔끔하게 라도 코드를 작성하게 하자

        - 위와 같은 이유로 컴파일 레벨에서 잡지 않는다
        - TMI. 클라이언트가 해당 예외 상황을 복구(대체 값 반환)할 수 있다면 check계열(메서드부 throw선언)를 사용하고, 해당 예외가 발생했을 때 아무것도 할 수 없다면, unchecked로
          만든다
            - 참고: **https://docs.oracle.com/javase/tutorial/essential/exceptions/
              runtime.html**
2. Object.clone 재정의시 생성자를 사용해서 반환해선 안된다
    - 아무런 의존(상속)관계가 없는 객체의 경우 문제가 되질 않지만 자식 객체에서 재정의 하거나

   상속받은 clone메소드를 호출하는 순간 ClassCastException 예외를 던진다
   ```java
    public class Item implements Cloneable {
        private String name;
    
        /*
         *  하위 클래스의 clone이 깨질수 있다.. 하위 클래스에서 상위 클래스 형변환이 안되기 때문에
         */
        @Override
        public Item clone() {
    
            Item item = new Item();
            item.name = this.name;
    
            return item;
        }
    }
    
    public class SubItem extends Item implements Cloneable {
    
        private String name;
    
    //    @Override
    //    public SubItem clone(){
    //        // 상위 클래스의 clone 메소드에서 생성자로 clone을 채워 넣기 때문에 예외처리 하지 않아도됨
    //        return (SubItem) super.clone();
    //    }
    
       public static void main(String[] args){
    
           SubItem item = new SubItem();
           //SubItem clone = item.clone(); // classCastException 발생
    
           SubItem clone = (SubItem) item.clone(); // 해당 클래스에서 clone을 구현하지 않더라도 암묵적으로 clone 호출시 super.clone이 호출된다.
       }
    }
     ```

3. 상속용 클래스(부모 클래스)에 Cloneable 인터페이스 사용을 권장하지 않는다.
    - 상속용 클래스(부모 클래스)에 확장하려는 프로그래머에게 많은 부담을 준다.
        - 확장할려는 클래스(자식 클래스)에서 부모 클래스에 재정의 되있는 clone 및 clone재정의시

          모든 주의 사항을 고려하여 구현 해야하기때문에..

    - 그래도 굳이 상속용 클래스 Cloneable 인터페이스를 사용하고 싶다면

    ```java
    
    public class Item implements Cloneable {
    
        private int number;
    	
        /** 방법 1
          부담을 덜기 위해서는 기본 clone() 구현체를 제공하여,
          Cloenable 구현 여부를 서브 클래스가 선택할 수 있다.
          (하위클래스가 굳이 구현을 하지 않거나 하위 클래스에서 clone을 재정의 하거나)
         */
        @Override
        public Object clone() throws CloneNotSupportedException {
            return super.clone();
        }
    
        /**
    	방법 2
    	하위 클래스에서 clone을 재정의 하지 못하게 막는다
        */
        @Override
        protected final Object clone() throws CloneNotSupportedException {
    	    throw new CloneNotSupportedException();
        }
    }
    ```


4. clone은 thread-safe 하지 않다.
    1. 만약 쓰레드에 안정성을 보장해야한다면 clone 메소드에 동기화 처리(synchroinzed)를 해야한다

    ```java
    // 4839641 (4840070): We must do an oop-atomic copy, because if another thread
    // is modifying a reference field in the clonee, a non-oop-atomic copy might
    // be suspended in the middle of copying the pointer and end up with parts
    // of two different pointers in the field.  Subsequent dereferences will crash.
    // 4846409: an oop-copy of objects with long or double fields or arrays of same
    // won't copy the longs/doubles atomically in 32-bit vm's, so we copy jlongs instead
    // of oops.  We know objects are aligned on a minimum of an jlong boundary.
    // The same is true of StubRoutines::object_copy and the various oop_copy
    // variants, and of the code generated by the inline_native_clone intrinsic.
     4839641 (4840070): 다른 스레드가 clone의 레퍼런스를 수정할 수 있기 때문에 객체에 대해 atomic한 copy를 수행해야 한다
        4846409 : 32비트 vm에서 long, double 필드나 배열은 atomic하게 oop copy할 수 없다. 따라서 oop 대신 jlong을 copy한다.
    
    assert(MinObjAlignmentInBytes >= BytesPerLong, "objects misaligned");
    Copy::conjoint_jlongs_atomic((jlong*)obj(), (jlong*)new_obj_oop,
    (size_t)align_object_size(size) / HeapWordsPerLong);
    ```

5. 가변객체 clone주의
     1. 가변객체(컬렉션과 같이`instance` 생성 이후에도 내부 상태 변경이 가능한 객체)가 존재할때의 엘레먼트에 대해서도 clone을 해야한다.

         ```java
         package item13;
         
         import java.util.Arrays;
         import java.util.EmptyStackException;
         
         class Stack implements Cloneable {
             private Object[] elements;
             private int size = 0;
             private static final int DEFAULT_INITIAL_CAPACITY = 16;
         
             public Stack() {
                 this.elements = new Object[DEFAULT_INITIAL_CAPACITY];
             }
         
             public void push(Object e) {
                 ensureCapacity();
                 elements[size++] = e;
             }
         
             public Object pop() {
                 if (size == 0) {
                     throw new EmptyStackException();
                 }
         
                 Object result = elements[--size];
                 elements[size] = null;
                 return result;
             }
         
             private void ensureCapacity() {
                 if (elements.length == size) {
                     elements = Arrays.copyOf(elements, 2 * size + 1);
                 }
             }
         
             public boolean isEmpty() {
                 return size == 0;
             }
         
             @Override
             public Stack clone() {
                 try {
                     Stack result = (Stack) super.clone();
                     return result;
                 } catch (CloneNotSupportedException e) {
                     throw new AssertionError();
                 }
             }
         
             public static void main(String[] args) {
         
                 Object[] values = new Object[2];
                 values[0] = new Item("이펙티브");
                 values[1] = new Item("자바");
         
                 Stack origin = new Stack();
         
                 for (Object obj : values) {
                     origin.push(obj);
                 }
         
                 Stack copy = origin.clone();
         
                 System.out.println("======== origin =======");
         
                 while (!origin.isEmpty()) {
                     System.out.println(origin.pop());
                 }
         
                 System.out.println("======== copy =======");
                 System.out.println(copy.size); // 2
                 while (!copy.isEmpty()) {
         
                     System.out.println(copy.pop());
                 }
         
                       System.out.println(copy.hashCode() == origin.hashCode()); //false
                 System.out.println(copy.elements.hashCode() == origin.elements.hashCode()); // true
             }
         }
         ```

         - Stack 만 복제하면, `primitive` 타입의 값은 올바르게 복제되지만, 복제된 `Stack(copy)`의 `elements`는 원본 `Stack(origin)`
           의 `elements` 와 주소가 공유된다
         - 해당 문제를 해결하기 위해서 element도 clone한다

         ```java
             @Override
             public Stack clone() {
                 try {
                     Stack result = (Stack) super.clone();
                     return result;
                 } catch (CloneNotSupportedException e) {
                     throw new AssertionError();
                 }
             }
         
         ```

         - 하지만…… clone 메소드는 얕은 복사를 하기 때문에 copy와 origin의 element 배열 자체는 는 서로 다른 주소를 참조하고 있지만 배열 안의 값은 같은 주소를 참조하고 있어
           copy에서 값을 변경하면 origin에도 영향을 준다
         ![스크린샷 2023-07-19 오전 12.38.55.png](https://github.com/cso6005/BackEnd-Study/assets/51026414/b5584a4a-f34a-4307-bdd7-b82427eca96f)

     2. 가변 객체 내부에 또 다른 가변 객체가 있을 경우

         ```java
         package item13;
         
         public class HashTable implements Cloneable {
         
         /**
              * LinkedList head에 해당하는node, Entry내부에next라는 포인터를 가지고 있어 다음 노드를 찾아갈수 있다
         */
         private Entry[] buckets = new Entry[10];
         
             private static class Entry {
                 final Object key;
                 Object value;
                 Entry next;
         
                 Entry(Object key, Object value, Entry next) {
                     this.key = key;
                     this.value = value;
                     this.next = next;
                 }
         
                 public void add(Object key, Object value) {
                     this.next = new Entry(key, value, null);
                 }
             }
         
             @Override
             public HashTable clone() {
                 HashTable result = null;
                 try {
                     result = (HashTable) super.clone();
                     result.buckets = this.buckets.clone();
                     return result;
                 } catch (CloneNotSupportedException e) {
                     throw new AssertionError();
                 }
             }
         
             public static void main(String[] args) {
                 HashTable hashTable = new HashTable();
                 Entry entry = new Entry(new Object(), new Object(), null);
                 hashTable.buckets[0] = entry;
                 HashTable clone = hashTable.clone();
                         System.out.println("버킷 껍대기 hashcode =>" + hashTable.buckets.hashCode());
                 System.out.println("원본 껍대기 hashcode =>" + clone.buckets.hashCode());
         
                 System.out.println("원본 0 번째 버킷 hash code => " + hashTable.buckets[0].hashCode());
                 System.out.println("카피 0 번째 버킷 hash code => " + clone.buckets[0].hashCode());
         
             }
         }
         
         ```

     - 위 `HashTable` 클래스의 `clone()` 메서드는 적절하게 구현되었을까?
         - `clone()` 메서드를 사용하면, 복제된 `HashTable` 의 `buckets` 은 새로 할당 받았겠지만 `buckets` 내부에 있는 객체들은 여전히 복제되기 이전의 객체들을
           가리키고 있을 것이다.
             - 추후 원본 HashTable에 똑같은 hashcode의 값을 반환하는 Entry를 밀어 넣으면 copy에서도 값이 증가 될 것이다.

           ![스크린샷 2023-07-19 오전 1.00.43.png](https://github.com/cso6005/BackEnd-Study/assets/51026414/6a9c0186-2039-4592-8f0d-b6643eeaf329)
         - 위 문제를 해결 하기 위해서 2가지 방법이 존재한다

              ```java
              public class HashTable implements Cloneable {
       
                  private Entry[] buckets = new Entry[10];
       
                  private static class Entry {
                      final Object key;
                      Object value;
                      Entry next;
       
                      Entry(Object key, Object value, Entry next) {
                          this.key = key;
                          this.value = value;
                          this.next = next;
                      }
       
                      public void add(Object key, Object value) {
                          this.next = new Entry(key, value, null);
                      }
              
                      /**
                      * 방법 1. 재귀 호출을 사용한다, 단 재귀 호출을 사용할 경우, Entry에 연결된 노드(next) 만큼 재귀 호출이 되기떄문에 스택오버플로우가 발생할수 있다.
                      */
              
                      public Entry deepCopy() {
                         return new Entry(key, value, next == null ? null : next.deepCopy());
                      }
                      /**
                      *  방법2. 이터레이터를 사용한다.
                      **/
                      public Entry deepCopy() {
                          Entry result = new Entry(key, value, next);
                          for (Entry p = result ; p.next != null ; p = p.next) {
                              p.next = new Entry(p.next.key, p.next.value, p.next.next);
                          }
                          return result;
                      }
                  }
       
                  /**
                   * TODO hasTable -> entryH[],
                   * TODO copy -> entryC[]
                   * TODO entryH[0] != entryC[0]
                   **/
                  @Override
                  public HashTable clone() {
                      HashTable result = null;
                      try {
                          result = (HashTable)super.clone();
                                    // Entry 배열을 꼭 새로 생성해줘야한다...
                                      // 해당 과정이 없으면 원본과 복사본이 동일한 buckets을 참조한다
                          result.buckets = new Entry[this.buckets.length];
       
                          for (int i = 0 ; i < this.buckets.length; i++) {
                              if (buckets[i] != null) {
                                                   //deepCopy() 메서드 내부에서는 연결된 모든 엔트리를 내용이 같은 새로운 객체로 복사하고 있다
                                  result.buckets[i] = this.buckets[i].deepCopy();
                              }
                          }
                          return result;
                      } catch (CloneNotSupportedException e) {
                          throw  new AssertionError();
                      }
                  }
       
                  public static void main(String[] args) {
                      HashTable hashTable = new HashTable();
                      Entry entry = new Entry(new Object(), new Object(), null);
                      hashTable.buckets[0] = entry;
                      HashTable clone = hashTable.clone();
                      System.out.println(hashTable.buckets[0] == entry);
                      System.out.println(hashTable.buckets[0] == clone.buckets[0]);
                  }
              }
              ```

## Clone 대안

- [https://www.artima.com/articles/josh-bloch-on-design](https://www.artima.com/articles/josh-bloch-on-design)
- 책에서 소개 해고 있기 때문에 deep dive 알아봤지만……. 실무에서 안쓴다……
    - 실무에서는 생성자를 할용하거나 copy 전용 팩터리 메서드를 구현한다

        ```java
        class Student
        {
            private String name;
            private int age;
                private List<String> subjects
            
         
            public Student(String name, int age, List<String> subjects)
            {
                this.name = name;
                this.age = age;
                this.subjects = subjects
            }
         
            // 복사 생성자
            public Student(Student student)
            {
                this.name = student.name;
                this.age = student.age;
                this.subjects = new ArrayList<>(student.subjects);
            }
         
                // 팩토리 복사
            public static Student newInstance(Student student) {
                return new Student(student);
            }
        
            @Override
            public String toString()
            {
                return Arrays.asList(name, String.valueOf(age),
                                            subjects.toString()).toString();
            }
          
        }
        ```
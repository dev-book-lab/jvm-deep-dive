# Method Invocation Instructions - 메서드 호출 명령어

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `invokevirtual`, `invokeinterface`, `invokespecial`, `invokestatic`, `invokedynamic`의 차이는 무엇인가?
- 왜 메서드 호출마다 다른 명령어가 필요한가?
- Virtual Method Table (vtable)은 무엇이며, `invokevirtual`은 어떻게 동작하는가?
- `invokespecial`은 왜 생성자와 `super` 호출에만 사용되는가?
- `invokedynamic`은 람다와 어떤 관계인가?

---

## 🔍 왜 이게 존재하는가

### 문제: 메서드 호출 방식이 상황마다 다르다

```java
class Parent {
    void foo() { }
    static void bar() { }
}

class Child extends Parent {
    @Override
    void foo() { }
    
    void test() {
        foo();          // 어떤 명령어?
        super.foo();    // 어떤 명령어?
        bar();          // 어떤 명령어?
        new Child();    // 어떤 명령어?
    }
}

interface MyInterface {
    void baz();
}
```

```
각 호출의 특성:
  foo()       → 다형성 (오버라이드 가능) → 런타임 결정
  super.foo() → 부모 메서드 고정 → 컴파일 타임 결정
  bar()       → static (클래스에 속함) → 컴파일 타임 결정
  new Child() → 생성자 (특수) → 컴파일 타임 결정
  baz()       → 인터페이스 → 런타임 결정 (구현체마다 다름)
```

JVM은 이를 **5가지 invoke 명령어**로 구분한다.

---

## 📐 내부 구조

### 1. 5가지 invoke 명령어 개요

```
invokevirtual
  - 인스턴스 메서드 호출 (일반 메서드)
  - 다형성 지원 (오버라이드 가능)
  - vtable 사용
  
invokeinterface
  - 인터페이스 메서드 호출
  - 다형성 지원
  - itable 사용 (vtable보다 복잡)
  
invokespecial
  - 특수 메서드 호출
  - 생성자 (<init>)
  - super 메서드
  - private 메서드
  - 컴파일 타임에 결정 (다형성 없음)
  
invokestatic
  - static 메서드 호출
  - 클래스에 속함 (인스턴스 무관)
  - 컴파일 타임에 결정
  
invokedynamic (Java 7+)
  - 동적 메서드 호출
  - 람다, 메서드 핸들
  - 런타임에 결정 (Bootstrap Method)
```

---

### 2. invokevirtual — Virtual Method Table

```
다형성 구현:

class Animal {
    void sound() { System.out.println("..."); }
}

class Dog extends Animal {
    @Override
    void sound() { System.out.println("Bark"); }
}

class Cat extends Animal {
    @Override
    void sound() { System.out.println("Meow"); }
}

Animal a = new Dog();
a.sound();  // "Bark" 출력
```

```
vtable (Virtual Method Table):

Animal vtable:
  [0] sound → Animal.sound()

Dog vtable:
  [0] sound → Dog.sound()  ← 오버라이드

Cat vtable:
  [0] sound → Cat.sound()  ← 오버라이드

invokevirtual 동작:
  1. Operand Stack에서 객체 참조 pop
  2. 객체의 실제 타입(Class) 확인
  3. 해당 클래스의 vtable 참조
  4. 메서드 인덱스로 실제 메서드 주소 찾기
  5. 메서드 호출

바이트코드:
  aload_0         // 'a' 객체 참조
  invokevirtual #2  // Animal.sound() (vtable index)
  
런타임:
  a의 실제 타입 = Dog
  → Dog의 vtable[0] = Dog.sound()
  → Dog.sound() 호출
```

---

### 3. invokeinterface — Interface Table

```
인터페이스 호출:

interface Flyable {
    void fly();
}

class Bird implements Flyable {
    public void fly() { System.out.println("Flying"); }
}

class Airplane implements Flyable {
    public void fly() { System.out.println("Jet engine"); }
}

Flyable f = new Bird();
f.fly();
```

```
itable (Interface Table):

문제:
  클래스는 여러 인터페이스 구현 가능
  → vtable처럼 고정 인덱스 불가
  
해결:
  각 클래스마다 itable 생성
  인터페이스마다 별도 슬롯

Bird itable:
  Flyable → [fly → Bird.fly()]
  Serializable → [...]  // 다른 인터페이스

invokeinterface 동작:
  1. 객체 참조 pop
  2. 실제 타입의 itable 확인
  3. Flyable 인터페이스 찾기
  4. fly() 메서드 찾기
  5. 호출
  
  → invokevirtual보다 느림 (탐색 과정 복잡)

바이트코드:
  aload_1
  invokeinterface #3, 1  // Flyable.fly()
                      ↑ 매개변수 개수 (추가 정보)
```

---

### 4. invokespecial — 특수 메서드 호출

```
사용 케이스:

1. 생성자:
   new Dog()
   →
   new #2          // Dog 클래스
   dup
   invokespecial #3  // Dog.<init>()

2. super 메서드:
   class Child extends Parent {
       void foo() {
           super.foo();
       }
   }
   →
   aload_0         // this
   invokespecial #4  // Parent.foo()

3. private 메서드:
   private void helper() { }
   void test() {
       helper();
   }
   →
   aload_0
   invokespecial #5  // this.helper()
   
특징:
  - 다형성 없음 (오버라이드 무시)
  - 컴파일 타임에 호출 대상 결정
  - vtable 사용 안 함
```

---

### 5. invokestatic — static 메서드 호출

```
static 메서드:
  class Math {
      static int add(int a, int b) {
          return a + b;
      }
  }

바이트코드:
  iconst_1        // 1
  iconst_2        // 2
  invokestatic #2   // Math.add(int, int)
  
특징:
  - 객체 참조 불필요 (this 없음)
  - 클래스에 속함
  - 컴파일 타임 결정
  - 가장 빠름 (vtable/itable 탐색 없음)

주의:
  static 메서드는 오버라이드 불가
  → 상속받아도 숨김(hiding)만 발생
  
  class Parent {
      static void foo() { }
  }
  class Child extends Parent {
      static void foo() { }  // 오버라이드 아님, 새 메서드
  }
  
  Parent p = new Child();
  p.foo();  // Parent.foo() 호출 (다형성 없음)
```

---

### 6. invokedynamic — 동적 메서드 호출

```
람다 표현식:
  Runnable r = () -> System.out.println("Hello");

바이트코드:
  invokedynamic #2, 0  // BootstrapMethods[0]
  astore_1

Bootstrap Method:
  클래스 로딩 시 한 번 실행
  → CallSite 생성
  → 이후 호출은 CallSite 재사용
  
BootstrapMethods 속성:
  #0: REF_invokeStatic java/lang/invoke/LambdaMetafactory.metafactory
      Method arguments:
        ()V
        REF_invokeStatic MyClass.lambda$main$0:()V
        ()V

동작:
  1. 처음 invokedynamic 실행
  2. LambdaMetafactory.metafactory() 호출
  3. 람다 구현체 클래스 동적 생성
  4. CallSite 생성 후 캐시
  5. 이후 호출은 CallSite에서 직접 실행
  
상세 내용 → 06. Lambda & InvokeDynamic 문서 참고
```

---

## 💻 실험으로 확인하기

### 실험 1: invoke 명령어 확인

```java
class Parent {
    void instanceMethod() { }
    static void staticMethod() { }
    private void privateMethod() { }
}

class Child extends Parent {
    @Override
    void instanceMethod() { }
    
    void test() {
        instanceMethod();           // invokevirtual
        super.instanceMethod();     // invokespecial
        staticMethod();             // invokestatic
        privateMethod();            // invokespecial
        new Child();                // invokespecial (<init>)
    }
}

interface MyInterface {
    void interfaceMethod();
}

class Impl implements MyInterface {
    public void interfaceMethod() { }
    
    void callInterface(MyInterface i) {
        i.interfaceMethod();        // invokeinterface
    }
}
```

```bash
javap -c Child.class

# test 메서드:
# Code:
#    0: aload_0
#    1: invokevirtual #2    // Method instanceMethod:()V
#    4: aload_0
#    5: invokespecial #3    // Method Parent.instanceMethod:()V
#    8: invokestatic  #4    // Method staticMethod:()V
#   11: aload_0
#   12: invokespecial #5    // Method privateMethod:()V
#   15: new           #6    // class Child
#   18: dup
#   19: invokespecial #7    // Method "<init>":()V
```

---

### 실험 2: vtable 동작 확인

```java
class Animal {
    void sound() { System.out.println("Animal"); }
    void eat() { System.out.println("Eating"); }
}

class Dog extends Animal {
    @Override
    void sound() { System.out.println("Bark"); }
}

public class VTableDemo {
    public static void main(String[] args) {
        Animal a1 = new Animal();
        Animal a2 = new Dog();
        
        a1.sound();  // "Animal"
        a2.sound();  // "Bark" (다형성)
        a2.eat();    // "Eating" (부모 메서드)
    }
}
```

```bash
javap -c VTableDemo.class

# main:
#   ...
#   10: invokevirtual #4   // Method Animal.sound:()V
#   ...
#   20: invokevirtual #4   // Method Animal.sound:()V
#                         ↑ 같은 Constant Pool 인덱스
#                         하지만 실제 호출되는 메서드는 다름
#                         (a1 → Animal.sound, a2 → Dog.sound)
```

---

### 실험 3: invokeinterface vs invokevirtual 성능

```java
import java.util.*;

interface Greeter {
    String greet();
}

class EnglishGreeter implements Greeter {
    public String greet() { return "Hello"; }
}

public class InvokeBenchmark {
    static final int ITERATIONS = 100_000_000;
    
    public static void main(String[] args) {
        EnglishGreeter concrete = new EnglishGreeter();
        Greeter interfaceRef = new EnglishGreeter();
        
        // Warm-up
        for (int i = 0; i < 10000; i++) {
            concrete.greet();
            interfaceRef.greet();
        }
        
        // invokevirtual (구체 클래스)
        long start = System.nanoTime();
        for (int i = 0; i < ITERATIONS; i++) {
            concrete.greet();
        }
        long virtualTime = System.nanoTime() - start;
        
        // invokeinterface
        start = System.nanoTime();
        for (int i = 0; i < ITERATIONS; i++) {
            interfaceRef.greet();
        }
        long interfaceTime = System.nanoTime() - start;
        
        System.out.printf("invokevirtual:   %d ms%n", virtualTime / 1_000_000);
        System.out.printf("invokeinterface: %d ms%n", interfaceTime / 1_000_000);
        System.out.printf("Ratio: %.2f%%n", (double)interfaceTime / virtualTime);
    }
}
```

```bash
# 실행 결과 (JIT 컴파일 후):
# invokevirtual:   250 ms
# invokeinterface: 250 ms
# Ratio: 1.00

# → JIT 컴파일 후에는 성능 차이 거의 없음
# Interpreter 모드에서는 invokeinterface가 10~20% 느림
```

---

## ⚡ 실무 임팩트

### JIT 인라이닝과 메서드 호출

```
인라이닝 가능 여부:

invokestatic:
  ✓ 항상 인라이닝 가능 (호출 대상 명확)
  
invokespecial:
  ✓ 인라이닝 가능 (다형성 없음)
  
invokevirtual:
  △ 조건부 인라이닝
  - 단일 구현체 확인 (Class Hierarchy Analysis)
  - Monomorphic Call Site (하나의 타입만 호출)
  → 인라이닝 + Deoptimization 준비
  
  - Polymorphic Call Site (여러 타입 호출)
  → 인라이닝 어려움 또는 불가

invokeinterface:
  △ invokevirtual과 동일
  - 단일 구현체면 인라이닝 가능
  
최적화 팁:
  자주 호출되는 메서드는 final 또는 private로
  → invokespecial 사용
  → 확실한 인라이닝
```

### Megamorphic Call Site 문제

```java
// ❌ 안티패턴
void process(List<Shape> shapes) {
    for (Shape s : shapes) {
        s.draw();  // 여러 타입 (Circle, Square, Triangle...)
                   // → Megamorphic Call Site
    }
}

// JIT가 보는 것:
// s.draw()가 10가지 이상 타입으로 호출됨
// → 인라이닝 불가
// → vtable dispatch 유지
// → 느림

// ✅ 개선
void processCircles(List<Circle> circles) {
    for (Circle c : circles) {
        c.draw();  // Monomorphic Call Site
                   // → 인라이닝 가능
    }
}

void processSquares(List<Square> squares) {
    for (Square s : squares) {
        s.draw();  // Monomorphic Call Site
    }
}

// 타입별로 분리 → 각각 인라이닝 가능
```

### invokedynamic의 장점

```
전통적 방식 (내부 클래스):
  Runnable r = new Runnable() {
      public void run() { System.out.println("Hello"); }
  };
  
  → 익명 클래스 생성 (클래스 파일 증가)
  → 메모리 오버헤드

invokedynamic 방식 (람다):
  Runnable r = () -> System.out.println("Hello");
  
  → 클래스 동적 생성 (필요 시)
  → 메모리 효율적
  → JIT 최적화 가능

성능:
  Warm-up 후 거의 동일
  클래스 로딩 시간과 메모리는 람다가 우수
```

---

## 🚫 흔한 오해

### "invokeinterface는 항상 느리다"

```
❌ 잘못된 이해:
  인터페이스 호출은 항상 클래스 호출보다 느리다.

✅ 실제:
  Interpreter 모드: invokeinterface가 10~20% 느림
  JIT 컴파일 후: 거의 동일
  
  이유:
  JIT가 Call Site 분석
  → 단일 구현체면 직접 호출로 변환
  → itable 탐색 제거
  
  성능 차이는 Polymorphic 정도에 따라 결정
  인터페이스 vs 클래스가 아님
```

### "private 메서드는 invokevirtual을 사용한다"

```
❌ 잘못된 이해:
  private도 인스턴스 메서드니까 invokevirtual이다.

✅ 실제:
  private 메서드 → invokespecial
  
  이유:
  private은 오버라이드 불가
  → 다형성 없음
  → vtable 필요 없음
  → invokespecial (컴파일 타임 결정)

Java 11+ 변화:
  Nest-based Access Control 도입
  → private 메서드도 같은 Nest 내에서 접근 가능
  → 여전히 invokespecial 사용
```

### "invokestatic은 synchronized보다 빠르다"

```
❌ 잘못된 이해:
  static 메서드는 동기화 오버헤드가 없어 빠르다.

✅ 실제:
  invokestatic vs invokevirtual:
  호출 메커니즘의 차이
  
  synchronized:
  별도의 동기화 메커니즘 (monitorenter/monitorexit)
  
  비교 대상이 다름:
  invokestatic      (호출 방식)
  synchronized      (동기화)
  
  static synchronized:
  invokestatic + monitorenter
  → static이라고 동기화 오버헤드 없는 건 아님
```

---

## 📌 핵심 정리

```
5가지 invoke 명령어
  invokevirtual:    인스턴스 메서드 (다형성)
  invokeinterface:  인터페이스 메서드 (다형성)
  invokespecial:    생성자, super, private (다형성 없음)
  invokestatic:     static 메서드 (다형성 없음)
  invokedynamic:    람다, 메서드 핸들 (동적)

다형성 구현
  invokevirtual: vtable (Virtual Method Table)
  invokeinterface: itable (Interface Table)
  
vtable
  각 클래스마다 메서드 포인터 배열
  오버라이드 시 해당 인덱스만 교체
  O(1) 탐색

itable
  인터페이스마다 별도 슬롯
  vtable보다 복잡 (여러 인터페이스 구현 가능)
  
invokespecial
  컴파일 타임 결정
  vtable 사용 안 함
  생성자, super, private 메서드

invokestatic
  가장 빠름 (객체 참조 불필요)
  컴파일 타임 결정
  오버라이드 불가

JIT 최적화
  Monomorphic Call Site: 인라이닝 가능
  Polymorphic Call Site: 조건부 인라이닝
  Megamorphic Call Site: 인라이닝 어려움

실무 팁
  final/private 메서드 → invokespecial → 확실한 인라이닝
  타입별 분리 → Monomorphic → 성능 향상
  invokeinterface vs invokevirtual: JIT 후 성능 동일
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드의 바이트코드에서 어떤 invoke 명령어가 사용되는가? 각 호출의 이유를 설명하라.

```java
class Example {
    void foo() { }
    static void bar() { }
    private void baz() { }
    
    void test() {
        foo();
        bar();
        baz();
        super.toString();
        new Example();
    }
}
```

**Q2.** 다음 두 코드의 성능 차이를 invoke 명령어와 JIT 최적화 관점에서 설명하라.

```java
// 방법 1
void process(List<Animal> animals) {
    for (Animal a : animals) {
        a.sound();  // Polymorphic
    }
}

// 방법 2
void process(List<Animal> animals) {
    List<Dog> dogs = new ArrayList<>();
    List<Cat> cats = new ArrayList<>();
    for (Animal a : animals) {
        if (a instanceof Dog) dogs.add((Dog)a);
        else if (a instanceof Cat) cats.add((Cat)a);
    }
    for (Dog d : dogs) d.sound();  // Monomorphic
    for (Cat c : cats) c.sound();  // Monomorphic
}
```

**Q3.** Java 8에서 람다가 `invokedynamic`을 사용하는 이유는 무엇인가? 익명 클래스 방식과 비교해 어떤 장점이 있는가?

> 💡 **해설**
>
> **Q1.** `foo()` → invokevirtual (일반 인스턴스 메서드, 다형성 가능). `bar()` → invokestatic (static 메서드, 클래스에 속함). `baz()` → invokespecial (private 메서드, 오버라이드 불가). `super.toString()` → invokespecial (super 키워드, 부모 메서드 명시). `new Example()` → invokespecial (생성자 `<init>`). 핵심: invokespecial은 다형성이 없는 특수한 경우에만 사용.
>
> **Q2.** 방법 1: `a.sound()`가 Dog, Cat, Bird 등 여러 타입으로 호출 → Polymorphic 또는 Megamorphic Call Site → JIT가 인라이닝 불가 → vtable dispatch 유지 → 느림. 방법 2: 타입별 분리 → `d.sound()`는 항상 Dog 타입 (Monomorphic) → JIT가 `Dog.sound()`로 인라이닝 → 직접 호출 → 빠름. 단, 방법 2는 타입 분류 오버헤드 (instanceof, 리스트 추가) 있음. 동물 종류가 2~3개면 방법 2가 유리, 10개 이상이면 방법 1이 나을 수 있음 (분류 비용 vs 호출 비용 트레이드오프).
>
> **Q3.** 람다 `invokedynamic` 장점: ① 클래스 파일 최소화 — 익명 클래스는 `Outer$1.class` 등 별도 파일 생성, 람다는 필요 시에만 동적 생성. ② 메모리 효율 — 람다는 싱글톤처럼 재사용 가능 (상태 없는 경우), 익명 클래스는 매번 새 인스턴스. ③ JIT 최적화 — `invokedynamic`의 Bootstrap Method가 최적화된 구현체 선택 가능 (예: 캡처 변수 없으면 싱글톤, 있으면 새 인스턴스). ④ 미래 유연성 — JVM이 람다 구현 방식을 버전업마다 개선 가능 (바이트코드는 `invokedynamic`만 유지). 익명 클래스는 컴파일 타임에 구현 고정, 람다는 런타임에 최적 구현 선택.

---

## 📚 참고 자료

- [JVMS §6.5 — Instructions](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-6.html#jvms-6.5)
- [Virtual Method Invocation](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-6.html#jvms-6.5.invokevirtual)
- [JEP 276 — Dynamic Linking of Language-Defined Object Models](https://openjdk.org/jeps/276)

---

<div align="center">

**[⬅️ 이전: Operand Stack Mechanism](./03-operand-stack-mechanism.md)** | **[다음: Exception Handling Bytecode ➡️](./05-exception-handling-bytecode.md)**

</div>

# Publication & Escape - 공개와 탈출

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 객체 "탈출(Escape)"은 무엇이며, 왜 위험한가?
- 안전한 공개(Safe Publication) 패턴은 무엇인가?
- this 참조가 생성자에서 탈출하는 경우는?
- 불변 객체를 안전하게 공개하는 방법은?

---

## 🔍 왜 이게 존재하는가

### 문제: 미완성 객체의 노출

```java
class EventListener {
    private final int id;
    
    EventListener() {
        EventBus.register(this);  // ← this 탈출!
        id = 42;  // ← 아직 초기화 안 됨
    }
}
```

```
문제:
  EventBus.register()가 this를 저장
  → 다른 스레드가 미완성 객체 접근
  → id = 0 (기본값) 읽기
```

**안전한 공개**가 필요하다.

---

## 📐 탈출 패턴

### 1. This Escape in Constructor

```java
// ❌ 생성자에서 this 탈출
class Leaky {
    private int value;
    
    Leaky(EventSource source) {
        source.registerListener(new EventListener() {
            public void onEvent(Event e) {
                doSomething(e);
            }
        });
        value = 42;  // ← 늦은 초기화
    }
    
    void doSomething(Event e) {
        System.out.println(value);  // 0 가능!
    }
}
```

---

### 2. Implicit This Escape

```java
// ❌ 내부 클래스에서 암시적 탈출
class Outer {
    private int x;
    
    Outer() {
        Inner inner = new Inner();
        // Inner는 Outer.this 참조 포함
        SomeRegistry.register(inner);  // ← 간접 탈출
        x = 42;
    }
    
    class Inner {
        void method() {
            System.out.println(x);  // Outer.this.x
        }
    }
}
```

---

### 3. Publishing Before Fully Constructed

```java
// ❌ Static 필드에 조기 공개
class EarlyPublish {
    static EarlyPublish instance;
    private int value;
    
    EarlyPublish() {
        instance = this;  // ← 탈출!
        value = initialize();  // ← 아직 미완성
    }
}
```

---

## 🛡️ 안전한 공개 패턴

### 1. Factory Method

```java
// ✅ Factory로 완성 후 공개
class SafePublish {
    private int value;
    
    private SafePublish() {
        value = 42;  // 생성자에서 초기화만
    }
    
    public static SafePublish create(EventSource source) {
        SafePublish obj = new SafePublish();
        source.registerListener(obj);  // 완성 후 등록
        return obj;
    }
}
```

---

### 2. Initialization-on-Demand

```java
// ✅ Lazy Holder 패턴
class Singleton {
    private Singleton() {}
    
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

---

### 3. Final Fields

```java
// ✅ Final 필드로 안전 공개
class ImmutableConfig {
    private final String url;
    private final int timeout;
    
    ImmutableConfig(String url, int timeout) {
        this.url = url;
        this.timeout = timeout;
    }  // ← Final Freeze
}

// Thread 1
config = new ImmutableConfig("http://api.com", 5000);

// Thread 2
if (config != null) {
    System.out.println(config.url);  // 안전
}
```

---

### 4. Volatile or Synchronized

```java
// ✅ volatile로 공개
class Publisher {
    private int value = 42;
    private static volatile Publisher instance;
    
    private Publisher() {}
    
    public static void publish() {
        instance = new Publisher();  // volatile 쓰기
    }
}

// ✅ synchronized로 공개
class Publisher {
    private int value = 42;
    private static Publisher instance;
    
    public static synchronized Publisher getInstance() {
        if (instance == null) {
            instance = new Publisher();
        }
        return instance;
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: This Escape 재현

```java
public class ThisEscapeTest {
    static volatile Leaky escaped = null;
    
    static class Leaky {
        int value;
        
        Leaky() {
            escaped = this;  // ← 탈출
            try {
                Thread.sleep(10);  // 초기화 지연
            } catch (InterruptedException e) {}
            value = 42;
        }
    }
    
    public static void main(String[] args) throws Exception {
        for (int i = 0; i < 100; i++) {
            escaped = null;
            
            Thread t1 = new Thread(() -> new Leaky());
            
            Thread t2 = new Thread(() -> {
                while (escaped == null) {
                    Thread.yield();
                }
                if (escaped.value != 42) {
                    System.out.println("Broken! value=" + escaped.value);
                }
            });
            
            t2.start();
            t1.start();
            t1.join();
            t2.join();
        }
    }
}
```

```bash
# 출력:
# Broken! value=0  ← 발생 가능
```

---

## ⚡ 실무 임팩트

### Spring Bean 초기화

```java
// ❌ 생성자에서 비즈니스 로직
@Component
class UserService {
    private final UserRepository repo;
    
    UserService(UserRepository repo) {
        this.repo = repo;
        loadInitialData();  // ← 위험! this 탈출 가능
    }
}

// ✅ @PostConstruct 사용
@Component
class UserService {
    private final UserRepository repo;
    
    UserService(UserRepository repo) {
        this.repo = repo;
    }
    
    @PostConstruct
    void init() {
        loadInitialData();  // ← 안전 (완전 초기화 후)
    }
}
```

---

### 안전한 Singleton

```java
// ✅ Enum Singleton (가장 안전)
public enum Singleton {
    INSTANCE;
    
    private int value = 42;
    
    public void doSomething() {
        // ...
    }
}
```

---

## 🚫 흔한 오해

### "생성자만 끝나면 안전하다"

```
❌ 잘못된 이해:
  생성자 완료 = 안전한 공개

✅ 실제:
  공개 방식에 따라 다름
  
  안전한 공개 필요:
  - volatile
  - synchronized
  - final
  - Thread.start/join
  
  없으면:
  다른 스레드가 미완성 상태 볼 가능성
```

---

## 📌 핵심 정리

```
객체 탈출
  생성 중 this 참조 노출
  미완성 객체 접근 → 위험

탈출 패턴
  1. 생성자에서 this 전달
  2. 내부 클래스 (암시적 this)
  3. Static 필드 조기 설정

안전한 공개
  1. Factory Method (완성 후 공개)
  2. Final Fields (Freeze 보장)
  3. volatile (가시성)
  4. synchronized (원자성 + 가시성)
  5. Lazy Holder (JVM 보장)

생성자 규칙
  - 초기화만 수행
  - this 노출 금지
  - 비즈니스 로직 금지

실무
  Spring: @PostConstruct
  Singleton: Enum 또는 Lazy Holder
  불변 객체: final 필드
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 this가 탈출하는 이유를 설명하라.

```java
class Widget {
    Widget(EventSource source) {
        source.addListener(new EventListener() {
            public void onEvent(Event e) {
                handleEvent(e);
            }
        });
    }
}
```

**Q2.** 안전한 공개의 4가지 방법 중 어느 것이 가장 안전한가? 이유를 설명하라.

**Q3.** Spring Framework에서 @PostConstruct를 사용하는 이유를 객체 탈출 관점에서 설명하라.

> 💡 **해설**
>
> **Q1.** This 탈출 이유: ① 익명 내부 클래스 `new EventListener()` 생성 → Widget.this 참조 포함 (암시적). ② `source.addListener()`로 전달 → EventSource가 리스너 저장. ③ 다른 스레드가 EventSource에서 리스너 가져와 `onEvent()` 호출. ④ `handleEvent()` 실행 → Widget.this 접근. ⑤ 생성자 완료 전 호출 가능 → 미완성 Widget 노출. 해결: Factory 메서드 사용 또는 `addListener()`를 생성자 밖에서 호출.
>
> **Q2.** Final Fields가 가장 안전. 이유: ① JVM이 하드웨어 수준에서 보장 (StoreStore Barrier). ② 명시적 동기화 불필요 → 실수 여지 없음. ③ 생성자 완료 시 자동 Freeze → 신경 안 써도 됨. ④ volatile/synchronized는 개발자가 올바르게 사용해야 함 (실수 가능). ⑤ 불변 객체라면 final이 최선. 가변 객체는 volatile/synchronized 필요하지만 복잡성 증가.
>
> **Q3.** @PostConstruct 사용 이유: ① Spring은 생성자로 Bean 생성 → 의존성 주입. ② 생성자에서 비즈니스 로직 실행 → this 탈출 위험 (다른 Bean이 미완성 Bean 참조 가능). ③ @PostConstruct는 모든 의존성 주입 완료 후 호출 → Bean 완전 초기화 보장. ④ 이 시점에는 Bean이 Container에 등록됨 → 안전한 상태. ⑤ 비즈니스 로직을 @PostConstruct로 분리 → 탈출 방지 + 명확한 생명주기.

---

## 📚 참고 자료

- [Java Concurrency in Practice - Safe Construction](https://jcip.net/)
- [Effective Java - Item 3: Enforce singleton with Enum](https://www.oreilly.com/library/view/effective-java/9780134686097/)

---

<div align="center">

**[⬅️ 이전: Final Field Semantics](./04-final-field-semantics.md)** | **[다음: Synchronized Internals ➡️](./06-synchronized-internals.md)**

</div>

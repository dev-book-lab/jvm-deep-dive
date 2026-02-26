# Final Field Semantics - Final 필드 의미론

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- final 필드는 어떻게 스레드 안전성을 보장하는가?
- 생성자 완료 후 final 필드가 보장되는 범위는?
- final 필드의 "동결(Freeze)" 의미론은 무엇인가?
- 언제 final로 충분하고, 언제 volatile이 필요한가?

---

## 🔍 왜 이게 존재하는가

### 문제: 불변 객체의 안전한 공개

```java
class ImmutablePoint {
    private final int x;
    private final int y;
    
    ImmutablePoint(int x, int y) {
        this.x = x;
        this.y = y;
    }
}

// Thread 1
point = new ImmutablePoint(1, 2);

// Thread 2
if (point != null) {
    System.out.println(point.x);  // 1 보장?
}
```

final은 **불변 객체의 안전한 공개**를 보장한다.

---

## 📐 Final 필드 보장

### 1. Final Field Freeze

```
생성자 완료 시 final 필드 "동결"

class Point {
    private final int x;
    private final int y;
    
    Point(int x, int y) {
        this.x = x;
        this.y = y;
    }  // ← 여기서 Freeze
}

보장:
  생성자 완료 후 final 필드 값
  → 모든 스레드가 일관되게 봄
  (volatile 없이도)

Memory Barrier:
  생성자 끝에 StoreStore Barrier 삽입
  → 캐시 플러시
```

---

### 2. Safe Publication without Synchronization

```java
// ✅ 안전 (final)
class Config {
    private final String url;
    private final int timeout;
    
    Config(String url, int timeout) {
        this.url = url;
        this.timeout = timeout;
    }
}

// Thread 1
config = new Config("http://api.com", 5000);

// Thread 2
if (config != null) {
    System.out.println(config.url);  // "http://api.com" 보장
}
```

```java
// ❌ 불안전 (non-final)
class Config {
    private String url;
    private int timeout;
    
    Config(String url, int timeout) {
        this.url = url;
        this.timeout = timeout;
    }
}

// Thread 2
if (config != null) {
    System.out.println(config.url);  // null 가능!
}
```

---

### 3. Final의 한계

```
Final이 보장하지 않는 것:

1. 참조 객체의 내부 변경
   final List<String> list = new ArrayList<>();
   list.add("item");  // ← 이건 보장 안 됨

2. 생성자 탈출
   class Leaky {
       final int x;
       Leaky() {
           x = 42;
           register(this);  // ← 탈출 (위험)
       }
   }

3. Reflection 변경
   Field field = obj.getClass().getDeclaredField("x");
   field.setAccessible(true);
   field.set(obj, newValue);  // ← final 무시
```

---

### 4. Final vs Volatile

```
Final:
  - 불변 값
  - 생성자 완료 후 보장
  - 변경 불가
  
Volatile:
  - 가변 값
  - 매 읽기/쓰기 보장
  - 변경 가능

예:

// Final (설정값)
final String API_URL = "http://api.com";

// Volatile (상태)
volatile boolean running = true;
```

---

## 💻 실험으로 확인하기

### 실험 1: Final 안전성

```java
public class FinalSafetyTest {
    static class ImmutablePoint {
        private final int x;
        private final int y;
        
        ImmutablePoint(int x, int y) {
            this.x = x;
            this.y = y;
        }
    }
    
    static class MutablePoint {
        private int x;
        private int y;
        
        MutablePoint(int x, int y) {
            this.x = x;
            this.y = y;
        }
    }
    
    private static ImmutablePoint immutable;
    private static MutablePoint mutable;
    
    public static void main(String[] args) throws Exception {
        // 1000번 반복
        for (int i = 0; i < 1000; i++) {
            immutable = null;
            mutable = null;
            
            Thread t1 = new Thread(() -> {
                immutable = new ImmutablePoint(42, 84);
                mutable = new MutablePoint(42, 84);
            });
            
            Thread t2 = new Thread(() -> {
                ImmutablePoint p1 = immutable;
                if (p1 != null && (p1.x != 42 || p1.y != 84)) {
                    System.out.println("Immutable broken!");
                }
                
                MutablePoint p2 = mutable;
                if (p2 != null && (p2.x != 42 || p2.y != 84)) {
                    System.out.println("Mutable broken! x=" + p2.x + " y=" + p2.y);
                }
            });
            
            t1.start();
            t2.start();
            t1.join();
            t2.join();
        }
    }
}
```

```bash
# 출력:
# Mutable broken! x=0 y=84  ← 발생 가능
# (Immutable는 문제 없음)
```

---

### 실험 2: 생성자 탈출 문제

```java
public class ConstructorEscapeTest {
    static class Leaky {
        final int x;
        
        Leaky() {
            x = 42;
            Registry.register(this);  // ← 탈출!
        }
    }
    
    static class Registry {
        static volatile Leaky instance;
        static void register(Leaky obj) {
            instance = obj;
        }
    }
    
    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(() -> {
            new Leaky();
        });
        
        Thread t2 = new Thread(() -> {
            while (true) {
                Leaky obj = Registry.instance;
                if (obj != null) {
                    if (obj.x != 42) {
                        System.out.println("Broken! x=" + obj.x);
                    }
                    break;
                }
            }
        });
        
        t2.start();
        t1.start();
        t1.join();
        t2.join();
    }
}
```

---

## ⚡ 실무 임팩트

### 불변 객체 패턴

```java
// ✅ 완전 불변
public final class ImmutableConfig {
    private final String url;
    private final int timeout;
    private final List<String> servers;
    
    public ImmutableConfig(String url, int timeout, List<String> servers) {
        this.url = url;
        this.timeout = timeout;
        this.servers = List.copyOf(servers);  // 방어적 복사
    }
    
    public String getUrl() { return url; }
    public int getTimeout() { return timeout; }
    public List<String> getServers() { return servers; }  // 불변 리스트
}
```

---

### 생성자 탈출 방지

```java
// ❌ 탈출
class Publisher {
    final List<String> items;
    
    Publisher() {
        items = new ArrayList<>();
        EventBus.register(this);  // ← 위험! 생성 중 탈출
    }
}

// ✅ 안전
class Publisher {
    final List<String> items;
    
    private Publisher() {
        items = new ArrayList<>();
    }
    
    static Publisher create() {
        Publisher p = new Publisher();
        EventBus.register(p);  // ← 생성 후 등록
        return p;
    }
}
```

---

## 🚫 흔한 오해

### "final은 성능 최적화다"

```
❌ 잘못된 이해:
  final을 쓰면 JIT가 최적화해서 빠르다

✅ 실제:
  성능 차이 거의 없음
  
  final의 진짜 가치:
  - 불변성 보장
  - 스레드 안전성
  - 의도 명확화
  
  성능은 부수 효과
```

---

### "final List는 불변이다"

```
❌ 잘못된 이해:
  final List<String> list
  → list 내용 변경 불가

✅ 실제:
  참조만 불변
  
  final List<String> list = new ArrayList<>();
  list.add("item");  // ← 가능!
  list = new ArrayList<>();  // ← 불가능
  
  진짜 불변:
  final List<String> list = List.of("a", "b");
  또는
  final List<String> list = Collections.unmodifiableList(...);
```

---

## 📌 핵심 정리

```
Final Field 보장
  생성자 완료 후 값 동결
  모든 스레드가 일관되게 봄
  volatile 없이 안전한 공개

Memory Barrier
  생성자 끝: StoreStore Barrier
  → 캐시 플러시

안전한 공개
  final 필드는 동기화 불필요
  참조 != null이면 값 보장

한계
  1. 참조 객체 내부 변경 보장 안 함
  2. 생성자 탈출 위험
  3. Reflection으로 변경 가능

Final vs Volatile
  Final: 불변 값 (설정, 상수)
  Volatile: 가변 값 (상태, 플래그)

실무 패턴
  불변 객체: 모든 필드 final
  방어적 복사: List.copyOf()
  생성자 탈출 방지: Factory 메서드
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드가 스레드 안전하지 않은 이유를 설명하라.

```java
class Config {
    private final List<String> items = new ArrayList<>();
    
    Config(List<String> items) {
        this.items.addAll(items);
    }
}
```

**Q2.** Final 필드의 "Freeze" 의미론이 생성자 탈출 시 깨지는 이유를 설명하라.

**Q3.** 다음 중 final만으로 충분한 것과 추가 동기화가 필요한 것을 구분하라.
- A: 불변 설정 객체 (url, timeout)
- B: 캐시 맵 (final Map<K, V>)
- C: 불변 리스트 (final List.of(...))

> 💡 **해설**
>
> **Q1.** 불안전한 이유: ① items 참조는 final (불변) ✓. ② 하지만 ArrayList 내용은 가변 ✗. ③ 외부에서 `config.items.add("new")` 가능 → 변경됨. ④ 또한 생성자 매개변수 items가 외부에서 변경되면 영향 받음. 해결: `this.items = List.copyOf(items)` (불변 복사) + getter에서 `Collections.unmodifiableList()` 반환.
>
> **Q2.** 생성자 탈출 문제: ① Final Freeze는 생성자 완료 시 발동 (StoreStore Barrier). ② 생성자 중간에 this 공개 → 다른 스레드가 미완성 객체 접근. ③ 예: `x = 42; register(this);` → register() 시점에 x는 설정됐지만 Freeze 전. ④ 다른 스레드가 register된 객체 읽기 → x = 0 (기본값) 볼 가능성. ⑤ Freeze는 생성자 끝에서만 보장되므로, 중간 탈출은 위험.
>
> **Q3.** A (설정 객체): final 충분 — 모든 필드가 불변 기본 타입/String → 생성 후 변경 없음 → 안전한 공개. B (캐시 맵): 추가 동기화 필요 — `final Map`은 참조만 불변, 내용 가변 (`map.put()` 가능) → ConcurrentHashMap 또는 synchronized 필요. C (불변 리스트): final 충분 — `List.of()`는 불변 리스트 → 내용 변경 불가 → 안전.

---

## 📚 참고 자료

- [JLS §17.5 Final Field Semantics](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html#jls-17.5)
- [Safe Publication and Safe Initialization](https://shipilev.net/blog/2014/safe-public-construction/)

---

<div align="center">

**[⬅️ 이전: Volatile Deep Dive](./03-volatile-deep-dive.md)** | **[다음: Publication & Escape ➡️](./05-publication-and-escape.md)**

</div>

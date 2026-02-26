# ThreadLocal Internals - ThreadLocal 내부 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- ThreadLocal은 어떻게 스레드별 저장소를 제공하는가?
- ThreadLocalMap의 내부 구조는 어떻게 되는가?
- 메모리 누수는 왜 발생하며, 어떻게 방지하는가?
- InheritableThreadLocal과의 차이는 무엇인가?

---

## 🔍 왜 이게 존재하는가

### 문제: 스레드별 데이터가 필요하다

```java
// ❌ 공유 변수 (동기화 필요)
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

synchronized (sdf) {
    String result = sdf.format(date);
}

// ✅ 스레드별 인스턴스
ThreadLocal<SimpleDateFormat> sdf = ThreadLocal.withInitial(
    () -> new SimpleDateFormat("yyyy-MM-dd")
);

String result = sdf.get().format(date);  // 동기화 불필요
```

ThreadLocal은 **스레드별 격리된 저장소**다.

---

## 📐 내부 구조

### 1. Thread 클래스 필드

```java
public class Thread {
    // 각 Thread가 가진 Map
    ThreadLocal.ThreadLocalMap threadLocals = null;
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
}
```

---

### 2. ThreadLocalMap 구조

```java
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;
        
        Entry(ThreadLocal<?> k, Object v) {
            super(k);  // WeakReference
            value = v;
        }
    }
    
    private Entry[] table;  // Hash Table
    private int size;
    private int threshold;
}

특징:
  - Open Addressing (Linear Probing)
  - Entry의 key는 WeakReference
  - HashMap과 다름 (체이닝 없음)
```

---

### 3. get() / set() 동작

```java
// ThreadLocal.get()
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = t.threadLocals;
    
    if (map != null) {
        Entry e = map.getEntry(this);  // this = ThreadLocal 인스턴스
        if (e != null) {
            return (T) e.value;
        }
    }
    
    return setInitialValue();
}

// ThreadLocal.set()
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = t.threadLocals;
    
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}

// ThreadLocalMap.set() - Linear Probing
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len - 1);
    
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        
        if (k == key) {
            e.value = value;  // 업데이트
            return;
        }
        
        if (k == null) {
            replaceStaleEntry(key, value, i);  // 재활용
            return;
        }
    }
    
    tab[i] = new Entry(key, value);
    size++;
}
```

---

### 4. WeakReference와 메모리 누수

```
Entry 구조:
  WeakReference<ThreadLocal<?>> → ThreadLocal 인스턴스
  Object value → 실제 데이터

시나리오 1: 정상 (GC됨)
  ThreadLocal<String> tl = new ThreadLocal<>();
  tl.set("value");
  tl = null;  // Strong Reference 제거
  
  → ThreadLocal 인스턴스 GC됨 (WeakReference)
  → Entry의 key = null
  → 다음 set()/get()/remove()에서 정리

시나리오 2: 누수 (GC 안 됨)
  ThreadLocal<BigData> tl = new ThreadLocal<>();
  tl.set(new BigData());  // 100MB
  tl = null;
  
  Thread가 종료 안 함 (Thread Pool)
  → Entry의 value = BigData (Strong Reference)
  → GC 안 됨
  → 메모리 누수!

해결:
  tl.remove();  // 명시적 제거
```

---

## 💻 실험으로 확인하기

### 실험 1: ThreadLocal 기본 동작

```java
public class ThreadLocalTest {
    static ThreadLocal<Integer> threadLocal = ThreadLocal.withInitial(() -> 0);
    
    public static void main(String[] args) throws Exception {
        threadLocal.set(100);
        System.out.println("Main: " + threadLocal.get());  // 100
        
        Thread t1 = new Thread(() -> {
            System.out.println("T1 initial: " + threadLocal.get());  // 0
            threadLocal.set(200);
            System.out.println("T1 after set: " + threadLocal.get());  // 200
        });
        
        Thread t2 = new Thread(() -> {
            System.out.println("T2: " + threadLocal.get());  // 0
        });
        
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        
        System.out.println("Main final: " + threadLocal.get());  // 100
    }
}
```

---

### 실험 2: 메모리 누수 재현

```java
public class ThreadLocalLeakTest {
    static class BigData {
        byte[] data = new byte[10 * 1024 * 1024];  // 10MB
    }
    
    static ThreadLocal<BigData> threadLocal = new ThreadLocal<>();
    
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(10);
        
        for (int i = 0; i < 100; i++) {
            executor.submit(() -> {
                threadLocal.set(new BigData());
                // threadLocal.remove();  ← 제거 안 함 (누수!)
            });
        }
        
        Thread.sleep(1000);
        
        System.gc();
        Thread.sleep(1000);
        
        printMemory();  // 1GB 메모리 사용 (100 × 10MB)
    }
    
    static void printMemory() {
        Runtime rt = Runtime.getRuntime();
        long used = (rt.totalMemory() - rt.freeMemory()) / (1024 * 1024);
        System.out.println("Used: " + used + " MB");
    }
}
```

---

## ⚡ 실무 임팩트

### SimpleDateFormat 안전 사용

```java
// ❌ 공유 (비안전)
static SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

// ✅ ThreadLocal
static ThreadLocal<SimpleDateFormat> sdf = ThreadLocal.withInitial(
    () -> new SimpleDateFormat("yyyy-MM-dd")
);

public String format(Date date) {
    return sdf.get().format(date);
}
```

---

### Transaction Context

```java
public class TransactionContext {
    private static ThreadLocal<Transaction> context = new ThreadLocal<>();
    
    public static void begin() {
        context.set(new Transaction());
    }
    
    public static Transaction get() {
        return context.get();
    }
    
    public static void commit() {
        Transaction tx = context.get();
        if (tx != null) {
            tx.commit();
            context.remove();  // ← 필수!
        }
    }
}
```

---

### Spring @RequestScope

```java
// Spring 내부적으로 ThreadLocal 사용
@Component
@RequestScope  // Request마다 새 인스턴스
public class RequestScopedBean {
    // HTTP Request 동안만 유효
}

// 내부 구현 (간략화):
static ThreadLocal<Map<String, Object>> requestBeans = new ThreadLocal<>();
```

---

## 🚫 흔한 오해

### "ThreadLocal은 Thread-Safe하다"

```
❌ 잘못된 이해:
  ThreadLocal을 쓰면 동기화 불필요

✅ 실제:
  ThreadLocal의 value가 공유 객체면 위험
  
  // ❌ 위험
  static ThreadLocal<List<String>> list = 
      ThreadLocal.withInitial(() -> sharedList);
  
  // ✅ 안전
  static ThreadLocal<List<String>> list = 
      ThreadLocal.withInitial(() -> new ArrayList<>());
```

---

### "ThreadLocal은 자동으로 정리된다"

```
❌ 잘못된 이해:
  Thread 종료 시 자동 정리됨

✅ 실제:
  Thread Pool에서는 종료 안 함
  
  Thread Pool:
  Thread 재사용 → threadLocals 유지
  → 메모리 누수
  
  해결:
  명시적 remove() 필수
```

---

## 📌 핵심 정리

```
ThreadLocal
  스레드별 격리된 저장소
  Thread.threadLocals (ThreadLocalMap)

ThreadLocalMap
  Entry[] table (Open Addressing)
  Entry: WeakReference<ThreadLocal>, value
  Linear Probing 충돌 해결

get() / set()
  현재 Thread의 threadLocals 접근
  this (ThreadLocal 인스턴스)를 key로

WeakReference
  ThreadLocal 인스턴스를 약하게 참조
  GC 가능
  하지만 value는 Strong Reference

메모리 누수
  Thread Pool + remove() 안 함
  → value가 계속 유지
  → 누수

해결
  remove() 명시적 호출
  try-finally 패턴

InheritableThreadLocal
  자식 스레드가 부모 값 상속
  Thread 생성 시 복사

실무
  SimpleDateFormat, Transaction, RequestScope
  Thread Pool 사용 시 remove() 필수
```

---

## 🤔 생각해볼 문제

**Q1.** ThreadLocalMap이 WeakReference를 사용하는 이유를 설명하라. Strong Reference를 사용하면 어떤 문제가 발생하는가?

**Q2.** 다음 코드에서 메모리 누수가 발생하는가? 이유를 설명하라.

```java
ExecutorService executor = Executors.newFixedThreadPool(10);
ThreadLocal<Connection> connLocal = new ThreadLocal<>();

executor.submit(() -> {
    Connection conn = getConnection();
    connLocal.set(conn);
    // work...
});
```

**Q3.** InheritableThreadLocal과 일반 ThreadLocal의 차이를 설명하고, 언제 사용하는가?

> 💡 **해설**
>
> **Q1.** WeakReference 이유: ThreadLocal 인스턴스 GC 허용. Strong Reference 사용 시: `ThreadLocal<String> tl = new ThreadLocal<>(); tl.set("value"); tl = null;` → tl 변수는 null이지만 Entry의 key가 Strong Reference → ThreadLocal 인스턴스 GC 안 됨 → Entry도 제거 안 됨 → value도 유지 → 메모리 누수. WeakReference: tl = null → GC 가능 → Entry의 key = null → 다음 set()/get()에서 정리. 하지만 value는 여전히 Strong → remove() 필요.
>
> **Q2.** 메모리 누수 발생. 이유: ① ExecutorService의 Thread는 재사용 (종료 안 함). ② `connLocal.set(conn)` → Thread.threadLocals에 Entry 추가. ③ 작업 완료 후 remove() 안 함 → Entry 유지. ④ Connection은 계속 참조됨 (Strong) → GC 안 됨. ⑤ 다음 작업에서 같은 Thread 사용 → 이전 Connection 유지. 해결: `try { connLocal.set(conn); work(); } finally { connLocal.remove(); }`.
>
> **Q3.** InheritableThreadLocal: 자식 스레드가 부모의 ThreadLocal 값 상속. 동작: `Thread child = new Thread(...)` → 부모 Thread.inheritableThreadLocals 복사 → 자식 Thread.inheritableThreadLocals. 사용: 부모-자식 간 Context 전달 (예: 보안 컨텍스트, 트랜잭션 ID). 주의: Thread Pool에서는 부모가 누구인지 불명확 → 예측 불가. 일반 ThreadLocal: 상속 없음, 각 Thread 독립. 선택: 상속 필요 → InheritableThreadLocal, 독립 필요 → ThreadLocal.

---

## 📚 참고 자료

- [ThreadLocal Best Practices](https://www.baeldung.com/java-threadlocal)
- [ThreadLocal Memory Leaks](https://blog.codecentric.de/en/2019/02/threadlocal-memory-leak/)

---

<div align="center">

**[⬅️ 이전: Thread States & Scheduler](./06-thread-states-and-scheduler.md)** | **[다음: Virtual Threads (Loom) ➡️](./08-virtual-threads-loom.md)**

</div>

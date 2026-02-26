# CAS & Atomic Operations - CAS와 원자 연산

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CAS (Compare-And-Swap)는 무엇이며, 어떻게 동작하는가?
- CPU의 CMPXCHG 명령어는 어떻게 구현되는가?
- ABA 문제는 무엇이며, 어떻게 해결하는가?
- AtomicInteger는 내부에서 어떻게 동작하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: Lock 없이 원자성을 보장할 수 있는가

```java
// ❌ Lock 사용 (느림)
synchronized (lock) {
    if (value == expected) {
        value = newValue;
    }
}

// ✅ CAS (빠름)
compareAndSwap(value, expected, newValue);
```

CAS는 **Lock-Free 동기화**의 핵심이다.

---

## 📐 CAS 동작 원리

### 1. CAS 알고리즘

```
Compare-And-Swap (의사 코드):

boolean CAS(int* addr, int expected, int newValue) {
    if (*addr == expected) {
        *addr = newValue;
        return true;  // 성공
    } else {
        return false;  // 실패
    }
}

특징:
  원자적 실행 (Atomic)
  → 중간에 끼어들 수 없음
```

---

### 2. CPU 명령어 (x86)

```
CMPXCHG (Compare and Exchange):

lock cmpxchg [addr], newValue

동작:
  1. EAX 레지스터 vs [addr] 비교
  2. 같으면: [addr] = newValue, ZF=1
  3. 다르면: EAX = [addr], ZF=0

lock 프리픽스:
  - 메모리 버스 잠금
  - 캐시 일관성 보장
  - ~20ns 비용

예:
  mov eax, expected
  lock cmpxchg [value], newValue
  jz success  // ZF=1이면 성공
```

---

### 3. Java CAS (Unsafe)

```java
// sun.misc.Unsafe
public final class Unsafe {
    public final native boolean 
        compareAndSwapInt(Object o, long offset, 
                         int expected, int newValue);
    
    public final native boolean 
        compareAndSwapLong(Object o, long offset, 
                          long expected, long newValue);
    
    public final native boolean 
        compareAndSwapObject(Object o, long offset, 
                            Object expected, Object newValue);
}

// 내부: JNI → CMPXCHG
```

---

### 4. AtomicInteger 구현

```java
public class AtomicInteger {
    private volatile int value;
    private static final Unsafe unsafe = ...;
    private static final long valueOffset;
    
    static {
        valueOffset = unsafe.objectFieldOffset(
            AtomicInteger.class.getDeclaredField("value"));
    }
    
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }
    
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
}

// Unsafe.getAndAddInt (Java 8+)
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!compareAndSwapInt(o, offset, v, v + delta));
    return v;
}
```

---

## 💻 실험으로 확인하기

### 실험 1: CAS 성능 vs synchronized

```java
import java.util.concurrent.atomic.AtomicInteger;

public class CASBenchmark {
    static AtomicInteger atomicCounter = new AtomicInteger(0);
    static int syncCounter = 0;
    static Object lock = new Object();
    
    public static void main(String[] args) throws Exception {
        // AtomicInteger (CAS)
        long start = System.nanoTime();
        for (int i = 0; i < 10_000_000; i++) {
            atomicCounter.incrementAndGet();
        }
        long casTime = System.nanoTime() - start;
        
        // synchronized
        start = System.nanoTime();
        for (int i = 0; i < 10_000_000; i++) {
            synchronized (lock) {
                syncCounter++;
            }
        }
        long syncTime = System.nanoTime() - start;
        
        System.out.println("CAS: " + casTime / 1_000_000 + "ms");
        System.out.println("Synchronized: " + syncTime / 1_000_000 + "ms");
        System.out.println("Speedup: " + (double)syncTime / casTime + "x");
    }
}
```

```bash
# 출력 (단일 스레드):
# CAS: 50ms
# Synchronized: 200ms
# Speedup: 4.0x
```

---

### 실험 2: ABA 문제 재현

```java
import java.util.concurrent.atomic.AtomicInteger;

public class ABAProblem {
    static AtomicInteger value = new AtomicInteger(100);
    
    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(() -> {
            int oldValue = value.get();  // A (100)
            System.out.println("T1: Read " + oldValue);
            
            try {
                Thread.sleep(100);  // 중간에 변경 발생
            } catch (InterruptedException e) {}
            
            boolean success = value.compareAndSet(oldValue, oldValue + 1);
            System.out.println("T1: CAS " + (success ? "Success" : "Failed"));
        });
        
        Thread t2 = new Thread(() -> {
            try {
                Thread.sleep(50);
            } catch (InterruptedException e) {}
            
            value.set(200);  // A → B
            System.out.println("T2: Changed to 200");
            
            value.set(100);  // B → A (원래 값 복원)
            System.out.println("T2: Changed back to 100");
        });
        
        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
```

```bash
# 출력:
# T1: Read 100
# T2: Changed to 200
# T2: Changed back to 100
# T1: CAS Success  ← ABA 문제! (100→200→100)
```

---

## ⚡ 실무 임팩트

### Lock-Free Stack

```java
public class LockFreeStack<T> {
    private static class Node<T> {
        final T item;
        Node<T> next;
        
        Node(T item) {
            this.item = item;
        }
    }
    
    private final AtomicReference<Node<T>> top = new AtomicReference<>();
    
    public void push(T item) {
        Node<T> newHead = new Node<>(item);
        Node<T> oldHead;
        
        do {
            oldHead = top.get();
            newHead.next = oldHead;
        } while (!top.compareAndSet(oldHead, newHead));
    }
    
    public T pop() {
        Node<T> oldHead;
        Node<T> newHead;
        
        do {
            oldHead = top.get();
            if (oldHead == null) {
                return null;
            }
            newHead = oldHead.next;
        } while (!top.compareAndSet(oldHead, newHead));
        
        return oldHead.item;
    }
}
```

---

### ABA 문제 해결 — AtomicStampedReference

```java
import java.util.concurrent.atomic.AtomicStampedReference;

public class ABASolution {
    private AtomicStampedReference<Integer> value = 
        new AtomicStampedReference<>(100, 0);  // (value, stamp)
    
    public void update() {
        int[] stampHolder = new int[1];
        int oldValue = value.get(stampHolder);
        int oldStamp = stampHolder[0];
        
        int newValue = oldValue + 1;
        int newStamp = oldStamp + 1;  // Stamp 증가
        
        value.compareAndSet(oldValue, newValue, oldStamp, newStamp);
    }
}

// A → B → A 발생해도
// Stamp이 다르므로 CAS 실패
```

---

## 🚫 흔한 오해

### "CAS는 항상 Lock보다 빠르다"

```
❌ 잘못된 이해:
  CAS가 무조건 성능 우수

✅ 실제:
  경쟁 심하면 느림
  
  경쟁 없음:
  CAS: ~20ns
  synchronized: ~50ns
  → CAS 2.5배 빠름
  
  경쟁 심함 (10 스레드):
  CAS: Spin (재시도) → CPU 낭비
  synchronized: Sleep (대기) → CPU 절약
  
  권장:
  경쟁 적음: CAS
  경쟁 많음: synchronized
```

---

### "volatile만 있으면 CAS 가능"

```
❌ 잘못된 이해:
  volatile int value;
  → CAS 사용 가능

✅ 실제:
  CPU 명령어 필요 (CMPXCHG)
  
  volatile:
  가시성만 보장
  
  CAS:
  가시성 + 원자성
  → CPU 명령어 필수
  
  Java:
  Unsafe.compareAndSwapInt() 필요
```

---

## 📌 핵심 정리

```
CAS
  Compare-And-Swap
  원자적 비교 + 교체
  Lock-Free 동기화

CPU 명령어
  x86: CMPXCHG
  lock 프리픽스
  메모리 버스 잠금

Java CAS
  Unsafe.compareAndSwapInt/Long/Object
  AtomicInteger/Long/Reference

AtomicInteger
  volatile int value
  CAS 기반 increment/decrement
  Spin 재시도

ABA 문제
  A → B → A 변경 감지 못 함
  AtomicStampedReference로 해결
  (value, stamp) 쌍 비교

성능
  경쟁 적음: CAS 빠름 (~20ns)
  경쟁 많음: synchronized 유리
  
Lock-Free
  CAS 기반 자료구조
  Stack, Queue, List
  높은 처리량
```

---

## 🤔 생각해볼 문제

**Q1.** AtomicInteger.incrementAndGet()의 내부 구현을 CAS 재시도 루프로 설명하라. 경쟁이 심할 때 무슨 일이 발생하는가?

**Q2.** ABA 문제가 실제로 문제가 되는 시나리오를 Lock-Free Stack 예시로 설명하라.

**Q3.** 다음 중 어느 것이 더 효율적인가? 이유를 설명하라.
- A: AtomicInteger 1개를 10 스레드가 공유
- B: ThreadLocal<Integer> 10개 (스레드당 1개)

> 💡 **해설**
>
> **Q1.** incrementAndGet() 구현: ① `v = value` (읽기). ② `newValue = v + 1`. ③ `CAS(value, v, newValue)` — 성공 시 종료, 실패 시 ①로 재시도. 경쟁 심할 때: Thread 1: v=0 읽음. Thread 2: v=0 읽음, CAS 성공 (value=1). Thread 1: CAS 실패 (v=0이지만 value=1) → 재시도. Thread 3: v=1 읽음, CAS 성공 (value=2). Thread 1: 또 실패 → 계속 Spin. 결과: CPU 낭비 (Busy Waiting). 10 스레드면 9개가 계속 재시도 → 비효율. synchronized가 더 나음 (Sleep).
>
> **Q2.** ABA 문제 (Lock-Free Stack): Thread 1: top 읽음 (A → B → C). Thread 2: pop() 2번 (A, B 제거) → top = C. Thread 3: push(A) → top = A → C (A 재사용). Thread 1: CAS(top, A, X) — A가 그대로니까 성공! 하지만 Stack 구조 변경됨 (A → B → C였는데 A → C). B가 사라짐 → 메모리 누수 또는 댕글링 포인터. 해결: AtomicStampedReference — (top, stamp) 비교 → Stamp 달라서 CAS 실패.
>
> **Q3.** B (ThreadLocal)가 훨씬 효율적. 이유: A (AtomicInteger 공유) — 10 스레드가 CAS 경쟁 → 9개 재시도 → CPU 낭비, 캐시 무효화 (False Sharing 가능). B (ThreadLocal) — 각 스레드가 독립 변수 → 경쟁 없음, CAS 불필요, 캐시 친화적. 단, 최종 합산 필요 시 추가 비용. 결론: 독립 계산이면 ThreadLocal, 공유 필수면 AtomicInteger (하지만 LongAdder 고려).

---

## 📚 참고 자료

- [Java Atomic Variables](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/package-summary.html)
- [CAS in Practice](https://www.baeldung.com/java-atomic-variables)

---

<div align="center">

**[⬅️ 이전: Lock: Biased → Thin → Fat](./02-lock-biased-thin-fat.md)** | **[다음: False Sharing & Cache Line ➡️](./04-false-sharing-and-cache-line.md)**

</div>

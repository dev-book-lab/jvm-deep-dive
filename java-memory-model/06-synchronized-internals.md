# Synchronized Internals - Synchronized 내부 동작

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- synchronized가 삽입하는 Memory Barrier는 무엇인가?
- 모니터 락의 메모리 의미론은 무엇을 보장하는가?
- synchronized의 성능 비용은 얼마나 되는가?
- 언제 synchronized를 사용하고, 언제 Lock을 사용하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 원자성 + 가시성 모두 필요

```java
int counter = 0;

void increment() {
    counter++;  // 3단계: read-modify-write
}
```

```
volatile: 가시성만 보장
AtomicInteger: 원자성만 보장 (단일 변수)

synchronized: 원자성 + 가시성
  → 복합 연산 보호
  → 다중 변수 원자성
```

synchronized는 **완전한 동기화**를 제공한다.

---

## 📐 Synchronized 메모리 의미론

### 1. Monitor Lock Rule

```
unlock happens-before lock

Thread 1:
synchronized (obj) {
    x = 1;
    y = 2;
}  // unlock

Thread 2:
synchronized (obj) {  // lock
    System.out.println(x);  // 1 보장
    System.out.println(y);  // 2 보장
}

보장:
  Thread 1의 모든 변경
  → Thread 2가 확실히 봄
```

---

### 2. Memory Barrier 삽입

```
synchronized 진입:
  LoadLoad Barrier
  [acquire lock]
  LoadStore Barrier

synchronized 종료:
  StoreStore Barrier
  [release lock]
  StoreLoad Barrier

효과:
  - 락 전 명령어 재정렬 금지
  - 락 후 명령어 재정렬 금지
  - 캐시 플러시
  - 가시성 보장
```

---

### 3. Object Header & Monitor

```
Java 객체 구조:

[Mark Word (8 bytes)][Class Pointer (4 bytes)][Fields...]

Mark Word (Normal):
[hash:25][age:4][biased:1][lock:2][...]

Lock States:
  00: Lightweight Lock
  01: Unlocked or Biased
  10: Heavyweight Lock (Monitor)
  11: GC Mark

Heavyweight Lock:
  Mark Word → Monitor 포인터
  Monitor: OS Mutex (Futex)
```

---

### 4. 성능 비용

```
Fast Path (Biased Lock):
  - 단일 스레드 독점
  - CAS 1회
  - ~10ns

Lightweight Lock:
  - 경쟁 없음
  - CAS 2회 (진입/퇴출)
  - ~50ns

Heavyweight Lock:
  - 경쟁 있음
  - OS Mutex
  - ~1000ns (1μs)

Context Switch:
  - Wait/Notify
  - ~10,000ns (10μs)
```

---

## 💻 실험으로 확인하기

### 실험 1: synchronized 성능 측정

```java
public class SyncBenchmark {
    private int counter = 0;
    private final Object lock = new Object();
    
    public static void main(String[] args) {
        SyncBenchmark bench = new SyncBenchmark();
        
        // Warm-up
        for (int i = 0; i < 100000; i++) {
            bench.synchronizedIncrement();
        }
        
        // 측정
        long start = System.nanoTime();
        for (int i = 0; i < 10_000_000; i++) {
            bench.synchronizedIncrement();
        }
        long elapsed = System.nanoTime() - start;
        
        System.out.println("Time: " + elapsed / 1_000_000 + "ms");
        System.out.println("Per operation: " + elapsed / 10_000_000 + "ns");
    }
    
    void synchronizedIncrement() {
        synchronized (lock) {
            counter++;
        }
    }
}
```

```bash
# 출력:
# Time: 200ms
# Per operation: 20ns  ← Biased Lock
```

---

### 실험 2: 경쟁 시 성능

```java
public class ContentionBenchmark {
    private int counter = 0;
    private final Object lock = new Object();
    
    public static void main(String[] args) throws Exception {
        ContentionBenchmark bench = new ContentionBenchmark();
        
        Thread[] threads = new Thread[4];
        for (int i = 0; i < 4; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1_000_000; j++) {
                    bench.increment();
                }
            });
        }
        
        long start = System.nanoTime();
        for (Thread t : threads) t.start();
        for (Thread t : threads) t.join();
        long elapsed = System.nanoTime() - start;
        
        System.out.println("Time: " + elapsed / 1_000_000 + "ms");
        System.out.println("Per operation: " + elapsed / 4_000_000 + "ns");
    }
    
    void increment() {
        synchronized (lock) {
            counter++;
        }
    }
}
```

```bash
# 출력:
# Time: 1500ms
# Per operation: 375ns  ← Heavyweight Lock
```

---

## ⚡ 실무 임팩트

### synchronized vs Lock

```java
// ✅ synchronized (간단)
synchronized (lock) {
    // Critical Section
}

// ✅ ReentrantLock (고급 기능)
Lock lock = new ReentrantLock();
lock.lock();
try {
    // Critical Section
} finally {
    lock.unlock();
}

선택 기준:
  synchronized: 대부분의 경우
  Lock: tryLock(), 타임아웃, 공정성 필요
```

---

### 세밀한 락 분할

```java
// ❌ 굵은 락
class Counter {
    private int read = 0;
    private int write = 0;
    
    synchronized void incrementRead() {
        read++;
    }
    
    synchronized void incrementWrite() {
        write++;
    }
}

// ✅ 세밀한 락
class Counter {
    private int read = 0;
    private int write = 0;
    private final Object readLock = new Object();
    private final Object writeLock = new Object();
    
    void incrementRead() {
        synchronized (readLock) {
            read++;
        }
    }
    
    void incrementWrite() {
        synchronized (writeLock) {
            write++;
        }
    }
}
```

---

## 🚫 흔한 오해

### "synchronized는 항상 느리다"

```
❌ 잘못된 이해:
  synchronized는 무조건 성능 저하

✅ 실제:
  Biased Lock: ~10ns (매우 빠름)
  경쟁 없으면: ~50ns
  
  문제:
  경쟁 심할 때만 느림 (~1000ns)
  
  대부분 경우:
  synchronized로 충분
```

---

### "메서드 전체를 synchronized하면 안전하다"

```
❌ 잘못된 이해:
  synchronized method면 모든 게 안전

✅ 실제:
  메서드 내부만 보호
  
  class Counter {
      synchronized int getAndIncrement() {
          return counter++;
      }
  }
  
  // ❌ 불안전
  if (counter.get() == 0) {
      counter.increment();
  }
  // ← get()과 increment() 사이 갭
  
  // ✅ 안전
  synchronized (counter) {
      if (counter.get() == 0) {
          counter.increment();
      }
  }
```

---

## 📌 핵심 정리

```
Memory Barrier
  진입: LoadLoad, LoadStore
  종료: StoreStore, StoreLoad

Monitor Lock Rule
  unlock happens-before lock
  → 완전한 가시성 보장

Object Header
  Mark Word에 Lock 상태 저장
  Biased → Lightweight → Heavyweight

성능
  Biased: ~10ns
  Lightweight: ~50ns
  Heavyweight: ~1000ns
  경쟁 심하면 매우 느림

synchronized vs Lock
  synchronized: 대부분 충분
  Lock: tryLock, 타임아웃, 공정성

실무
  세밀한 락 분할
  임계 영역 최소화
  경쟁 회피
```

---

## 🤔 생각해볼 문제

**Q1.** synchronized가 삽입하는 Memory Barrier의 역할을 설명하라. 왜 4개가 필요한가?

**Q2.** 다음 코드의 문제점을 찾고, synchronized로 해결하라.

```java
class BankAccount {
    private int balance = 1000;
    
    void withdraw(int amount) {
        if (balance >= amount) {
            balance -= amount;
        }
    }
}
```

**Q3.** Biased Lock → Heavyweight Lock 전환이 발생하는 시나리오를 설명하라.

> 💡 **해설**
>
> **Q1.** 4개 Memory Barrier 역할: ① 진입 LoadLoad — 락 획득 전 읽기가 락 후 읽기보다 먼저 실행 안 되도록 (재정렬 방지). ② 진입 LoadStore — 락 획득 전 읽기가 Critical Section 쓰기보다 늦게 실행 안 되도록. ③ 종료 StoreStore — Critical Section 쓰기가 락 해제 후 쓰기보다 먼저 실행되도록 (가시성). ④ 종료 StoreLoad — 락 해제 쓰기가 다음 락 획득 읽기보다 먼저 보이도록 (전체 순서 보장). 4개 모두 필요한 이유: 재정렬 가능한 모든 조합 차단 + 캐시 일관성.
>
> **Q2.** 문제점: Check-Then-Act 패턴 — `if (balance >= amount)` 체크와 `balance -= amount` 실행 사이 갭. Thread 1: balance=1000 체크 통과. Thread 2: balance=1000 체크 통과. Thread 1: balance=500 (1000-500). Thread 2: balance=0 (500-500). → 잔액 -500 가능. 해결: `synchronized void withdraw(int amount) { if (balance >= amount) { balance -= amount; } }` — 전체 원자성 보장.
>
> **Q3.** Biased → Heavyweight 전환: ① 초기: Thread 1이 락 획득 → Biased Lock (Thread ID 저장). ② Thread 1이 반복 진입 → CAS 없이 빠르게 획득 (~10ns). ③ Thread 2가 같은 락 요청 → Bias 취소 (Revoke). ④ Lightweight Lock 시도 → CAS로 Stack Lock Record 설정. ⑤ Thread 1이 아직 락 보유 중 → CAS 실패. ⑥ Spin (짧은 대기) → 여전히 실패. ⑦ Heavyweight Lock으로 전환 (Inflation) → Monitor 생성, OS Mutex 사용. ⑧ Thread 2는 Wait Queue에서 대기 (~1000ns).

---

## 📚 참고 자료

- [Biased Locking in HotSpot](https://blogs.oracle.com/dave/biased-locking-in-hotspot)
- [Synchronization and Object Locking](https://wiki.openjdk.org/display/HotSpot/Synchronization)

---

<div align="center">

**[⬅️ 이전: Publication & Escape](./05-publication-and-escape.md)** | **[다음: Memory Barriers ➡️](./07-memory-barriers.md)**

</div>

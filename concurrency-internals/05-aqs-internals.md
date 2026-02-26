# AQS Internals - AQS 내부 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- AQS (AbstractQueuedSynchronizer)는 무엇이며, 왜 중요한가?
- CLH Queue는 어떻게 동작하는가?
- ReentrantLock, Semaphore, CountDownLatch는 AQS를 어떻게 활용하는가?
- Exclusive vs Shared 모드의 차이는 무엇인가?

---

## 🔍 왜 이게 존재하는가

### 문제: Lock을 매번 직접 구현하는가?

```java
// ReentrantLock, Semaphore, CountDownLatch...
// 모두 비슷한 패턴:
// - 대기 큐
// - 상태 관리
// - park/unpark

중복 제거:
  공통 프레임워크 필요
  → AQS
```

AQS는 **동기화 도구의 공통 기반**이다.

---

## 📐 내부 구조

### 1. AQS 핵심 개념

```java
public abstract class AbstractQueuedSynchronizer {
    // 동기화 상태
    private volatile int state;
    
    // 대기 큐 (CLH Queue 변형)
    private transient volatile Node head;
    private transient volatile Node tail;
    
    // CAS 연산
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
    
    // 서브클래스가 구현
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
    
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
}
```

---

### 2. CLH Queue (대기 큐)

```
CLH Lock Queue:

head → [Thread A] → [Thread B] → [Thread C] ← tail

Node 구조:
  - thread: 대기 중인 스레드
  - waitStatus: 상태 (CANCELLED, SIGNAL, ...)
  - prev/next: 이중 연결 리스트
  - nextWaiter: Condition Queue용

waitStatus:
  CANCELLED (1): 취소됨
  SIGNAL (-1): 다음 노드 깨워야 함
  CONDITION (-2): Condition Queue 대기
  PROPAGATE (-3): Shared 모드 전파
  0: 초기 상태
```

---

### 3. Acquire (획득) 과정

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

과정:
1. tryAcquire() 시도
   → 성공: 즉시 리턴
   → 실패: 큐 진입

2. addWaiter()
   → 큐 끝에 Node 추가 (CAS)

3. acquireQueued()
   for (;;) {
       if (predecessor == head && tryAcquire()) {
           // 획득 성공
           setHead(node);
           return;
       }
       
       if (shouldParkAfterFailedAcquire()) {
           parkAndCheckInterrupt();  // LockSupport.park()
       }
   }
```

---

### 4. Release (해제) 과정

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);  // 다음 노드 깨움
        return true;
    }
    return false;
}

unparkSuccessor():
  head의 next 찾기
  → LockSupport.unpark(thread)
  → 대기 스레드 깨움
```

---

### 5. ReentrantLock 구현

```java
public class ReentrantLock {
    private final Sync sync;
    
    abstract static class Sync extends AbstractQueuedSynchronizer {
        // state: 락 획득 횟수 (재진입)
        
        abstract void lock();
        
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            
            if (c == 0) {  // 락 해제됨
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            } else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;  // 재진입
                setState(nextc);
                return true;
            }
            return false;
        }
        
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            
            boolean free = false;
            if (c == 0) {  // 완전 해제
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
    }
}
```

---

### 6. Semaphore 구현

```java
public class Semaphore {
    private final Sync sync;
    
    abstract static class Sync extends AbstractQueuedSynchronizer {
        // state: 남은 permits

        Sync(int permits) {
            setState(permits);
        }
        
        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
        
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (compareAndSetState(current, next))
                    return true;
            }
        }
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: AQS 상태 관찰

```java
import java.util.concurrent.locks.ReentrantLock;
import java.lang.reflect.Field;

public class AQSStateTest {
    public static void main(String[] args) throws Exception {
        ReentrantLock lock = new ReentrantLock();
        
        Field syncField = ReentrantLock.class.getDeclaredField("sync");
        syncField.setAccessible(true);
        Object sync = syncField.get(lock);
        
        Field stateField = sync.getClass().getSuperclass().getDeclaredField("state");
        stateField.setAccessible(true);
        
        System.out.println("Initial state: " + stateField.get(sync));
        
        lock.lock();
        System.out.println("After lock: " + stateField.get(sync));  // 1
        
        lock.lock();  // 재진입
        System.out.println("After reentrant: " + stateField.get(sync));  // 2
        
        lock.unlock();
        System.out.println("After unlock: " + stateField.get(sync));  // 1
        
        lock.unlock();
        System.out.println("After full release: " + stateField.get(sync));  // 0
    }
}
```

---

### 실험 2: Fair vs Unfair Lock

```java
import java.util.concurrent.locks.ReentrantLock;

public class FairVsUnfairTest {
    static void testLock(ReentrantLock lock, String name) throws Exception {
        Runnable task = () -> {
            for (int i = 0; i < 3; i++) {
                lock.lock();
                try {
                    System.out.println(name + " - " + 
                        Thread.currentThread().getName());
                } finally {
                    lock.unlock();
                }
            }
        };
        
        Thread t1 = new Thread(task, "T1");
        Thread t2 = new Thread(task, "T2");
        
        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
    
    public static void main(String[] args) throws Exception {
        System.out.println("=== Unfair Lock ===");
        testLock(new ReentrantLock(false), "Unfair");
        
        System.out.println("\n=== Fair Lock ===");
        testLock(new ReentrantLock(true), "Fair");
    }
}
```

---

## ⚡ 실무 임팩트

### CountDownLatch 활용

```java
import java.util.concurrent.CountDownLatch;

public class ParallelTask {
    public static void main(String[] args) throws Exception {
        int N = 10;
        CountDownLatch startSignal = new CountDownLatch(1);
        CountDownLatch doneSignal = new CountDownLatch(N);
        
        for (int i = 0; i < N; i++) {
            new Thread(() -> {
                try {
                    startSignal.await();  // 시작 대기
                    doWork();
                    doneSignal.countDown();  // 완료 신호
                } catch (InterruptedException e) {}
            }).start();
        }
        
        System.out.println("준비 완료");
        startSignal.countDown();  // 모두 시작
        
        doneSignal.await();  // 모두 완료 대기
        System.out.println("전체 완료");
    }
    
    static void doWork() {
        // 작업 수행
    }
}
```

---

## 🚫 흔한 오해

### "AQS는 직접 사용한다"

```
❌ 잘못된 이해:
  AQS를 직접 상속해서 사용

✅ 실제:
  j.u.c 클래스 사용
  
  직접 사용: 복잡, 오류 위험
  ReentrantLock 사용: 간단, 안전
  
  특수한 경우만:
  커스텀 동기화 도구 필요 시
```

---

## 📌 핵심 정리

```
AQS
  AbstractQueuedSynchronizer
  동기화 도구의 공통 기반
  state + CLH Queue

CLH Queue
  이중 연결 리스트
  Node: thread, waitStatus, prev/next
  FIFO 대기

Acquire
  1. tryAcquire() 시도
  2. 실패 시 큐 진입
  3. park() 대기
  4. unpark() 후 재시도

Release
  1. tryRelease()
  2. 성공 시 다음 노드 unpark()

ReentrantLock
  state = 재진입 횟수
  Exclusive 모드

Semaphore
  state = 남은 permits
  Shared 모드

CountDownLatch
  state = 남은 카운트
  Shared 모드

Fair vs Unfair
  Fair: FIFO 보장 (느림)
  Unfair: 바로 시도 (빠름, 기본)
```

---

## 🤔 생각해볼 문제

**Q1.** ReentrantLock의 state 값이 2라면 무엇을 의미하는가? 어떤 상황에서 이 값이 나오는가?

**Q2.** Fair Lock과 Unfair Lock의 성능 차이는 왜 발생하는가? CLH Queue 관점에서 설명하라.

**Q3.** CountDownLatch의 countDown()이 AQS의 어떤 메서드를 호출하는가? state 변화를 추적하라.

> 💡 **해설**
>
> **Q1.** state=2는 재진입 2회. 같은 스레드가 lock.lock()을 2번 호출 → state=1 → state=2. 시나리오: `lock.lock(); doSomething(); lock.lock(); doOther(); lock.unlock(); lock.unlock();`. unlock()도 2번 호출해야 완전 해제 (state=0). 다른 스레드는 state=0이 될 때까지 대기. ReentrantLock이 "재진입 가능"한 이유.
>
> **Q2.** Fair Lock: CLH Queue 순서 엄격히 준수 → hasQueuedPredecessors() 체크 (큐에 선행자 있는지) → 있으면 대기 → 공정하지만 느림 (체크 비용). Unfair Lock: 큐 확인 없이 바로 CAS 시도 → 성공하면 큐 건너뛰기 → 불공정하지만 빠름 (Context Switch 감소). 성능 차이: Fair는 체크 + 항상 큐 거침, Unfair는 즉시 획득 시도 → 2~5배 빠름. Starvation 가능성: Unfair는 일부 스레드가 계속 못 얻을 수 있음.
>
> **Q3.** countDown() 호출 체인: ① `latch.countDown()` → ② `sync.releaseShared(1)` (AQS) → ③ `tryReleaseShared(1)` (CountDownLatch.Sync) → ④ CAS로 state 감소 (state--). state 변화: 초기 state=N (예: 3) → countDown() 호출 시마다 state-- (3→2→1→0). state=0 도달 → releaseShared()가 대기 중인 모든 스레드 unpark() → await() 리턴. Shared 모드라서 모두 깨움 (Exclusive는 1개만).

---

## 📚 참고 자료

- [AQS Paper - Doug Lea](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)
- [Java Concurrency in Practice](https://jcip.net/)

---

<div align="center">

**[⬅️ 이전: False Sharing & Cache Line](./04-false-sharing-and-cache-line.md)** | **[다음: Thread States & Scheduler ➡️](./06-thread-states-and-scheduler.md)**

</div>

# Object Monitor - 객체 모니터

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Object Monitor는 무엇이며, 어떻게 구성되는가?
- Entry Set과 Wait Set의 차이는 무엇인가?
- `wait()`, `notify()`, `notifyAll()`은 내부에서 어떻게 동작하는가?
- Monitor를 사용한 동기화의 성능 비용은?

---

## 🔍 왜 이게 존재하는가

### 문제: 스레드 간 협력이 필요하다

```java
// Producer-Consumer 문제
synchronized (queue) {
    while (queue.isFull()) {
        // 어떻게 기다릴까?
    }
    queue.add(item);
}
```

```
Busy Waiting: CPU 낭비
Sleep: 타이밍 어려움

해결책:
  Monitor Pattern
  → wait/notify로 협력
```

Object Monitor는 **스레드 협력의 핵심 메커니즘**이다.

---

## 📐 내부 구조

### 1. Monitor 구조

```
ObjectMonitor (C++ 구조):

┌─────────────────────────┐
│    Object Header        │
│  [Mark Word → Monitor*] │
└─────────────────────────┘
            ↓
┌─────────────────────────┐
│   ObjectMonitor         │
├─────────────────────────┤
│ _owner: Thread*         │ ← 락 소유자
│ _EntryList: Thread*     │ ← 대기 중 (Entry Set)
│ _WaitSet: Thread*       │ ← wait() 중 (Wait Set)
│ _recursions: int        │ ← 재진입 횟수
│ _count: int             │ ← 대기 스레드 수
└─────────────────────────┘

상태:
  _owner == null: 락 해제됨
  _owner != null: 락 보유 중
```

---

### 2. Entry Set vs Wait Set

```
Entry Set (_EntryList):
  synchronized 블록 진입 대기
  
  Thread 1: synchronized (obj) { ... }  ← Owner
  Thread 2: synchronized (obj) { ... }  ← Entry Set
  Thread 3: synchronized (obj) { ... }  ← Entry Set

Wait Set (_WaitSet):
  wait() 호출 후 대기
  
  Thread 4: obj.wait();  ← Wait Set
  Thread 5: obj.wait();  ← Wait Set

차이:
  Entry Set: 락 획득 경쟁
  Wait Set: notify() 대기
```

---

### 3. synchronized 동작

```
synchronized (obj) 진입:

1. Mark Word 확인
   Lightweight Lock? → CAS 시도
   실패? → Monitor로 Inflate

2. Monitor 진입 시도
   if (_owner == null) {
       _owner = currentThread;
       // 진입 성공
   } else {
       _EntryList에 추가
       park();  // OS 대기
   }

3. 재진입
   if (_owner == currentThread) {
       _recursions++;
       // 재진입 허용
   }

종료:
   _recursions--;
   if (_recursions == 0) {
       _owner = null;
       _EntryList에서 1개 깨움
   }
```

---

### 4. wait() / notify() 동작

```
obj.wait():

1. 선행 조건 확인
   if (_owner != currentThread) {
       throw IllegalMonitorStateException;
   }

2. Wait Set 이동
   _owner = null;  // 락 해제
   _WaitSet에 추가
   _EntryList에서 1개 깨움 (다른 스레드에 기회)

3. 대기
   park();  // OS 대기

4. notify() 후 깨어남
   _WaitSet에서 제거
   _EntryList로 이동 (락 재획득 경쟁)

obj.notify():

1. 선행 조건 확인
   if (_owner != currentThread) {
       throw IllegalMonitorStateException;
   }

2. Wait Set에서 1개 깨움
   _WaitSet.head를 _EntryList로 이동
   unpark(thread);  // OS 깨움

obj.notifyAll():
   _WaitSet의 모든 스레드를 _EntryList로 이동
```

---

### 5. Monitor 생명주기

```
1. Object 생성
   Mark Word: Unlocked (001)

2. 첫 synchronized
   Biased Lock (101) → Thin Lock (00)

3. 경쟁 발생
   Monitor 생성 (Inflate)
   Mark Word → Monitor 포인터 (10)

4. 사용 완료
   Monitor는 일반적으로 회수 안 됨
   (Object 삭제 시 같이 삭제)

비용:
  Inflate: ~1000ns (Monitor 생성)
  park/unpark: ~1000ns (OS 호출)
```

---

## 💻 실험으로 확인하기

### 실험 1: wait/notify 동작 확인

```java
public class MonitorTest {
    static Object lock = new Object();
    static boolean ready = false;
    
    public static void main(String[] args) throws Exception {
        Thread consumer = new Thread(() -> {
            synchronized (lock) {
                while (!ready) {
                    try {
                        System.out.println("Consumer waiting...");
                        lock.wait();  // Wait Set
                    } catch (InterruptedException e) {}
                }
                System.out.println("Consumer done!");
            }
        });
        
        Thread producer = new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {}
            
            synchronized (lock) {
                ready = true;
                System.out.println("Producer notifying...");
                lock.notify();  // Wait Set → Entry Set
            }
        });
        
        consumer.start();
        Thread.sleep(100);
        producer.start();
        
        consumer.join();
        producer.join();
    }
}
```

```bash
# 출력:
# Consumer waiting...
# Producer notifying...
# Consumer done!
```

---

### 실험 2: Monitor Inflation 관찰

```java
public class InflationTest {
    public static void main(String[] args) throws Exception {
        Object obj = new Object();
        
        System.out.println("Before sync: " + 
            ClassLayout.parseInstance(obj).toPrintable());
        
        synchronized (obj) {
            System.out.println("Lightweight: " + 
                ClassLayout.parseInstance(obj).toPrintable());
        }
        
        // 경쟁 유발
        Thread t1 = new Thread(() -> {
            synchronized (obj) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {}
            }
        });
        
        t1.start();
        Thread.sleep(100);
        
        synchronized (obj) {  // 경쟁 → Inflate
            System.out.println("Heavyweight: " + 
                ClassLayout.parseInstance(obj).toPrintable());
        }
    }
}
```

---

## ⚡ 실무 임팩트

### Producer-Consumer 패턴

```java
class BoundedQueue<T> {
    private Queue<T> queue = new LinkedList<>();
    private int capacity;
    
    BoundedQueue(int capacity) {
        this.capacity = capacity;
    }
    
    synchronized void put(T item) throws InterruptedException {
        while (queue.size() == capacity) {
            wait();  // Wait Set
        }
        queue.add(item);
        notifyAll();  // Consumer 깨움
    }
    
    synchronized T take() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();  // Wait Set
        }
        T item = queue.remove();
        notifyAll();  // Producer 깨움
        return item;
    }
}
```

---

### notify vs notifyAll

```java
// ❌ notify (일부만 깨움)
synchronized (lock) {
    ready = true;
    lock.notify();  // 1개만 깨움
}
// → 여러 대기 스레드 중 1개만 깨어남
// → 나머지는 영원히 대기 가능

// ✅ notifyAll (모두 깨움)
synchronized (lock) {
    ready = true;
    lock.notifyAll();  // 모두 깨움
}
// → 모든 대기 스레드 깨어남
// → 조건 확인 후 재대기

권장:
  notifyAll 사용 (안전)
  notify는 특수한 경우만
```

---

## 🚫 흔한 오해

### "wait()는 락을 유지한다"

```
❌ 잘못된 이해:
  wait()는 락을 가진 채 대기

✅ 실제:
  wait()는 락을 해제
  
  synchronized (obj) {
      obj.wait();  // ← 락 해제!
  }
  
  이유:
  다른 스레드가 notify() 호출하려면
  락을 획득해야 함
  
  wait() 후 깨어나면:
  락을 재획득해야 진행
```

---

### "notify()는 즉시 깨운다"

```
❌ 잘못된 이해:
  notify() 호출 시 즉시 대기 스레드 실행

✅ 실제:
  notify() 호출자가 락 해제 후
  
  synchronized (obj) {
      obj.notify();
      // ← 여기서는 아직 락 보유
      // ← 대기 스레드 못 깨어남
  }  // ← 여기서 락 해제
     // ← 이제 대기 스레드 깨어남
```

---

## 📌 핵심 정리

```
Object Monitor
  C++ ObjectMonitor 구조
  _owner, _EntryList, _WaitSet

Entry Set
  synchronized 진입 대기
  락 획득 경쟁

Wait Set
  wait() 호출 후 대기
  notify() 대기

wait()
  1. 락 해제
  2. Wait Set 추가
  3. park() 대기
  4. notify() 후 Entry Set 이동

notify()
  Wait Set → Entry Set (1개)

notifyAll()
  Wait Set → Entry Set (전체)

성능 비용
  Monitor Inflate: ~1000ns
  park/unpark: ~1000ns
  Heavyweight Lock 유지

실무 패턴
  Producer-Consumer
  notifyAll 권장
  while 루프로 재확인
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 IllegalMonitorStateException이 발생하는 이유를 설명하라.

```java
Object obj = new Object();
obj.wait();
```

**Q2.** Entry Set과 Wait Set에서 스레드가 깨어나는 순서는 보장되는가? 공정성(Fairness) 관점에서 설명하라.

**Q3.** notify() 대신 notifyAll()을 사용해야 하는 이유를 "Lost Wake-Up" 문제와 연결해 설명하라.

> 💡 **해설**
>
> **Q1.** IllegalMonitorStateException 이유: `obj.wait()` 호출 전 락을 보유해야 함. Monitor의 _owner가 currentThread여야 wait() 가능. 코드는 synchronized 없이 wait() 호출 → _owner != currentThread → 예외 발생. 올바른 코드: `synchronized (obj) { obj.wait(); }`. 이유: wait()는 락을 해제하는 동작 → 먼저 락을 가져야 해제 가능. notify()도 동일하게 synchronized 블록 내에서만 호출 가능.
>
> **Q2.** 순서 보장 안 됨 (Unfair). Entry Set: LIFO 스택 구조 (최근 진입이 우선) → 공정하지 않음. Wait Set: FIFO 큐 구조 (먼저 wait()한 것이 우선) → 상대적으로 공정. 하지만 Wait Set에서 Entry Set 이동 후 경쟁 → 최종 순서 보장 안 됨. Starvation 가능: 계속 락 못 얻는 스레드 존재 가능. 공정성 필요 시: ReentrantLock(true) 사용 → FIFO 보장.
>
> **Q3.** Lost Wake-Up 문제: 여러 조건으로 대기 중인 스레드들 → notify() 1개만 깨움 → 잘못된 스레드 깨어날 가능성. 예: Producer/Consumer 혼재 → notify()가 Producer 깨움 (Consumer 필요한데) → Consumer 영원히 대기. notifyAll() 해결: 모든 대기 스레드 깨움 → 각자 조건 확인 (while 루프) → 조건 맞는 스레드만 진행, 나머지 재대기. 비용: 불필요한 깨우기 발생하지만 안전성 확보. 결론: notify()는 단일 조건, 단일 대기자만 있을 때만 안전.

---

## 📚 참고 자료

- [HotSpot ObjectMonitor](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/runtime/objectMonitor.hpp)
- [Java Monitor Pattern](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html#jls-17.1)

---

<div align="center">

**[다음: Lock: Biased → Thin → Fat ➡️](./02-lock-biased-thin-fat.md)**

</div>

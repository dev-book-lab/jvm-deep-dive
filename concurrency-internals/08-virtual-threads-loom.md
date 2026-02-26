# Virtual Threads (Project Loom) - 가상 스레드

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Virtual Thread는 무엇이며, Platform Thread와 어떻게 다른가?
- Carrier Thread는 무엇이며, 어떻게 동작하는가?
- Structured Concurrency는 무엇인가?
- Pinning 문제는 무엇이며, 어떻게 회피하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: Thread의 비용이 너무 높다

```java
// Platform Thread (전통적)
ExecutorService executor = Executors.newFixedThreadPool(1000);

// 1000개 OS Thread 생성
// → 각각 1~2MB Stack
// → 1~2GB 메모리
// → Context Switch 비용

// Virtual Thread (Java 21+)
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

// 수백만 개 생성 가능
// → 각각 ~KB Stack
// → 메모리 효율적
// → Context Switch 없음 (대부분)
```

Virtual Thread는 **경량 스레드**다.

---

## 📐 내부 구조

### 1. Platform vs Virtual Thread

```
Platform Thread (OS Thread):
  1:1 매핑
  OS가 스케줄링
  Stack: 1~2MB
  생성 비용: ~1ms
  최대: 수천 개

Virtual Thread:
  M:N 매핑 (M virtual → N carrier)
  JVM이 스케줄링
  Stack: 동적 (~KB)
  생성 비용: ~1μs
  최대: 수백만 개
```

---

### 2. Carrier Thread

```
Carrier Thread:
  Virtual Thread를 실행하는 Platform Thread
  ForkJoinPool 기반
  기본: CPU 코어 수

┌────────────────────────────┐
│   Virtual Thread 1         │
│   Virtual Thread 2         │
│   Virtual Thread 3         │
│         ...                │
│   Virtual Thread 1,000,000 │
└────────────────────────────┘
         ↓ Mount/Unmount
┌────────────────────────────┐
│ Carrier Thread 1 (Platform)│
│ Carrier Thread 2 (Platform)│
│ Carrier Thread 8 (Platform)│
└────────────────────────────┘

Mount:
  Virtual Thread → Carrier Thread 할당
  
Unmount:
  Blocking I/O 시 Carrier 해제
  → 다른 Virtual Thread 실행
```

---

### 3. Virtual Thread 생성

```java
// Java 21+

// 방법 1: Thread.ofVirtual()
Thread vt = Thread.ofVirtual().start(() -> {
    System.out.println("Virtual Thread");
});

// 방법 2: Thread.startVirtualThread()
Thread.startVirtualThread(() -> {
    System.out.println("Virtual Thread");
});

// 방법 3: Executor
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
executor.submit(() -> {
    System.out.println("Virtual Thread");
});

// 방법 4: ThreadFactory
ThreadFactory factory = Thread.ofVirtual().factory();
Thread vt = factory.newThread(() -> {
    System.out.println("Virtual Thread");
});
```

---

### 4. Blocking I/O 처리

```java
// Platform Thread
Thread platformThread = new Thread(() -> {
    socket.read(buffer);  // Blocking
    // → Thread가 Block됨
    // → Carrier (OS Thread) Block됨
    // → 비효율적
});

// Virtual Thread
Thread virtualThread = Thread.ofVirtual().start(() -> {
    socket.read(buffer);  // Blocking
    // → Virtual Thread만 Block됨
    // → Carrier는 Unmount
    // → 다른 Virtual Thread 실행
    // → 효율적!
});

내부 동작:
  1. socket.read() 호출
  2. JVM이 감지 (Blocking I/O)
  3. Virtual Thread Unmount
  4. Carrier Thread 해제
  5. I/O 완료 시 Virtual Thread Mount
  6. 실행 재개
```

---

### 5. Pinning (고정) 문제

```java
// ❌ Pinning 발생
synchronized (lock) {
    // Blocking I/O
    socket.read(buffer);  
    // → Virtual Thread가 Carrier에 고정됨
    // → Carrier Block됨
    // → 비효율적
}

// ✅ Pinning 회피
Lock lock = new ReentrantLock();
lock.lock();
try {
    socket.read(buffer);
    // → Virtual Thread Unmount 가능
    // → Carrier 해제 가능
} finally {
    lock.unlock();
}

Pinning 원인:
  1. synchronized 블록
  2. Native 메서드 호출
```

---

### 6. Structured Concurrency

```java
// Java 21+ (Preview)
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<String> user = scope.fork(() -> fetchUser());
    Future<String> order = scope.fork(() -> fetchOrder());
    
    scope.join();  // 모든 작업 완료 대기
    scope.throwIfFailed();  // 실패 시 예외
    
    return new Result(user.resultNow(), order.resultNow());
}
// scope 종료 시 모든 Virtual Thread 자동 취소

특징:
  - 작업 계층 구조
  - 자동 리소스 정리
  - 예외 전파
  - Timeout 지원
```

---

## 💻 실험으로 확인하기

### 실험 1: Virtual vs Platform 성능

```java
import java.time.Duration;

public class VirtualThreadBenchmark {
    public static void main(String[] args) throws Exception {
        int tasks = 100_000;
        
        // Platform Thread (Pool)
        long start = System.currentTimeMillis();
        try (var executor = Executors.newFixedThreadPool(200)) {
            for (int i = 0; i < tasks; i++) {
                executor.submit(() -> {
                    try {
                        Thread.sleep(Duration.ofMillis(10));
                    } catch (InterruptedException e) {}
                });
            }
        }
        long platformTime = System.currentTimeMillis() - start;
        
        // Virtual Thread
        start = System.currentTimeMillis();
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < tasks; i++) {
                executor.submit(() -> {
                    try {
                        Thread.sleep(Duration.ofMillis(10));
                    } catch (InterruptedException e) {}
                });
            }
        }
        long virtualTime = System.currentTimeMillis() - start;
        
        System.out.println("Platform: " + platformTime + "ms");
        System.out.println("Virtual: " + virtualTime + "ms");
        System.out.println("Speedup: " + (platformTime / virtualTime) + "x");
    }
}
```

```bash
# 출력:
# Platform: 50000ms
# Virtual: 1000ms
# Speedup: 50x
```

---

### 실험 2: Pinning 감지

```java
// -Djdk.tracePinnedThreads=full
public class PinningTest {
    static Object lock = new Object();
    
    public static void main(String[] args) throws Exception {
        Thread.ofVirtual().start(() -> {
            synchronized (lock) {
                try {
                    Thread.sleep(1000);  // Blocking
                } catch (InterruptedException e) {}
            }
        }).join();
    }
}
```

```bash
# 실행
java -Djdk.tracePinnedThreads=full PinningTest

# 출력:
# Thread[#21,ForkJoinPool-1-worker-1,5,CarrierThreads]
#     java.base/java.lang.VirtualThread$VThreadContinuation.onPinned
#     ...
#     PinningTest.main (PinningTest.java:8) ← synchronized 위치
```

---

## ⚡ 실무 임팩트

### HTTP 서버 (대량 연결)

```java
// 전통적 방식 (Platform Thread Pool)
ExecutorService executor = Executors.newFixedThreadPool(200);

serverSocket.accept((socket) -> {
    executor.submit(() -> handleRequest(socket));
});
// 최대 200개 동시 연결

// Virtual Thread 방식
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

serverSocket.accept((socket) -> {
    executor.submit(() -> handleRequest(socket));
});
// 수백만 개 동시 연결 가능
```

---

### Reactive → Imperative 전환

```java
// Before: Reactive (복잡)
Mono<User> user = webClient.get("/user").retrieve().bodyToMono(User.class);
Mono<Order> order = webClient.get("/order").retrieve().bodyToMono(Order.class);

return Mono.zip(user, order, (u, o) -> new Result(u, o));

// After: Virtual Thread (간단)
User user = httpClient.send(request("/user")).body();
Order order = httpClient.send(request("/order")).body();

return new Result(user, order);
```

---

## 🚫 흔한 오해

### "Virtual Thread는 항상 빠르다"

```
❌ 잘못된 이해:
  Virtual Thread가 무조건 성능 우수

✅ 실제:
  I/O-bound 작업만 유리
  
  CPU-bound:
  Platform Thread가 더 나음
  (오버헤드 없음)
  
  I/O-bound:
  Virtual Thread가 압도적
  (Blocking 시 Unmount)
```

---

### "Virtual Thread는 무제한 생성 가능"

```
❌ 잘못된 이해:
  메모리 제약 없이 무한정 생성

✅ 실제:
  메모리 한계 존재
  
  Virtual Thread 1개: ~KB
  1M 개: ~GB
  
  제한:
  - Heap 메모리
  - Carrier Thread 수
  - OS 리소스
```

---

## 📌 핵심 정리

```
Virtual Thread
  경량 스레드 (Java 21+)
  M:N 매핑 (M virtual → N carrier)
  Stack 동적 할당 (~KB)
  수백만 개 생성 가능

Carrier Thread
  Virtual Thread 실행하는 Platform Thread
  ForkJoinPool 기반
  기본: CPU 코어 수

Mount/Unmount
  Mount: Virtual → Carrier 할당
  Unmount: Blocking I/O 시 해제
  → 다른 Virtual Thread 실행

Pinning
  synchronized 블록 내 Blocking
  → Carrier에 고정됨
  → 비효율적
  회피: ReentrantLock 사용

Structured Concurrency
  작업 계층 구조
  자동 리소스 정리
  예외 전파

성능
  I/O-bound: 50~100배 빠름
  CPU-bound: Platform과 비슷
  
실무 적용
  HTTP 서버 (대량 연결)
  Reactive → Imperative 전환
  Pinning 회피 필수
```

---

## 🤔 생각해볼 문제

**Q1.** Virtual Thread가 Blocking I/O에서 자동으로 Unmount되는데, synchronized 블록에서는 왜 Unmount되지 않는가?

**Q2.** 다음 코드에서 Virtual Thread의 장점을 설명하라.

```java
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
for (int i = 0; i < 1_000_000; i++) {
    executor.submit(() -> {
        httpClient.send(request);  // 100ms Blocking
    });
}
```

**Q3.** CPU-bound 작업에서 Virtual Thread를 사용하면 오히려 느려질 수 있는 이유를 설명하라.

> 💡 **해설**
>
> **Q1.** synchronized는 JVM Monitor 사용 → Native 구현 → JVM이 Unmount 제어 불가. Blocking I/O는 JVM이 감지 가능 (Socket 등 Java API) → Unmount 가능. synchronized 내부: JNI 호출 → Native Mutex → JVM 레벨에서 투명하지 않음 → Pinning. ReentrantLock: Pure Java → JVM이 완전히 제어 → Unmount 가능. 해결: synchronized 대신 ReentrantLock 사용.
>
> **Q2.** Virtual Thread 장점: ① 1M 개 생성 가능 — Platform Thread면 1M × 2MB = 2TB 메모리 (불가능). Virtual Thread는 1M × ~1KB = 1GB (가능). ② Blocking 시 Unmount — httpClient.send() Blocking (100ms) → Carrier 해제 → 다른 Virtual Thread 실행. 8 Carrier로 1M 요청 처리 가능. Platform Thread면 Pool 크기만큼만 동시 처리. ③ 간단한 코드 — Imperative 스타일 유지, Reactive 복잡성 없음.
>
> **Q3.** CPU-bound에서 느린 이유: ① Virtual Thread도 결국 Carrier (Platform Thread)에서 실행 → CPU 코어 수 제한. ② Carrier 전환 오버헤드 — Virtual Thread 전환 시 Stack 저장/복원 비용. Platform Thread는 전환 없음. ③ 시나리오: 8 코어, 100 Virtual Thread (CPU-bound) → 계속 Carrier 전환 → 오버헤드만 증가. Platform Thread 8개가 더 효율적 (전환 없음). 결론: CPU-bound는 Platform Thread, I/O-bound는 Virtual Thread.

---

## 📚 참고 자료

- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [Project Loom](https://wiki.openjdk.org/display/loom)
- [Virtual Threads Guide](https://inside.java/2021/05/10/virtual-threads/)

---

<div align="center">

**[⬅️ 이전: ThreadLocal Internals](./07-thread-local-internals.md)** | **[다음: Safepoint Mechanism ➡️](./09-safepoint-mechanism.md)**

</div>

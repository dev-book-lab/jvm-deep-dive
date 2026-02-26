# Thread States & Scheduler - 스레드 상태와 스케줄러

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- OS 스레드 상태와 JVM 스레드 상태의 차이는 무엇인가?
- Context Switching 비용은 얼마나 되는가?
- Thread.sleep(), Object.wait(), LockSupport.park()의 차이는?
- 스레드 스케줄링은 어떻게 동작하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 스레드는 항상 실행 중인가?

```
CPU: 8개
스레드: 1000개

모든 스레드가 동시 실행?
→ 불가능!

해결:
  OS 스케줄러가 시분할
  → Context Switching
```

스레드 상태는 **스케줄링의 핵심**이다.

---

## 📐 스레드 상태

### 1. OS 스레드 상태

```
┌─────────┐
│  NEW    │
└────┬────┘
     ↓
┌─────────┐
│ RUNNABLE│ ←──┐
└────┬────┘    │
     ↓         │
┌─────────┐    │
│ RUNNING │────┘
└────┬────┘
     ↓
┌──────────┐
│TERMINATED│
└──────────┘

세부 상태:
  RUNNABLE: Ready Queue 대기
  RUNNING: CPU 실행 중
  BLOCKED: I/O 대기
  WAITING: Signal 대기
```

---

### 2. JVM Thread.State

```java
public enum State {
    NEW,           // 생성됨, 시작 전
    RUNNABLE,      // 실행 가능 (OS RUNNABLE + RUNNING)
    BLOCKED,       // synchronized 대기
    WAITING,       // wait(), park() 무한 대기
    TIMED_WAITING, // sleep(), wait(timeout) 시간 대기
    TERMINATED     // 종료됨
}
```

---

### 3. 상태 전이

```
NEW
 ↓ start()
RUNNABLE ←──────────────────────┐
 ↓ synchronized (경쟁)           │
BLOCKED ────────────────────────┘
 ↓ 락 획득                       
RUNNABLE                         
 ↓ wait() / park()              
WAITING ────────────────────────┐
 ↓ notify() / unpark()          │
RUNNABLE ←──────────────────────┘
 ↓ 실행 완료
TERMINATED
```

---

### 4. RUNNABLE vs RUNNING

```
JVM RUNNABLE:
  OS RUNNABLE (Ready) + RUNNING (Executing)
  
예:
  Thread t = new Thread(...);
  t.start();
  
  System.out.println(t.getState());  // RUNNABLE
  
  하지만:
  실제 CPU에서 실행 중? 아닐 수 있음
  → Ready Queue 대기 중일 수도
  
JVM은 구분 안 함:
  두 상태 모두 "실행 가능"
```

---

### 5. Context Switching

```
Context Switch:
  CPU가 Thread A → Thread B로 전환

과정:
  1. Thread A 레지스터 저장
  2. Thread B 레지스터 복원
  3. Thread B 실행

비용:
  직접 비용: 1~2μs (레지스터 저장/복원)
  간접 비용: 10~100μs (캐시 무효화)

발생 원인:
  - Time Slice 소진 (보통 10ms)
  - I/O 대기
  - Lock 대기
  - sleep(), wait()
```

---

## 💻 실험으로 확인하기

### 실험 1: Thread State 관찰

```java
public class ThreadStateTest {
    public static void main(String[] args) throws Exception {
        Thread t = new Thread(() -> {
            synchronized (ThreadStateTest.class) {
                try {
                    ThreadStateTest.class.wait();
                } catch (InterruptedException e) {}
            }
        });
        
        System.out.println("NEW: " + t.getState());
        
        t.start();
        Thread.sleep(100);
        System.out.println("WAITING: " + t.getState());
        
        synchronized (ThreadStateTest.class) {
            ThreadStateTest.class.notify();
        }
        
        t.join();
        System.out.println("TERMINATED: " + t.getState());
    }
}
```

---

### 실험 2: Context Switch 비용 측정

```java
public class ContextSwitchBenchmark {
    static volatile boolean flag = false;
    
    public static void main(String[] args) throws Exception {
        // 단일 스레드 (Switch 없음)
        long start = System.nanoTime();
        for (int i = 0; i < 10_000_000; i++) {
            flag = !flag;
        }
        long singleTime = System.nanoTime() - start;
        
        // 두 스레드 (Switch 빈번)
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 5_000_000; i++) {
                flag = true;
                Thread.yield();  // Switch 유도
            }
        });
        
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 5_000_000; i++) {
                flag = false;
                Thread.yield();
            }
        });
        
        start = System.nanoTime();
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        long switchTime = System.nanoTime() - start;
        
        System.out.println("Single: " + singleTime / 1_000_000 + "ms");
        System.out.println("Switch: " + switchTime / 1_000_000 + "ms");
        System.out.println("Overhead: " + (switchTime / singleTime) + "x");
    }
}
```

---

## ⚡ 실무 임팩트

### sleep() vs wait() vs park()

```java
// sleep(): 스레드 일시 정지
Thread.sleep(1000);  // 1초
// → TIMED_WAITING
// → 락 유지

// wait(): 조건 대기
synchronized (obj) {
    obj.wait();  // 무한 대기
    obj.wait(1000);  // 1초 대기
}
// → WAITING / TIMED_WAITING
// → 락 해제

// park(): Low-level 대기
LockSupport.park();
LockSupport.parkNanos(1_000_000_000L);  // 1초
// → WAITING / TIMED_WAITING
// → AQS 내부 사용
```

---

### Thread Pool 크기 결정

```
CPU-bound:
  스레드 수 = CPU 코어 수
  (또는 코어 수 + 1)

I/O-bound:
  스레드 수 = CPU 코어 수 × (1 + 대기시간/실행시간)
  
  예:
  실행: 10ms
  I/O 대기: 90ms
  → 코어 수 × (1 + 90/10) = 코어 수 × 10

과도한 스레드:
  Context Switch 증가
  → 성능 저하
```

---

## 🚫 흔한 오해

### "RUNNABLE = 실행 중"

```
❌ 잘못된 이해:
  RUNNABLE 상태는 CPU에서 실행 중

✅ 실제:
  실행 가능 (Ready + Running)
  
  RUNNABLE 스레드:
  - CPU 실행 중 (일부)
  - Ready Queue 대기 (대부분)
  
  JVM은 구분 안 함
```

---

### "Context Switch는 매우 저렴하다"

```
❌ 잘못된 이해:
  Context Switch는 1~2μs로 무시 가능

✅ 실제:
  간접 비용이 큼
  
  직접 비용: 1~2μs
  간접 비용:
  - L1/L2 캐시 무효화
  - TLB 무효화
  - Branch Prediction 초기화
  → 10~100μs 추가
  
  빈번하면:
  성능 크게 저하
```

---

## 📌 핵심 정리

```
JVM Thread State
  NEW: 생성, 시작 전
  RUNNABLE: 실행 가능
  BLOCKED: synchronized 대기
  WAITING: 무한 대기
  TIMED_WAITING: 시간 대기
  TERMINATED: 종료

OS vs JVM
  OS: RUNNABLE, RUNNING 구분
  JVM: RUNNABLE로 통합

Context Switch
  Thread A → Thread B 전환
  직접 비용: 1~2μs
  간접 비용: 10~100μs
  빈번하면 성능 저하

sleep() vs wait() vs park()
  sleep(): 락 유지, 시간 대기
  wait(): 락 해제, 조건 대기
  park(): Low-level, AQS 사용

Thread Pool
  CPU-bound: 코어 수
  I/O-bound: 코어 수 × (1 + 대기/실행)
  
스케줄링
  Time Slice: ~10ms
  우선순위: 1~10 (기본 5)
  선점형 멀티태스킹
```

---

## 🤔 생각해볼 문제

**Q1.** synchronized 블록에서 대기 중인 스레드의 상태는 BLOCKED인데, ReentrantLock.lock()에서 대기 중인 스레드의 상태는 WAITING이다. 왜 차이가 나는가?

**Q2.** Context Switch 비용을 줄이기 위한 방법 3가지를 제시하라.

**Q3.** Thread.sleep(0)과 Thread.yield()의 차이는 무엇인가? 각각 언제 사용하는가?

> 💡 **해설**
>
> **Q1.** synchronized: JVM 내장 Monitor 사용 → OS에게 "락 대기 중"으로 보고 → OS가 BLOCKED 상태로 관리. ReentrantLock: AQS 사용 → LockSupport.park() 호출 → OS에게 "일반 대기"로 보고 → WAITING 상태. 차이: synchronized는 JVM 레벨 동기화, ReentrantLock은 Java 레벨 동기화. 결과는 동일 (대기)하지만 메커니즘이 다름 → 상태 다름.
>
> **Q2.** Context Switch 줄이기: ① 스레드 수 최적화 — 과도한 스레드 생성 금지 → CPU 코어 수 고려. ② Spin Lock 사용 — 짧은 대기는 park() 대신 Spin → Context Switch 회피 (예: CAS 재시도). ③ Batching — 여러 작업을 모아서 한 번에 처리 → I/O 횟수 감소 → Switch 감소. ④ Thread Pool 재사용 — 스레드 생성/종료 비용 제거. ⑤ Lock-Free 자료구조 — Lock 대기 제거 → Switch 감소.
>
> **Q3.** Thread.sleep(0): OS에게 "Time Slice 양보" 제안 → OS가 판단 (다른 RUNNABLE 스레드 있으면 Switch, 없으면 계속 실행). Thread.yield(): JVM에게 "양보"힌트 → 구현 의존적 (일부 JVM은 무시). 차이: sleep(0)은 OS 레벨, yield()는 JVM 레벨. 사용: sleep(0)은 명확한 양보, yield()는 힌트 (보장 안 됨). 실무: 둘 다 거의 사용 안 함 (스케줄러 믿기).

---

## 📚 참고 자료

- [Java Thread States](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.State.html)
- [Context Switch Cost](https://blog.tsunanet.net/2010/11/how-long-does-it-take-to-make-context.html)

---

<div align="center">

**[⬅️ 이전: AQS Internals](./05-aqs-internals.md)** | **[다음: ThreadLocal Internals ➡️](./07-thread-local-internals.md)**

</div>

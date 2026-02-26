# Lock: Biased → Thin → Fat - 락 상태 전이

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Biased, Thin, Fat Lock의 차이는 무엇인가?
- Mark Word가 어떻게 변화하며 락 상태를 표현하는가?
- 각 Lock 타입의 성능 비용은 얼마나 되는가?
- Biased Lock이 deprecated된 이유는 무엇인가?

---

## 🔍 왜 이게 존재하는가

### 문제: synchronized는 비싸다 (과거)

```
Java 1.5 이전:
  synchronized = Heavyweight Lock
  → OS Mutex 사용
  → ~1000ns 비용
  → 성능 저하

목표:
  경쟁 없는 경우 최적화
  → Biased/Thin Lock
```

Lock 최적화는 **단계적 성능 개선**이다.

---

## 📐 Lock 상태 전이

### 1. Mark Word 구조 (64bit)

```
Unlocked (001):
[hash:25][age:4][biased:0][lock:01]

Biased Lock (101):
[thread:54][epoch:2][age:4][biased:1][lock:01]

Thin Lock (00):
[ptr:62][lock:00]
  ↑ Stack Lock Record 포인터

Fat Lock (10):
[ptr:62][lock:10]
  ↑ ObjectMonitor 포인터
```

---

### 2. Lock 상태 전이 흐름

```
┌─────────┐
│Unlocked │ (001)
└────┬────┘
     │ First synchronized
     ↓
┌─────────┐
│ Biased  │ (101)
└────┬────┘
     │ Thread contention
     ↓
┌─────────┐
│  Thin   │ (00)
└────┬────┘
     │ Lock contention
     ↓
┌─────────┐
│  Fat    │ (10)
└─────────┘

되돌아가기:
  Fat → Thin: 없음
  Thin → Biased: 없음
  (한 방향만 가능)
```

---

### 3. Biased Lock (편향 락)

```
목적:
  단일 스레드가 반복 획득 시 최적화

동작:
  첫 synchronized:
    Mark Word에 Thread ID 저장
    biased = 1, lock = 01
  
  재진입:
    Thread ID 확인만
    → CAS 불필요
    → 매우 빠름 (~5ns)

Revoke (해제):
  다른 스레드가 획득 시도
  → Safepoint에서 Bias 해제
  → Thin Lock으로 전환

비용:
  획득: ~5ns
  Revoke: ~10,000ns (Safepoint)
```

---

### 4. Thin Lock (경량 락)

```
목적:
  경쟁 없는 짧은 Critical Section

동작:
  진입:
    Stack에 Lock Record 할당
    CAS로 Mark Word 변경
    Mark Word → Lock Record 포인터
  
  종료:
    CAS로 Mark Word 복원
  
  재진입:
    Stack에 추가 Lock Record
    (Recursion 추적)

성공 조건:
  다른 스레드가 락 보유 안 함

실패:
  Fat Lock으로 Inflate

비용:
  획득: ~50ns (CAS 2회)
  Inflate: ~1000ns
```

---

### 5. Fat Lock (중량 락)

```
목적:
  경쟁 있는 경우 OS 동기화

동작:
  ObjectMonitor 생성
  Mark Word → Monitor 포인터
  
  대기:
    Entry Set에 추가
    park() (OS 대기)
  
  깨어남:
    unpark() (OS 깨움)
    락 재획득 시도

비용:
  획득: ~1000ns (OS 호출)
  Context Switch: ~10,000ns
```

---

## 💻 실험으로 확인하기

### 실험 1: Lock 상태 관찰

```java
import org.openjdk.jol.info.ClassLayout;

public class LockStateTest {
    public static void main(String[] args) throws Exception {
        Object obj = new Object();
        
        System.out.println("=== Unlocked ===");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        
        synchronized (obj) {
            System.out.println("=== Thin Lock ===");
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        }
        
        // Biased Lock 활성화 (기본 4초 후)
        Thread.sleep(5000);
        Object obj2 = new Object();
        
        synchronized (obj2) {
            System.out.println("=== Biased Lock ===");
            System.out.println(ClassLayout.parseInstance(obj2).toPrintable());
        }
        
        // Fat Lock 유발
        Thread t = new Thread(() -> {
            synchronized (obj2) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {}
            }
        });
        t.start();
        Thread.sleep(100);
        
        synchronized (obj2) {
            System.out.println("=== Fat Lock ===");
            System.out.println(ClassLayout.parseInstance(obj2).toPrintable());
        }
    }
}
```

---

### 실험 2: Lock 성능 측정

```java
public class LockBenchmark {
    static Object biasedObj = new Object();
    static Object thinObj = new Object();
    static Object fatObj = new Object();
    static volatile boolean flag = false;
    
    public static void main(String[] args) throws Exception {
        // Biased Lock 준비
        Thread.sleep(5000);
        synchronized (biasedObj) {}
        
        // Fat Lock 준비
        Thread t = new Thread(() -> {
            synchronized (fatObj) {
                while (!flag) {
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {}
                }
            }
        });
        t.start();
        Thread.sleep(100);
        
        // Biased Lock 측정
        long start = System.nanoTime();
        for (int i = 0; i < 10_000_000; i++) {
            synchronized (biasedObj) {}
        }
        long biasedTime = System.nanoTime() - start;
        
        // Thin Lock 측정
        start = System.nanoTime();
        for (int i = 0; i < 10_000_000; i++) {
            synchronized (thinObj) {}
        }
        long thinTime = System.nanoTime() - start;
        
        flag = true;
        t.join();
        
        // Fat Lock 측정
        start = System.nanoTime();
        for (int i = 0; i < 10_000_000; i++) {
            synchronized (fatObj) {}
        }
        long fatTime = System.nanoTime() - start;
        
        System.out.println("Biased: " + biasedTime / 10_000_000 + "ns/op");
        System.out.println("Thin: " + thinTime / 10_000_000 + "ns/op");
        System.out.println("Fat: " + fatTime / 10_000_000 + "ns/op");
    }
}
```

```bash
# 출력:
# Biased: 5ns/op
# Thin: 50ns/op
# Fat: 80ns/op (경쟁 없음)
```

---

## ⚡ 실무 임팩트

### Biased Lock Deprecated (Java 15+)

```
이유:
  1. 복잡성
     - Safepoint Revoke 비용
     - JIT 최적화 방해
  
  2. 현대 워크로드
     - 멀티스레드 증가
     - Biased Lock이 오히려 느림
  
  3. 대안
     - Thin Lock 충분히 빠름 (~50ns)
     - Lock Elision (JIT 최적화)

비활성화:
  Java 15+: 기본 비활성화
  -XX:+UseBiasedLocking (재활성화)
  
  Java 18+: 완전 제거
```

---

### Lock Coarsening (락 합치기)

```java
// ❌ Lock 반복
for (int i = 0; i < 1000; i++) {
    synchronized (obj) {
        sum += i;
    }
}

// ✅ JIT 최적화 (Lock Coarsening)
synchronized (obj) {
    for (int i = 0; i < 1000; i++) {
        sum += i;
    }
}
// → Lock 획득 1회로 최적화
```

---

## 🚫 흔한 오해

### "Biased Lock이 항상 빠르다"

```
❌ 잘못된 이해:
  Biased Lock은 무조건 성능 향상

✅ 실제:
  단일 스레드만 유리
  
  멀티스레드:
  Bias Revoke 비용 (~10,000ns)
  → Thin Lock보다 느림
  
  현대 애플리케이션:
  대부분 멀티스레드
  → Biased Lock deprecated
```

---

### "Fat Lock으로 가면 영원히 Fat"

```
❌ 잘못된 이해:
  Fat Lock은 절대 해제 안 됨

✅ 실제:
  Object 삭제 전까지 유지
  
  Deflation (축소):
  일부 GC에서 시도
  (G1 GC의 Concurrent Mark)
  
  하지만:
  대부분 Fat 유지
  (Deflation 비용이 큼)
```

---

## 📌 핵심 정리

```
Lock 타입
  Biased: 단일 스레드 최적화 (~5ns)
  Thin: 경쟁 없는 짧은 CS (~50ns)
  Fat: 경쟁 있는 경우 (~1000ns)

Mark Word
  Unlocked: 001
  Biased: 101 (Thread ID)
  Thin: 00 (Lock Record 포인터)
  Fat: 10 (Monitor 포인터)

전이
  Unlocked → Biased → Thin → Fat
  한 방향만 가능 (되돌아가기 없음)

Biased Lock
  Thread ID 저장
  재진입 시 CAS 불필요
  Revoke: Safepoint (~10,000ns)
  Java 15+: Deprecated

Thin Lock
  CAS 기반
  Stack Lock Record
  경쟁 시 Inflate

Fat Lock
  ObjectMonitor
  OS Mutex
  park/unpark

실무
  Biased Lock 비활성화 권장
  Thin Lock으로 충분
  Lock Coarsening 활용
```

---

## 🤔 생각해볼 문제

**Q1.** Biased Lock에서 다른 스레드가 락을 요청하면 Safepoint에서 Revoke가 발생한다. 왜 Safepoint가 필요한가?

**Q2.** Thin Lock의 재진입(Reentrant)은 어떻게 구현되는가? Stack Lock Record를 고려해 설명하라.

**Q3.** 다음 시나리오에서 어떤 Lock 타입이 사용될지 예측하라.
- A: 단일 스레드가 synchronized 블록을 1000번 반복
- B: 2개 스레드가 번갈아 synchronized 블록 실행 (경쟁 없음)
- C: 10개 스레드가 동시에 synchronized 블록 경쟁

> 💡 **해설**
>
> **Q1.** Safepoint 필요 이유: ① Bias Revoke는 Mark Word 변경 (Biased → Thin). ② 하지만 대상 Thread가 실행 중일 수 있음 → Stack 확인 필요 (Lock Record 위치). ③ Thread가 임의 시점에 있으면 Stack 상태 불명확 → 안전하지 않음. ④ Safepoint: 모든 Thread가 알려진 상태 (Safe Point) → Stack 안전하게 접근 가능. ⑤ Revoke 과정: Thread 정지 → Stack 스캔 → Lock Record 생성 → Mark Word 변경 → 재개. 비용: ~10,000ns (모든 Thread 정지).
>
> **Q2.** Thin Lock 재진입: ① 첫 진입: Stack에 Lock Record 1 생성 → Mark Word에 포인터 저장. ② 재진입: Stack에 Lock Record 2 추가 (Displaced Mark Word는 null) → 재진입 카운트. ③ 종료: Lock Record 2 제거. ④ 마지막 종료: Lock Record 1 제거 → CAS로 Mark Word 복원. Stack 구조: `[Lock Record 1: Mark Word 포인터][Lock Record 2: null][Lock Record 3: null]`. 재진입 횟수 = Lock Record 개수 - 1.
>
> **Q3.** A: Biased Lock (Java 14 이하) 또는 Thin Lock (Java 15+) — 단일 스레드 반복 → Biased 최적 (Thread ID만 확인). B: Thin Lock — 번갈아 실행, 경쟁 없음 → CAS 성공 → Thin 유지. Biased였다면 첫 전환 시 Revoke → Thin. C: Fat Lock — 10개 스레드 동시 경쟁 → Thin Lock CAS 실패 → Inflate → Fat Lock. Entry Set에 대기 → park/unpark. 결론: A는 Biased/Thin, B는 Thin, C는 Fat.

---

## 📚 참고 자료

- [Biased Locking in HotSpot](https://blogs.oracle.com/dave/biased-locking-in-hotspot)
- [JEP 374: Deprecate Biased Locking](https://openjdk.org/jeps/374)

---

<div align="center">

**[⬅️ 이전: Object Monitor](./01-object-monitor.md)** | **[다음: CAS & Atomic Operations ➡️](./03-cas-and-atomic-operations.md)**

</div>

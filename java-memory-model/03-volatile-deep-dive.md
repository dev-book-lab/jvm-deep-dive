# Volatile Deep Dive - Volatile 심층 분석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- volatile이 보장하는 것과 보장하지 않는 것은 무엇인가?
- volatile은 어떻게 가시성과 재정렬을 방지하는가?
- volatile의 성능 비용은 얼마나 되는가?
- 언제 volatile을 사용하고, 언제 synchronized를 사용하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 경량 동기화가 필요하다

```java
// synchronized는 무겁다
private boolean flag = false;

synchronized boolean getFlag() {
    return flag;
}

synchronized void setFlag(boolean value) {
    flag = value;
}

// volatile은 가볍다
private volatile boolean flag = false;
```

volatile은 **가벼운 동기화 메커니즘**이다.

---

## 📐 내부 구조

### 1. volatile이 보장하는 것

```
1. 가시성 (Visibility)
   한 스레드의 쓰기 → 다른 스레드의 읽기 즉시 반영
   
2. 재정렬 금지 (No Reordering)
   volatile 전후로 명령어 재정렬 금지
   
3. Happens-Before
   volatile 쓰기 happens-before 읽기

보장하지 않는 것:
   원자성 (Atomicity)
   → i++ 같은 복합 연산 불안전
```

---

### 2. Memory Barrier 삽입

```
volatile 읽기:
  LoadLoad Barrier
  [volatile read]
  LoadStore Barrier

volatile 쓰기:
  StoreStore Barrier
  [volatile write]
  StoreLoad Barrier

효과:
  - 캐시 플러시
  - 재정렬 방지
  - 가시성 보장

비용:
  읽기: ~5ns
  쓰기: ~20ns
  (synchronized: 50~100ns)
```

---

### 3. 재정렬 금지 규칙

```
일반 변수 재정렬: 허용

int a = 1;
int b = 2;
→ b = 2; a = 1; (재정렬 가능)

volatile 재정렬: 금지

volatile int v;

a = 1;
v = 2;  // volatile 쓰기
b = 3;

→ a = 1은 v = 2 전에 실행 (보장)
→ b = 3은 v = 2 후에 실행 (보장)

규칙:
  - volatile 쓰기 전 명령어는 앞으로 못 감
  - volatile 읽기 후 명령어는 뒤로 못 감
```

---

### 4. volatile이 보장하지 않는 것

```java
// ❌ 원자성 없음
volatile int counter = 0;

void increment() {
    counter++;  // 3단계: 읽기 → 증가 → 쓰기
}

// Thread 1: counter++ (0 → 1)
// Thread 2: counter++ (0 → 1)
// 결과: counter = 1 (손실!)

// ✅ 원자성 필요
AtomicInteger counter = new AtomicInteger(0);

void increment() {
    counter.incrementAndGet();  // CAS 사용
}
```

---

## 💻 실험으로 확인하기

### 실험 1: volatile 성능 측정

```java
public class VolatileBenchmark {
    private static volatile long volatileValue = 0;
    private static long normalValue = 0;
    
    public static void main(String[] args) {
        // volatile 쓰기
        long start = System.nanoTime();
        for (int i = 0; i < 10_000_000; i++) {
            volatileValue = i;
        }
        long volatileTime = System.nanoTime() - start;
        
        // 일반 쓰기
        start = System.nanoTime();
        for (int i = 0; i < 10_000_000; i++) {
            normalValue = i;
        }
        long normalTime = System.nanoTime() - start;
        
        System.out.println("Volatile: " + volatileTime / 1_000_000 + "ms");
        System.out.println("Normal: " + normalTime / 1_000_000 + "ms");
        System.out.println("Overhead: " + (volatileTime / normalTime) + "x");
    }
}
```

```bash
# 출력:
# Volatile: 150ms
# Normal: 20ms
# Overhead: 7.5x
```

---

### 실험 2: volatile 재정렬 방지

```java
public class ReorderingTest {
    private int a = 0;
    private volatile int v = 0;
    private int b = 0;
    
    void writer() {
        a = 1;  // ← v 전에 실행 보장
        v = 2;
        b = 3;  // ← v 후에 실행 보장
    }
}
```

---

## ⚡ 실무 임팩트

### 사용 패턴 1: 상태 플래그

```java
// ✅ 적합
class Server {
    private volatile boolean running = true;
    
    void run() {
        while (running) {
            handleRequest();
        }
    }
    
    void shutdown() {
        running = false;
    }
}
```

---

### 사용 패턴 2: 단순 상태 공유

```java
// ✅ 적합
class StatusHolder {
    private volatile int status = IDLE;
    
    void setStatus(int newStatus) {
        status = newStatus;  // 단순 쓰기
    }
    
    int getStatus() {
        return status;  // 단순 읽기
    }
}
```

---

### 안티패턴: 복합 연산

```java
// ❌ 부적합
class Counter {
    private volatile int count = 0;
    
    void increment() {
        count++;  // 복합 연산 → 불안전
    }
}

// ✅ 개선
class Counter {
    private final AtomicInteger count = new AtomicInteger(0);
    
    void increment() {
        count.incrementAndGet();
    }
}
```

---

## 🚫 흔한 오해

### "volatile은 synchronized의 대체제다"

```
❌ 잘못된 이해:
  volatile만 있으면 동기화 충분

✅ 실제:
  제한적 사용만 가능
  
  volatile 가능:
  - 단순 플래그
  - 단일 변수 읽기/쓰기
  
  synchronized 필요:
  - 복합 연산 (i++)
  - 다중 변수 원자성
  - Critical Section
```

---

### "volatile long/double은 특별하다"

```
❌ 잘못된 이해:
  volatile은 64bit 원자성만 보장

✅ 실제:
  volatile 없어도 대부분 CPU는 원자적
  
  volatile의 진짜 역할:
  - 가시성 보장
  - 재정렬 방지
  
  64bit 원자성:
  부수 효과일 뿐
```

---

## 📌 핵심 정리

```
volatile 보장
  1. 가시성 (캐시 플러시)
  2. 재정렬 금지
  3. Happens-Before

보장 안 함
  원자성 (i++ 불안전)

Memory Barrier
  읽기: LoadLoad, LoadStore
  쓰기: StoreStore, StoreLoad

성능
  읽기: ~5ns
  쓰기: ~20ns
  일반 변수 대비 5~10배

적합한 사용
  - 상태 플래그
  - 단순 상태 공유
  - 단일 변수 읽기/쓰기

부적합
  - 복합 연산
  - 다중 변수 원자성
  → synchronized 또는 Atomic 사용
```

---

## 🤔 생각해볼 문제

**Q1.** volatile int를 사용한 다음 코드가 안전하지 않은 이유를 설명하라.

```java
volatile int balance = 100;

void withdraw(int amount) {
    if (balance >= amount) {
        balance -= amount;
    }
}
```

**Q2.** volatile과 synchronized의 성능 차이를 Memory Barrier 관점에서 설명하라.

**Q3.** 다음 중 volatile로 충분한 것과 synchronized가 필요한 것을 구분하라.
- A: 스레드 중지 플래그
- B: 카운터 증가 (counter++)
- C: 최신 설정값 공유

> 💡 **해설**
>
> **Q1.** 불안전한 이유: ① Check-Then-Act 패턴 — `if (balance >= amount)` 체크와 `balance -= amount` 실행 사이에 갭 존재. ② Thread 1: balance=100 체크 통과. Thread 2: balance=100 체크 통과. Thread 1: balance=50 (100-50). Thread 2: balance=0 (50-50). ③ 결과: balance=-50 가능 (음수 잔액). volatile은 각 연산의 가시성만 보장, 전체 원자성 보장 안 함. 해결: synchronized 또는 AtomicInteger 사용.
>
> **Q2.** 성능 차이: volatile — StoreLoad Barrier 삽입 (~20ns), 캐시 플러시만. synchronized — Lock 획득/해제 (50~100ns), 모든 Memory Barrier 삽입, 추가로 Monitor 구조 관리 (Object Header 변경). 경쟁 없을 때: synchronized 2~5배 느림. 경쟁 있을 때: synchronized 10~100배 느림 (Wait/Notify 오버헤드). 결론: volatile이 훨씬 가볍지만 기능 제한적.
>
> **Q3.** A (플래그): volatile 충분 — 단순 boolean 읽기/쓰기, 복합 연산 없음. `volatile boolean stopped`. B (카운터): synchronized 필요 — `counter++`는 read-modify-write (복합 연산), volatile은 원자성 보장 안 함. `synchronized void increment()` 또는 `AtomicInteger`. C (설정값): volatile 충분 — 단일 참조 변경, 읽기/쓰기만. `volatile Config config`. 하지만 Config 내부 필드 변경은 별도 동기화 필요.

---

## 📚 참고 자료

- [Java Volatile Keyword](https://jenkov.com/tutorials/java-concurrency/volatile.html)
- [Understanding Volatile](https://www.baeldung.com/java-volatile)

---

<div align="center">

**[⬅️ 이전: Happens-Before](./02-happens-before.md)** | **[다음: Final Field Semantics ➡️](./04-final-field-semantics.md)**

</div>

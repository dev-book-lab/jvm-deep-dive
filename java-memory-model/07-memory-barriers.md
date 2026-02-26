# Memory Barriers - 메모리 배리어

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Memory Barrier(메모리 장벽)는 무엇이며, 왜 필요한가?
- LoadLoad, StoreStore, LoadStore, StoreLoad의 차이는?
- Java의 volatile, synchronized가 어떤 Barrier를 삽입하는가?
- CPU 명령어 수준에서 Barrier는 어떻게 구현되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: CPU 재정렬과 캐시 일관성

```
CPU는 성능을 위해 재정렬:
  Store Buffer
  Invalidation Queue
  Out-of-Order Execution

결과:
  멀티코어에서 예측 불가능한 동작
```

Memory Barrier는 **재정렬을 제한**한다.

---

## 📐 Memory Barrier 종류

### 1. LoadLoad Barrier

```
LoadLoad Barrier

Load1
LoadLoad Barrier
Load2

보장:
  Load1이 Load2보다 먼저 실행
  
예:
  x = a;  // Load1
  LoadLoad
  y = b;  // Load2
  
  → a 읽기가 b 읽기보다 먼저

CPU 명령어 (x86):
  (대부분 불필요, TSO 모델)
```

---

### 2. StoreStore Barrier

```
StoreStore Barrier

Store1
StoreStore Barrier
Store2

보장:
  Store1이 Store2보다 먼저 완료
  
예:
  a = 1;  // Store1
  StoreStore
  b = 2;  // Store2
  
  → a = 1이 캐시/메모리 반영 후 b = 2

CPU 명령어 (x86):
  (대부분 불필요, TSO 모델)
```

---

### 3. LoadStore Barrier

```
LoadStore Barrier

Load
LoadStore Barrier
Store

보장:
  Load가 Store보다 먼저 실행
  
예:
  x = a;  // Load
  LoadStore
  b = 2;  // Store
  
  → a 읽기 후 b 쓰기

CPU 명령어 (x86):
  (대부분 불필요, TSO 모델)
```

---

### 4. StoreLoad Barrier

```
StoreLoad Barrier

Store
StoreLoad Barrier
Load

보장:
  Store가 완전히 완료 후 Load 실행
  (가장 비싼 배리어)
  
예:
  a = 1;  // Store
  StoreLoad
  x = b;  // Load
  
  → a = 1이 모든 CPU에 보인 후 b 읽기

CPU 명령어 (x86):
  mfence (Memory Fence)
  lock addl $0, 0(%%rsp)
  
비용: ~20ns
```

---

## 🔧 Java 구문과 Barrier

### 1. volatile 쓰기

```
volatile int v;

a = 1;
b = 2;
[StoreStore Barrier]
v = 3;  // volatile write
[StoreLoad Barrier]

보장:
  - a, b 쓰기가 v 쓰기 전 완료
  - v 쓰기가 모든 CPU에 보임
```

---

### 2. volatile 읽기

```
volatile int v;

x = v;  // volatile read
[LoadLoad Barrier]
[LoadStore Barrier]
y = a;
z = b;

보장:
  - v 읽기가 a, b 읽기보다 먼저
  - v 읽기 시 최신 값 획득
```

---

### 3. synchronized

```
synchronized (obj) {
    [LoadLoad Barrier]
    [LoadStore Barrier]
    // Critical Section
    [StoreStore Barrier]
    [StoreLoad Barrier]
}

보장:
  모든 재정렬 방지
  완전한 가시성
```

---

### 4. final 필드

```
class Point {
    final int x;
    final int y;
    
    Point(int x, int y) {
        this.x = x;
        this.y = y;
        [StoreStore Barrier]  // 생성자 끝
    }
}

보장:
  final 필드 초기화 완료
```

---

## 💻 CPU별 Barrier 구현

### 1. x86/x64 (TSO 모델)

```
Total Store Ordering (TSO):
  - Load는 재정렬 안 됨
  - Store는 Store Buffer 사용
  - Store-Load만 재정렬 가능

필요한 Barrier:
  StoreLoad만 명시적 필요
  
  mfence    // Full Memory Barrier
  lfence    // Load Fence
  sfence    // Store Fence
  
volatile 쓰기:
  mov [addr], value
  lock addl $0, 0(%%rsp)  // StoreLoad
  
비용:
  mfence: ~20ns
```

---

### 2. ARM (Weak Ordering)

```
Weak Ordering:
  모든 재정렬 가능
  
필요한 Barrier:
  모든 종류
  
  dmb (Data Memory Barrier)
  dsb (Data Synchronization Barrier)
  isb (Instruction Synchronization Barrier)
  
volatile 쓰기:
  dmb ishst  // StoreStore
  str [addr], value
  dmb ish    // StoreLoad
  
비용:
  dmb: ~수십 ns
```

---

## ⚡ 실무 임팩트

### Lazy Initialization (DCL)

```java
// ❌ 잘못된 DCL
class Singleton {
    private static Singleton instance;
    
    static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                    // 재정렬 위험:
                    // 1. 메모리 할당
                    // 2. instance = 주소 (재정렬!)
                    // 3. 생성자 실행
                }
            }
        }
        return instance;
    }
}

// ✅ volatile로 해결
private static volatile Singleton instance;
```

---

### Happens-Before 체인

```java
int a = 0;
volatile int v = 0;
int b = 0;

// Thread 1
a = 1;
[StoreStore]
v = 2;  // volatile
[StoreLoad]

// Thread 2
[LoadLoad]
[LoadStore]
x = v;  // volatile
y = a;  // a = 1 보장
```

---

## 🚫 흔한 오해

### "x86은 Barrier가 필요 없다"

```
❌ 잘못된 이해:
  x86은 TSO라서 Barrier 불필요

✅ 실제:
  Store-Load는 재정렬됨
  
  필요한 경우:
  - volatile 쓰기 후 읽기
  - synchronized unlock/lock
  - Atomic 연산
  
  mfence 필수
```

---

### "Barrier는 성능을 크게 저하시킨다"

```
❌ 잘못된 이해:
  Barrier는 항상 느림

✅ 실제:
  x86에서 대부분 무료
  
  비용:
  - LoadLoad: ~0ns (TSO)
  - StoreStore: ~0ns (TSO)
  - LoadStore: ~0ns (TSO)
  - StoreLoad: ~20ns (mfence)
  
  ARM에서는 모두 비용
```

---

## 📌 핵심 정리

```
Memory Barrier 종류
  LoadLoad: Load 순서 보장
  StoreStore: Store 순서 보장
  LoadStore: Load → Store 순서
  StoreLoad: Store → Load 순서 (비쌈)

volatile
  쓰기: StoreStore + StoreLoad
  읽기: LoadLoad + LoadStore

synchronized
  진입: LoadLoad + LoadStore
  종료: StoreStore + StoreLoad

final
  생성자 끝: StoreStore

CPU별 구현
  x86 (TSO): StoreLoad만 필요 (mfence)
  ARM (Weak): 모든 Barrier 필요 (dmb)

비용 (x86)
  LoadLoad/StoreStore/LoadStore: ~0ns
  StoreLoad: ~20ns

실무
  volatile: Lazy Init, 플래그
  synchronized: Critical Section
  final: 불변 객체
```

---

## 🤔 생각해볼 문제

**Q1.** volatile 쓰기가 StoreStore와 StoreLoad를 모두 삽입하는 이유를 설명하라.

**Q2.** x86 (TSO)와 ARM (Weak Ordering)에서 다음 코드의 Barrier 비용을 비교하라.

```java
volatile int v;
a = 1;
v = 2;
x = v;
```

**Q3.** synchronized가 4개의 Memory Barrier를 모두 사용하는 이유를 Happens-Before 관점에서 설명하라.

> 💡 **해설**
>
> **Q1.** volatile 쓰기에 2개 Barrier 필요: ① StoreStore — volatile 쓰기 전 모든 일반 Store가 완료되도록 보장. 예: `a=1; v=2;` → a=1이 먼저 메모리 반영. ② StoreLoad — volatile 쓰기가 모든 CPU에 보인 후 다음 Load 실행. 예: `v=2; x=b;` → v=2가 모든 코어에 보인 후 b 읽기. StoreStore만으로 부족한 이유: volatile 쓰기 후 읽기에서 최신 값 보장 필요 → StoreLoad 필수. 결과: volatile 쓰기는 가장 강력한 동기화 (2개 Barrier).
>
> **Q2.** x86 비용: ① `a=1` → 0ns (일반 Store). ② StoreStore Barrier → 0ns (TSO라서 불필요). ③ `v=2` → 0ns (volatile Store). ④ StoreLoad Barrier → 20ns (mfence). ⑤ LoadLoad Barrier → 0ns (TSO). ⑥ `x=v` → 0ns (volatile Load). 총 비용: 20ns. ARM 비용: ① `a=1` → 0ns. ② StoreStore (dmb ishst) → 20ns. ③ `v=2` → 0ns. ④ StoreLoad (dmb ish) → 40ns. ⑤ LoadLoad (dmb ishld) → 20ns. ⑥ `x=v` → 0ns. 총 비용: 80ns (4배 차이). 결론: x86이 훨씬 저렴.
>
> **Q3.** synchronized에 4개 모두 필요: ① 진입 LoadLoad — 락 획득 전 읽기가 Critical Section 읽기보다 늦게 실행 안 되도록. ② 진입 LoadStore — 락 획득 전 읽기가 Critical Section 쓰기보다 늦게 실행 안 되도록. ③ 종료 StoreStore — Critical Section 쓰기가 락 해제보다 먼저 완료되도록. ④ 종료 StoreLoad — 락 해제가 다음 락 획득에 보이도록 (Monitor Lock Rule). Happens-Before: unlock happens-before lock → 모든 변경이 다음 락 획득자에게 보여야 함 → 4개 Barrier로 전방위 재정렬 차단.

---

## 📚 참고 자료

- [Memory Barriers: a Hardware View](http://www.rdrop.com/users/paulmck/scalability/paper/whymb.2010.07.23a.pdf)
- [JSR 133 Cookbook for Compiler Writers](http://gee.cs.oswego.edu/dl/jmm/cookbook.html)

---

<div align="center">

**[⬅️ 이전: Synchronized Internals](./06-synchronized-internals.md)** | **[홈으로 🏠](../README.md)**

</div>

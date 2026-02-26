# CPU Cache & Visibility Problem - CPU 캐시와 가시성 문제

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CPU 캐시 계층 구조는 어떻게 되어 있으며, 왜 필요한가?
- 멀티코어 환경에서 캐시 일관성 문제는 왜 발생하는가?
- 명령어 재정렬(Reordering)은 왜 발생하며, 어떤 문제를 일으키는가?
- Java Memory Model은 이를 어떻게 추상화하는가?
- Visibility Problem의 실제 예시는 무엇인가?

---

## 🔍 왜 이게 존재하는가

### 문제: 멀티코어 CPU의 예측 불가능한 동작

```java
class VisibilityProblem {
    private boolean ready = false;
    private int value = 0;
    
    // Thread 1
    void writer() {
        value = 42;
        ready = true;
    }
    
    // Thread 2
    void reader() {
        if (ready) {
            System.out.println(value);  // 42? 0? 무엇?
        }
    }
}
```

```
예상: 42 출력
실제: 0 출력 가능
      무한 루프 가능 (ready를 못 봄)

원인:
  - CPU 캐시
  - 명령어 재정렬
  - 메모리 모델
```

Java Memory Model은 **멀티코어 동시성을 추상화**한다.

---

## 📐 내부 구조

### 1. CPU 캐시 계층

```
CPU 캐시 계층 구조:

┌─────────────┐
│    CPU 0    │
├─────────────┤
│  L1 Cache   │  32KB, 1ns
│  (Private)  │
├─────────────┤
│  L2 Cache   │  256KB, 4ns
│  (Private)  │
├─────────────┤
│  L3 Cache   │  8MB, 40ns
│  (Shared)   │
└─────────────┘
       ↓
┌─────────────┐
│ Main Memory │  100ns
└─────────────┘

속도 차이:
  L1: 1ns
  RAM: 100ns (100배 느림)

필요성:
  CPU-RAM 속도 차이 극복
  → 캐시로 자주 쓰는 데이터 저장
```

---

### 2. 캐시 일관성 문제

```
멀티코어 환경:

CPU 0:              CPU 1:
L1: [x=0]          L1: [x=0]
     ↓                  ↓
      Main Memory: x=0

CPU 0 실행: x = 1
L1: [x=1]          L1: [x=0]
     ↓                  ↓
      Main Memory: x=0  (아직 미반영)

CPU 1 실행: read x
L1: [x=1]          L1: [x=0]  ← 0을 읽음!
     ↓                  ↓
      Main Memory: x=0

문제:
  CPU 1이 CPU 0의 변경을 못 봄
  → Visibility Problem
```

---

### 3. MESI 프로토콜

```
캐시 일관성 프로토콜 (MESI):

Modified (M):  수정됨, 다른 캐시에 없음
Exclusive (E): 독점, 수정 안 됨
Shared (S):    여러 캐시에 공유됨
Invalid (I):   무효화됨

예:
CPU 0: x = 1 (쓰기)
  → CPU 0 캐시: Modified
  → CPU 1 캐시: Invalid (무효화)
  
CPU 1: read x
  → CPU 0 캐시 확인 (Cache Coherence)
  → x = 1 가져옴

비용:
  캐시 무효화 메시지 (수십 ns)
  → 성능 저하
```

---

### 4. 명령어 재정렬

```
컴파일러/CPU 재정렬:

원본 코드:
  a = 1;
  b = 2;
  c = 3;

CPU 실행 순서:
  b = 2;  ← 재정렬!
  a = 1;
  c = 3;

이유:
  - CPU 파이프라인 최적화
  - Out-of-Order Execution
  - 메모리 대기 시간 숨김

단일 스레드: 문제 없음
  (의존성 없으면 순서 변경 OK)

멀티 스레드: 문제!
  다른 스레드가 중간 상태 관찰
```

---

### 5. Visibility Problem 실제 예시

```java
class StopThread {
    private boolean stopRequested = false;
    
    void backgroundThread() {
        while (!stopRequested) {
            // 작업 수행
        }
    }
    
    void stopThread() {
        stopRequested = true;
    }
}
```

```
문제:

Thread 1 (Background):
  L1 캐시: stopRequested = false
  → 무한 루프 (캐시에서만 읽음)

Thread 2 (Main):
  L1 캐시: stopRequested = true (쓰기)
  → Main Memory에 반영 안 됨
  → Thread 1이 못 봄

결과:
  Thread 1이 영원히 안 멈춤
```

---

### 6. Java Memory Model 추상화

```
JMM (Java Memory Model):

하드웨어 복잡성 숨김:
  - CPU 캐시
  - 명령어 재정렬
  - Store Buffer
  - Invalidation Queue

제공:
  추상화된 메모리 의미론
  
  Happens-Before 관계
  → A happens-before B
  → A의 변경을 B가 확실히 봄

구현:
  volatile, synchronized, final
  → 적절한 Memory Barrier 삽입
  → 캐시 플러시
  → 재정렬 금지
```

---

## 💻 실험으로 확인하기

### 실험 1: Visibility Problem 재현

```java
public class VisibilityTest {
    private static boolean stopRequested = false;
    
    public static void main(String[] args) throws Exception {
        Thread bg = new Thread(() -> {
            int i = 0;
            while (!stopRequested) {
                i++;
            }
            System.out.println("Stopped: " + i);
        });
        bg.start();
        
        Thread.sleep(1000);
        stopRequested = true;
        System.out.println("Stop requested");
        
        bg.join(5000);
        if (bg.isAlive()) {
            System.out.println("Thread still running!");
        }
    }
}
```

```bash
# 실행 (최적화 활성화)
java -server VisibilityTest

# 출력:
# Stop requested
# Thread still running!  ← 무한 루프
```

---

### 실험 2: volatile로 해결

```java
private static volatile boolean stopRequested = false;
```

```bash
# 실행
java VisibilityTest

# 출력:
# Stop requested
# Stopped: 1234567890  ← 정상 종료
```

---

### 실험 3: CPU 캐시 확인 (Linux)

```bash
# CPU 캐시 정보
lscpu | grep cache

# 출력:
# L1d cache:    32K
# L1i cache:    32K
# L2 cache:     256K
# L3 cache:     8192K

# Cache Line 크기
cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
# 64 (bytes)
```

---

## ⚡ 실무 임팩트

### False Sharing

```java
class Counter {
    volatile long value1;  // Cache Line 1
    volatile long value2;  // Cache Line 1 (같은 라인!)
}

// Thread 1: value1++
// Thread 2: value2++

// 문제:
// 같은 Cache Line → MESI 프로토콜 충돌
// → 성능 저하 (10배 느림)
```

```java
// 해결: Padding
class Counter {
    volatile long value1;
    long p1, p2, p3, p4, p5, p6, p7;  // Padding
    volatile long value2;  // 다른 Cache Line
}

// 또는 @Contended (Java 8+)
@Contended
volatile long value1;
```

---

### Lazy Initialization 문제

```java
class Singleton {
    private static Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();  // 문제!
        }
        return instance;
    }
}
```

```
재정렬 발생:

1. 메모리 할당
2. instance = 메모리 주소 (재정렬!)
3. 생성자 실행

Thread 1: instance = 주소 (생성자 전)
Thread 2: instance != null → 사용
         → NullPointerException
         (필드 초기화 안 됨)

해결:
  volatile 또는 Initialization-on-demand
```

---

## 🚫 흔한 오해

### "캐시는 자동으로 동기화된다"

```
❌ 잘못된 이해:
  CPU 캐시가 항상 일치한다.

✅ 실제:
  MESI 프로토콜은 "최종 일관성"
  
  즉시 동기화 아님:
  - Store Buffer 지연
  - Invalidation Queue 지연
  - 수십 ns ~ 수백 ns
  
  volatile/synchronized 없으면:
  무한정 불일치 가능
```

---

### "단일 변수 읽기/쓰기는 안전하다"

```
❌ 잘못된 이해:
  int x = 0; x = 1; 은 원자적이다.

✅ 실제:
  32bit 변수: 원자적 (대부분 CPU)
  64bit 변수 (long, double): 비원자적
  
  long x = 0;
  x = Long.MAX_VALUE;
  
  다른 스레드가 읽으면:
  상위 32bit + 하위 32bit 섞임
  → 쓰레기 값
  
  해결: volatile long
```

---

### "synchronized만 있으면 충분하다"

```
❌ 잘못된 이해:
  모든 공유 변수를 synchronized로 보호.

✅ 실제:
  성능 저하 심함
  
  Lock 비용:
  - 수십 ns (Fast Path)
  - 수백 ns (경쟁 시)
  
  volatile:
  - 수 ns (읽기)
  - 수십 ns (쓰기)
  
  적절히 조합:
  - 단순 플래그: volatile
  - 복합 연산: synchronized
  - 불변 객체: final
```

---

## 📌 핵심 정리

```
CPU 캐시 계층
  L1: 32KB, 1ns (Private)
  L2: 256KB, 4ns (Private)
  L3: 8MB, 40ns (Shared)
  RAM: 100ns

캐시 일관성
  MESI 프로토콜
  Modified, Exclusive, Shared, Invalid
  캐시 무효화 메시지 (수십 ns)

Visibility Problem
  멀티코어 환경에서 변경 미반영
  CPU 캐시 불일치
  무한 루프 가능

명령어 재정렬
  컴파일러/CPU 최적화
  Out-of-Order Execution
  멀티스레드에서 문제

Java Memory Model
  하드웨어 복잡성 추상화
  Happens-Before 관계
  volatile, synchronized, final

False Sharing
  같은 Cache Line 공유
  성능 저하 (10배)
  Padding으로 해결

실무 주의
  단일 변수도 불안전 (64bit)
  Lazy Init 재정렬 문제
  적절한 동기화 필수
```

---

## 🤔 생각해볼 문제

**Q1.** CPU 0과 CPU 1이 각각 다른 변수 x, y를 읽고 쓸 때, False Sharing이 발생할 수 있는가? Cache Line 크기를 고려해 설명하라.

**Q2.** 다음 코드에서 "Thread 2가 0을 출력"할 수 있는 이유를 CPU 캐시와 명령어 재정렬 관점에서 설명하라.

```java
int a = 0, b = 0;

// Thread 1
a = 1;
b = 1;

// Thread 2
if (b == 1) {
    System.out.println(a);  // 0 가능?
}
```

**Q3.** volatile을 사용하지 않고 Visibility Problem을 해결할 수 있는 방법 3가지를 제시하라.

> 💡 **해설**
>
> **Q1.** False Sharing 발생 가능. 이유: Cache Line 크기는 64 bytes. x, y가 연속된 메모리 (예: 같은 객체의 필드)면 같은 Cache Line에 위치 가능. CPU 0이 x 쓰기 → Cache Line Modified. CPU 1이 y 쓰기 → CPU 0의 Cache Line Invalid → 재로드. 서로 다른 변수지만 같은 Cache Line → MESI 충돌 → 성능 저하. 해결: 변수 사이에 Padding (56 bytes 이상) 추가 → 다른 Cache Line 배치.
>
> **Q2.** 0 출력 가능한 이유: ① CPU 캐시 — Thread 1이 a=1 쓰기 → CPU 0 캐시에만 (Main Memory 미반영). Thread 2가 a 읽기 → CPU 1 캐시에서 (아직 0). ② 명령어 재정렬 — Thread 1: b=1, a=1 순서로 재정렬. Thread 2: b==1 확인 → a 읽기 (아직 a=1 실행 전) → 0. 해결: volatile int a, b; 또는 synchronized 사용.
>
> **Q3.** volatile 없이 해결: ① synchronized 블록 — 모든 접근을 synchronized로 감싸기. Lock 획득/해제 시 Memory Barrier 삽입 → 캐시 플러시. ② AtomicBoolean 사용 — 내부적으로 volatile 사용하지만, API로 추상화. ③ Thread.join() — Thread 1 종료 후 Thread 2 시작. Join은 Happens-Before 보장 → Thread 1의 모든 변경을 Thread 2가 봄.

---

## 📚 참고 자료

- [JSR 133: Java Memory Model](https://www.cs.umd.edu/~pugh/java/memoryModel/)
- [CPU Caches and Why You Care](https://www.aristeia.com/TalkNotes/codedive-CPUCachesHandouts.pdf)
- [Memory Barriers: a Hardware View for Software Hackers](http://www.rdrop.com/users/paulmck/scalability/paper/whymb.2010.07.23a.pdf)

---

<div align="center">

**[다음: Happens-Before ➡️](./02-happens-before.md)**

</div>

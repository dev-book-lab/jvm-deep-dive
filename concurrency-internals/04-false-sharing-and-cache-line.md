# False Sharing & Cache Line - 거짓 공유와 캐시 라인

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- False Sharing은 무엇이며, 왜 성능 문제를 일으키는가?
- Cache Line은 무엇이며, 크기는 얼마나 되는가?
- @Contended 어노테이션은 어떻게 동작하는가?
- JMH로 False Sharing을 어떻게 측정하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 독립적인 변수인데 왜 느린가?

```java
class Counter {
    volatile long value1 = 0;  // Thread 1 전용
    volatile long value2 = 0;  // Thread 2 전용
}

// Thread 1: value1++
// Thread 2: value2++
// → 독립적인데도 10배 느림!
```

```
원인:
  같은 Cache Line 공유
  → MESI 프로토콜 충돌
  → False Sharing
```

False Sharing은 **캐시 일관성의 부작용**이다.

---

## 📐 Cache Line 구조

### 1. CPU 캐시 계층

```
CPU Cache 구조:

┌──────────┐
│   CPU    │
├──────────┤
│ L1 Cache │  32KB, 64-byte lines
│ (Private)│
├──────────┤
│ L2 Cache │  256KB, 64-byte lines
│ (Private)│
├──────────┤
│ L3 Cache │  8MB, 64-byte lines
│ (Shared) │
└──────────┘
      ↓
┌──────────┐
│   RAM    │
└──────────┘

Cache Line:
  데이터 전송 단위
  x86/x64: 64 bytes
  ARM: 64 bytes
```

---

### 2. Cache Line 구조

```
Cache Line (64 bytes):

┌─────┬─────┬─────┬───┬─────┐
│ 8B  │ 8B  │ 8B  │...│ 8B  │
└─────┴─────┴─────┴───┴─────┘
  0-7  8-15  16-23     56-63

예: long 변수 (8 bytes)
  1개 Cache Line에 8개 저장 가능

메모리 레이아웃:
  Object obj = new Object();
  
  [Object Header: 16B][field1: 8B][field2: 8B][...]
  └────────────── Cache Line 1 ──────────────┘
```

---

### 3. MESI 프로토콜 복습

```
MESI States:
  Modified (M): 수정됨, 독점
  Exclusive (E): 독점, 미수정
  Shared (S): 공유됨
  Invalid (I): 무효화됨

예: value1, value2가 같은 Cache Line

CPU 0:              CPU 1:
[value1, value2]    [value1, value2]
  M                   S

CPU 0: value1 = 1 (쓰기)
→ CPU 1 Cache Line: Invalid

CPU 1: value2 = 2 (쓰기)
→ CPU 0에서 Cache Line 재로드
→ 불필요한 동기화!
```

---

### 4. False Sharing 발생

```java
class FalseSharing {
    volatile long value1;  // Offset 0
    volatile long value2;  // Offset 8
    
    // 둘 다 같은 64-byte Cache Line
}

Thread 1 (CPU 0):
  for (int i = 0; i < 100_000_000; i++) {
      value1++;
  }

Thread 2 (CPU 1):
  for (int i = 0; i < 100_000_000; i++) {
      value2++;
  }

문제:
  value1, value2가 같은 Cache Line
  → 서로 영향
  → MESI Invalidation
  → 성능 10배 저하
```

---

### 5. Padding으로 해결

```java
class NoFalseSharing {
    volatile long value1;
    long p1, p2, p3, p4, p5, p6, p7;  // 56 bytes padding
    volatile long value2;
    
    // value1: Cache Line 1
    // value2: Cache Line 2 (분리됨)
}

결과:
  독립된 Cache Line
  → MESI 충돌 없음
  → 10배 빠름
```

---

### 6. @Contended 어노테이션

```java
import jdk.internal.vm.annotation.Contended;

class ContendedExample {
    @Contended
    volatile long value1;
    
    @Contended
    volatile long value2;
}

// JVM이 자동으로 Padding 삽입
// -XX:-RestrictContended 필요 (사용자 코드)

내부 동작:
  @Contended → 128 bytes padding (기본)
  -XX:ContendedPaddingWidth=128

Thread 클래스 예:
  @Contended("tlr")
  long threadLocalRandomSeed;
  → False Sharing 방지
```

---

## 💻 실험으로 확인하기

### 실험 1: False Sharing 측정 (JMH)

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@State(Scope.Group)
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
public class FalseSharingBenchmark {
    
    // False Sharing 발생
    static class FalseSharing {
        volatile long value1;
        volatile long value2;
    }
    
    // Padding으로 방지
    static class NoPadding {
        volatile long value1;
        long p1, p2, p3, p4, p5, p6, p7;
        volatile long value2;
    }
    
    FalseSharing falseSharing = new FalseSharing();
    NoPadding noPadding = new NoPadding();
    
    @Benchmark
    @Group("false")
    @GroupThreads(1)
    public void falseSharingWriter1() {
        falseSharing.value1++;
    }
    
    @Benchmark
    @Group("false")
    @GroupThreads(1)
    public void falseSharingWriter2() {
        falseSharing.value2++;
    }
    
    @Benchmark
    @Group("nopadding")
    @GroupThreads(1)
    public void noPaddingWriter1() {
        noPadding.value1++;
    }
    
    @Benchmark
    @Group("nopadding")
    @GroupThreads(1)
    public void noPaddingWriter2() {
        noPadding.value2++;
    }
}
```

```bash
# 실행
mvn clean install
java -jar target/benchmarks.jar FalseSharingBenchmark

# 결과:
# false:           10,000 ops/ms
# nopadding:      100,000 ops/ms  ← 10배 빠름
```

---

### 실험 2: Cache Line 크기 확인

```bash
# Linux
cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
# 64

# macOS
sysctl hw.cachelinesize
# hw.cachelinesize: 64

# Java
java -XX:+PrintFlagsFinal -version | grep CacheLineSize
# intx CacheLineSize = 64
```

---

### 실험 3: @Contended 효과

```java
import jdk.internal.vm.annotation.Contended;

public class ContendedTest {
    @Contended
    volatile long value1;
    
    @Contended
    volatile long value2;
    
    public static void main(String[] args) throws Exception {
        ContendedTest obj = new ContendedTest();
        
        // 필드 오프셋 확인
        Field f1 = ContendedTest.class.getDeclaredField("value1");
        Field f2 = ContendedTest.class.getDeclaredField("value2");
        
        long offset1 = unsafe.objectFieldOffset(f1);
        long offset2 = unsafe.objectFieldOffset(f2);
        
        System.out.println("value1 offset: " + offset1);
        System.out.println("value2 offset: " + offset2);
        System.out.println("Distance: " + (offset2 - offset1));
    }
}
```

```bash
# 실행
java -XX:-RestrictContended ContendedTest

# 출력:
# value1 offset: 144
# value2 offset: 280
# Distance: 136  ← 128 bytes padding
```

---

## ⚡ 실무 임팩트

### Disruptor (LMAX)

```java
// Disruptor RingBuffer
class RingBufferPadded {
    // Padding
    long p1, p2, p3, p4, p5, p6, p7;
    
    // Hot fields
    volatile long cursor;
    
    // Padding
    long p8, p9, p10, p11, p12, p13, p14;
}

// 성능:
// 초당 600만 메시지 처리
// False Sharing 제거로 달성
```

---

### ConcurrentHashMap (Java 8+)

```java
// CounterCell
@Contended
static final class CounterCell {
    volatile long value;
    // @Contended로 False Sharing 방지
}

// 멀티스레드 카운팅
// 각 스레드가 독립 CounterCell 사용
// → False Sharing 없음
```

---

### ThreadLocalRandom

```java
// Thread 클래스
@Contended("tlr")
long threadLocalRandomSeed;

@Contended("tlr")
int threadLocalRandomProbe;

@Contended("tlr")
int threadLocalRandomSecondarySeed;

// @Contended로 다른 Thread 필드와 분리
// → False Sharing 방지
```

---

## 🚫 흔한 오해

### "Padding은 메모리 낭비다"

```
❌ 잘못된 이해:
  Padding 56 bytes = 메모리 낭비

✅ 실제:
  성능 10배 향상 vs 56 bytes
  
  트레이드오프:
  메모리 56 bytes 증가
  → 성능 10배 향상
  → 충분히 가치 있음
  
  주의:
  수백만 객체면 고려 필요
  (56 MB per 1M objects)
```

---

### "@Contended는 항상 사용해야 한다"

```
❌ 잘못된 이해:
  모든 volatile에 @Contended

✅ 실제:
  Hot Path만 적용
  
  적용 기준:
  - 멀티스레드 경쟁
  - 높은 쓰기 빈도
  - 성능 병목
  
  불필요:
  - 단일 스레드
  - 읽기 전용
  - 낮은 빈도
```

---

## 📌 핵심 정리

```
Cache Line
  데이터 전송 단위
  x86/ARM: 64 bytes
  8개 long 저장 가능

False Sharing
  독립 변수가 같은 Cache Line
  → MESI 프로토콜 충돌
  → 성능 10배 저하

MESI 프로토콜
  Modified, Exclusive, Shared, Invalid
  Cache Line 단위 동기화
  → False Sharing 원인

해결 방법
  1. Padding (56 bytes)
  2. @Contended (128 bytes)
  3. 변수 분리 (다른 객체)

@Contended
  JVM 자동 Padding
  -XX:-RestrictContended
  Thread, ConcurrentHashMap 사용

성능 영향
  False Sharing: 10배 느림
  Padding: 10배 빠름
  메모리: 객체당 ~128 bytes

실무 적용
  Disruptor RingBuffer
  ConcurrentHashMap CounterCell
  ThreadLocalRandom
  Hot Path 변수만 적용
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 클래스에서 False Sharing이 발생하는 변수 쌍을 찾고, Padding으로 해결하라.

```java
class Statistics {
    volatile long readCount;   // 8 bytes, offset 16
    volatile long writeCount;  // 8 bytes, offset 24
    volatile long errorCount;  // 8 bytes, offset 32
    String name;               // 8 bytes, offset 40
}
```

**Q2.** @Contended를 사용할 때 `-XX:-RestrictContended` 플래그가 필요한 이유를 설명하라.

**Q3.** 100만 개 객체를 생성하는 애플리케이션에서 @Contended를 사용하면 메모리가 얼마나 증가하는가? 이것이 허용 가능한지 판단하라.

> 💡 **해설**
>
> **Q1.** False Sharing 발생: readCount (offset 16), writeCount (24), errorCount (32) — 모두 같은 64-byte Cache Line (offset 0~63). 멀티스레드에서 각각 증가 시 MESI 충돌. 해결: `volatile long readCount; long p1, p2, p3, p4, p5, p6, p7; volatile long writeCount; long p8, p9, p10, p11, p12, p13, p14; volatile long errorCount;` — 각 변수를 다른 Cache Line에 배치. 또는 `@Contended volatile long readCount/writeCount/errorCount;`.
>
> **Q2.** RestrictContended 이유: @Contended는 JDK 내부 API (jdk.internal.vm.annotation) — 기본적으로 JDK 클래스만 사용 가능. 사용자 코드에서 사용 시 무시됨. `-XX:-RestrictContended` — 제한 해제 → 사용자 코드도 @Contended 적용 가능. 이유: 메모리 오버헤드 방지 — 무분별한 사용 시 메모리 낭비 우려 → 제한. 실무: 성능 크리티컬한 곳만 신중히 사용.
>
> **Q3.** 메모리 증가: @Contended는 기본 128 bytes padding. 객체당 128 bytes × 1,000,000 = 128 MB 증가. 허용 가능 여부: ① 힙 크기 고려 — 4GB 힙이면 128MB는 3% (허용 가능). ② 성능 이득 — 10배 향상이면 128MB 가치 있음. ③ 대안 — 수동 Padding (56 bytes)으로 56MB만 증가. 결론: 성능이 크리티컬하고 힙 여유 있으면 허용 가능. 아니면 Hot Path만 선택적 적용.

---

## 📚 참고 자료

- [False Sharing (Martin Thompson)](https://mechanical-sympathy.blogspot.com/2011/07/false-sharing.html)
- [@Contended Annotation](https://shipilev.net/blog/2014/jmm-pragmatics/#_contended)
- [LMAX Disruptor](https://lmax-exchange.github.io/disruptor/)

---

<div align="center">

**[⬅️ 이전: CAS & Atomic Operations](./03-cas-and-atomic-operations.md)** | **[다음: AQS Internals ➡️](./05-aqs-internals.md)**

</div>

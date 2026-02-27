# Unsafe API - Unsafe API

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `sun.misc.Unsafe`는 무엇이며, 왜 존재하는가?
- Unsafe로 무엇을 할 수 있는가?
- JDK 내부 코드가 Unsafe를 사용하는 이유는?
- Unsafe의 위험성과 대안은 무엇인가?

---

## 🔍 왜 이게 존재하는가

### 문제: Java의 제약을 넘어야 한다

```
Java의 제약:
  - 메모리 직접 접근 불가
  - Null 체크 강제
  - 배열 범위 체크
  → 성능 오버헤드

Native 코드:
  빠르지만 JNI 비용

Unsafe:
  Java에서 직접 메모리 조작
  → 성능 최적화
```

Unsafe는 **저수준 메모리 조작 API**다.

---

## 📐 Unsafe 기능

### 1. 메모리 직접 조작

```java
Unsafe unsafe = getUnsafe();

// Off-Heap 메모리 할당
long address = unsafe.allocateMemory(1024);

// 메모리 쓰기
unsafe.putLong(address, 123456789L);

// 메모리 읽기
long value = unsafe.getLong(address);

// 메모리 해제
unsafe.freeMemory(address);

특징:
  GC 관리 밖
  수동 해제 필요
  빠른 접근
```

---

### 2. 객체 필드 직접 조작

```java
class Point {
    private int x;
    private int y;
}

Unsafe unsafe = getUnsafe();
Point p = new Point();

// 필드 오프셋
long xOffset = unsafe.objectFieldOffset(
    Point.class.getDeclaredField("x"));

// private 필드 직접 쓰기
unsafe.putInt(p, xOffset, 100);

// private 필드 직접 읽기
int x = unsafe.getInt(p, xOffset);

장점:
  Reflection 없이 접근
  매우 빠름 (수 ns)
```

---

### 3. CAS (Compare-And-Swap)

```java
// AtomicInteger 내부 구현
class AtomicInteger {
    private volatile int value;
    private static final Unsafe unsafe = ...;
    private static final long valueOffset;
    
    static {
        valueOffset = unsafe.objectFieldOffset(
            AtomicInteger.class.getDeclaredField("value"));
    }
    
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
}

CPU 명령어:
  CMPXCHG (x86)
  원자적 연산
```

---

### 4. 클래스/객체 생성

```java
// 생성자 없이 인스턴스 생성
Class<?> clazz = MyClass.class;
Object obj = unsafe.allocateInstance(clazz);
// 생성자 실행 안 됨!

// 클래스 정의
byte[] classBytes = loadClassBytes();
Class<?> clazz = unsafe.defineClass(
    "MyClass", classBytes, 0, classBytes.length, 
    classLoader, null);

용도:
  Serialization
  Proxy 생성
  Framework
```

---

### 5. 배열 직접 접근

```java
int[] array = new int[100];

// 배열 Base Offset
long baseOffset = unsafe.arrayBaseOffset(int[].class);

// 배열 Index Scale
long indexScale = unsafe.arrayIndexScale(int[].class);

// array[10] 직접 접근
long offset = baseOffset + 10 * indexScale;
unsafe.putInt(array, offset, 42);

int value = unsafe.getInt(array, offset);

장점:
  범위 체크 없음
  빠른 접근
```

---

## 💻 실험으로 확인하기

### 실험 1: Unsafe 획득

```java
import sun.misc.Unsafe;
import java.lang.reflect.Field;

public class UnsafeTest {
    public static Unsafe getUnsafe() throws Exception {
        Field f = Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        return (Unsafe) f.get(null);
    }
    
    public static void main(String[] args) throws Exception {
        Unsafe unsafe = getUnsafe();
        System.out.println("Unsafe: " + unsafe);
    }
}
```

---

### 실험 2: Off-Heap 메모리

```java
public class OffHeapTest {
    public static void main(String[] args) throws Exception {
        Unsafe unsafe = getUnsafe();
        
        // 1MB 할당
        long address = unsafe.allocateMemory(1024 * 1024);
        System.out.println("Allocated: " + address);
        
        // 데이터 쓰기
        for (int i = 0; i < 1024; i++) {
            unsafe.putLong(address + i * 8, i);
        }
        
        // 데이터 읽기
        long sum = 0;
        for (int i = 0; i < 1024; i++) {
            sum += unsafe.getLong(address + i * 8);
        }
        System.out.println("Sum: " + sum);
        
        // 해제
        unsafe.freeMemory(address);
    }
}
```

---

## ⚡ 실무 사용 예시

### JDK 내부 사용

```
AtomicInteger/AtomicLong:
  CAS 연산 (compareAndSwapInt)

DirectByteBuffer:
  Off-Heap 메모리 관리
  allocateMemory / freeMemory

Concurrent 자료구조:
  ConcurrentHashMap
  ConcurrentLinkedQueue
  → CAS 기반 Lock-Free

Serialization:
  allocateInstance (생성자 없이)
```

---

### 외부 라이브러리

```
Netty:
  DirectByteBuffer 풀링
  Zero-Copy I/O

Kafka:
  Off-Heap 버퍼
  고성능 메시지 처리

Apache Arrow:
  Columnar 메모리 레이아웃
  Off-Heap 데이터 구조
```

---

## 🚫 위험성과 대안

### 위험성

```
1. 메모리 손상
   잘못된 주소 접근 → JVM Crash

2. 보안 위험
   private 필드 접근
   메모리 보호 우회

3. 메모리 누수
   freeMemory 누락
   → Native Memory Leak

4. JVM 버전 의존
   내부 API → 변경 가능성
```

---

### 대안 (Java 9+)

```java
// VarHandle (Java 9+)
class Point {
    volatile int x;
    
    static final VarHandle X;
    static {
        X = MethodHandles.lookup()
            .findVarHandle(Point.class, "x", int.class);
    }
    
    void setX(int value) {
        X.set(this, value);  // Volatile 쓰기
    }
    
    boolean casX(int expect, int update) {
        return X.compareAndSet(this, expect, update);
    }
}

장점:
  안전한 API
  JVM 최적화
  Module System 호환
```

---

## 📌 핵심 정리

```
Unsafe
  sun.misc.Unsafe
  저수준 메모리 조작 API

기능
  Off-Heap 메모리 (allocate/free)
  객체 필드 직접 조작
  CAS 연산
  배열 직접 접근
  생성자 없이 인스턴스

성능
  Reflection보다 100배 빠름
  범위 체크 없음
  JNI 없이 Native 수준

사용처
  JDK: Atomic, DirectByteBuffer
  외부: Netty, Kafka, Arrow

위험
  메모리 손상 → Crash
  보안 우회
  메모리 누수

대안
  VarHandle (Java 9+)
  안전한 API
  동일 성능
```

---

## 🤔 생각해볼 문제

**Q1.** Unsafe를 사용해 Off-Heap 메모리를 할당했지만 `freeMemory()`를 호출하지 않았다. 어떤 문제가 발생하는가?

**Q2.** Reflection과 Unsafe의 필드 접근 성능 차이를 설명하라. 왜 Unsafe가 빠른가?

**Q3.** Java 9+ 환경에서 Unsafe 대신 VarHandle을 사용해야 하는 이유 3가지를 제시하라.

> 💡 **해설**
>
> **Q1.** freeMemory() 누락 문제: ① Off-Heap 메모리는 GC 관리 밖 → 자동 해제 안 됨. ② Native Memory Leak 발생 → 계속 증가. ③ OS 메모리 고갈 → 프로세스 종료 또는 OOM. ④ JVM Heap은 정상이지만 Native Memory 부족. ⑤ 탐지 어려움: JVM 툴로 안 보임. 해결: try-finally로 freeMemory() 보장.
>
> **Q2.** 성능 차이: ① Reflection — 메서드 호출, 보안 체크, Boxing/Unboxing → 수백 ns. ② Unsafe — 직접 메모리 오프셋 접근, 체크 없음 → 수 ns. ③ 차이: 100배 (Reflection 500ns vs Unsafe 5ns). 이유: Unsafe는 JVM 내부 최적화 경로, JIT가 인라인 가능.
>
> **Q3.** VarHandle 사용 이유: ① 안전성 — Unsafe는 잘못 사용 시 Crash, VarHandle은 체크 있음. ② Module System — Java 9+ 모듈에서 Unsafe 접근 제한, VarHandle은 공식 API. ③ 동일 성능 — VarHandle도 JVM 내부 최적화, Unsafe와 유사한 속도. ④ 유지보수 — Unsafe는 내부 API (변경 가능), VarHandle은 표준 API.

---

## 📚 참고 자료

- [Unsafe Javadoc](http://www.docjar.com/docs/api/sun/misc/Unsafe.html)
- [VarHandle Guide](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/invoke/VarHandle.html)

---

<div align="center">

**[⬅️ 이전: String Pool & Interning](./03-string-pool-interning.md)** | **[다음: Reflection & Performance ➡️](./05-reflection-and-performance.md)**

</div>

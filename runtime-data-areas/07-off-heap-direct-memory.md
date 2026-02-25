# Off-Heap & Direct Memory - 오프힙과 다이렉트 메모리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Off-Heap 메모리는 무엇이며, Heap과 어떻게 다른가?
- `ByteBuffer.allocateDirect()`는 어디에 메모리를 할당하는가?
- Off-Heap 메모리는 GC의 영향을 받지 않는가? 메모리 누수 위험은?
- `sun.misc.Unsafe`를 사용하는 이유와 위험성은 무엇인가?
- `-XX:MaxDirectMemorySize`는 무엇을 제어하며, 어떻게 설정해야 하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: Heap만으로는 해결하기 어려운 케이스들

```
문제 1: 대용량 파일 I/O
  1GB 파일을 읽어 네트워크로 전송
  
  Heap 사용 시:
  1. 파일 → OS 버퍼 (커널 공간)
  2. OS 버퍼 → JVM Heap (복사)
  3. JVM Heap → OS 소켓 버퍼 (복사)
  
  → 불필요한 복사 2회
  → 메모리 낭비, 성능 저하

문제 2: GC 압박
  수 GB의 캐시 데이터를 Heap에 저장
  → GC가 계속 스캔
  → STW 증가
  → 처리량 감소

문제 3: JNI 네이티브 라이브러리
  C/C++ 라이브러리와 메모리 공유
  → Heap 객체는 주소가 GC로 변경 가능
  → 네이티브 코드가 잘못된 주소 참조
```

JVM은 이를 **Off-Heap 메모리**로 해결한다.

---

## 📐 내부 구조

### 1. Heap vs Off-Heap

```
JVM 메모리 구조:

┌─────────────────────────────────────────┐
│          JVM Process                    │
├─────────────────────────────────────────┤
│  Heap (-Xmx로 제한)                       │
│  ┌─────────────────────────────────┐    │
│  │  Young Gen                      │    │
│  │  Old Gen                        │    │
│  │  → GC 관리                       │    │
│  │  → 객체 이동 가능                  │    │
│  └─────────────────────────────────┘    │
├─────────────────────────────────────────┤
│  Metaspace (네이티브 메모리)                │
│  → 클래스 메타데이터                         │
├─────────────────────────────────────────┤
│  Thread Stacks (네이티브 메모리)            │
│  → 스레드당 -Xss 크기                      │
├─────────────────────────────────────────┤
│  Direct Memory (네이티브 메모리)            │
│  ┌─────────────────────────────────┐    │
│  │  ByteBuffer.allocateDirect()    │    │
│  │  → GC 관리 안 함                  │    │
│  │  → 주소 고정                      │    │
│  │  → OS와 직접 통신 가능              │    │
│  └─────────────────────────────────┘    │
├─────────────────────────────────────────┤
│  Native Memory (기타)                    │
│  → JNI, Unsafe 할당                      │
└─────────────────────────────────────────┘

특징:
  Heap: JVM 관리, GC 대상, 주소 이동 가능
  Off-Heap: OS 관리, GC 안 함, 주소 고정
```

---

### 2. DirectByteBuffer 구조

```java
ByteBuffer direct = ByteBuffer.allocateDirect(1024);
```

```
내부 구조:

Heap:
┌─────────────────────────────────┐
│  DirectByteBuffer 객체           │
│  ┌─────────────────────────┐    │
│  │ address: 0x7f3a2b10000  │ ───┼──┐
│  │ capacity: 1024          │    │  │
│  │ cleaner: Cleaner 객체    │    │  │
│  └─────────────────────────┘    │  │
└─────────────────────────────────┘  │
                                     │
Off-Heap (Native Memory):            │
┌─────────────────────────────────┐  │
│  0x7f3a2b10000                  │ ←┘
│  [1024 bytes 버퍼]               │
│  → malloc()으로 할당              │
│  → GC가 접근 안 함                 │
└─────────────────────────────────┘

Cleaner:
  DirectByteBuffer 객체가 GC되면
  → Cleaner.clean() 호출
  → free() 실행 (네이티브 메모리 해제)
  
  하지만 즉시 해제는 아님
  → Full GC 또는 Reference 처리 시점
  → 메모리 누수 가능성
```

---

### 3. Direct Memory vs Heap Memory I/O

```
파일 읽기 예시:

Heap Buffer 사용:
┌──────────┐    복사1   ┌──────────┐    복사2   ┌──────────┐
│  File    │ ────────→ │OS Kernel │ ────────→ │JVM Heap  │
│(Disk)    │           │ Buffer   │           │ Buffer   │
└──────────┘           └──────────┘           └──────────┘
  
  FileChannel.read(heapBuffer):
  1. Disk → Kernel Buffer (DMA)
  2. Kernel Buffer → Heap Buffer (CPU 복사)
  → 2단계 필요

Direct Buffer 사용:
┌──────────┐    DMA    ┌──────────────────────┐
│  File    │ ────────→ │ Direct Buffer        │
│(Disk)    │           │ (네이티브 메모리)        │
└──────────┘           └──────────────────────┘
  
  FileChannel.read(directBuffer):
  1. Disk → Direct Buffer (DMA, 복사 없음)
  → 1단계 (Zero-Copy)

속도:
  대용량 I/O (MB~GB): Direct Buffer가 2~3배 빠름
  소량 I/O (KB 이하): Heap Buffer와 비슷 (할당 비용)
```

---

### 4. sun.misc.Unsafe

```java
import sun.misc.Unsafe;
import java.lang.reflect.Field;

public class UnsafeDemo {
    private static Unsafe unsafe;
    
    static {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            unsafe = (Unsafe) f.get(null);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
    
    public static void main(String[] args) {
        // Off-Heap 메모리 할당
        long address = unsafe.allocateMemory(1024);
        
        try {
            // 직접 메모리 조작
            unsafe.putInt(address, 42);
            int value = unsafe.getInt(address);
            System.out.println("Value: " + value);  // 42
            
        } finally {
            // 수동 해제 필수
            unsafe.freeMemory(address);
        }
    }
}
```

```
Unsafe의 용도:

1. 직접 메모리 할당/해제
   allocateMemory(size) → malloc()
   freeMemory(address)  → free()

2. 메모리 직접 읽기/쓰기
   putInt(address, value)
   getInt(address)
   
3. CAS (Compare-And-Swap)
   compareAndSwapInt(object, offset, expect, update)
   → AtomicInteger 내부 구현

4. 객체 메모리 복사
   copyMemory(src, dst, bytes)
   
위험성:
  - 잘못된 주소 접근 → Segmentation Fault (JVM 크래시)
  - 메모리 누수 (수동 해제 필수)
  - 타입 안전성 없음
  
  JDK 내부 코드(java.nio, java.util.concurrent)만 사용 권장
```

---

## 💻 실험으로 확인하기

### 실험 1: Direct vs Heap Buffer 성능 비교

```java
import java.io.*;
import java.nio.*;
import java.nio.channels.*;

public class BufferBenchmark {
    static final int SIZE = 100 * 1024 * 1024; // 100MB
    
    public static void main(String[] args) throws Exception {
        File file = new File("test.dat");
        
        // 테스트 파일 생성
        try (FileOutputStream fos = new FileOutputStream(file)) {
            fos.write(new byte[SIZE]);
        }
        
        // Heap Buffer
        long start = System.nanoTime();
        try (FileChannel channel = new FileInputStream(file).getChannel()) {
            ByteBuffer heap = ByteBuffer.allocate(8192);
            while (channel.read(heap) > 0) {
                heap.clear();
            }
        }
        long heapTime = System.nanoTime() - start;
        
        // Direct Buffer
        start = System.nanoTime();
        try (FileChannel channel = new FileInputStream(file).getChannel()) {
            ByteBuffer direct = ByteBuffer.allocateDirect(8192);
            while (channel.read(direct) > 0) {
                direct.clear();
            }
        }
        long directTime = System.nanoTime() - start;
        
        System.out.printf("Heap Buffer:   %d ms%n", heapTime / 1_000_000);
        System.out.printf("Direct Buffer: %d ms%n", directTime / 1_000_000);
        System.out.printf("Direct가 %.1f배 빠름%n", (double)heapTime / directTime);
        
        file.delete();
    }
}
```

```bash
# 실행 결과 예시:
# Heap Buffer:   350 ms
# Direct Buffer: 120 ms
# Direct가 2.9배 빠름
```

---

### 실험 2: Direct Memory 누수 재현

```java
public class DirectMemoryLeak {
    public static void main(String[] args) throws Exception {
        long allocated = 0;
        
        try {
            while (true) {
                ByteBuffer.allocateDirect(1024 * 1024); // 1MB
                allocated += 1;
                
                if (allocated % 100 == 0) {
                    System.out.println("Allocated: " + allocated + " MB");
                }
            }
        } catch (OutOfMemoryError e) {
            System.out.println("OOM after " + allocated + " MB");
            System.out.println("Error: " + e.getMessage());
        }
    }
}
```

```bash
# -XX:MaxDirectMemorySize 제한 설정
java -XX:MaxDirectMemorySize=512m DirectMemoryLeak

# 출력:
# Allocated: 100 MB
# Allocated: 200 MB
# ...
# Allocated: 500 MB
# OOM after 512 MB
# Error: Direct buffer memory

# DirectByteBuffer 객체는 Heap에 생성되지만
# 실제 버퍼는 Off-Heap에 할당
# → MaxDirectMemorySize 초과 시 OOM
```

---

### 실험 3: jcmd로 Direct Memory 사용량 확인

```bash
# JVM 프로세스 찾기
jps

# Direct Memory 사용량 확인
jcmd <pid> VM.native_memory

# 출력 예시:
# Internal (reserved=12345KB, committed=12345KB)
# - (malloc=10000KB #1234)
# - (mmap: reserved=2345KB, committed=2345KB)
#
# Direct buffer (reserved=524288KB, committed=524288KB)  ← Direct Memory
```

---

## ⚡ 실무 임팩트

### Direct Memory 사용 시기

```
Direct Buffer 사용 권장:
  ✓ 대용량 파일 I/O (수 MB 이상)
  ✓ 네트워크 통신 (소켓 I/O)
  ✓ 장시간 유지되는 버퍼
  ✓ JNI 네이티브 라이브러리와 메모리 공유

Heap Buffer 사용 권장:
  ✓ 소량 데이터 (<1KB)
  ✓ 단기 사용 버퍼
  ✓ 빈번한 할당/해제
  ✓ GC로 자동 관리 필요

이유:
  Direct Buffer 할당은 Heap보다 느림 (OS syscall)
  하지만 I/O는 훨씬 빠름 (복사 없음)
```

### Netty의 Direct Buffer 전략

```java
// Netty는 기본적으로 Direct Buffer 풀 사용
PooledByteBufAllocator allocator = PooledByteBufAllocator.DEFAULT;

// Direct Buffer 할당
ByteBuf buf = allocator.directBuffer(1024);
try {
    // 네트워크 I/O
    channel.writeAndFlush(buf);
} finally {
    buf.release();  // 풀로 반환 (재사용)
}

풀링 효과:
  - 할당/해제 비용 제거
  - Direct Memory 재사용
  - GC 압박 최소화
```

### MaxDirectMemorySize 설정

```bash
# 기본값: -Xmx와 동일 (또는 무제한)
java -Xmx4g MyApp
# → MaxDirectMemorySize도 ~4GB

# 명시적 설정 권장
java -Xmx4g -XX:MaxDirectMemorySize=1g MyApp

계산:
  총 메모리 = Heap + Metaspace + Direct Memory + Thread Stack + 기타
  
  예: 8GB 서버
  - Heap: 4GB
  - Direct Memory: 1GB (네트워크 버퍼)
  - Metaspace: 512MB
  - Thread Stack: 500MB (500 스레드 × 1MB)
  - 기타: 2GB
  → MaxDirectMemorySize=1g로 제한
```

### Off-Heap 메모리 누수 진단

```
증상:
  힙 사용량은 낮은데 프로세스 메모리(RSS)가 계속 증가

진단:

1. NMT (Native Memory Tracking) 활성화
   java -XX:NativeMemoryTracking=detail MyApp

2. 베이스라인 기록
   jcmd <pid> VM.native_memory baseline

3. 메모리 증가 후 diff 확인
   jcmd <pid> VM.native_memory summary.diff

4. Direct Buffer 누수 확인
   jmap -histo <pid> | grep DirectByteBuffer
   # DirectByteBuffer 객체가 많으면 누수 의심

해결:
  - DirectByteBuffer를 명시적으로 null 처리
  - System.gc() 호출 (강제 Cleaner 실행, 권장 안 함)
  - 버퍼 풀 사용 (Netty PooledByteBufAllocator)
```

---

## 🚫 흔한 오해

### "Direct Memory는 GC의 영향을 받지 않는다"

```
❌ 잘못된 이해:
  Direct Memory는 GC와 완전히 무관하다.

✅ 실제:
  DirectByteBuffer 객체 자체는 Heap에 존재
  → GC 대상
  
  GC 흐름:
  1. DirectByteBuffer 객체가 unreachable
  2. GC가 객체 수거
  3. Cleaner.clean() 호출
  4. free() 실행 → 네이티브 메모리 해제
  
  문제:
  - GC 발생 전까지 네이티브 메모리 유지
  - DirectByteBuffer 객체는 작아서 GC 트리거 늦음
  - 결과: 네이티브 메모리 누수처럼 보임
  
  해결:
  - 명시적으로 ((DirectBuffer)buf).cleaner().clean()
  - 또는 버퍼 풀 사용
```

### "Unsafe는 JDK 내부에서만 사용된다"

```
❌ 잘못된 이해:
  sun.misc.Unsafe는 JDK 코드만 쓴다.

✅ 실제:
  많은 라이브러리가 Unsafe 사용:
  - Netty (Zero-Copy, Direct Memory)
  - Apache Arrow (컬럼형 메모리)
  - Chronicle Map (Off-Heap 맵)
  - Hazelcast (Off-Heap 캐시)
  
  하지만:
  Java 9+에서 접근 제한 강화
  --add-opens 플래그 필요
  
  대안:
  Java 14+: Foreign Memory Access API (Incubator)
  Java 19+: Foreign Function & Memory API
```

### "Direct Buffer는 항상 빠르다"

```java
❌ 잘못된 이해:
  무조건 allocateDirect()를 써야 빠르다.

✅ 실제:
  케이스별 성능 비교:

소량 버퍼 (1KB):
  ByteBuffer.allocate(1024):      ~10ns
  ByteBuffer.allocateDirect(1024): ~1000ns
  → Heap이 100배 빠름 (할당만)

대용량 I/O (100MB):
  Heap Buffer:   350ms
  Direct Buffer: 120ms
  → Direct가 3배 빠름 (I/O 포함)

결론:
  버퍼 크기 > 수 KB && 장시간 유지 → Direct
  버퍼 크기 < 1KB || 단기 사용 → Heap
```

---

## 📌 핵심 정리

```
Off-Heap 메모리
  JVM Heap 밖의 네이티브 메모리
  GC가 직접 관리하지 않음
  주소 고정 (GC로 이동 안 됨)

Direct Memory
  ByteBuffer.allocateDirect()로 할당
  파일/네트워크 I/O 시 Zero-Copy
  대용량 I/O에서 2~3배 빠름

DirectByteBuffer 구조
  Heap: DirectByteBuffer 객체 (참조, Cleaner)
  Off-Heap: 실제 버퍼 (malloc)
  GC 시 Cleaner가 free() 호출

sun.misc.Unsafe
  직접 메모리 할당/조작
  JDK 내부 + 일부 라이브러리 사용
  위험: Segfault, 메모리 누수

MaxDirectMemorySize
  -XX:MaxDirectMemorySize=<size>
  기본: -Xmx와 동일 (또는 무제한)
  명시적 설정 권장

사용 시기
  Direct: 대용량 I/O, 장시간 유지
  Heap: 소량 데이터, 단기 사용

메모리 누수 진단
  jcmd VM.native_memory
  DirectByteBuffer 객체 수 (jmap -histo)
  NativeMemoryTracking (NMT)

풀링 권장
  Netty PooledByteBufAllocator
  할당/해제 비용 제거
```

---

## 🤔 생각해볼 문제

**Q1.** 1GB 파일을 읽어 네트워크로 전송하는 애플리케이션이 있다. Heap Buffer와 Direct Buffer 중 어느 것을 사용해야 하며, 버퍼 크기는 어떻게 설정해야 하는가? 메모리 복사 횟수와 연결해 설명하라.

**Q2.** DirectByteBuffer를 1000개 생성했는데 명시적으로 해제하지 않았다. GC가 발생하지 않으면 네이티브 메모리는 어떻게 되는가? 이를 방지하는 방법 3가지를 제시하라.

**Q3.** 컨테이너 메모리 8GB, JVM 힙 4GB로 설정한 서버에서 Direct Memory를 많이 사용한다. MaxDirectMemorySize를 어떻게 설정해야 하며, 그 이유를 총 메모리 계산식으로 설명하라.

> 💡 **해설**
>
> **Q1.** Direct Buffer 사용 권장. 이유: Heap Buffer는 File → Kernel Buffer → Heap Buffer → Kernel Socket Buffer → Network로 2회 복사. Direct Buffer는 File → Direct Buffer → Network로 1회만 복사 (Zero-Copy). 버퍼 크기: 8KB~64KB 권장. 너무 작으면 read() 호출 횟수 증가, 너무 크면 할당 비용 증가. 1GB 파일이라면 64KB 버퍼로 16,384회 read() 호출 (적절). 또한 Direct Buffer는 재사용하거나 풀링해야 할당 비용 최소화.
>
> **Q2.** DirectByteBuffer 객체는 Heap에 있지만 작아서 (수십 바이트) Young GC 트리거가 늦다. GC가 발생하지 않으면 Cleaner.clean()이 호출되지 않아 네이티브 메모리가 계속 증가한다. 방지 방법: ① 명시적 해제 — `((DirectBuffer)buf).cleaner().clean()` 호출 (sun.misc 접근 필요). ② 버퍼 풀 사용 — Netty의 PooledByteBufAllocator로 재사용. ③ MaxDirectMemorySize 설정 — 제한 초과 시 OOM으로 조기 발견. ④ System.gc() 주기 호출 (권장 안 함, STW 유발).
>
> **Q3.** 총 메모리 8GB = Heap 4GB + Metaspace + Direct Memory + Thread Stack + 기타. Metaspace ~512MB, Thread Stack (500 스레드 × 1MB) ~500MB, 기타 (Code Cache, Native) ~500MB. 남은 공간 = 8 - 4 - 0.5 - 0.5 - 0.5 = 2.5GB. Direct Memory는 1~1.5GB 정도로 설정 (`-XX:MaxDirectMemorySize=1g`). 2GB 이상 설정 시 컨테이너 OOM Killer 위험. Direct Memory를 많이 쓴다면 Heap을 3GB로 줄이고 Direct를 2GB로 늘리는 것도 가능하지만, Heap과 Direct의 실제 사용량을 모니터링 후 조정 권장.

---

## 📚 참고 자료

- [JEP 370 — Foreign-Memory Access API](https://openjdk.org/jeps/370)
- [Netty ByteBuf Documentation](https://netty.io/4.1/api/io/netty/buffer/ByteBuf.html)
- [Java NIO Direct Buffer Performance](https://mechanical-sympathy.blogspot.com/2012/07/native-cc-like-performance-for-java.html)

---

<div align="center">

**[⬅️ 이전: Object Layout In Memory](./06-object-layout-in-memory.md)** | **[홈으로 🏠](../README.md)**

</div>

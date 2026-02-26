# Safepoint Mechanism - 세이프포인트 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Safepoint는 무엇이며, 왜 필요한가?
- Time-To-Safepoint (TTSP)는 무엇이며, 지연 원인은?
- Safepoint가 필요한 JVM 작업은 무엇인가?
- 어떻게 Safepoint 문제를 분석하고 해결하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: JVM은 언제 안전하게 멈출 수 있는가?

```
GC 시작 시:
  모든 스레드를 멈춰야 함
  → Stop-The-World

언제 멈춰야 하는가?
  임의 시점? → 위험 (Stack 상태 불명확)
  특정 시점? → 안전 (Safepoint)
```

Safepoint는 **안전한 정지 지점**이다.

---

## 📐 Safepoint 개념

### 1. Safepoint란?

```
Safepoint:
  JVM이 스레드를 안전하게 멈출 수 있는 지점
  
특징:
  - Stack 상태가 명확
  - GC Root 추적 가능
  - Object 상태 일관성

위치:
  - 메서드 호출 후
  - 루프 백점프
  - Exception 발생 지점
  - JNI 호출 전후
```

---

### 2. Safepoint 폴링

```java
// JIT 컴파일 후 코드 (의사 코드)
for (int i = 0; i < N; i++) {
    // 루프 본문
    
    // Safepoint Poll (삽입됨)
    if (safepoint_required) {
        goto safepoint;
    }
}

safepoint:
  // 스레드 정지
  // GC 대기
  // 재개

Poll 구현 (x86):
  test %eax, [safepoint_page]
  → Safepoint 요청 시 페이지 보호
  → SIGSEGV 발생 → 처리
```

---

### 3. Time-To-Safepoint (TTSP)

```
TTSP:
  Safepoint 요청 → 모든 스레드 도달까지 시간

과정:
  1. JVM이 Safepoint 요청
  2. 각 스레드가 Safepoint 도달
  3. 모든 스레드 도달 완료
  4. GC 시작

지연 원인:
  - Counted Loop (긴 루프)
  - JNI 호출 중
  - Synchronized 블록 (긴 Critical Section)
```

---

### 4. Counted Loop 문제

```java
// ❌ Safepoint 없는 루프
for (int i = 0; i < Integer.MAX_VALUE; i++) {
    sum += i;  // 간단한 연산
}

// JIT 최적화:
// Counted Loop 인식
// → Safepoint Poll 제거 (성능)
// → 수십 초 동안 Safepoint 도달 안 함
// → TTSP 지연

// ✅ Safepoint 포함 루프
for (int i = 0; i < Integer.MAX_VALUE; i++) {
    sum += i;
    
    if (i % 1000 == 0) {
        // 복잡한 연산 (JIT가 Counted Loop로 안 봄)
        // → Safepoint Poll 유지
    }
}

// 또는 JVM 옵션
-XX:+UseCountedLoopSafepoints (Java 8+, 기본 활성화)
```

---

### 5. Safepoint 필요 작업

```
Stop-The-World 작업:
  - GC (모든 타입)
  - Deoptimization
  - Bias Lock Revocation
  - Thread Dump (jstack)
  - Heap Dump (jmap)
  - Class Redefinition (디버거)
  - JIT Compilation (일부)

No Safepoint 작업:
  - 일반 애플리케이션 코드
  - JNI Native 코드 (실행 중)
  - Concurrent GC (일부 단계)
```

---

## 💻 실험으로 확인하기

### 실험 1: TTSP 측정

```bash
# Safepoint 로그 활성화
java -Xlog:safepoint:file=safepoint.log \
     -XX:+UnlockDiagnosticVMOptions \
     -XX:+LogVMOutput \
     MyApp

# safepoint.log:
# [safepoint] Application time: 1.234s
# [safepoint] Entering safepoint: 0.005s  ← TTSP
# [safepoint] GC: 0.050s
# [safepoint] Leaving safepoint: 0.001s
# Total pause: 0.056s
```

---

### 실험 2: Counted Loop 문제

```java
public class CountedLoopTest {
    static long sum = 0;
    
    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            // Counted Loop (Safepoint Poll 없음)
            for (int i = 0; i < Integer.MAX_VALUE; i++) {
                sum += i;
            }
        });
        
        worker.start();
        Thread.sleep(1000);
        
        // Thread Dump (Safepoint 필요)
        long start = System.currentTimeMillis();
        Thread.getAllStackTraces();  // jstack 유사
        long ttsp = System.currentTimeMillis() - start;
        
        System.out.println("TTSP: " + ttsp + "ms");
        // 출력: TTSP: 5000ms ← 매우 길음!
    }
}
```

---

### 실험 3: Safepoint 분석

```bash
# 상세 Safepoint 로그
java -Xlog:safepoint=trace:file=safepoint_trace.log \
     MyApp

# safepoint_trace.log:
# [safepoint] Safepoint sync time: 5234 ms
# [safepoint] Spin time: 5234 ms  ← 특정 스레드 대기
# [safepoint] Thread: "worker-1" [0x...] ← 문제 스레드
```

---

## ⚡ 실무 임팩트

### GC Pause 분석

```bash
# GC 로그
[gc] GC(10) Pause Young 50ms

# 실제 Pause 구성:
Total Pause: 50ms
  TTSP: 30ms  ← Safepoint 대기
  GC: 20ms    ← 실제 GC

문제:
  TTSP가 GC보다 김
  → Safepoint 최적화 필요
```

---

### 긴 루프 최적화

```java
// ❌ 문제 코드
for (int i = 0; i < data.length; i++) {
    process(data[i]);  // 간단한 연산
}

// ✅ 해결 1: Batch 처리
for (int i = 0; i < data.length; i += 1000) {
    int end = Math.min(i + 1000, data.length);
    for (int j = i; j < end; j++) {
        process(data[j]);
    }
    // 1000개마다 Safepoint Poll
}

// ✅ 해결 2: 메서드 호출 (Safepoint)
for (int i = 0; i < data.length; i++) {
    processBatch(data, i);  // 메서드 호출 → Safepoint
}
```

---

### JNI 호출 주의

```java
// Native 메서드
public native void longComputation();

// 호출
longComputation();  // 수 초 실행

문제:
  Native 코드 실행 중
  → Safepoint 도달 안 함
  → TTSP 지연

해결:
  Native 코드에서 주기적으로 Java 호출
  → Safepoint 기회 제공
```

---

## 🚫 흔한 오해

### "Safepoint는 GC만 사용한다"

```
❌ 잘못된 이해:
  Safepoint는 GC 전용

✅ 실제:
  다양한 작업에 사용
  
  Safepoint 필요:
  - GC (모든 타입)
  - Thread Dump (jstack)
  - Heap Dump (jmap)
  - Deoptimization
  - Bias Lock Revocation
  
  의외로 빈번함
```

---

### "Safepoint Poll은 오버헤드가 크다"

```
❌ 잘못된 이해:
  Safepoint Poll은 성능 저하

✅ 실제:
  매우 저렴
  
  비용:
  test %eax, [safepoint_page]
  → 1~2 CPU 사이클
  → ~1ns
  
  최적화:
  Branch Prediction
  → 대부분 Not Taken
  → 거의 무료
```

---

## 📌 핵심 정리

```
Safepoint
  JVM이 스레드를 안전하게 멈추는 지점
  Stack 상태 명확, GC Root 추적 가능

위치
  메서드 호출 후
  루프 백점프
  Exception 발생
  JNI 호출 전후

Safepoint Poll
  if (safepoint_required) goto safepoint
  JIT 컴파일 시 삽입
  비용: ~1ns

TTSP
  Time-To-Safepoint
  요청 → 모든 스레드 도달 시간
  지연 원인: Counted Loop, JNI, 긴 CS

Counted Loop 문제
  JIT 최적화로 Safepoint Poll 제거
  → 수 초 도달 안 함
  해결: -XX:+UseCountedLoopSafepoints

필요 작업
  GC, Thread Dump, Heap Dump
  Deoptimization, Bias Lock Revocation

분석
  -Xlog:safepoint
  TTSP 측정
  문제 스레드 식별

최적화
  긴 루프 분할 (Batch)
  메서드 호출 삽입
  JNI 주기적 Java 호출
```

---

## 🤔 생각해볼 문제

**Q1.** GC Pause가 100ms인데, 실제 GC 시간은 20ms다. 나머지 80ms는 무엇이며, 어떻게 줄이는가?

**Q2.** 다음 코드에서 Safepoint 도달이 지연되는 이유를 설명하고, 해결 방법을 제시하라.

```java
for (int i = 0; i < 1_000_000_000; i++) {
    sum += i * 2;
}
```

**Q3.** JNI Native 메서드가 실행 중일 때 GC가 시작하면 어떻게 되는가? Safepoint 관점에서 설명하라.

> 💡 **해설**
>
> **Q1.** 나머지 80ms는 TTSP (Time-To-Safepoint). 과정: GC 요청 (0ms) → 모든 스레드 Safepoint 도달 대기 (80ms) → GC 실행 (20ms). 지연 원인: 특정 스레드가 Counted Loop, JNI 호출 중. 줄이는 방법: ① 긴 루프 분할 (1000개마다 메서드 호출). ② UseCountedLoopSafepoints 확인 (기본 활성화). ③ JNI 호출 최소화. ④ -Xlog:safepoint로 문제 스레드 식별 → 코드 수정.
>
> **Q2.** 지연 이유: Counted Loop — JIT가 `i < 1B, i++, i*2` 패턴 인식 → Safepoint Poll 제거 (최적화). 10억 번 반복 → 수 초 실행 → Safepoint 도달 안 함 → TTSP 수 초. 해결: ① Batch 처리: `for (int i = 0; i < 1B; i += 1000) { for (int j = i; j < i+1000; j++) { sum += j*2; } }`. ② 메서드 호출 삽입: `for (...) { compute(i); }` (compute에서 sum += i*2). ③ -XX:+UseCountedLoopSafepoints 확인 (Java 8+는 기본). ④ 복잡한 연산 추가 (JIT가 Counted Loop로 안 봄).
>
> **Q3.** JNI 실행 중 GC: ① JNI 호출 전 스레드는 Safepoint 도달 (JNI transition). ② Native 코드 실행 중 → JVM 관점에서 "Safe" (Stack 변경 없음). ③ GC 시작 가능 (Native 스레드는 대기 안 함). ④ Native 코드 종료 후 Java 복귀 시 → Safepoint 체크. 문제: 긴 Native 호출은 TTSP 지연 원인 아님 (이미 Safe). 하지만 복귀 후 Safepoint 도달까지 지연 가능. 해결: Native에서 주기적으로 Java 호출 → Safepoint 기회.

---

## 📚 참고 자료

- [Safepoints in HotSpot](https://shipilev.net/jvm/anatomy-quarks/22-safepoint-polls/)
- [Understanding GC Pauses](https://www.infoq.com/articles/Java_Garbage_Collection_Distilled/)
- [JVM Safepoint](https://medium.com/@unmeshvjoshi/understanding-jvm-safepoint-3c3d7b0e1ac8)

---

<div align="center">

**[⬅️ 이전: Virtual Threads (Loom)](./08-virtual-threads-loom.md)** | **[홈으로 🏠](../README.md)**

</div>

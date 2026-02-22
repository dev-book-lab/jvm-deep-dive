# TLAB (Thread-Local Allocation Buffer) - 스레드 로컬 할당 버퍼

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 멀티스레드 환경에서 객체를 Eden에 할당할 때 왜 동기화가 필요한가?
- TLAB는 어떻게 동기화 없이 빠른 할당을 가능하게 하는가?
- TLAB 크기는 어떻게 결정되며, 너무 크거나 작으면 어떤 문제가 생기는가?
- 객체가 TLAB보다 크면 어떻게 할당되는가?
- `-XX:+UseTLAB`를 끄면 성능이 얼마나 차이나는가?

---

## 🔍 왜 이게 존재하는가

### 문제: Eden은 모든 스레드가 공유하는 공간이다

```
멀티스레드 객체 할당 시나리오:

Thread 1: new MyObject() → Eden의 빈 공간 탐색
Thread 2: new MyObject() → Eden의 빈 공간 탐색
Thread 3: new MyObject() → Eden의 빈 공간 탐색

문제:
  세 스레드가 동시에 같은 위치를 "빈 공간"으로 인식
  → 서로 덮어쓰기 → 메모리 손상
  
해결책 1: Eden 전체에 Lock
  synchronized (eden) {
      allocate(size);
  }
  
  문제:
  - 모든 객체 생성이 직렬화됨
  - Lock 경합(contention) 심각
  - 처리량(throughput) 폭락
```

```
실측 데이터:
  Lock 없이: 1억 객체 생성 → 0.5초
  Lock 사용: 1억 객체 생성 → 5초 (10배 느림)
  
  고성능 애플리케이션에서는 치명적
```

JVM은 이를 **스레드별 전용 할당 공간**으로 해결한다.

---

## 📐 내부 구조

### 1. TLAB 기본 구조

```
Eden 구조 (TLAB 활성화 시):

┌─────────────────────────────────────────────────────┐
│                    Eden Space                       │
├──────────────┬──────────────┬──────────────┬────────┤
│ Thread 1     │ Thread 2     │ Thread 3     │ Shared │
│   TLAB       │   TLAB       │   TLAB       │  Eden  │
│ ┌──────────┐ │ ┌──────────┐ │ ┌──────────┐ │        │
│ │ top      │ │ │ top      │ │ │ top      │ │        │
│ │    ↓     │ │ │    ↓     │ │ │    ↓     │ │        │
│ │ [A][B]   │ │ │ [X][Y]   │ │ │ [M][N]   │ │ (Lock) │
│ │          │ │ │          │ │ │          │ │        │
│ │  end     │ │ │  end     │ │ │  end     │ │        │
│ └──────────┘ │ └──────────┘ │ └──────────┘ │        │
└──────────────┴──────────────┴──────────────┴────────┘

각 스레드는 자신의 TLAB 내에서만 할당
→ Lock 불필요 → Bump-the-pointer로 고속 할당
```

---

### 2. TLAB 할당 메커니즘

```
Bump-the-pointer 기법:

Thread 1의 TLAB:
┌──────────────────────────┐
│ start                    │
│   ↓                      │
│ [Obj1][Obj2]             │ ← top (다음 할당 위치)
│                          │
│                     end→ │
└──────────────────────────┘

new MyObject(size=16 bytes) 실행:

1. size <= (end - top) 확인 (공간 충분?)
2. result = top
3. top += size  (포인터만 이동, Lock 없음!)
4. return result

→ 단 3개 명령어로 할당 완료
→ 나노초 단위 속도
```

---

### 3. TLAB 크기 결정

```
TLAB 크기 계산:

기본 크기:
  Eden 크기 / (스레드 수 * TLABWasteTargetPercent)
  
예:
  Eden = 100MB
  스레드 = 8개
  TLABWasteTargetPercent = 1%
  → TLAB 크기 = 100MB / (8 * 100) ≈ 128KB

동적 조정:
  JVM이 런타임에 각 스레드의 할당 패턴 관찰
  → TLAB 크기 자동 조정
  
플래그:
  -XX:TLABSize=<size>         (고정 크기 설정, 권장 안 함)
  -XX:+ResizeTLAB             (동적 크기 조정, 기본 활성화)
  -XX:TLABWasteTargetPercent  (낭비 허용 비율, 기본 1%)
```

---

### 4. TLAB 재할당 (Refill)

```
TLAB가 가득 찼을 때:

현재 TLAB:
┌──────────────────────────┐
│ [Obj1][Obj2][Obj3][Obj4] │ ← top == end (가득 참)
└──────────────────────────┘

새 객체 할당 요청 (32 bytes):

1. 현재 TLAB의 남은 공간 < 32 bytes
2. 기존 TLAB를 Eden으로 반환 (남은 공간은 waste)
3. Eden의 공유 영역에서 새 TLAB 할당 (Lock 필요!)
4. 새 TLAB에서 객체 할당

Slow Path vs Fast Path:
  Fast Path: TLAB 내 할당 (Lock 없음, 99% 케이스)
  Slow Path: TLAB 재할당 (Lock 있음, 1% 케이스)
  
  → 전체적으로는 Lock 경합이 거의 없음
```

---

### 5. 큰 객체 처리

```
객체 크기 > TLAB 크기인 경우:

TLAB를 우회해 직접 Eden의 공유 영역에 할당

new byte[1MB]  // TLAB 크기가 128KB라면?

1. TLAB 크기 확인 → 불가능 판단
2. Eden 공유 영역에서 직접 할당 (Lock 필요)
3. 또는 즉시 Old Generation에 할당

이것이 "큰 객체를 자주 생성하면 느리다"는 이유
→ Lock 경합 + Old Generation 직행 가능
```

---

## 💻 실험으로 확인하기

### 실험 1: TLAB 활성화 vs 비활성화 성능 비교

```java
public class TLABBenchmark {
    static final int THREADS = 8;
    static final int OBJECTS_PER_THREAD = 10_000_000;
    
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== 멀티스레드 객체 생성 벤치마크 ===");
        
        long start = System.nanoTime();
        
        Thread[] threads = new Thread[THREADS];
        for (int i = 0; i < THREADS; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < OBJECTS_PER_THREAD; j++) {
                    new byte[128]; // 128 바이트 객체
                }
            });
            threads[i].start();
        }
        
        for (Thread t : threads) {
            t.join();
        }
        
        long elapsed = System.nanoTime() - start;
        long totalObjects = THREADS * OBJECTS_PER_THREAD;
        
        System.out.printf("총 객체: %d개%n", totalObjects);
        System.out.printf("소요 시간: %.2f초%n", elapsed / 1e9);
        System.out.printf("초당 할당: %.0f개/초%n", totalObjects / (elapsed / 1e9));
    }
}
```

```bash
# TLAB 활성화 (기본)
java -XX:+UseTLAB TLABBenchmark
# 출력 예: 소요 시간: 0.85초, 초당 할당: 94,000,000개/초

# TLAB 비활성화
java -XX:-UseTLAB TLABBenchmark
# 출력 예: 소요 시간: 8.50초, 초당 할당: 9,400,000개/초
# → 10배 느림!
```

---

### 실험 2: TLAB 통계 확인

```bash
# TLAB 관련 JVM 플래그
java -XX:+PrintTLAB \
     -XX:+UseTLAB \
     -Xlog:gc+tlab=trace \
     MyApp

# 출력 예시:
# TLAB: gc thread: 0x00007f8a4c001000 [id=12345] desired_size: 131KB slow allocs: 5 refill waste: 1.2KB ...
# TLAB totals: thrds: 8  refills: 1234 max: 200 slow: 45 ...

# 주요 지표:
# desired_size: TLAB 목표 크기
# refills:      TLAB 재할당 횟수 (낮을수록 좋음)
# slow allocs:  Slow Path 할당 횟수 (낮을수록 좋음)
# waste:        낭비된 공간 (refill 시 버려진 공간)
```

---

### 실험 3: JMH로 정밀 측정

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Thread)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
@Fork(value = 1, jvmArgs = {"-Xms2g", "-Xmx2g"})
public class TLABAllocationBench {
    
    @Benchmark
    public Object allocateSmall() {
        return new byte[128]; // TLAB에서 할당
    }
    
    @Benchmark
    public Object allocateLarge() {
        return new byte[1024 * 1024]; // TLAB 우회, 직접 Eden
    }
}
```

```bash
# TLAB 활성화
mvn clean install
java -jar target/benchmarks.jar -jvmArgs "-XX:+UseTLAB"

# 결과 예시:
# allocateSmall  thrpt   5  250000000  ops/s  (TLAB, 매우 빠름)
# allocateLarge  thrpt   5    5000000  ops/s  (Slow Path, 50배 느림)
```

---

## ⚡ 실무 임팩트

### 고처리량 시스템에서 TLAB의 중요성

```
웹 서버 시나리오:
  요청당 수백 개의 임시 객체 생성
  1000 RPS (Requests Per Second) 서버
  → 초당 100만 개 객체 할당
  
  TLAB 없이 Lock으로 동기화:
  - Lock 경합으로 처리량 70% 감소
  - CPU 코어 증가해도 성능 개선 제한 (Lock Bottleneck)
  
  TLAB 사용:
  - Lock 없이 병렬 할당
  - CPU 코어에 비례한 성능 확장 가능
```

### TLAB 튜닝이 필요한 경우

```
대부분의 경우:
  기본 설정(-XX:+UseTLAB)으로 충분
  JVM이 자동으로 최적화

튜닝이 필요한 경우:
  1. 스레드가 매우 많은 경우 (100+)
     → Eden 대비 스레드 수 과다
     → TLAB 크기가 너무 작아짐
     → Refill 빈번 발생
     
     해결: Eden 크기 증가 (-XX:NewSize)
  
  2. 큰 객체를 자주 생성
     → TLAB 우회가 빈번
     → Lock 경합 증가
     
     해결: 객체 풀링 또는 재사용 패턴 적용
  
  3. TLAB waste가 높은 경우
     → -XX:TLABWasteTargetPercent 조정 (기본 1%)
     → 너무 낮으면 Refill 빈번, 너무 높으면 메모리 낭비
```

### JFR로 TLAB 병목 탐지

```bash
# JFR 프로파일링
java -XX:StartFlightRecording=filename=tlab.jfr,settings=profile MyApp

# JFR 파일 분석 (JDK Mission Control)
# "TLAB Allocation" 이벤트 확인:
# - Outside TLAB: TLAB 우회 할당 빈도
# - Allocation Rate: 스레드별 할당 속도
# - TLAB Size: 각 스레드의 TLAB 크기 분포

# Slow Path가 5% 이상이면 문제 신호
# → 큰 객체 생성을 줄이거나 객체 풀링 고려
```

---

## 🚫 흔한 오해

### "TLAB는 GC를 방지한다"

```
❌ 잘못된 이해:
  TLAB에 할당된 객체는 GC되지 않는다.

✅ 실제:
  TLAB는 할당 속도를 높이는 메커니즘일 뿐
  GC와는 무관
  
  TLAB에 할당된 객체도:
  - Young GC 대상
  - 참조가 끊어지면 수거됨
  - Promotion도 정상 발생
  
  TLAB의 역할:
  할당 시 Lock 경합 제거 → 빠른 할당
  GC 동작에는 영향 없음
```

### "TLAB 크기를 크게 하면 성능이 좋아진다"

```
❌ 잘못된 이해:
  -XX:TLABSize를 크게 → 할당 속도 향상

✅ 실제:
  TLAB가 너무 크면:
  1. Eden을 너무 많이 차지
     → 다른 스레드의 TLAB 공간 부족
     → 스레드 간 불균형
  
  2. Refill 빈도 감소 (좋음)
     하지만 waste 증가 (나쁨)
     → 전체적으로 메모리 효율 저하
  
  3. 스레드가 죽으면 TLAB 전체가 waste
     → Thread Pool에서 문제
  
  권장:
  기본 설정 유지 (-XX:+ResizeTLAB)
  JVM이 런타임에 자동 조정
  수동 설정은 특수한 경우만
```

### "단일 스레드 애플리케이션에는 TLAB가 불필요하다"

```
❌ 잘못된 이해:
  스레드가 하나면 TLAB를 끌 수 있다.

✅ 실제:
  단일 스레드여도 TLAB는 여전히 유용
  
  이유:
  1. Bump-the-pointer는 Lock과 무관하게 빠름
     (단순 포인터 연산)
  
  2. JVM 내부 스레드들 (GC Thread 등)도 객체 할당
     → 단일 애플리케이션 스레드 ≠ 단일 JVM 스레드
  
  3. 코드 경로 단순화
     TLAB 활성화가 기본이므로 코드 최적화가 이에 맞춰짐
  
  결론:
  -XX:+UseTLAB는 항상 켜두는 것이 권장
```

---

## 📌 핵심 정리

```
TLAB의 존재 이유
  멀티스레드 환경에서 Eden 할당 시 Lock 경합 제거
  스레드별 전용 할당 공간 제공

할당 메커니즘
  Fast Path: TLAB 내 Bump-the-pointer (Lock 없음, 99%)
  Slow Path: TLAB 재할당 또는 큰 객체 할당 (Lock 있음, 1%)

TLAB 크기
  기본: Eden / (스레드 수 * 100)
  동적 조정: JVM이 런타임에 최적화 (권장)
  수동 설정: 특수한 경우만 (-XX:TLABSize)

성능 영향
  TLAB 활성화: 멀티스레드 할당 속도 10배 향상
  TLAB 비활성화: Lock 경합으로 처리량 폭락

큰 객체 문제
  객체 크기 > TLAB → Slow Path (Lock)
  해결: 객체 재사용, 풀링, 크기 최적화

실무 체크리스트
  기본 설정 유지 (-XX:+UseTLAB, -XX:+ResizeTLAB)
  JFR로 "Outside TLAB" 비율 확인 (5% 이하 권장)
  큰 객체 빈번 생성 시 설계 재검토
```

---

## 🤔 생각해볼 문제

**Q1.** 스레드 100개, Eden 1GB 환경에서 각 스레드가 10MB씩 TLAB를 할당받으면 무슨 일이 생기는가? TLAB 크기 결정 공식과 연결해 설명하라.

**Q2.** 객체 풀(Object Pool)을 사용하면 TLAB의 이점이 사라지는가? 풀에서 객체를 가져오는 것과 `new`로 생성하는 것의 TLAB 관점 차이를 설명하라.

**Q3.** `-XX:TLABWasteTargetPercent=10`으로 설정하면 어떤 일이 생기는가? Refill 빈도와 메모리 낭비 사이의 트레이드오프를 설명하라.

> 💡 **해설**
>
> **Q1.** 100개 스레드 * 10MB = 1GB로 Eden 전체를 TLAB가 차지한다. 이 경우 공유 Eden 공간이 없어 큰 객체 할당이 즉시 Old로 가거나 Full GC를 유발한다. 또한 스레드 간 TLAB 크기 불균형이 생겨 일부 스레드는 빠르게 Refill하고 다른 스레드는 남는 공간이 많아 비효율적이다. 기본 공식 `Eden / (스레드 수 * 100)`을 쓰면 각 스레드는 약 100KB만 받아 전체 Eden의 10%만 TLAB로 사용하고 나머지는 공유 공간으로 남는다.
>
> **Q2.** 객체 풀 사용 시 `new` 호출이 없어 TLAB 할당 자체가 발생하지 않는다. 대신 풀에서 `get()`/`return()` 호출만 있다. TLAB의 이점(Lock 없는 빠른 할당)이 사라지지만, 풀 자체가 캐싱으로 할당 비용을 완전히 제거하므로 더 빠를 수 있다. 단, 풀 관리 비용(풀 크기 조정, 메모리 낭비)이 있고, 대부분의 경우 TLAB의 빠른 할당과 Young GC의 효율적 수거가 더 나은 선택이다. 큰 객체(1MB+) 또는 생성 비용이 큰 객체(DB 커넥션)에만 풀링이 유리하다.
>
> **Q3.** `TLABWasteTargetPercent=10`은 TLAB 공간의 10%까지 낭비를 허용한다는 의미다. 기본 1%에서 10%로 올리면: Refill 빈도 감소 (좋음) — TLAB를 더 오래 사용하므로 Slow Path 진입이 줄어든다. 메모리 낭비 증가 (나쁨) — Refill 시 버려지는 공간이 커진다. 예를 들어 TLAB 100KB에서 90KB 사용 후 Refill하면 10KB가 waste다. 트레이드오프: 할당 속도 vs 메모리 효율. 메모리가 넉넉하고 할당 속도가 중요하면 높이고, 메모리가 빡빡하면 낮춘다. 기본 1%가 대부분 최적.

---

## 📚 참고 자료

- [HotSpot TLAB Implementation](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/gc/shared/threadLocalAllocBuffer.cpp)
- [JEP 307 — Parallel Full GC for G1](https://openjdk.org/jeps/307) *(TLAB 관련 설명 포함)*
- [JVM Performance Tuning — TLAB](https://docs.oracle.com/en/java/javase/17/gctuning/garbage-first-g1-garbage-collector1.html)

---

<div align="center">

**[⬅️ 이전: Heap Structure](./01-heap-structure.md)** | **[다음: Stack And Frames ➡️](./03-stack-and-frames.md)**

</div>

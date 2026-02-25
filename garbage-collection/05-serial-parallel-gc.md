# Serial & Parallel GC - 시리얼과 병렬 GC

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Serial GC와 Parallel GC의 차이는 무엇인가?
- Stop-The-World (STW)는 왜 발생하며, 얼마나 걸리는가?
- Parallel GC는 어떻게 멀티스레드를 활용하는가?
- Young/Old Generation 각각의 GC 알고리즘은 무엇인가?
- 언제 Serial을 쓰고, 언제 Parallel을 쓰는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 기본적인 GC가 필요하다

```
요구사항:
  1. 간단한 구현
  2. 예측 가능한 동작
  3. 작은 메모리 환경에서도 동작

Serial GC:
  - 단일 스레드
  - 가장 단순
  - 메모리 오버헤드 최소
  
Parallel GC:
  - 멀티 스레드
  - Throughput 최대화
  - 서버 환경 최적화
```

Serial/Parallel GC는 **가장 기본적이고 단순한** GC다.

---

## 📐 내부 구조

### 1. Serial GC 개요

```
활성화:
  -XX:+UseSerialGC

특징:
  - 단일 GC 스레드
  - Stop-The-World (STW)
  - 가장 오래된 GC

Young Generation:
  Copy 알고리즘 (Copying Collector)
  
Old Generation:
  Mark-Sweep-Compact

사용 케이스:
  - 클라이언트 애플리케이션
  - 작은 힙 (< 100MB)
  - 단일 CPU 환경
  - 임베디드 디바이스
```

---

### 2. Serial GC - Young Generation (Copy 알고리즘)

```
구조:
  Eden + Survivor 0 + Survivor 1

Minor GC 과정:

1. Eden + From Survivor 스캔
   ┌────────┬──────────┬──────────┐
   │  Eden  │ From (S0)│ To (S1)  │
   │ [A][B] │   [C]    │          │
   └────────┴──────────┴──────────┘

2. 살아있는 객체 복사
   GC Root → A, C
   
   ┌────────┬──────────┬──────────┐
   │  Eden  │ From (S0)│ To (S1)  │
   │        │          │ [A][C]   │
   └────────┴──────────┴──────────┘

3. Eden + From Survivor 비움
   ┌────────┬──────────┬──────────┐
   │  Eden  │ From (S0)│ To (S1)  │
   │ [Empty]│ [Empty]  │ [A][C]   │
   └────────┴──────────┴──────────┘

4. From/To 역할 교환
   다음 GC에서 S1 → From, S0 → To

장점:
  - Compaction 자동 (복사 과정에서)
  - Fragmentation 없음
  
단점:
  - 공간 낭비 (Survivor 2개 필요)

Age 증가:
  Minor GC 생존 시 age++
  age == 15 → Old로 Promotion
```

---

### 3. Serial GC - Old Generation (Mark-Sweep-Compact)

```
Major/Full GC 과정:

1. Mark Phase
   GC Root에서 Reachable 객체 표시
   
2. Sweep Phase
   Unmarked 객체 제거
   
3. Compact Phase
   살아있는 객체를 힙 앞쪽으로 이동
   
   Before:
   [A][Dead][B][Dead][C][Dead]
   
   After:
   [A][B][C][Free Free Free]

시간:
  10GB 힙: 수 초 ~ 수십 초 (STW)
  
특징:
  - 단일 스레드 실행
  - 애플리케이션 완전 정지
  - Fragmentation 없음
```

---

### 4. Parallel GC 개요

```
활성화:
  -XX:+UseParallelGC (기본, Java 8+)

특징:
  - 멀티 GC 스레드
  - Throughput 최대화
  - Young/Old 모두 병렬

별칭:
  Throughput Collector
  
Young Generation:
  Parallel Copy (Multi-threaded)
  
Old Generation:
  Parallel Mark-Sweep-Compact

GC 스레드 수:
  -XX:ParallelGCThreads=N
  기본: CPU 코어 수
  
  예:
  8 코어 → 8 GC 스레드
```

---

### 5. Parallel GC - Young Generation

```
Parallel Scavenge:

Minor GC (8 스레드 예시):

1. Eden 분할
   Thread 1: Eden[0~12.5%]
   Thread 2: Eden[12.5~25%]
   ...
   Thread 8: Eden[87.5~100%]

2. 병렬 복사
   각 스레드가 독립적으로:
   - 살아있는 객체 스캔
   - Survivor로 복사
   - Age 증가

3. 동기화 지점
   모든 스레드 완료 대기

시간:
  1 스레드: 100ms
  8 스레드: ~15ms (이론상 12.5ms)
  
  Speedup = 100 / 15 ≈ 6.7배
  (Linear scaling 아님, 동기화 오버헤드)

효과:
  Young GC Pause Time 단축
  → Throughput 향상
```

---

### 6. Parallel GC - Old Generation

```
Parallel Mark-Compact:

1. Parallel Mark
   Thread 1: GC Root[0~25%]
   Thread 2: GC Root[25~50%]
   ...
   
2. Parallel Sweep
   Thread 1: Old[0~25%]
   Thread 2: Old[25~50%]
   ...

3. Parallel Compact
   Thread 1: Region[0~25%] 이동
   Thread 2: Region[25~50%] 이동
   ...

시간:
  10GB Old, 1 스레드: 10초
  10GB Old, 8 스레드: ~2초
  
  5배 향상 (8배 아님, Compact 병렬화 어려움)

주의:
  여전히 STW
  → 애플리케이션 정지
  → 사용자 경험 저하 (큰 힙)
```

---

### 7. Stop-The-World (STW)

```
STW 발생 이유:
  객체 그래프 일관성 유지
  
  예:
  GC 중 객체 참조 변경
  A → B (원래)
  A → C (변경)
  
  → GC가 B를 못 봄
  → B가 Unreachable로 잘못 판단
  → 버그 (Dangling Pointer)

해결:
  애플리케이션 스레드 모두 정지
  → Safe Point에서 대기
  → GC 완료까지

Safe Point:
  - 메서드 호출
  - 루프 백점프
  - Exception 발생
  
  JVM이 스레드 상태 안전하게 변경 가능한 지점

STW 단계:
  1. Safe Point 도달 대기
  2. GC 실행
  3. 애플리케이션 재개

시간:
  Safe Point 대기: 수 ms
  GC 실행: 수십 ms ~ 수 초
  
  총 Pause Time = 대기 + GC
```

---

## 💻 실험으로 확인하기

### 실험 1: Serial vs Parallel 성능 비교

```java
public class GCBenchmark {
    public static void main(String[] args) throws Exception {
        List<byte[]> list = new ArrayList<>();
        
        long start = System.currentTimeMillis();
        
        for (int i = 0; i < 10000; i++) {
            byte[] data = new byte[1024 * 1024];  // 1MB
            list.add(data);
            
            if (i % 100 == 0) {
                list.clear();  // GC 유발
            }
        }
        
        long elapsed = System.currentTimeMillis() - start;
        System.out.println("Time: " + elapsed + " ms");
    }
}
```

```bash
# Serial GC (1 스레드)
java -Xmx2g -XX:+UseSerialGC GCBenchmark
# Time: 15000 ms

# Parallel GC (8 스레드)
java -Xmx2g -XX:+UseParallelGC GCBenchmark
# Time: 5000 ms (3배 빠름)
```

---

### 실험 2: GC 로그 분석

```bash
java -Xlog:gc*:file=gc.log \
     -XX:+UseParallelGC \
     MyApp

# gc.log (Parallel GC):
# [gc] GC(0) Pause Young (Allocation Failure)
# [gc] GC(0) Using 8 workers  ← 병렬 스레드
# [gc] GC(0) Eden: 512M -> 0M
# [gc] GC(0) 50ms  ← Pause Time

# Serial GC:
# [gc] GC(0) Pause Young
# [gc] GC(0) Using 1 worker  ← 단일 스레드
# [gc] GC(0) 400ms
```

---

### 실험 3: STW 시간 측정

```bash
java -Xlog:safepoint:file=safepoint.log \
     MyApp

# safepoint.log:
# [safepoint] Entering safepoint: 3ms
# [safepoint] GC executing: 50ms
# [safepoint] Leaving safepoint: 1ms
# Total Pause: 54ms
```

---

## ⚡ 실무 임팩트

### Serial GC 사용 케이스

```
적합:
  - 작은 힙 (< 100MB)
  - 단일 CPU
  - 클라이언트 애플리케이션
  - 임베디드 디바이스
  - 컨테이너 (CPU 제한)

부적합:
  - 서버 애플리케이션
  - 큰 힙 (> 1GB)
  - 멀티 코어 환경

예:
  Docker 컨테이너 (CPU 0.5 core)
  → Parallel GC 스레드 낭비
  → Serial GC 사용
  
  -XX:+UseSerialGC
```

---

### Parallel GC 사용 케이스

```
적합:
  - Throughput 중시
  - 배치 작업
  - 멀티 코어 서버
  - 큰 힙 (수 GB)

부적합:
  - 낮은 Latency 요구
  - 실시간 애플리케이션
  - 웹 서버 (99th percentile < 100ms)

예:
  데이터 분석 배치
  → 처리량 최대화
  → Pause Time 무관
  
  -XX:+UseParallelGC
  -XX:ParallelGCThreads=16
```

---

### GC 스레드 수 튜닝

```
기본값:
  ParallelGCThreads = CPU 코어 수
  
  8 코어 → 8 GC 스레드

문제:
  GC 중 CPU 100% 사용
  → 다른 작업 블록킹

튜닝:
  # GC 스레드 절반으로
  -XX:ParallelGCThreads=4
  
  장점:
  - CPU 여유 확보
  - 다른 프로세스와 공존
  
  단점:
  - GC 시간 증가 (약 2배)

권장:
  전용 서버: 기본값 (CPU 코어 수)
  공유 서버: 코어 수 / 2
  컨테이너: CPU limit 고려
```

---

## 🚫 흔한 오해

### "Parallel GC는 STW가 없다"

```
❌ 잘못된 이해:
  Parallel은 병렬이니까 애플리케이션이 계속 실행된다.

✅ 실제:
  여전히 STW 발생
  
  Parallel의 의미:
  "GC 스레드가 병렬" (애플리케이션 아님)
  
  과정:
  1. 애플리케이션 정지 (STW)
  2. GC 스레드 8개 병렬 실행
  3. 애플리케이션 재개
  
  장점:
  GC 시간 단축 (8배 아님, 2~5배)
  → STW 시간 단축
  
  STW는 여전히 존재
```

---

### "Serial GC는 항상 느리다"

```
❌ 잘못된 이해:
  Serial은 단일 스레드라 무조건 느리다.

✅ 실제:
  작은 힙에서는 더 빠를 수 있음
  
  100MB 힙:
  Serial: 10ms (단일 스레드, 오버헤드 없음)
  Parallel: 15ms (스레드 생성/동기화 오버헤드)
  
  10GB 힙:
  Serial: 10초
  Parallel: 2초
  
  Break-even:
  힙 크기 ~500MB
  
  권장:
  힙 < 500MB: Serial
  힙 > 500MB: Parallel
```

---

### "ParallelGCThreads를 많이 할수록 빠르다"

```
❌ 잘못된 이해:
  GC 스레드를 CPU 코어보다 많이 하면 더 빠르다.

✅ 실제:
  코어 수 이상은 오히려 느림
  
  8 코어 서버:
  ParallelGCThreads=8:  50ms
  ParallelGCThreads=16: 80ms (느림)
  
  이유:
  - Context Switching 증가
  - CPU 경쟁
  - 캐시 효율 저하
  
  최적값:
  CPU 코어 수 이하
  또는
  CPU 코어 수 × 0.75
```

---

## 📌 핵심 정리

```
Serial GC
  단일 GC 스레드
  Stop-The-World
  Young: Copy 알고리즘
  Old: Mark-Sweep-Compact
  사용: 작은 힙, 단일 CPU

Parallel GC
  멀티 GC 스레드
  Throughput 최대화
  Young/Old 병렬 처리
  사용: 큰 힙, 멀티 코어

Copy 알고리즘 (Young)
  Eden + Survivor 2개
  살아있는 객체만 복사
  Compaction 자동
  Fragmentation 없음

Mark-Sweep-Compact (Old)
  Mark: Reachable 표시
  Sweep: Unmarked 제거
  Compact: 객체 이동
  시간: 힙 크기에 비례

Stop-The-World
  GC 중 애플리케이션 정지
  Safe Point에서 대기
  시간: 수십 ms ~ 수 초
  Serial/Parallel 모두 STW

GC 스레드 수
  기본: CPU 코어 수
  튜닝: -XX:ParallelGCThreads=N
  최적: 코어 수 이하

성능 비교
  Serial: 작은 힙에서 빠름
  Parallel: 큰 힙에서 2~5배 빠름
  Break-even: ~500MB
```

---

## 🤔 생각해볼 문제

**Q1.** 8 코어 서버에서 Parallel GC를 사용할 때, Minor GC 시간이 100ms에서 20ms로 단축되었다. 왜 8배가 아닌 5배만 빨라졌는가?

**Q2.** Serial GC의 Copy 알고리즘에서 Survivor 영역이 2개 필요한 이유를 설명하라. 1개만 사용하면 어떤 문제가 발생하는가?

**Q3.** 다음 환경 중 어느 것에 Serial GC가 더 적합한가? 이유를 설명하라.
- 환경 A: 8 코어, 4GB 힙, 배치 작업
- 환경 B: 1 코어, 256MB 힙, Docker 컨테이너

> 💡 **해설**
>
> **Q1.** Linear Scaling 안 되는 이유: ① 동기화 오버헤드 — 8개 스레드가 Survivor 영역에 객체 복사 시 동기화 필요 (Lock contention). ② Amdahl's Law — GC에도 병렬화 불가능한 부분 존재 (Safe Point 대기, 초기화/정리). ③ 메모리 대역폭 — 8개 스레드가 동시에 메모리 접근 → 대역폭 포화. ④ 캐시 경쟁 — CPU 캐시 공유 → 캐시 미스 증가. 실제 Speedup: 5배 (62.5% 효율). 이론 최대: 8배 (100% 효율). Gap: 37.5% (동기화/메모리/캐시 오버헤드).
>
> **Q2.** Survivor 2개 이유 (Copy 알고리즘): Eden + From Survivor의 살아있는 객체를 To Survivor로 복사 → Eden + From 비움. 다음 GC에서 역할 교환 (From ↔ To). 1개만 사용 시: Eden의 객체를 Survivor로 복사 → Eden 비움. 다음 GC: Eden + Survivor의 객체를... 어디로? Survivor가 가득 참 → Compaction 필요 (느림). 또는 Old로 조기 Promotion (Premature Promotion). 2개 사용: Ping-pong 방식으로 항상 빈 공간 확보 → Compaction 불필요 → 빠름.
>
> **Q3.** 환경 B (1 코어, 256MB)가 Serial GC에 적합. 이유: 환경 A — 8 코어, 큰 힙 → Parallel GC가 압도적 유리 (2~5배 빠름). Throughput 중시 배치 → Parallel GC 최적. 환경 B — 1 코어 → Parallel GC 스레드 낭비 (Context Switching만 증가). 작은 힙 (256MB) → GC 시간 짧음 (수 ms) → Parallel 오버헤드가 이득보다 큼. Docker 리소스 제약 → Serial GC가 메모리 절약 (Parallel은 스레드 오버헤드). 결론: B는 Serial, A는 Parallel.

---

## 📚 참고 자료

- [HotSpot Serial GC](https://docs.oracle.com/en/java/javase/17/gctuning/available-collectors.html#GUID-45794DA6-AB96-4856-A96D-FDE5F7DEE498)
- [Parallel GC Tuning](https://docs.oracle.com/en/java/javase/17/gctuning/parallel-collector1.html)
- [Understanding GC Pauses](https://www.infoq.com/articles/Java_Garbage_Collection_Distilled/)

---

<div align="center">

**[⬅️ 이전: Generational Hypothesis](./04-generational-hypothesis.md)** | **[다음: CMS GC & Problems ➡️](./06-cms-gc-and-problems.md)**

</div>

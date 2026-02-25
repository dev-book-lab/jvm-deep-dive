# CMS GC & Problems - CMS GC와 문제점

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CMS (Concurrent Mark Sweep) GC는 무엇이며, 왜 혁신적이었는가?
- Concurrent Marking은 어떻게 동작하며, STW를 어떻게 줄이는가?
- Concurrent Mode Failure는 왜 발생하며, 어떤 결과를 초래하는가?
- Fragmentation 문제는 왜 발생하며, 어떻게 해결하는가?
- CMS가 deprecated된 이유와 G1 GC 탄생 배경은?

---

## 🔍 왜 이게 존재하는가

### 문제: Parallel GC의 한계

```
Parallel GC (10GB 힙):
  Full GC → 2초 STW
  
  웹 서버 요구사항:
  99th percentile < 100ms
  
  2초 STW → 요구사항 20배 초과
  → 사용자 경험 최악
  → Timeout 발생
```

```
목표:
  STW 시간 최소화
  → Low Latency GC

아이디어:
  GC를 애플리케이션과 "동시에" 실행
  → Concurrent GC
```

CMS는 **첫 Concurrent GC**다.

---

## 📐 내부 구조

### 1. CMS GC 개요

```
활성화:
  -XX:+UseConcMarkSweepGC (Java 8)
  Java 9+: Deprecated
  Java 14+: 제거됨

특징:
  - Concurrent Marking
  - Old Generation 전용
  - Low Latency 목표
  - Fragmentation 발생

Young Generation:
  ParNew (Parallel Copy)
  → STW (수십 ms)

Old Generation:
  CMS (Concurrent Mark Sweep)
  → Mostly Concurrent (대부분 동시)
```

---

### 2. CMS 단계 (7단계)

```
1. Initial Mark (STW)
   GC Root 직접 참조 객체만 Mark
   시간: 매우 짧음 (수 ms)

2. Concurrent Mark
   애플리케이션 실행 중 Mark 확장
   시간: 길지만 (수백 ms ~ 수 초) Concurrent

3. Concurrent Preclean
   Dirty Card 처리 (일부)
   시간: 짧음, Concurrent

4. Remark (STW)
   Final Mark (누락 객체 없도록)
   시간: 짧음 (수십 ms)

5. Concurrent Sweep
   애플리케이션 실행 중 Unmarked 제거
   시간: 길지만 Concurrent

6. Concurrent Reset
   다음 GC 준비
   시간: 매우 짧음, Concurrent

7. (선택) Concurrent Abortable Preclean
   Remark 시간 단축 위해
   시간: 가변, Concurrent

STW 단계: Initial Mark, Remark만
나머지: Concurrent (애플리케이션 실행 중)
```

---

### 3. Concurrent Marking 원리

```
문제:
  애플리케이션 실행 중 객체 그래프 변경
  
  예:
  Mark 중:
  A → B (Mark 완료)
  
  애플리케이션:
  A → C (새 참조)
  B → null (참조 제거)
  
  결과:
  C가 누락 → 잘못 GC될 위험

해결: Write Barrier

Write Barrier:
  객체 참조 변경 시 JVM이 개입
  
  oldObj.field = newObj;
  
  컴파일 후:
  if (CMS_active) {
      mark_card(oldObj);  // Card Table
  }
  oldObj.field = newObj;

Card Table:
  객체가 속한 Card를 Dirty로 표시
  → Remark 단계에서 재스캔
```

---

### 4. Initial Mark vs Remark

```
Initial Mark (STW):
  
  목적:
  GC Root 직접 참조만 Mark
  
  예:
  GC Root → A, D
  
  Mark:
  A (marked)
  D (marked)
  
  시간: 수 ms (빠름)

Concurrent Mark:
  
  A → B, C (탐색)
  D → E (탐색)
  
  하지만 애플리케이션 실행 중
  → 참조 변경 가능
  → Card Table에 기록

Remark (STW):
  
  목적:
  Concurrent Mark 중 변경 반영
  
  Dirty Card 재스캔:
  Card 5 (dirty) → F 발견
  
  Final Mark:
  A, B, C, D, E, F
  
  시간: 수십 ms (Initial Mark보다 길지만 짧음)
```

---

### 5. Concurrent Sweep

```
Sweep 단계:

애플리케이션 실행 중:
  Thread 1: 비즈니스 로직
  Thread 2: 비즈니스 로직
  ...
  GC Thread: Sweep (Unmarked 제거)

Free List 생성:
  Unmarked 객체 주소를 Free List에 추가
  
  힙: [A(marked)][B(unmarked)][C(marked)][D(unmarked)]
  
  Free List: [B의 주소, 크기], [D의 주소, 크기]

할당:
  새 객체 요청
  → Free List에서 적절한 크기 블록 찾기
  → 할당

문제: Fragmentation
  [A][Free 10KB][B][Free 5KB][C][Free 8KB]
  
  새 객체 15KB 요청
  → 실패 (연속 공간 없음)
  → Concurrent Mode Failure
```

---

### 6. Concurrent Mode Failure

```
발생 원인:
  
  1. Old Generation이 가득 참
     CMS 실행 중 Old 공간 부족
     → Promotion 실패
  
  2. Fragmentation
     큰 객체 할당 실패
     → 연속 공간 없음

증상:
  
  [CMS concurrent mark]
  [Full GC (Concurrent Mode Failure)]
  ← CMS 중단, Full GC로 Fallback
  
  시간: 수 초 ~ 수십 초 (STW)
  → Low Latency 목표 실패

Full GC (Serial Old):
  
  단일 스레드 Mark-Sweep-Compact
  → 매우 느림
  → Compaction으로 Fragmentation 해결
  
  10GB 힙: 10초 이상 STW

원인 분석:
  
  -XX:+PrintGCDetails
  
  Concurrent Mode Failure 빈번
  → CMSInitiatingOccupancyFraction 조정 필요
```

---

### 7. CMSInitiatingOccupancyFraction

```
CMS 시작 시점:
  
  Old Generation 사용률이 임계값 초과
  → CMS 시작
  
  기본값: 92% (Java 8)
  -XX:CMSInitiatingOccupancyFraction=75

예:
  
  Old Gen 10GB
  Threshold 75%
  
  Old 사용률 7.5GB 도달
  → CMS 시작
  
  CMS 중 Promotion 계속 발생
  → Old 증가
  
  CMS 완료 전 10GB 도달
  → Concurrent Mode Failure

튜닝:
  
  Failure 발생 시:
  Threshold 낮추기 (92% → 70%)
  
  -XX:CMSInitiatingOccupancyFraction=70
  -XX:+UseCMSInitiatingOccupancyOnly
  
  장점: 여유 공간 확보
  단점: CMS 빈번 실행 (CPU 사용 증가)
```

---

## 💻 실험으로 확인하기

### 실험 1: CMS GC 동작 확인

```bash
java -Xmx2g -Xmn500m \
     -XX:+UseConcMarkSweepGC \
     -Xlog:gc*:file=cms.log \
     MyApp

# cms.log:
# [gc] GC(0) Pause Initial Mark
# [gc] GC(0) Concurrent Mark
# [gc] GC(0) Concurrent Preclean
# [gc] GC(0) Pause Remark
# [gc] GC(0) Concurrent Sweep
# [gc] GC(0) Concurrent Reset
```

---

### 실험 2: Concurrent Mode Failure 유발

```java
public class CMSFailureTest {
    static List<byte[]> longLived = new ArrayList<>();
    
    public static void main(String[] args) throws Exception {
        // Old Generation을 빠르게 채움
        for (int i = 0; i < 2000; i++) {
            byte[] data = new byte[1024 * 1024];  // 1MB
            longLived.add(data);
            
            // 중간에 Young 객체도 생성 (Promotion 유발)
            for (int j = 0; j < 100; j++) {
                new byte[1024];
            }
            
            Thread.sleep(1);
        }
    }
}
```

```bash
java -Xmx2g -Xmn500m \
     -XX:+UseConcMarkSweepGC \
     -XX:CMSInitiatingOccupancyFraction=92 \
     -Xlog:gc* \
     CMSFailureTest

# 출력:
# [gc] GC Pause Initial Mark
# [gc] Concurrent Mark
# [gc] Concurrent Mode Failure  ← Failure 발생
# [gc] Full GC (Allocation Failure)
# Pause: 5.2s  ← 매우 긴 STW
```

---

### 실험 3: Fragmentation 확인

```bash
java -XX:+UseConcMarkSweepGC \
     -XX:+PrintHeapAtGC \
     -Xlog:gc*:file=heap.log \
     MyApp

# heap.log 분석:
# Old Generation:
# Free space: 2GB
# Largest free block: 50MB
# → Fragmentation 심각 (연속 공간 부족)
```

---

## ⚡ 실무 임팩트

### CMS 튜닝 가이드

```
기본 설정:
  -XX:+UseConcMarkSweepGC
  -XX:+UseParNewGC (Young)

Concurrent Mode Failure 방지:
  -XX:CMSInitiatingOccupancyFraction=70
  -XX:+UseCMSInitiatingOccupancyOnly
  
  → Old 70% 도달 시 CMS 시작
  → 30% 여유 확보

Remark 시간 단축:
  -XX:+CMSScavengeBeforeRemark
  
  → Remark 전 Young GC
  → Young → Old 참조 감소
  → Remark 시간 단축

GC 스레드 수:
  -XX:ParallelCMSThreads=4
  
  → Concurrent 단계 스레드 수
  → 기본: (ParallelGCThreads + 3) / 4

메모리 여유 확보:
  -Xmx4g (힙 크게)
  -Xmn1g (Young 적절히)
  
  → Old 공간 확보
  → Failure 가능성 감소
```

---

### CMS vs Parallel GC 선택

```
CMS 선택:
  - Low Latency 중시
  - 웹 서버, API 서버
  - 99th percentile < 100ms
  - 큰 힙 (4GB+)

Parallel GC 선택:
  - Throughput 중시
  - 배치 작업
  - Pause Time 무관
  - 작은 힙 (< 2GB)

주의:
  CMS는 deprecated (Java 9+)
  → G1 GC로 마이그레이션 권장
```

---

## 🚫 흔한 오해

### "CMS는 STW가 없다"

```
❌ 잘못된 이해:
  Concurrent이니까 STW가 전혀 없다.

✅ 실제:
  Initial Mark, Remark는 STW
  
  STW 단계:
  Initial Mark: 5ms
  Remark: 50ms
  
  Concurrent 단계:
  Concurrent Mark: 2초 (STW 아님)
  Concurrent Sweep: 1초 (STW 아님)
  
  총 STW: 55ms
  → Parallel GC (2000ms)보다 짧음
  
  "Low Pause"이지 "No Pause" 아님
```

---

### "CMS는 항상 Parallel보다 빠르다"

```
❌ 잘못된 이해:
  Concurrent이니까 무조건 빠르다.

✅ 실제:
  Throughput은 Parallel이 더 높음
  
  CMS:
  - Concurrent 단계 CPU 사용
  → 애플리케이션 스레드와 경쟁
  → 전체 처리량 5~10% 감소
  
  Parallel:
  - GC 중 CPU 100% 사용
  → 애플리케이션 정지
  → 전체 처리량 최대
  
  벤치마크 (배치):
  Parallel: 10분
  CMS: 11분 (10% 느림)
  
  선택:
  Latency → CMS
  Throughput → Parallel
```

---

### "Concurrent Mode Failure는 버그다"

```
❌ 잘못된 이해:
  Failure가 발생하면 설정이 잘못됐다.

✅ 실제:
  CMS의 구조적 한계
  
  불가피한 상황:
  - 갑작스런 트래픽 증가
  - 큰 객체 대량 할당
  - Old Generation 압박
  
  완전 방지 불가:
  CMSInitiatingOccupancyFraction=50
  → Old 50% 시 시작
  → 50% 여유
  → 그래도 Failure 가능 (희귀)
  
  대응:
  Threshold 낮추기
  힙 크게 설정
  G1 GC로 전환
```

---

## 📌 핵심 정리

```
CMS GC
  Concurrent Mark Sweep
  Old Generation 전용
  Low Latency 목표
  Java 14+: 제거됨

핵심 단계
  Initial Mark (STW): GC Root 직접 참조
  Concurrent Mark: 애플리케이션 실행 중 Mark
  Remark (STW): 누락 객체 보정
  Concurrent Sweep: 애플리케이션 실행 중 Sweep

장점
  STW 시간 최소 (수십 ms)
  Pause Time 예측 가능
  대형 힙에 적합

단점
  Fragmentation (Compaction 없음)
  Concurrent Mode Failure (Old 부족)
  Throughput 감소 (5~10%)
  CPU 사용 증가 (Concurrent 단계)

Write Barrier
  객체 참조 변경 감지
  Card Table에 기록
  Remark에서 재스캔

Concurrent Mode Failure
  CMS 중 Old 부족
  → Full GC Fallback (Serial Old)
  → 수 초 ~ 수십 초 STW
  → Low Latency 목표 실패

튜닝
  CMSInitiatingOccupancyFraction=70
  CMSScavengeBeforeRemark
  힙 크게 설정
  G1 GC로 전환 권장

Deprecated
  Java 9+: Deprecated
  Java 14+: 제거
  대안: G1 GC, ZGC
```

---

## 🤔 생각해볼 문제

**Q1.** CMS의 Concurrent Mark 단계에서 애플리케이션이 `A → B` 참조를 제거했다. B가 GC되지 않도록 보호하는 메커니즘을 설명하라.

**Q2.** Concurrent Mode Failure가 발생하면 왜 Serial Old GC로 Fallback하는가? Parallel Old를 사용하지 않는 이유는?

**Q3.** CMS에서 CMSInitiatingOccupancyFraction을 50%로 낮추면 어떤 장단점이 있는가? Concurrent Mode Failure 빈도와 CPU 사용률을 고려하라.

> 💡 **해설**
>
> **Q1.** Write Barrier로 보호. Concurrent Mark 중 `A.field = null` 실행 시: ① Write Barrier 발동 → A가 속한 Card를 Dirty로 표시 (Card Table). ② Concurrent Mark는 B를 이미 Mark했거나, 아직 안 했을 수 있음. ③ Remark 단계 (STW): Dirty Card 재스캔 → A 확인 → A → B 참조 없음 확인. ④ 하지만 B는 이미 Marked → 살아있음으로 유지. ⑤ B가 다른 참조도 없으면 다음 GC에서 제거. 핵심: Concurrent Mark는 Conservative (보수적) — 의심스러우면 살려둠 → False Positive OK, False Negative는 안 됨.
>
> **Q2.** CMS는 Old Generation 전용 알고리즘 — Young은 ParNew (별도). Concurrent Mode Failure 시: ① CMS 중단 (Mark/Sweep 미완성). ② Old Generation Compaction 필요 (Fragmentation 해결). ③ HotSpot에 내장된 Compaction 알고리즘: Serial Old (단일 스레드 Mark-Sweep-Compact). Parallel Old를 사용 안 하는 이유: CMS와 Parallel은 별개 GC 구현 — CMS는 Parallel Old와 통합 안 됨. 기술적 부채로 Serial Old만 Fallback 가능. 결과: 매우 느린 Full GC (수 초~수십 초).
>
> **Q3.** 50%로 낮추면: 장점 — ① Concurrent Mode Failure 대폭 감소 (50% 여유 → CMS 완료 시간 충분). ② Promotion 여유 확보. 단점 — ① CMS 빈번 실행 (Old 50% 도달이 빠름). ② CPU 사용률 증가 (Concurrent Mark/Sweep 자주). ③ Throughput 감소 (5~10% → 10~15%). ④ GC 로그 증가. 권장 설정: 70~80% (균형). 50%는 너무 공격적 — Failure는 줄지만 오버헤드 큼. 92% (기본)는 너무 느슨 — Failure 위험.

---

## 📚 참고 자료

- [CMS Collector Deprecated](https://openjdk.org/jeps/291)
- [Understanding CMS GC Logs](https://blogs.oracle.com/jonthecollector/our-collectors)
- [CMS Tuning Guide](https://docs.oracle.com/en/java/javase/11/gctuning/concurrent-mark-sweep-cms-collector.html)

---

<div align="center">

**[⬅️ 이전: Serial & Parallel GC](./05-serial-parallel-gc.md)** | **[다음: G1 GC Deep Dive ➡️](./07-g1-gc-deep-dive.md)**

</div>

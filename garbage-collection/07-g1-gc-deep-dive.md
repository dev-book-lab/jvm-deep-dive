# G1 GC Deep Dive - G1 GC 심층 분석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- G1 (Garbage First) GC는 무엇이며, CMS의 문제를 어떻게 해결하는가?
- Region 기반 구조는 어떻게 동작하며, 왜 혁신적인가?
- Pause Prediction Model은 무엇이며, 목표 시간을 어떻게 맞추는가?
- Mixed GC와 Full GC의 차이는 무엇인가?
- 언제 G1을 사용하고, 어떻게 튜닝하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: CMS의 한계

```
CMS 문제:
  1. Fragmentation
     Compaction 없음
     → Concurrent Mode Failure
     → Full GC (수 초 STW)
  
  2. 예측 불가능한 Pause
     Young/Old 크기 고정
     → Pause Time 조절 어려움
  
  3. Full Heap Scan
     Card Table 스캔 부담

목표:
  - Fragmentation 해결
  - Pause Time 예측 가능
  - 큰 힙 (6GB+) 지원
```

G1 GC는 **Region 기반 + Pause Prediction**으로 해결한다.

---

## 📐 내부 구조

### 1. G1 GC 개요

```
활성화:
  -XX:+UseG1GC (Java 9+ 기본)

특징:
  - Region 기반 힙 구조
  - Pause Time 목표 설정 가능
  - Concurrent + Incremental
  - Compaction 지원

목표:
  -XX:MaxGCPauseMillis=200 (기본)
  → 200ms 이하 Pause 목표

Young + Old 통합:
  CMS처럼 분리 안 함
  → 모든 Region이 Young/Old 역할 가능
```

---

### 2. Region 기반 구조

```
힙을 작은 Region으로 분할:

Region 크기:
  1MB, 2MB, 4MB, 8MB, 16MB, 32MB
  자동 계산: 힙 크기 / 2048
  
  예:
  8GB 힙 → 4MB Region → 2048개 Region

Region 타입:
  ┌──────┬──────┬──────┬──────┬──────┐
  │ Eden │ Eden │Surv. │ Old  │ Old  │
  ├──────┼──────┼──────┼──────┼──────┤
  │ Old  │Humong│Humong│ Eden │ Free │
  └──────┴──────┴──────┴──────┴──────┘

Eden: 새 객체 할당
Survivor: Young GC 생존 객체
Old: Promotion된 객체
Humongous: 큰 객체 (Region 크기 50% 이상)
Free: 비어있음 (재할당 가능)

유연성:
  Region 역할이 동적으로 변경
  
  GC 전: Eden Region
  GC 후: Free Region → Old로 재할당 가능
```

---

### 3. Young GC (Evacuation Pause)

```
Young GC 과정:

1. Eden + Survivor Region 선택
   ┌──────┬──────┬──────┬──────┐
   │ Eden │ Eden │Surv. │ Old  │
   └──────┴──────┴──────┴──────┘
   
2. 살아있는 객체 복사 (Evacuation)
   Eden → Survivor (age++)
   Survivor → Survivor (age++)
   Survivor → Old (age==15)
   
   ┌──────┬──────┬──────┬──────┬──────┐
   │ Free │ Free │Surv. │ Old  │ Old  │
   └──────┴──────┴──────┴──────┴──────┘
   
3. 원래 Region 비움
   Eden, Survivor Region → Free

STW:
  수십 ms (목표 시간 내)

병렬 처리:
  -XX:ParallelGCThreads=8
  → 8개 스레드 병렬 Evacuation
```

---

### 4. Concurrent Marking Cycle

```
목적:
  Old Region의 Live 객체 비율 계산
  → Garbage 많은 Region 찾기

단계:

1. Initial Mark (STW, Young GC와 통합)
   GC Root 직접 참조 Mark
   시간: 수 ms

2. Root Region Scan (Concurrent)
   Survivor Region 스캔
   (Young → Old 참조)

3. Concurrent Mark (Concurrent)
   Reachable 객체 Mark
   시간: 수백 ms ~ 수 초

4. Remark (STW)
   SATB (Snapshot-At-The-Beginning) 처리
   시간: 수십 ms

5. Cleanup (STW + Concurrent)
   Region 통계 계산
   빈 Region 회수

결과:
  각 Region의 Live 비율 파악
  → Mixed GC에서 활용
```

---

### 5. Mixed GC

```
Mixed GC:
  Young + Old (Garbage 많은 Region) 동시 수집

트리거:
  Concurrent Marking 완료 후
  Old Region 중 Garbage 많은 것 선택

선택 기준:
  Garbage 비율 높은 순서
  
  Region A: 90% Garbage
  Region B: 80% Garbage
  Region C: 60% Garbage
  
  → A, B 우선 선택 (Garbage First!)

과정:
  1. Young Region 전체 + Old Region 일부 선택
     (Pause Time 목표 고려)
  
  2. Evacuation (복사)
     선택된 Region → 새 Region
  
  3. 원래 Region 비움 (Compaction 효과)

여러 번 반복:
  한 번에 모든 Old 처리 안 함
  → Incremental (점진적)
  → Pause Time 목표 달성
```

---

### 6. Pause Prediction Model

```
목표:
  -XX:MaxGCPauseMillis=200 (200ms)

동작:

1. 과거 GC 통계 수집
   각 Region Evacuation 시간
   
   Region A: 10ms
   Region B: 15ms
   Region C: 12ms

2. Young Region 크기 조정
   목표 200ms
   - Young 많이: Pause 길어짐
   - Young 적게: Pause 짧아짐
   
   자동 조절:
   Eden Region 개수 = f(pause_goal, statistics)

3. Mixed GC Region 선택
   목표 200ms 내에서 최대한 많은 Old 수집
   
   예:
   예상: A(10ms) + B(15ms) + C(12ms) + Young(150ms) = 187ms
   → A, B, C 선택
   
   D(20ms) 추가하면 207ms 초과
   → D는 다음 Mixed GC로 연기

4. 피드백 루프
   실제 Pause 측정
   → 모델 업데이트
   → 다음 GC에 반영

결과:
  Pause Time이 목표에 수렴
  (완벽하지 않지만 근접)
```

---

### 7. Full GC (Serial Old)

```
Full GC 발생 조건:

1. Evacuation Failure
   복사할 공간 부족
   → "to-space exhausted"

2. Humongous Allocation 실패
   큰 객체 할당 불가

3. Concurrent Mode Failure (드뭄)
   Marking 완료 전 Old 가득 참

Full GC 동작:
  Serial Old (단일 스레드)
  Mark-Sweep-Compact
  시간: 수 초 ~ 수십 초 (매우 느림)

회피 방법:
  1. 힙 크게 설정
  2. Humongous 객체 줄이기
  3. Mixed GC 빈도 증가
     -XX:G1MixedGCCountTarget=8 (기본)
```

---

## 💻 실험으로 확인하기

### 실험 1: G1 GC 동작 확인

```bash
java -Xmx4g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -Xlog:gc*:file=g1.log \
     MyApp

# g1.log:
# [gc] GC(0) Pause Young (Normal) (G1 Evacuation Pause)
# [gc] GC(0) Eden: 512M -> 0M
# [gc] GC(0) Survivor: 0M -> 64M
# [gc] GC(0) 50ms
# 
# [gc] GC(5) Concurrent Marking Cycle
# [gc] GC(5) Pause Initial Mark
# [gc] GC(5) Concurrent Root Region Scan
# [gc] GC(5) Concurrent Mark
# [gc] GC(5) Pause Remark
# [gc] GC(5) Pause Cleanup
#
# [gc] GC(6) Pause Young (Mixed)
# [gc] GC(6) Eden: 512M -> 0M
# [gc] GC(6) Old: 1024M -> 800M
# [gc] GC(6) 120ms
```

---

### 실험 2: Pause Time 목표 조정

```java
public class G1PauseTest {
    public static void main(String[] args) throws Exception {
        List<byte[]> list = new ArrayList<>();
        
        for (int i = 0; i < 10000; i++) {
            byte[] data = new byte[1024 * 100];  // 100KB
            list.add(data);
            
            if (i % 100 == 0) {
                list.clear();
            }
        }
    }
}
```

```bash
# 목표 100ms
java -Xmx2g -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=100 \
     -Xlog:gc \
     G1PauseTest
# 출력: Pause 50~100ms

# 목표 500ms
java -Xmx2g -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=500 \
     -Xlog:gc \
     G1PauseTest
# 출력: Pause 100~300ms (더 긴 Pause 허용)
```

---

### 실험 3: Humongous Region 확인

```bash
java -Xmx4g -XX:+UseG1GC \
     -XX:G1HeapRegionSize=2m \
     -Xlog:gc+heap=trace \
     MyApp

# 로그:
# Humongous object allocation:
# Region size: 2MB
# Object size: 1.5MB (>50%)
# → Humongous Region 할당
```

---

## ⚡ 실무 임팩트

### G1 GC 튜닝 가이드

```
기본 설정:
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200

Pause Time 조정:
  낮은 Latency (웹 서버):
  -XX:MaxGCPauseMillis=100
  
  높은 Throughput (배치):
  -XX:MaxGCPauseMillis=500

Region 크기:
  -XX:G1HeapRegionSize=16m
  
  Humongous 많으면: Region 크게
  객체 대부분 작으면: Region 작게

Mixed GC 빈도:
  -XX:G1MixedGCCountTarget=8 (기본)
  -XX:G1MixedGCLiveThresholdPercent=85
  
  Old Region Live > 85% → Mixed GC 대상 제외

Concurrent 스레드:
  -XX:ConcGCThreads=2
  
  기본: ParallelGCThreads / 4
  CPU 여유 있으면 증가

힙 크기:
  -Xmx8g -Xms8g (고정)
  
  → 힙 크기 변동 없음
  → GC 예측 안정
```

---

### G1 vs CMS vs Parallel 선택

```
G1 GC:
  - 중간 Latency (50~200ms)
  - 예측 가능한 Pause
  - 큰 힙 (6GB+)
  - 웹 서버, API 서버

CMS (deprecated):
  - 낮은 Latency (10~50ms)
  - 예측 불가능
  - Fragmentation 문제
  → G1으로 전환 권장

Parallel GC:
  - 높은 Throughput
  - 긴 Pause (수 초)
  - 배치 작업
  - 작은 힙 (< 4GB)

ZGC/Shenandoah:
  - 매우 낮은 Latency (< 10ms)
  - 초대형 힙 (수백 GB)
  - 최신 Java (11+)
  - 실시간성 중요
```

---

## 🚫 흔한 오해

### "G1은 항상 목표 Pause Time을 지킨다"

```
❌ 잘못된 이해:
  MaxGCPauseMillis=100이면 항상 100ms 이하다.

✅ 실제:
  목표일 뿐, 보장 아님
  
  초과 가능한 경우:
  - Full GC 발생
  - Evacuation Failure
  - 목표가 너무 낮음 (< 10ms)
  - 힙이 너무 작음
  
  통계 (정상):
  90%: 목표 이하
  9%: 목표의 1.5배
  1%: 목표의 2배 이상
  
  완벽한 보장은 불가능
  → ZGC로 가야 함 (< 10ms 보장)
```

---

### "G1은 Fragmentation이 없다"

```
❌ 잘못된 이해:
  Region 기반이라 Fragmentation 없다.

✅ 실제:
  Region 내부 Fragmentation 존재
  
  Region A:
  [Live][Free][Live][Free][Live]
  
  → Region 내부는 단편화 가능
  
  하지만:
  Mixed GC에서 Evacuation (복사)
  → 자동 Compaction
  → Region 수준 Fragmentation 해결
  
  Humongous 문제:
  연속된 여러 Region 필요
  → 할당 실패 가능
  
  완전 제거는 아니지만
  CMS보다 훨씬 나음
```

---

### "G1은 작은 힙에 적합하다"

```
❌ 잘못된 이해:
  G1은 모든 힙 크기에 최적이다.

✅ 실제:
  큰 힙 (6GB+)에 최적화됨
  
  작은 힙 (< 2GB):
  - Region 오버헤드
  - Pause Prediction 부정확
  - Parallel GC가 나음
  
  큰 힙 (> 6GB):
  - Region 기반 장점
  - Incremental GC
  - Pause Time 제어
  
  권장:
  2GB 미만: Parallel GC
  2~6GB: G1 또는 Parallel
  6GB 이상: G1 GC
```

---

## 📌 핵심 정리

```
G1 GC
  Garbage First
  Region 기반 구조
  Pause Time 목표 설정 가능
  Java 9+ 기본 GC

Region 기반
  힙을 작은 Region으로 분할
  크기: 1~32MB (자동 계산)
  타입: Eden, Survivor, Old, Humongous, Free
  유연한 역할 변경

Young GC
  Eden + Survivor Region Evacuation
  STW (수십 ms)
  병렬 처리

Concurrent Marking
  Old Region Live 비율 계산
  Mostly Concurrent
  Remark만 STW

Mixed GC
  Young + Old (Garbage 많은 것) 동시 수집
  Incremental (여러 번 나눠서)
  Compaction 효과

Pause Prediction Model
  과거 통계로 예측
  Young 크기 자동 조정
  Mixed GC Region 선택
  목표 Pause Time 달성

Full GC
  Evacuation Failure 시
  Serial Old (매우 느림)
  회피가 중요

튜닝
  MaxGCPauseMillis (기본 200ms)
  G1HeapRegionSize
  G1MixedGCCountTarget
  힙 크기 충분히 확보

선택 기준
  큰 힙 (6GB+)
  예측 가능한 Pause 필요
  중간 Latency (50~200ms)
  웹 서버, API 서버
```

---

## 🤔 생각해볼 문제

**Q1.** G1 GC의 Pause Prediction Model이 MaxGCPauseMillis=100ms 목표를 맞추기 위해 Young Generation 크기를 어떻게 조절하는가?

**Q2.** Mixed GC에서 Garbage 90% Region과 Garbage 60% Region 중 어느 것을 먼저 선택하는가? 그 이유를 "Garbage First" 철학과 연결해 설명하라.

**Q3.** G1 GC에서 Full GC를 회피하려면 어떻게 튜닝해야 하는가? 힙 크기, Region 크기, Mixed GC 빈도를 고려하라.

> 💡 **해설**
>
> **Q1.** Pause Prediction 메커니즘: ① 과거 Young GC 통계 수집 (Eden Region 1개당 평균 5ms 소요). ② 목표 100ms → Eden Region 최대 20개 할당 가능 (100ms / 5ms = 20). ③ 현재 Eden 30개 → Pause 150ms 예상 → 목표 초과. ④ 다음 GC: Eden 15개로 축소 → Pause 75ms 예상. ⑤ 실제 Pause 85ms → 통계 업데이트. ⑥ 다음 GC: Eden 17개 (미세 조정). 결과: 점진적으로 목표에 수렴 (완벽하지 않지만 근접).
>
> **Q2.** 90% Garbage Region 우선 선택. 이유: "Garbage First" — 가장 효율적인 Region (Garbage 많은 것) 먼저 수집 → 적은 Pause로 많은 메모리 회수. 90% Region: 10% Live 복사 → 90% 회수. 60% Region: 40% Live 복사 → 60% 회수. 같은 Pause Time이면 90% Region이 2배 효율. Incremental 전략: 한 번에 모든 Old 안 함 → 효율 높은 것부터 → Pause Time 목표 달성하면서 메모리 최대 회수.
>
> **Q3.** Full GC 회피 전략: ① 힙 크기 증가 (-Xmx16g 등) → Old 여유 확보 → Evacuation Failure 방지. ② Region 크기 증가 (-XX:G1HeapRegionSize=32m) → Humongous 임계값 증가 (16MB까지 일반 객체) → Humongous 할당 실패 감소. ③ Mixed GC 빈도 증가 (-XX:G1MixedGCCountTarget=16) → Old Region 더 자주 수집 → Garbage 누적 방지. ④ Concurrent 스레드 증가 (-XX:ConcGCThreads=4) → Marking 더 빨리 → Old 압박 감소. ⑤ Pause 목표 완화 (-XX:MaxGCPauseMillis=300) → Mixed GC에서 더 많은 Old 수집 가능.

---

## 📚 참고 자료

- [Getting Started with G1 GC](https://www.oracle.com/technical-resources/articles/java/g1gc.html)
- [G1 GC Tuning Guide](https://docs.oracle.com/en/java/javase/17/gctuning/garbage-first-g1-garbage-collector1.html)
- [Understanding G1 GC Logs](https://www.redhat.com/en/blog/understanding-g1-gc-logs)

---

<div align="center">

**[⬅️ 이전: CMS GC & Problems](./06-cms-gc-and-problems.md)** | **[다음: ZGC Deep Dive ➡️](./08-zgc-deep-dive.md)**

</div>

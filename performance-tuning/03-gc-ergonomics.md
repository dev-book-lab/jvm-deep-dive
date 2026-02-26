# GC Ergonomics - GC 자동 튜닝

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- GC Ergonomics는 무엇이며, 어떻게 동작하는가?
- JVM이 자동으로 조정하는 것은 무엇인가?
- Ergonomics의 목표와 한계는?
- 언제 수동 튜닝이 필요한가?

---

## 🔍 왜 이게 존재하는가

### 문제: 최적의 GC 설정은 복잡하다

```
수동 튜닝:
  - Young/Old 크기
  - GC 스레드 수
  - Pause Time 목표
  - ...수십 개 파라미터

대부분 사용자:
  최적 설정 모름
  → 기본값 사용
  → 성능 저하

해결:
  JVM이 자동 튜닝
  → Ergonomics
```

Ergonomics는 **JVM의 자동 최적화**다.

---

## 📐 Ergonomics 동작 원리

### 1. 자동 설정 항목

```
JVM 시작 시:
  - GC 선택 (G1/Parallel/...)
  - 힙 크기 (Xms, Xmx)
  - Young/Old 비율
  - GC 스레드 수
  - Pause Time 목표

실행 중:
  - Young Gen 크기 조정
  - Tenuring Threshold
  - Survivor 크기
```

---

### 2. GC 선택 (Java 9+)

```
기본 GC:
  Java 9+: G1 GC
  Java 8: Parallel GC

자동 선택 기준:
  - 서버 클래스 머신 (2GB+ RAM, 2+ CPU)
    → G1 GC (Java 9+)
  
  - 클라이언트 클래스 머신
    → Serial GC

확인:
  java -XX:+PrintFlagsFinal -version | grep "Use.*GC"
  # UseG1GC = true {ergonomic}
```

---

### 3. 힙 크기 자동 설정

```
Xms (초기 힙):
  물리 메모리 / 64
  최소: 8MB

Xmx (최대 힙):
  물리 메모리 / 4
  최대: 1GB (32bit), 무제한 (64bit)

예:
  RAM: 16GB
  → Xms: 256MB
  → Xmx: 4GB

확인:
  java -XX:+PrintFlagsFinal -version | grep HeapSize
  # InitialHeapSize = 268435456 {ergonomic}
  # MaxHeapSize = 4294967296 {ergonomic}
```

---

### 4. GC 스레드 수

```
ParallelGCThreads:
  CPU ≤ 8: CPU 코어 수
  CPU > 8: 8 + (CPU - 8) × 5/8

예:
  8 코어: 8 스레드
  16 코어: 8 + (16-8) × 5/8 = 13 스레드

ConcGCThreads (G1, CMS):
  ParallelGCThreads / 4

확인:
  java -XX:+PrintFlagsFinal -version | grep GCThreads
  # ParallelGCThreads = 8 {ergonomic}
  # ConcGCThreads = 2 {ergonomic}
```

---

### 5. Adaptive Sizing (G1)

```
G1 GC의 자동 조정:

1. Young Gen 크기
   목표: MaxGCPauseMillis 달성
   
   Pause 길면 → Young 축소
   Pause 짧으면 → Young 확대

2. Mixed GC 빈도
   Old Gen 압박 → Mixed GC 빈도 증가

3. Marking Threshold
   InitiatingHeapOccupancyPercent 조정
   
과정:
  GC 실행 → 통계 수집 → 다음 GC 반영
```

---

### 6. Parallel GC Ergonomics

```
목표:
  MaxGCPauseMillis (기본: 무제한)
  GCTimeRatio (기본: 99, GC 1%)

조정:
  1. Young Gen 크기
     Pause 목표 달성 위해 조정
  
  2. Tenuring Threshold
     Promotion Rate 최소화
  
  3. Survivor 크기
     객체 수명에 따라 조정

UseAdaptiveSizePolicy (기본: true)
  자동 조정 활성화
```

---

## 💻 실험으로 확인하기

### 실험 1: Ergonomic 플래그 확인

```bash
# 자동 설정된 플래그
java -XX:+PrintFlagsFinal -version | grep ergonomic

# 출력 예:
# bool UseG1GC = true {ergonomic}
# uintx InitialHeapSize = 268435456 {ergonomic}
# uintx MaxHeapSize = 4294967296 {ergonomic}
# uintx ParallelGCThreads = 8 {ergonomic}
```

---

### 실험 2: Adaptive Sizing 관찰

```bash
# G1 GC Adaptive Sizing 로그
java -Xlog:gc+ergo*:file=ergo.log \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=100 \
     MyApp

# ergo.log:
# [gc,ergo] Eden: 512M -> 400M (pause too long)
# [gc,ergo] Survivor: 64M -> 80M
# [gc,ergo] IHOP: 45 -> 40 (old pressure)
```

---

## ⚡ 실무 임팩트

### Ergonomics 신뢰 vs 수동 튜닝

```
Ergonomics 충분:
  - 일반적인 웹 서버
  - 표준 워크로드
  - 리소스 충분
  
수동 튜닝 필요:
  - 극한 성능 요구
  - 특수 워크로드 (메모리 패턴)
  - 리소스 제약 (컨테이너)
  - Latency < 10ms 목표

권장:
  1. Ergonomics로 시작
  2. 모니터링
  3. 문제 발견 시 수동 튜닝
```

---

### 컨테이너에서 Ergonomics

```bash
# Kubernetes Pod: 2GB RAM, 2 CPU
# JVM Ergonomics (Java 10+)

-XX:+UseContainerSupport  # 기본 활성화

자동 설정:
  MaxHeapSize = 2GB × 0.25 = 512MB (Java 8)
  MaxHeapSize = 2GB × 0.25 = 512MB (Java 10+, 수정됨)
  
Java 10+ 개선:
  -XX:MaxRAMPercentage=75.0 (기본: 25%)
  → 2GB × 0.75 = 1.5GB

수동 권장:
  -Xms1500m -Xmx1500m
  (Ergonomics보다 명확)
```

---

## 🚫 흔한 오해

### "Ergonomics는 항상 최적이다"

```
❌ 잘못된 이해:
  Ergonomics면 튜닝 불필요

✅ 실제:
  일반적 케이스에 적합
  특수 케이스는 수동 필요
  
예:
  초저지연 (< 10ms)
  → ZGC 명시 필요
  
  메모리 제약 (컨테이너)
  → 힙 크기 명시 권장
```

---

### "Ergonomics는 실행 중 최적화한다"

```
❌ 잘못된 이해:
  실행 중 모든 설정 자동 조정

✅ 실제:
  일부만 동적 조정
  
동적 조정:
  - Young Gen 크기 (Adaptive Sizing)
  - Tenuring Threshold
  - Survivor 크기

고정:
  - GC 종류
  - 최대 힙 크기
  - GC 스레드 수 (대부분)
```

---

## 📌 핵심 정리

```
GC Ergonomics
  JVM 자동 튜닝
  기본 설정 최적화
  워크로드 적응

자동 설정
  GC 선택 (G1/Parallel)
  힙 크기 (RAM/4, RAM/64)
  GC 스레드 (CPU 기반)
  Young/Old 비율

Adaptive Sizing
  G1: Young Gen, IHOP
  Parallel: Pause 목표 기반
  실행 중 통계로 조정

목표
  MaxGCPauseMillis (G1, Parallel)
  GCTimeRatio (Parallel)

확인
  -XX:+PrintFlagsFinal
  {ergonomic} 태그

수동 튜닝 필요
  초저지연 (ZGC)
  컨테이너 (힙 명시)
  특수 워크로드

Best Practice
  Ergonomics로 시작
  모니터링 지속
  문제 시 수동 튜닝
  최소한의 플래그
```

---

## 🤔 생각해볼 문제

**Q1.** 16GB RAM, 16 Core 서버에서 JVM을 기본 설정으로 시작하면 힙 크기와 GC 스레드는 각각 얼마나 되는가?

**Q2.** G1 GC의 Adaptive Sizing이 Young Gen 크기를 줄이는 조건을 설명하라.

**Q3.** 컨테이너 환경 (4GB RAM)에서 Ergonomics에만 의존하는 것이 위험한 이유를 설명하라.

> 💡 **해설**
>
> **Q1.** Ergonomic 설정: ① 힙 크기 — Xms: 16GB / 64 = 256MB. Xmx: 16GB / 4 = 4GB. ② GC 스레드 — ParallelGCThreads: 8 + (16-8) × 5/8 = 13. ConcGCThreads: 13 / 4 = 3. ③ GC: G1 GC (Java 9+, 서버 클래스). 확인: `java -XX:+PrintFlagsFinal -version | grep -E "HeapSize|GCThreads|UseG1GC"`.
>
> **Q2.** Young Gen 축소 조건: ① Pause Time > MaxGCPauseMillis — Young GC 시간이 목표 초과. ② G1은 통계 분석 — Young 크기와 Pause 상관관계 파악. ③ Young 크면 GC 시간 증가 → 축소로 Pause 단축. ④ 예: 목표 100ms, 실제 150ms → Young 512MB → 400MB 축소. ⑤ 피드백 루프 — 다음 GC 결과 반영, 점진적 조정.
>
> **Q3.** 컨테이너 Ergonomics 위험: ① Java 8: MaxHeapSize = RAM / 4 — 4GB / 4 = 1GB. 하지만 Container Limit 무시 → Native 메모리 추가 → 총 1.5GB+ → OOM Killer. ② Java 10+: MaxRAMPercentage=25% (기본) — 4GB × 0.25 = 1GB. 너무 보수적 → 메모리 낭비. ③ 권장: 명시적 설정 `-Xms3g -Xmx3g` (75% 활용) + `-XX:MaxRAMPercentage=75.0` 또는 명시.

---

## 📚 참고 자료

- [Ergonomics in HotSpot VM](https://docs.oracle.com/en/java/javase/17/gctuning/ergonomics.html)
- [G1 GC Ergonomics](https://www.oracle.com/technical-resources/articles/java/g1gc.html)

---

<div align="center">

**[⬅️ 이전: Heap Sizing Strategy](./02-heap-sizing-strategy.md)** | **[다음: Profiling with JFR ➡️](./04-profiling-with-jfr.md)**

</div>

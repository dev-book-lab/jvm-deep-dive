# GC Tuning Flags - GC 튜닝 플래그

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 주요 GC 튜닝 플래그는 무엇이며, 언제 사용하는가?
- 힙 크기, GC 스레드, Pause Time 목표를 어떻게 설정하는가?
- GC 로그를 어떻게 활성화하고 분석하는가?
- 각 GC별 주요 튜닝 포인트는 무엇인가?
- 실무에서 어떻게 체계적으로 튜닝하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 기본 설정은 범용적이다

```
기본 GC 설정:
  모든 애플리케이션에 적합하도록
  → 중간값 선택
  
  하지만:
  - 웹 서버: Latency 중시
  - 배치: Throughput 중시
  - 실시간: Ultra Low Latency
  
  → 각각 다른 튜닝 필요
```

체계적인 **GC 튜닝 방법론**이 필요하다.

---

## 📐 주요 플래그 카테고리

### 1. 힙 크기 설정

```
기본 힙 크기:
  -Xms<size>    초기 힙
  -Xmx<size>    최대 힙
  
  예: -Xms4g -Xmx4g
  
권장:
  Xms = Xmx (고정)
  → 힙 크기 변동 없음
  → GC 예측 안정
  → 성능 일정

Young Generation:
  -Xmn<size>    Young 크기 (고정)
  -XX:NewRatio=N  Old/Young 비율
  
  예: -Xmn1g (Young 1GB)
  예: -XX:NewRatio=2 (Old 2, Young 1)

Metaspace (Java 8+):
  -XX:MetaspaceSize=<size>     초기
  -XX:MaxMetaspaceSize=<size>  최대
  
  예: -XX:MaxMetaspaceSize=512m
  
  기본: 무제한 (Native Memory)
  → OOM 방지 위해 제한 권장
```

---

### 2. GC 선택

```
GC 선택:
  -XX:+UseSerialGC       Serial GC
  -XX:+UseParallelGC     Parallel GC (기본, Java 8)
  -XX:+UseG1GC           G1 GC (기본, Java 9+)
  -XX:+UseConcMarkSweepGC CMS (deprecated)
  -XX:+UseZGC            ZGC (Java 11+)
  -XX:+UseShenandoahGC   Shenandoah (Java 12+)

선택 기준:
  Throughput: Parallel GC
  Balanced: G1 GC
  Low Latency: ZGC, Shenandoah
  Small Heap: Serial GC
```

---

### 3. GC 스레드 수

```
Parallel GC 스레드:
  -XX:ParallelGCThreads=N
  
  기본: CPU 코어 수
  권장: 코어 수 또는 코어 수 × 0.75
  
  예: 8 코어
  -XX:ParallelGCThreads=8 (기본)
  -XX:ParallelGCThreads=6 (CPU 여유 확보)

Concurrent GC 스레드:
  -XX:ConcGCThreads=N
  
  기본: ParallelGCThreads / 4
  G1, CMS, ZGC, Shenandoah에서 사용
  
  예: -XX:ConcGCThreads=2
```

---

### 4. Pause Time 목표

```
G1 GC:
  -XX:MaxGCPauseMillis=N
  
  기본: 200ms
  범위: 10~1000ms
  
  예:
  -XX:MaxGCPauseMillis=100  (웹 서버)
  -XX:MaxGCPauseMillis=500  (배치)
  
  주의:
  너무 낮게 (< 50ms) → 목표 미달성
  너무 높게 (> 500ms) → Throughput 저하

ZGC/Shenandoah:
  MaxGCPauseMillis 무시
  → 항상 < 10ms 목표
```

---

### 5. G1 GC 튜닝

```
Region 크기:
  -XX:G1HeapRegionSize=N
  
  범위: 1MB ~ 32MB
  기본: 힙 크기 / 2048
  
  Humongous 많으면: 크게 (16MB, 32MB)
  작은 객체 많으면: 작게 (2MB, 4MB)

Initiating Heap Occupancy:
  -XX:InitiatingHeapOccupancyPercent=N
  
  기본: 45 (Old 45% 시 Marking 시작)
  
  Full GC 빈번: 낮추기 (40, 35)
  CPU 부담: 높이기 (50, 55)

Mixed GC:
  -XX:G1MixedGCCountTarget=N
  
  기본: 8 (Mixed GC 8번에 나눠 수집)
  
  Pause 짧게: 증가 (16)
  Pause 길게: 감소 (4)

Old Region 선택:
  -XX:G1MixedGCLiveThresholdPercent=N
  
  기본: 85 (Live > 85%면 제외)
  
  더 많은 Old 수집: 낮추기 (80)
```

---

### 6. Parallel GC 튜닝

```
GC 시간 비율:
  -XX:GCTimeRatio=N
  
  기본: 99 (GC 시간 1%, 애플리케이션 99%)
  
  Throughput 중시: 증가 (199 → GC 0.5%)
  Latency 중시: 감소 (49 → GC 2%)

Adaptive Size Policy:
  -XX:+UseAdaptiveSizePolicy (기본)
  -XX:-UseAdaptiveSizePolicy (비활성화)
  
  활성화: Young/Old 크기 자동 조정
  비활성화: 수동 제어

Max Pause:
  -XX:MaxGCPauseMillis=N
  
  Parallel GC에서도 사용 가능
  하지만 목표 달성 보장 안 됨
```

---

### 7. ZGC 튜닝

```
기본 설정:
  -XX:+UseZGC
  -Xmx16g

Concurrent 스레드:
  -XX:ConcGCThreads=N
  
  기본: CPU / 8
  권장: 2~4

Uncommit 메모리:
  -XX:ZUncommitDelay=N
  
  기본: 300초
  N초 사용 안 하면 OS에 반환
  
  컨테이너: 짧게 (60초)
  전용 서버: 길게 (600초)

Large Pages:
  -XX:+UseLargePages
  
  성능: 5~10% 향상
  설정: OS Large Pages 활성화 필요
```

---

### 8. GC 로그

```
Java 8:
  -XX:+PrintGCDetails
  -XX:+PrintGCDateStamps
  -Xloggc:gc.log
  
  예:
  -XX:+PrintGCDetails \
  -XX:+PrintGCDateStamps \
  -Xloggc:/var/log/gc.log

Java 9+:
  -Xlog:gc*:file=gc.log:time,uptime,level,tags
  
  간단:
  -Xlog:gc:gc.log
  
  상세:
  -Xlog:gc*=debug:gc.log

GC 로그 회전:
  -XX:+UseGCLogFileRotation
  -XX:NumberOfGCLogFiles=10
  -XX:GCLogFileSize=100M
  
  → 100MB마다 회전, 최대 10개 유지
```

---

## 💻 실무 튜닝 시나리오

### 시나리오 1: 웹 서버 (Low Latency)

```
요구사항:
  - 99th percentile < 100ms
  - 힙 8GB
  - 멀티 코어 (16 core)

튜닝:
  -Xms8g -Xmx8g
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=100
  -XX:ParallelGCThreads=12
  -XX:ConcGCThreads=3
  -Xlog:gc*:file=/var/log/gc.log

모니터링:
  Pause Time 분포 확인
  99th < 100ms 달성 여부
  
  초과 시:
  MaxGCPauseMillis=80 (더 낮게)
  힙 크기 증가 (10g)
```

---

### 시나리오 2: 배치 작업 (Throughput)

```
요구사항:
  - 처리량 최대화
  - Pause Time 무관
  - 힙 4GB

튜닝:
  -Xms4g -Xmx4g
  -XX:+UseParallelGC
  -XX:ParallelGCThreads=16
  -XX:GCTimeRatio=199
  -Xlog:gc:gc.log

효과:
  GC 시간 0.5% 이하
  처리량 최대화
  
  Full GC 시간 길어도 OK
  → 전체 처리 시간 단축
```

---

### 시나리오 3: 대용량 힙 (100GB+)

```
요구사항:
  - 100GB 힙
  - Pause < 10ms
  - 실시간 거래

튜닝:
  -Xms100g -Xmx100g
  -XX:+UseZGC
  -XX:ConcGCThreads=4
  -XX:+UseLargePages
  -Xlog:gc*:gc.log

모니터링:
  Pause Time < 10ms 확인
  Throughput 5% 감소 허용
  
  Full GC 발생 시:
  힙 증가 (120g)
  ConcGCThreads 증가 (6)
```

---

## 🚫 흔한 오해

### "플래그를 많이 쓸수록 좋다"

```
❌ 잘못된 이해:
  튜닝 플래그를 최대한 많이 설정한다.

✅ 실제:
  기본값이 대부분 적절
  
  필수만 설정:
  - 힙 크기 (Xms, Xmx)
  - GC 선택 (UseG1GC 등)
  - GC 로그
  
  나머지: 문제 발생 시만
  
  과도한 튜닝:
  - JVM 최적화 방해
  - 업그레이드 시 문제
  - 유지보수 어려움
```

---

## 📌 핵심 정리

```
필수 플래그
  힙: -Xms -Xmx (같게)
  GC: -XX:+UseG1GC (또는 다른 GC)
  로그: -Xlog:gc:gc.log

힙 크기
  고정: Xms = Xmx
  Young: -Xmn 또는 NewRatio
  Metaspace: MaxMetaspaceSize

GC 선택
  Throughput: Parallel
  Balanced: G1
  Low Latency: ZGC/Shenandoah

스레드 수
  ParallelGCThreads: CPU 코어 수
  ConcGCThreads: ParallelGCThreads / 4

Pause 목표
  G1: MaxGCPauseMillis (기본 200ms)
  ZGC: 항상 < 10ms

G1 튜닝
  G1HeapRegionSize
  InitiatingHeapOccupancyPercent
  G1MixedGCCountTarget

ZGC 튜닝
  ConcGCThreads
  UseLargePages
  ZUncommitDelay

튜닝 순서
  1. 기본 설정으로 시작
  2. GC 로그 분석
  3. 병목 찾기
  4. 필수 플래그만 조정
  5. 효과 측정
  6. 반복
```

---

## 🤔 생각해볼 문제

**Q1.** 웹 서버에서 `-Xms4g -Xmx8g`로 설정했더니 GC Pause가 불규칙하다. 왜 `Xms = Xmx`로 고정하는 것이 권장되는가?

**Q2.** G1 GC에서 `-XX:MaxGCPauseMillis=50`으로 설정했더니 목표를 자주 초과한다. 어떤 튜닝 옵션을 조정해야 하는가? 3가지 방법을 제시하라.

**Q3.** 다음 두 설정 중 어느 것이 더 나은가? Throughput과 Pause Time을 고려하라.
- 설정 A: `-XX:ParallelGCThreads=16 -XX:ConcGCThreads=4`
- 설정 B: `-XX:ParallelGCThreads=8 -XX:ConcGCThreads=8`
(8 코어 서버 기준)

> 💡 **해설**
>
> **Q1.** Xms < Xmx 문제점: ① 힙 크기가 동적으로 변함 (4GB ↔ 8GB) → GC가 힙 확장/축소 판단 → 예측 불가능한 Pause. ② 확장 시점에 Full GC 발생 가능 (힙 재구성) → 수 초 STW. ③ OS 메모리 할당/해제 오버헤드. Xms = Xmx 장점: ① 힙 크기 고정 → GC 동작 일정. ② 확장/축소 없음 → Pause 예측 가능. ③ JVM 시작 시 메모리 한 번에 할당 → 런타임 오버헤드 없음. 권장: 항상 Xms = Xmx.
>
> **Q2.** MaxGCPauseMillis=50 달성 방법: ① 힙 크기 증가 (`-Xmx8g` → `-Xmx12g`) — 여유 공간 확보 → GC 빈도 감소 → Pause 단축. ② G1HeapRegionSize 조정 (`-XX:G1HeapRegionSize=4m`) — 작은 Region → Evacuation 시간 단축. ③ G1MixedGCCountTarget 증가 (`-XX:G1MixedGCCountTarget=16`) — Mixed GC를 16번에 나눠 수행 → 한 번당 Pause 감소. ④ Young Gen 축소 (간접적) — MaxGCPauseMillis가 낮으면 G1이 자동으로 Young 줄임. 주의: 50ms는 매우 낮음 → 달성 어려울 수 있음 → ZGC 고려.
>
> **Q3.** 설정 A가 더 나음. 이유: ParallelGCThreads — STW 단계 (Young GC, Remark 등) 병렬 처리 → 8 코어에서 16은 과도 (Context Switching 증가). 8이 적정. ConcGCThreads — Concurrent 단계 (Marking, Cleanup) → 애플리케이션과 공존 → 4개면 충분 (CPU 절반 양보). 설정 A: Parallel 16 (과도), Concurrent 4 (적정) → 혼합. 설정 B: Parallel 8 (적정), Concurrent 8 (과도) → Concurrent 단계에서 CPU 독점 → 애플리케이션 느려짐. 최적: `-XX:ParallelGCThreads=8 -XX:ConcGCThreads=2` (또는 기본값 사용).

---

## 📚 참고 자료

- [JVM Options Explorer](https://chriswhocodes.com/)
- [GC Tuning Guide](https://docs.oracle.com/en/java/javase/17/gctuning/)
- [GC Easy - GC Log Analyzer](https://gceasy.io/)

---

<div align="center">

**[⬅️ 이전: Shenandoah GC](./09-shenandoah-gc.md)** | **[다음: GC Log Analysis ➡️](./11-gc-log-analysis.md)**

</div>

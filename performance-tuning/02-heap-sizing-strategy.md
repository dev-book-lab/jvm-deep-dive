# Heap Sizing Strategy - 힙 크기 결정 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 적절한 힙 크기는 어떻게 결정하는가?
- Young/Old Generation 비율은 어떻게 설정하는가?
- 컨테이너 환경에서 힙 크기 주의사항은?
- 힙이 너무 크거나 작을 때 문제는 무엇인가?

---

## 🔍 왜 이게 존재하는가

### 문제: 얼마나 큰 힙이 필요한가?

```
너무 작은 힙:
  빈번한 GC
  OutOfMemoryError

너무 큰 힙:
  긴 GC Pause
  메모리 낭비

적절한 힙:
  GC 빈도 적절
  Pause Time 수용 가능
```

힙 크기는 **성능과 안정성의 균형점**이다.

---

## 📐 힙 크기 결정 방법론

### 1. 데이터 기반 결정

```
방법:
  1. 기본 힙으로 시작 (예: 2GB)
  2. 실제 부하 테스트
  3. GC 로그 분석
  4. 힙 크기 조정
  5. 반복

측정 지표:
  - GC 빈도 (Minor/Major)
  - GC Pause Time (Avg/Max/P99)
  - Heap 사용률 (After GC)
  - Throughput (GC 시간 비율)

목표:
  - Minor GC: 1분당 10~30회
  - Major GC: 시간당 1회 미만
  - Pause: < 목표 시간 (예: 100ms)
  - After GC: 힙의 30~70% 사용
```

---

### 2. 워크로드 기반 산정

```
공식:
  Heap = Live Data × Factor

Live Data:
  Full GC 후 남은 객체 크기
  
Factor:
  3~4배 (일반적)
  5~6배 (안전)

예:
  Live Data = 2GB
  Factor = 4
  → Heap = 8GB

추가 고려:
  + Metaspace (~256MB)
  + Native Memory
  + OS 예약
  = 총 메모리
```

---

### 3. Young/Old 비율

```
기본 비율:
  Young : Old = 1 : 2
  (전체 힙의 1/3 Young)

조정 기준:
  객체 수명 짧음 (웹 서버)
  → Young 크게 (1:1.5)
  → Minor GC 빈도 감소
  
  객체 수명 김 (캐시)
  → Old 크게 (1:3)
  → Promotion 줄이기

설정:
  -Xmn<size>  (Young 고정)
  -XX:NewRatio=2  (Old/Young = 2)
```

---

## 💻 실전 힙 크기 결정

### 시나리오 1: 웹 서버

```
요구사항:
  동시 사용자: 1000명
  Request당 메모리: ~100KB
  응답 시간: < 100ms

계산:
  Active Request: 1000 × 100KB = 100MB
  Buffer: 3배 = 300MB
  Live Data: ~500MB

힙 크기:
  500MB × 4 = 2GB (최소)
  안전: 4GB

설정:
  -Xms4g -Xmx4g
  -Xmn1g (Young 1GB)
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=100
```

---

### 시나리오 2: 배치 작업

```
요구사항:
  데이터 처리량: 10GB
  메모리 처리: 일괄 로드
  응답 시간: 무관

계산:
  일시 메모리: ~3GB
  Live Data: ~1GB
  
힙 크기:
  3GB × 2 = 6GB (최소)
  안전: 8~12GB

설정:
  -Xms12g -Xmx12g
  -Xmn4g (Young 크게)
  -XX:+UseParallelGC
```

---

### 시나리오 3: 컨테이너

```
컨테이너 제약:
  Memory Limit: 2GB
  CPU: 2 core

힙 크기 계산:
  Container: 2GB
  OS: ~200MB
  Native: ~200MB
  Heap: ~1.6GB (80%)

설정:
  -XX:MaxRAMPercentage=75.0
  (2GB × 75% = 1.5GB)
  
  또는
  -Xms1500m -Xmx1500m
  -XX:+UseContainerSupport
```

---

## 🔧 힙 크기 검증

### GC 로그 분석

```bash
# GC 로그 수집
-Xlog:gc*:file=gc.log:time,uptime

# 분석 (gceasy.io 또는 수동)
grep "Pause" gc.log | awk '{print $NF}' | sort -n

# 지표 확인:
1. GC 빈도
   Minor GC: 10초~1분 간격
   Major GC: 시간당 < 1회

2. Pause Time
   Avg: < 목표 (예: 50ms)
   P99: < 목표 × 2 (예: 100ms)

3. Heap 사용률 (After GC)
   30~70% → 적절
   < 30% → 힙 너무 큼
   > 70% → 힙 부족
```

---

### 조정 판단

```
힙 증가 신호:
  - Major GC 빈번 (시간당 > 2회)
  - After GC > 70%
  - OutOfMemoryError 발생
  - Promotion Rate 높음

힙 감소 고려:
  - After GC < 30%
  - Major GC 드물 (하루 < 1회)
  - GC Pause 길음 (> 1초)
  - 메모리 낭비 심각

Young 증가:
  - Minor GC 빈번 (초당 > 2회)
  - Promotion Rate 높음

Old 증가:
  - Major GC 빈번
  - Old Gen 압박
```

---

## ⚠️ 컨테이너 환경 주의사항

### Kubernetes/Docker 설정

```yaml
# Kubernetes Pod
resources:
  limits:
    memory: "2Gi"
    cpu: "2"
  requests:
    memory: "2Gi"
    cpu: "2"
```

```bash
# JVM 설정 (Java 10+)
-XX:+UseContainerSupport  (기본 활성화)
-XX:MaxRAMPercentage=75.0

# 또는 명시적
-Xms1500m -Xmx1500m

주의:
  Xmx > Container Limit
  → OOM Killer (컨테이너 종료)
```

---

### Container-Aware JVM

```bash
# Java 8u191+ / Java 10+
-XX:+UseContainerSupport

기능:
  CPU Limit 인식
  Memory Limit 인식
  → GC 스레드 자동 조정

확인:
  docker run -m 2g openjdk:17 \
    java -XX:+PrintFlagsFinal -version | grep MaxHeapSize
  
  # 출력: MaxHeapSize = ~1.5GB (75%)
```

---

## 🚫 흔한 실수

### 실수 1: Xms ≠ Xmx

```bash
# ❌ 동적 힙
-Xms512m -Xmx4g

문제:
  힙 확장 시 Full GC
  GC 동작 불안정
  성능 예측 어려움

# ✅ 고정 힙
-Xms4g -Xmx4g
```

---

### 실수 2: 컨테이너 Limit 무시

```bash
# Container: 2GB
# JVM: -Xmx3g

결과:
  JVM 메모리 > Container Limit
  → OOM Killer
  → Pod 재시작

해결:
  -Xmx1500m (75% of 2GB)
  또는
  -XX:MaxRAMPercentage=75.0
```

---

### 실수 3: 과도하게 큰 힙

```bash
# 256GB RAM 서버
# JVM: -Xmx200g

문제:
  Full GC 시 수 분 Pause
  → 애플리케이션 정지
  
해결:
  적절한 크기 (8~32GB)
  또는 ZGC 사용
```

---

## 📌 핵심 정리

```
힙 크기 공식
  Heap = Live Data × 3~4배

측정 지표
  Live Data: Full GC 후 크기
  GC 빈도: Minor 10~30/분, Major < 1/시간
  Pause Time: P99 < 목표
  After GC: 30~70% 사용

Young/Old 비율
  기본: 1:2 (Young 1/3)
  웹 서버: 1:1.5 (Young 크게)
  캐시: 1:3 (Old 크게)

컨테이너
  MaxRAMPercentage=75.0
  UseContainerSupport (기본)
  Xmx < Container Limit

검증
  GC 로그 분석
  After GC 사용률
  Pause Time 분포

조정
  After GC > 70% → 힙 증가
  After GC < 30% → 힙 감소
  Minor GC 빈번 → Young 증가
  Major GC 빈번 → Old 증가

Best Practice
  Xms = Xmx (필수)
  데이터 기반 결정
  점진적 조정
  모니터링 지속
```

---

## 🤔 생각해볼 문제

**Q1.** Full GC 후 힙 사용량이 3GB인 애플리케이션의 적절한 힙 크기를 계산하라. (안전 여유 고려)

**Q2.** 컨테이너 Memory Limit이 4GB일 때, 적절한 JVM 힙 크기와 설정 플래그를 제시하라.

**Q3.** GC 로그에서 "After GC: 85%"가 지속적으로 나타난다. 어떤 조치를 취해야 하는가?

> 💡 **해설**
>
> **Q1.** 힙 크기 계산: Live Data = 3GB. 공식: Heap = Live Data × Factor. 일반적 Factor = 4 → 3GB × 4 = 12GB. 안전 여유 Factor = 5 → 3GB × 5 = 15GB. 권장: `-Xms12g -Xmx12g` (또는 안전하게 16GB). 이유: 12GB면 After GC ~25% (3/12), 15GB면 ~20% — 둘 다 적절 (30~70% 범위).
>
> **Q2.** 컨테이너 4GB 힙 설정: ① MaxRAMPercentage 사용: `-XX:MaxRAMPercentage=75.0 -XX:+UseContainerSupport` → 4GB × 75% = 3GB 힙. ② 명시적 설정: `-Xms3g -Xmx3g -XX:+UseContainerSupport`. 이유: Container 4GB - OS/Native (1GB) = 힙 3GB. 75%가 안전 (OOM Killer 회피). UseContainerSupport로 CPU/Memory Limit 인식.
>
> **Q3.** "After GC: 85%" 조치: ① 힙 증가 — 현재 After GC > 70% (압박) → 힙 부족 신호. Live Data = 현재 힙 × 0.85. 목표 After GC = 50% → 필요 힙 = Live Data / 0.5 = 현재 힙 × 1.7. 예: 10GB 힙, After GC 85% → Live 8.5GB → 필요 힙 17GB. ② Young 증가 — Promotion Rate 확인, 높으면 Young 증가로 Old 압박 완화. ③ 메모리 누수 확인 — Live Data가 계속 증가하면 Heap Dump 분석.

---

## 📚 참고 자료

- [GC Tuning Guide](https://docs.oracle.com/en/java/javase/17/gctuning/)
- [Container Support](https://www.eclipse.org/openj9/docs/xcontainersupport/)

---

<div align="center">

**[⬅️ 이전: JVM Flags Complete Guide](./01-jvm-flags-complete-guide.md)** | **[다음: GC Ergonomics ➡️](./03-gc-ergonomics.md)**

</div>

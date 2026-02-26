# JVM Flags Complete Guide - JVM 플래그 완전 가이드

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JVM 플래그의 종류와 구분은 무엇인가?
- 필수 플래그와 선택 플래그는 무엇인가?
- 실전에서 가장 많이 사용하는 플래그는?
- 플래그 조합의 Best Practice는?

---

## 🔍 왜 이게 존재하는가

### 문제: JVM 기본 설정은 범용적이다

```
기본 설정:
  모든 워크로드에 적합하도록 타협
  → 최적은 아님

실제 요구사항:
  - 웹 서버: Low Latency
  - 배치: High Throughput
  - 컨테이너: 리소스 제약
  
  → 각각 다른 플래그 필요
```

JVM 플래그는 **워크로드별 최적화**의 핵심이다.

---

## 📐 플래그 분류

### 1. 플래그 타입

```
Standard Flags (-):
  -cp, -classpath
  -Xms, -Xmx
  모든 JVM 구현에서 지원

Non-Standard Flags (-X):
  -Xms, -Xmx
  -Xss
  HotSpot 전용, 호환성 보장

Experimental Flags (-XX:):
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  HotSpot 전용, 변경 가능성

Diagnostic Flags (-XX:+UnlockDiagnosticVMOptions):
  -XX:+PrintAssembly
  디버깅/분석용
```

---

### 2. 필수 플래그 (모든 환경)

```bash
# 힙 크기 (필수)
-Xms4g -Xmx4g
  초기/최대 힙 같게 (고정)
  → 힙 크기 변동 없음
  → GC 예측 안정

# GC 로그 (필수)
-Xlog:gc*:file=/var/log/gc.log:time,uptime,level,tags
  GC 로그 파일 생성
  → 문제 분석 필수

# Out-Of-Memory 시 행동
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heapdump.hprof
  OOM 시 Heap Dump 생성
  → 메모리 누수 분석

# Error 파일
-XX:ErrorFile=/var/log/hs_err_pid%p.log
  JVM Crash 시 로그
```

---

### 3. GC 선택 플래그

```bash
# G1 GC (기본, Java 9+)
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=16m

# ZGC (Ultra Low Latency)
-XX:+UseZGC
-XX:ConcGCThreads=4

# Parallel GC (High Throughput)
-XX:+UseParallelGC
-XX:ParallelGCThreads=8

# Serial GC (작은 힙)
-XX:+UseSerialGC
```

---

### 4. 힙 튜닝 플래그

```bash
# Young Generation
-Xmn2g
  Young Gen 크기 고정

-XX:NewRatio=2
  Old/Young 비율 (Old=2, Young=1)

-XX:SurvivorRatio=8
  Eden/Survivor 비율 (Eden=8, Survivor=1+1)

# Metaspace
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
  클래스 메타데이터 크기
```

---

### 5. GC 세부 튜닝

```bash
# G1 GC
-XX:InitiatingHeapOccupancyPercent=45
  Old 45% 시 Marking 시작

-XX:G1MixedGCCountTarget=8
  Mixed GC 분할 횟수

-XX:G1MixedGCLiveThresholdPercent=85
  Live > 85% Region 제외

# Parallel GC
-XX:GCTimeRatio=99
  GC 시간 1% 목표 (99:1)

-XX:MaxGCPauseMillis=200
  Pause 목표 (보장 아님)

# 공통
-XX:ParallelGCThreads=8
  Parallel GC 스레드 수

-XX:ConcGCThreads=2
  Concurrent GC 스레드 수
```

---

### 6. JIT 컴파일러 플래그

```bash
# Tiered Compilation (기본 활성화)
-XX:+TieredCompilation
  C1 (빠른 컴파일) + C2 (최적화)

# Compilation Threshold
-XX:CompileThreshold=10000
  메서드 호출 10000회 후 컴파일

# Code Cache
-XX:ReservedCodeCacheSize=256m
  컴파일된 코드 저장 공간

# Inline
-XX:MaxInlineSize=35
  인라인 가능한 메서드 크기

-XX:FreqInlineSize=325
  자주 호출되는 메서드 인라인 크기
```

---

### 7. 디버깅/프로파일링 플래그

```bash
# JFR (Java Flight Recorder)
-XX:StartFlightRecording=duration=60s,filename=recording.jfr
  60초 프로파일링

# GC 상세 로그
-Xlog:gc*=debug:file=gc_debug.log

# Class Loading
-Xlog:class+load:file=class_load.log

# JIT Compilation
-XX:+PrintCompilation
-XX:+LogCompilation

# Assembly 출력 (진단)
-XX:+UnlockDiagnosticVMOptions
-XX:+PrintAssembly
-XX:+PrintInlining
```

---

## 💻 실전 플래그 조합

### 조합 1: 웹 서버 (Low Latency)

```bash
java -Xms8g -Xmx8g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=100 \
     -XX:ParallelGCThreads=8 \
     -XX:ConcGCThreads=2 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/heapdump.hprof \
     -Xlog:gc*:file=/var/log/gc.log:time,uptime \
     -jar app.jar

포인트:
  - G1 GC (Pause < 100ms)
  - 힙 고정 (8GB)
  - GC 로그/Heap Dump
```

---

### 조합 2: 배치 작업 (High Throughput)

```bash
java -Xms16g -Xmx16g \
     -XX:+UseParallelGC \
     -XX:ParallelGCThreads=16 \
     -XX:GCTimeRatio=199 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -Xlog:gc:file=/var/log/gc.log \
     -jar batch.jar

포인트:
  - Parallel GC (Throughput 최대)
  - GC 시간 0.5% 목표
  - Pause Time 무관
```

---

### 조합 3: 컨테이너 (리소스 제약)

```bash
java -Xms512m -Xmx512m \
     -XX:+UseSerialGC \
     -XX:MaxRAMPercentage=75.0 \
     -XX:+UseContainerSupport \
     -XX:+HeapDumpOnOutOfMemoryError \
     -Xlog:gc:file=/var/log/gc.log \
     -jar app.jar

포인트:
  - Serial GC (작은 힙)
  - Container 인식 (CPU/Memory)
  - MaxRAMPercentage (컨테이너 제한의 75%)
```

---

### 조합 4: 초저지연 (Ultra Low Latency)

```bash
java -Xms32g -Xmx32g \
     -XX:+UseZGC \
     -XX:ConcGCThreads=4 \
     -XX:+UseLargePages \
     -XX:+HeapDumpOnOutOfMemoryError \
     -Xlog:gc*:file=/var/log/gc.log \
     -jar trading.jar

포인트:
  - ZGC (Pause < 10ms)
  - Large Pages (성능 향상)
  - 큰 힙 (32GB+)
```

---

## 🔧 플래그 확인 방법

### 현재 JVM 플래그 확인

```bash
# 실행 중인 JVM 플래그
jcmd <pid> VM.flags

# 모든 플래그 (기본값 포함)
java -XX:+PrintFlagsFinal -version | grep G1

# 특정 플래그 확인
java -XX:+PrintFlagsFinal -version | grep MaxHeapSize

# Ergonomics (자동 설정)
java -XX:+PrintFlagsFinal -version | grep ergonomic
```

---

### 플래그 충돌 확인

```bash
# 충돌하는 플래그
java -XX:+UseG1GC -XX:+UseZGC -version
# WARNING: Option UseG1GC was deprecated...

# 유효하지 않은 플래그
java -XX:InvalidFlag=123 -version
# Error: Could not create the Java Virtual Machine.
```

---

## 🚫 흔한 실수

### 실수 1: Xms ≠ Xmx

```bash
# ❌ 나쁜 예
-Xms1g -Xmx8g

문제:
  힙 크기 변동 (1GB ↔ 8GB)
  → GC 동작 예측 불가
  → 확장 시 Full GC 가능

# ✅ 좋은 예
-Xms8g -Xmx8g

장점:
  힙 고정 → GC 안정
```

---

### 실수 2: GC 중복 설정

```bash
# ❌ 나쁜 예
-XX:+UseG1GC -XX:+UseZGC

결과:
  마지막 플래그만 적용 (ZGC)
  혼란 초래

# ✅ 좋은 예
-XX:+UseG1GC
```

---

### 실수 3: 과도한 튜닝

```bash
# ❌ 나쁜 예
-XX:+UseG1GC \
-XX:MaxGCPauseMillis=50 \
-XX:G1HeapRegionSize=8m \
-XX:InitiatingHeapOccupancyPercent=35 \
-XX:G1MixedGCCountTarget=16 \
-XX:G1MixedGCLiveThresholdPercent=80 \
...30개 플래그

문제:
  JVM 자동 최적화 방해
  복잡도 증가
  유지보수 어려움

# ✅ 좋은 예
-Xms8g -Xmx8g \
-XX:+UseG1GC \
-XX:MaxGCPauseMillis=100

원칙:
  필수만 설정
  나머지는 JVM에 맡김
```

---

## 📌 핵심 정리

```
필수 플래그
  -Xms, -Xmx (힙 크기, 같게)
  -Xlog:gc (GC 로그)
  -XX:+HeapDumpOnOutOfMemoryError

GC 선택
  G1: 균형 (기본)
  ZGC: 초저지연
  Parallel: Throughput
  Serial: 작은 힙

GC 튜닝
  MaxGCPauseMillis (G1, Parallel)
  ParallelGCThreads (병렬도)
  InitiatingHeapOccupancyPercent (G1)

JIT 튜닝
  TieredCompilation (기본)
  ReservedCodeCacheSize
  CompileThreshold

디버깅
  -XX:+PrintFlagsFinal
  -XX:StartFlightRecording
  -XX:+PrintCompilation

실전 조합
  웹 서버: G1 + MaxGCPauseMillis=100
  배치: Parallel + GCTimeRatio=199
  컨테이너: Serial + MaxRAMPercentage
  초저지연: ZGC + UseLargePages

Best Practice
  Xms = Xmx (필수)
  필수 플래그만 설정
  GC 로그 활성화
  OOM Heap Dump
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 플래그 조합에서 문제를 찾고, 올바르게 수정하라.

```bash
java -Xms2g -Xmx8g \
     -XX:+UseG1GC \
     -XX:+UseZGC \
     -XX:MaxGCPauseMillis=10 \
     -jar app.jar
```

**Q2.** 컨테이너 환경 (1 CPU, 1GB RAM)에 적합한 JVM 플래그 조합을 제시하라.

**Q3.** `-XX:+PrintFlagsFinal`의 출력에서 "ergonomic"이 표시된 플래그는 무엇을 의미하는가?

> 💡 **해설**
>
> **Q1.** 문제점: ① Xms(2g) ≠ Xmx(8g) — 힙 크기 변동 → GC 불안정. ② UseG1GC + UseZGC 중복 — 마지막(ZGC)만 적용, 혼란. ③ MaxGCPauseMillis=10 — 너무 낮음, ZGC는 이 플래그 무시. 올바른 조합: `java -Xms8g -Xmx8g -XX:+UseZGC -jar app.jar` (ZGC는 자동으로 < 10ms 보장).
>
> **Q2.** 컨테이너 플래그: `java -Xms768m -Xmx768m -XX:+UseSerialGC -XX:MaxRAMPercentage=75.0 -XX:+UseContainerSupport -XX:+HeapDumpOnOutOfMemoryError -Xlog:gc:file=/var/log/gc.log -jar app.jar`. 이유: ① Serial GC (1 CPU, 작은 힙에 적합). ② MaxRAMPercentage=75% (1GB의 75% = 768MB). ③ UseContainerSupport (컨테이너 제한 인식). ④ 힙 고정 (768MB). ⑤ GC 로그 + OOM Dump.
>
> **Q3.** "ergonomic" 플래그: JVM이 자동으로 설정한 값. 예: `-XX:ParallelGCThreads=8 {ergonomic}` — 사용자가 명시 안 함, JVM이 CPU 코어 수(8)에 따라 자동 설정. 확인: `java -XX:+PrintFlagsFinal -version | grep ergonomic`. 의미: JVM Ergonomics (자동 튜닝)가 동작 중. 사용자가 명시하면 `{product}` 또는 `{manageable}` 표시.

---

## 📚 참고 자료

- [JVM Options Explorer](https://chriswhocodes.com/)
- [HotSpot VM Options](https://www.oracle.com/java/technologies/javase/vmoptions-jsp.html)
- [G1 GC Tuning Guide](https://docs.oracle.com/en/java/javase/17/gctuning/)

---

<div align="center">

**[다음: Heap Sizing Strategy ➡️](./02-heap-sizing-strategy.md)**

</div>

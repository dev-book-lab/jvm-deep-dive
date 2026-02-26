# Profiling with async-profiler - async-profiler 프로파일링

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- async-profiler는 무엇이며, JFR과 어떻게 다른가?
- CPU, 메모리, Lock 프로파일링은 어떻게 하는가?
- alloc 모드로 GC 압박을 어떻게 찾는가?
- Flame Graph를 어떻게 해석하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: JFR의 한계

```
JFR 장점:
  - 낮은 오버헤드
  - 프로덕션 사용 가능

JFR 한계:
  - SafePoint Bias (부정확)
  - Native 코드 프로파일링 제한
  - 일부 Lock 정보 누락

async-profiler:
  - SafePoint Bias 없음
  - Native 코드 포함
  - 더 정확한 프로파일링
```

async-profiler는 **정확한 저수준 프로파일러**다.

---

## 📐 async-profiler 개요

### 1. 특징

```
기술:
  - AsyncGetCallTrace (HotSpot API)
  - 비동기 샘플링
  - SafePoint 불필요

장점:
  - 정확한 CPU 프로파일링
  - Native 코드 포함
  - 낮은 오버헤드 (< 5%)

모드:
  - cpu: CPU 프로파일링
  - alloc: 메모리 할당
  - lock: Lock Contention
  - wall: Wall-clock Time

출력:
  - Flame Graph (HTML)
  - JFR 포맷
  - Collapsed Stack
```

---

### 2. 설치

```bash
# 다운로드
wget https://github.com/async-profiler/async-profiler/releases/download/v2.9/async-profiler-2.9-linux-x64.tar.gz

tar -xzf async-profiler-2.9-linux-x64.tar.gz
cd async-profiler-2.9-linux-x64

# 권한 설정 (필요시)
sudo sysctl kernel.perf_event_paranoid=1
sudo sysctl kernel.kptr_restrict=0
```

---

### 3. 기본 사용법

```bash
# CPU 프로파일링 (60초)
./profiler.sh -d 60 -f flamegraph.html <pid>

# 메모리 할당 프로파일링
./profiler.sh -d 60 -e alloc -f alloc.html <pid>

# Lock 프로파일링
./profiler.sh -d 60 -e lock -f lock.html <pid>

# 시작/중지 분리
./profiler.sh start <pid>
./profiler.sh stop -f output.html <pid>
```

---

## 💻 프로파일링 모드

### 1. CPU 프로파일링

```bash
# CPU 샘플링
./profiler.sh -d 60 -e cpu -f cpu.html <pid>

출력:
  Flame Graph
  - X축: CPU 시간 비율
  - Y축: 호출 스택
  - 넓은 블록 = Hot Spot

분석:
  1. 가장 넓은 블록 찾기
  2. 호출 경로 추적
  3. 최적화 대상 결정

예:
  MyService.process() (60%)
    ↓
  JSON.parse() (50%)
    ↓
  최적화: JSON 파서 교체
```

---

### 2. 메모리 할당 프로파일링 (alloc)

```bash
# Allocation 프로파일링
./profiler.sh -d 60 -e alloc -f alloc.html <pid>

출력:
  할당이 많은 경로 표시

분석:
  1. 가장 많이 할당하는 메서드
  2. 할당 타입 (byte[], String, ...)
  3. 불필요한 할당 찾기

예:
  ┌─────────────────────────┐
  │ handleRequest()         │
  ├─────────────┬───────────┤
  │toByteArray()│ other...  │
  │   (80%)     │  (20%)    │
  └─────────────┴───────────┘
  
  → byte[] 과다 할당
  → 버퍼 재사용으로 개선
```

---

### 3. Lock 프로파일링

```bash
# Lock Contention
./profiler.sh -d 60 -e lock -f lock.html <pid>

출력:
  Lock 대기 시간 표시

분석:
  1. 대기 시간 많은 Lock
  2. Lock 위치 (코드)
  3. 경쟁 패턴

예:
  synchronized(cache) {
    ...
  }
  
  대기 시간: 40%
  → ConcurrentHashMap 교체
```

---

### 4. Wall-clock Time

```bash
# Wall-clock (실시간)
./profiler.sh -d 60 -e wall -f wall.html <pid>

차이:
  CPU: CPU 실행 시간만
  Wall: 전체 시간 (I/O 대기 포함)

용도:
  I/O Bound 애플리케이션
  → CPU는 낮지만 느린 경우
```

---

## 🔥 Flame Graph 분석

### 심화 해석

```
Flame Graph 색상:
  - Java: 녹색
  - JVM (GC, JIT): 노랑
  - Kernel: 빨강
  - Native Library: 파랑

패턴:
  1. Flat Top (평평한 상단)
     → CPU 집중 메서드
     → 최적화 대상
  
  2. Tower (타워형)
     → 깊은 호출 스택
     → 재귀 또는 복잡한 로직
  
  3. Wide Base (넓은 하단)
     → 다양한 코드 경로
     → 균등한 CPU 분산

검색:
  Ctrl+F → 메서드명 입력
  → 해당 메서드 하이라이트
```

---

## ⚡ 실무 사례

### 사례 1: JSON 파싱 병목

```bash
# CPU 프로파일링
./profiler.sh -d 60 -e cpu -f cpu.html <pid>

발견:
  Jackson.parse() : 45% CPU
  
상세 분석:
  ┌─────────────────────────┐
  │ RestController.handle() │
  ├─────────────────────────┤
  │ ObjectMapper.readValue()│ 45%
  ├──────────┬──────────────┤
  │ parse()  │ deserialize()│
  └──────────┴──────────────┘

해결:
  1. Jackson → fastjson (벤치마크)
  2. 결과 캐싱 (동일 요청)
  
효과:
  45% CPU → 15% CPU
  Throughput 2배 증가
```

---

### 사례 2: 메모리 할당 압박

```bash
# Allocation 프로파일링
./profiler.sh -d 60 -e alloc -f alloc.html <pid>

발견:
  byte[] : 2GB/s
  String : 500MB/s

추적:
  ┌──────────────────────────┐
  │ processData()            │
  ├──────────────────────────┤
  │ String.getBytes()        │ 80%
  │   → new byte[]           │
  └──────────────────────────┘

문제:
  반복적으로 byte[] 생성
  → GC 압박

해결:
  ByteBuffer 재사용
  → Pooling

효과:
  GC Pause 50ms → 10ms
  Allocation Rate 80% 감소
```

---

### 사례 3: Lock Contention

```bash
# Lock 프로파일링
./profiler.sh -d 60 -e lock -f lock.html <pid>

발견:
  ConcurrentHashMap.put() : 30% Wait

상세:
  Segment Lock 경쟁
  (Java 7 ConcurrentHashMap)

해결:
  Java 8+ ConcurrentHashMap
  (CAS 기반, 더 적은 Lock)

효과:
  Throughput 50% 증가
```

---

## 🔧 고급 옵션

### 필터링

```bash
# 특정 패키지만
./profiler.sh -d 60 \
  --include 'com/myapp/*' \
  -f filtered.html <pid>

# 특정 패키지 제외
./profiler.sh -d 60 \
  --exclude 'java/*,sun/*' \
  -f app_only.html <pid>

# Java 메서드만 (Native 제외)
./profiler.sh -d 60 \
  --jfrsync=default \
  -f java_only.html <pid>
```

---

### JFR 호환 모드

```bash
# JFR 포맷 출력
./profiler.sh -d 60 -o jfr -f output.jfr <pid>

# JMC로 열기
jmc output.jfr
```

---

## 🚫 흔한 실수

### "CPU 100%인데 Flame Graph가 비어있다"

```
❌ 원인:
  I/O 대기, Lock 대기
  → CPU 사용 안 함
  
✅ 해결:
  wall 모드 사용
  
  ./profiler.sh -e wall -f wall.html <pid>
  
  또는 lock 모드
  ./profiler.sh -e lock -f lock.html <pid>
```

---

## 📌 핵심 정리

```
async-profiler
  비동기 샘플링 프로파일러
  SafePoint Bias 없음
  Native 코드 포함

모드
  cpu: CPU 샘플링
  alloc: 메모리 할당
  lock: Lock Contention
  wall: Wall-clock Time

사용법
  ./profiler.sh -d 60 -e cpu -f output.html <pid>

Flame Graph
  X축: 시간 비율
  Y축: 호출 스택
  넓은 블록 = Hot Spot

분석 순서
  1. Flame Graph 확인
  2. Hot Spot 찾기
  3. 호출 경로 추적
  4. 최적화

실무 활용
  CPU: 병목 메서드
  alloc: GC 압박 원인
  lock: 동시성 문제

Best Practice
  프로덕션: cpu 모드 (5% 오버헤드)
  개발: alloc 모드로 최적화
  문제 시: wall/lock 모드
```

---

## 🤔 생각해볼 문제

**Q1.** CPU 프로파일링에서 GC 관련 메서드가 30%를 차지한다. alloc 모드로 어떻게 원인을 찾는가?

**Q2.** async-profiler의 wall 모드와 cpu 모드의 차이를 설명하고, 각각 언제 사용하는가?

**Q3.** Flame Graph에서 "flat top" 패턴을 발견했다. 이것이 의미하는 것과 최적화 방법을 설명하라.

> 💡 **해설**
>
> **Q1.** GC 압박 원인 찾기: ① CPU 프로파일링 — GC 30% → 메모리 할당 과다 의심. ② alloc 모드 실행: `./profiler.sh -e alloc -f alloc.html <pid>`. ③ Flame Graph 확인 — 가장 넓은 블록 = 할당 많은 메서드. 예: `processRequest() → byte[] allocation (80%)`. ④ 해당 메서드로 이동 → 코드 분석. ⑤ 해결: byte[] 재사용 (ByteBuffer Pool), String 최적화 (StringBuilder). ⑥ 재측정 — alloc 감소 → GC CPU 감소.
>
> **Q2.** wall vs cpu: ① cpu 모드 — CPU 실행 시간만 측정 (I/O 대기 제외). 스레드가 Running 상태일 때만 샘플링. ② wall 모드 — 전체 시간 측정 (I/O 대기, Lock 대기 포함). 실제 경과 시간. ③ 사용: CPU-bound (계산 집약) → cpu 모드. I/O-bound (네트워크, 디스크) → wall 모드. Lock 대기 의심 → lock 모드 또는 wall 모드. ④ 예: CPU 20%인데 느림 → wall 모드로 I/O 대기 확인.
>
> **Q3.** Flat Top 패턴: ① 의미 — 호출 스택이 해당 메서드에서 끝남 (더 이상 하위 호출 없음) → CPU 집중 메서드. ② 예: `compute()` 메서드가 복잡한 계산 수행 → 하위 호출 없이 CPU 소비. ③ 최적화 방법: 알고리즘 개선 (O(n²) → O(n log n)), 불필요한 연산 제거, 캐싱 (결과 재사용), 병렬화 (멀티스레드), Native 라이브러리 사용 (JNI). ④ 확인: 메서드 내부 코드 프로파일링 (라인별).

---

## 📚 참고 자료

- [async-profiler GitHub](https://github.com/async-profiler/async-profiler)
- [Profiling Java Applications](https://www.brendangregg.com/blog/2014-06-12/java-flame-graphs.html)

---

<div align="center">

**[⬅️ 이전: Profiling with JFR](./04-profiling-with-jfr.md)** | **[다음: Memory Leak Analysis ➡️](./06-memory-leak-analysis.md)**

</div>

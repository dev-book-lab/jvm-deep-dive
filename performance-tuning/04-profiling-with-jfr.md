# Profiling with JFR - Java Flight Recorder 프로파일링

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JFR (Java Flight Recorder)은 무엇이며, 왜 사용하는가?
- JFR로 어떻게 프로파일링하는가?
- Flame Graph는 어떻게 읽는가?
- JDK Mission Control은 어떻게 사용하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 프로덕션 성능 분석이 어렵다

```
전통적 프로파일러:
  - 높은 오버헤드 (10~50%)
  - 프로덕션 사용 불가
  
JFR:
  - 매우 낮은 오버헤드 (< 1%)
  - 프로덕션에서 항상 실행 가능
  - 상세한 JVM 이벤트 기록
```

JFR은 **프로덕션 프로파일링 도구**다.

---

## 📐 JFR 개요

### 1. JFR 특징

```
오버헤드:
  - 기본: < 1%
  - 상세 설정: 1~3%
  
이벤트:
  - CPU 사용 (Method Profiling)
  - 메모리 할당
  - GC 이벤트
  - Lock Contention
  - I/O
  - Exception

Java 버전:
  - Java 8: 상용 라이센스 필요 (무료화됨)
  - Java 11+: 완전 무료, 오픈소스
```

---

### 2. JFR 활성화

```bash
# 시작 시 활성화
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr \
     MyApp

# 실행 중 시작
jcmd <pid> JFR.start duration=60s filename=recording.jfr

# 실행 중 덤프
jcmd <pid> JFR.dump filename=recording.jfr

# 중지
jcmd <pid> JFR.stop
```

---

### 3. JFR 설정 프로필

```bash
# default (낮은 오버헤드)
-XX:StartFlightRecording=settings=default

# profile (상세, 2~3% 오버헤드)
-XX:StartFlightRecording=settings=profile

# 커스텀 설정
-XX:StartFlightRecording=settings=myprofile.jfc

myprofile.jfc:
  <?xml version="1.0" encoding="UTF-8"?>
  <configuration>
    <event name="jdk.CPUSample">
      <setting name="enabled">true</setting>
      <setting name="period">10 ms</setting>
    </event>
  </configuration>
```

---

## 💻 JFR 실습

### 실습 1: 기본 레코딩

```bash
# 1분간 레코딩
jcmd <pid> JFR.start duration=60s filename=app.jfr

# 레코딩 확인
jcmd <pid> JFR.check

# 출력:
# Recording 1: name=Recording-1 duration=60s (running)
```

---

### 실습 2: 연속 레코딩 (Continuous)

```bash
# 무한 레코딩 (순환 버퍼)
jcmd <pid> JFR.start name=continuous maxage=1h maxsize=100M

# 필요 시 덤프
jcmd <pid> JFR.dump name=continuous filename=snapshot.jfr

# 중지
jcmd <pid> JFR.stop name=continuous
```

---

## 🔥 Flame Graph 분석

### Flame Graph 읽기

```
Flame Graph 구조:

┌─────────────────────────────────────┐
│        main()                       │ ← 최상위 (호출 시작)
├─────────────────┬───────────────────┤
│ processRequest()│   handleData()    │
├────────┬────────┼─────────┬─────────┤
│ query()│ parse()│ encode()│ write() │ ← 최하위 (CPU 소비)
└────────┴────────┴─────────┴─────────┘

X축: CPU 시간 비율 (넓이 = 많은 시간)
Y축: 호출 스택 깊이

색상:
  - 빨강/주황: Hot Spot (CPU 많이 씀)
  - 파랑/녹색: 정상
```

---

### Hot Spot 찾기

```
분석 순서:

1. 넓은 블록 찾기
   → 전체 CPU 시간 많이 차지

2. 호출 스택 추적
   → main → ... → hot method

3. 최적화 대상 결정
   - 자주 호출 + 느린 메서드
   - 불필요한 연산
   - 비효율적 알고리즘

예:
┌─────────────────────────────────┐
│        service()                │
├──────────────────┬──────────────┤
│  JSON.parse()    │  process()   │ ← parse() 넓음!
│    (80%)         │   (20%)      │
└──────────────────┴──────────────┘

결론: JSON 파싱이 병목
```

---

## 🎛️ JDK Mission Control

### JMC 사용법

```bash
# JMC 실행
jmc

# .jfr 파일 열기
File → Open File → recording.jfr
```

### 주요 탭

```
1. Automated Analysis
   - 자동 문제 탐지
   - GC 압박, Hot Method, Lock Contention
   
2. Method Profiling
   - Flame Graph
   - Top Methods (CPU 시간 순)
   
3. Memory
   - Allocation Rate
   - Allocation by Class
   - GC 통계
   
4. Lock Instances
   - Lock Contention
   - 대기 시간
   
5. I/O
   - File I/O
   - Socket I/O
```

---

## ⚡ 실무 분석 예시

### 예시 1: CPU 병목 찾기

```
문제:
  애플리케이션 느림

JFR 분석:
  1. Method Profiling 탭
  2. Flame Graph 확인
  
발견:
  ┌───────────────────────────┐
  │  processData()            │
  ├──────────────┬────────────┤
  │ regex.match()│ other...   │
  │   (70%)      │   (30%)    │
  └──────────────┴────────────┘

결론:
  정규표현식 매칭이 70% CPU 사용
  
해결:
  - 정규표현식 컴파일 캐싱
  - 또는 더 빠른 파서 사용
```

---

### 예시 2: 메모리 할당 압박

```
문제:
  GC 빈번, Pause 길음

JFR 분석:
  1. Memory 탭
  2. Allocation by Class
  
발견:
  byte[] : 5GB/s
  String : 2GB/s
  
  → TLAB Allocation in new TLAB
    매우 빈번

결론:
  byte[] 과다 할당
  
추적:
  Method Profiling → Allocation
  → parseJson() 메서드
  
해결:
  - byte[] 재사용 (풀링)
  - 불필요한 복사 제거
```

---

### 예시 3: Lock Contention

```
문제:
  Throughput 낮음 (멀티스레드)

JFR 분석:
  1. Lock Instances 탭
  2. Java Monitor Blocked
  
발견:
  synchronized(cache) {
    ...
  }
  
  대기 시간: 30% (전체 실행 시간)

결론:
  Cache 락 경쟁 심각
  
해결:
  - ConcurrentHashMap 사용
  - 락 분할 (Striped Lock)
```

---

## 🚫 흔한 실수

### "JFR은 항상 켜둔다"

```
❌ 잘못된 이해:
  모든 이벤트를 항상 기록

✅ 실제:
  필요할 때만 활성화
  
연속 레코딩:
  - 순환 버퍼 (1시간, 100MB)
  - 문제 발생 시 덤프
  - 오버헤드 < 1%
  
상세 레코딩:
  - 문제 재현 시
  - 짧은 시간 (1~5분)
  - 오버헤드 2~3%
```

---

## 📌 핵심 정리

```
JFR
  Java Flight Recorder
  오버헤드 < 1%
  프로덕션 사용 가능

활성화
  -XX:StartFlightRecording
  jcmd JFR.start/dump/stop

프로필
  default: 낮은 오버헤드
  profile: 상세 (2~3%)

이벤트
  CPU (Method Profiling)
  메모리 (Allocation)
  GC, Lock, I/O, Exception

Flame Graph
  X축: CPU 시간
  Y축: 호출 스택
  넓은 블록 = Hot Spot

JMC
  JDK Mission Control
  .jfr 파일 분석
  자동 문제 탐지

실무 분석
  CPU: Flame Graph → Hot Method
  메모리: Allocation by Class
  Lock: Monitor Blocked

Best Practice
  연속 레코딩 (1h, 100MB)
  문제 시 덤프
  짧은 상세 레코딩
  Flame Graph 우선 확인
```

---

## 🤔 생각해볼 문제

**Q1.** JFR Flame Graph에서 JSON.parse() 메서드가 전체 CPU의 40%를 차지한다. 최적화 방법 3가지를 제시하라.

**Q2.** "Allocation in new TLAB" 이벤트가 초당 10,000회 발생한다. 이것이 문제인 이유와 해결 방법을 설명하라.

**Q3.** 프로덕션에서 JFR을 연속으로 실행할 때 권장 설정 (maxage, maxsize)을 제시하고, 이유를 설명하라.

> 💡 **해설**
>
> **Q1.** JSON.parse() 최적화: ① 파서 변경 — Jackson → fastjson 또는 Gson → Jackson. 벤치마크로 검증. ② 캐싱 — 동일 JSON 반복 파싱 시 결과 캐싱 (LRU Cache). ③ Streaming API — DOM 파싱 대신 SAX/Stream 파싱으로 메모리 절약 + 속도 향상. ④ 불필요한 필드 제외 — @JsonIgnore로 파싱 범위 축소. ⑤ String Pool — 반복되는 문자열 인턴.
>
> **Q2.** TLAB 과다 할당 문제: ① TLAB (Thread-Local Allocation Buffer) — 스레드별 Eden 할당 버퍼. 가득 차면 새로 할당 → "new TLAB" 이벤트. ② 초당 10,000회 → 메모리 할당 과다 → GC 압박. ③ 원인: 임시 객체 과다 생성 (byte[], String). ④ 해결: JFR Allocation by Class 확인 → 할당 많은 클래스 추적 → 객체 재사용 (풀링), 불변 객체로 전환, StringBuilder 사용.
>
> **Q3.** 연속 JFR 설정: `jcmd <pid> JFR.start name=continuous maxage=1h maxsize=500M`. 이유: ① maxage=1h — 1시간 이상 오래된 데이터 삭제 (순환 버퍼). 최근 데이터만 유지. ② maxsize=500M — 최대 500MB 제한 → 디스크 압박 방지. ③ 조합: 1시간 또는 500MB 중 먼저 도달 시 삭제. ④ 오버헤드 < 1% 유지. ⑤ 문제 발생 시 JFR.dump로 즉시 스냅샷.

---

## 📚 참고 자료

- [JFR Official Guide](https://docs.oracle.com/javacomponents/jmc-5-4/jfr-runtime-guide/)
- [JMC Tutorial](https://www.oracle.com/java/technologies/jdk-mission-control.html)
- [Flame Graph](https://www.brendangregg.com/flamegraphs.html)

---

<div align="center">

**[⬅️ 이전: GC Ergonomics](./03-gc-ergonomics.md)** | **[다음: Profiling with async-profiler ➡️](./05-profiling-with-async-profiler.md)**

</div>

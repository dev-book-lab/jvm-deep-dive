# GC Log Analysis - GC 로그 분석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- GC 로그를 어떻게 읽고 해석하는가?
- STW 시간, Throughput, Memory 사용량을 어떻게 측정하는가?
- 메모리 누수를 GC 로그에서 어떻게 탐지하는가?
- GC 로그 분석 도구는 무엇이 있는가?
- 실무에서 GC 문제를 어떻게 진단하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: GC 문제는 로그로 드러난다

```
증상:
  - 응답 시간 느려짐
  - CPU 사용률 높음
  - OOM 발생

원인 파악:
  GC 로그 분석 필수
  → Pause Time
  → GC 빈도
  → 메모리 사용 패턴
```

**GC 로그 분석**은 성능 문제 해결의 핵심이다.

---

## 📐 GC 로그 읽기

### 1. G1 GC 로그 예시

```
[2024-02-25T10:30:15.123+0900][0.500s][info][gc] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 512M->64M(2048M) 45.123ms
```

```
해석:
  [2024-02-25T10:30:15.123+0900]  타임스탬프
  [0.500s]                        JVM 시작 후 0.5초
  [info][gc]                      로그 레벨, 태그
  GC(0)                           GC 번호
  Pause Young (Normal)            Young GC
  (G1 Evacuation Pause)           GC 타입
  512M->64M(2048M)                힙: 512MB → 64MB (전체 2GB)
  45.123ms                        Pause Time
```

---

### 2. Mixed GC 로그

```
[gc] GC(5) Pause Young (Mixed) (G1 Evacuation Pause)
[gc] GC(5) Eden: 1024M -> 0M
[gc] GC(5) Survivor: 128M -> 128M
[gc] GC(5) Old: 1536M -> 1200M
[gc] GC(5) 1024M->1200M(4096M) 120.456ms
```

```
해석:
  Mixed GC (Young + Old)
  Eden: 1024MB → 0MB (전부 비움)
  Old: 1536MB → 1200MB (336MB 회수)
  총 Pause: 120ms
```

---

### 3. Full GC 로그

```
[gc] GC(10) Pause Full (Allocation Failure)
[gc] GC(10) 3800M->1200M(4096M) 5234.567ms
```

```
해석:
  Full GC 발생
  원인: Allocation Failure (할당 실패)
  힙: 3800MB → 1200MB (2600MB 회수)
  Pause: 5.2초 (매우 길음)
  
  문제:
  Full GC는 드물어야 함
  → 시간당 1회 미만
  → 빈번하면 튜닝 필요
```

---

### 4. Concurrent Marking 로그

```
[gc] GC(3) Concurrent Cycle
[gc] GC(3) Pause Initial Mark 0.5ms
[gc] GC(3) Concurrent Mark 1234.567ms
[gc] GC(3) Pause Remark 23.456ms
[gc] GC(3) Pause Cleanup 1.234ms
[gc] GC(3) Concurrent Cleanup 45.678ms
```

```
해석:
  Concurrent Marking 주기
  STW: Initial Mark (0.5ms), Remark (23ms), Cleanup (1ms)
  Concurrent: Mark (1.2s), Cleanup (45ms)
  
  총 STW: 25ms (짧음)
```

---

## 💻 GC 로그 분석 메트릭

### 1. Pause Time 분석

```bash
# GC 로그에서 Pause Time 추출
grep "Pause" gc.log | awk '{print $NF}' | sed 's/ms//'

# 통계 계산 (Python)
import statistics

pauses = [45.1, 52.3, 48.7, 120.4, 55.2]  # ms

print(f"Min: {min(pauses)} ms")
print(f"Max: {max(pauses)} ms")
print(f"Avg: {statistics.mean(pauses):.2f} ms")
print(f"P95: {statistics.quantiles(pauses, n=20)[18]:.2f} ms")
print(f"P99: {statistics.quantiles(pauses, n=100)[98]:.2f} ms")

# 출력:
# Min: 45.1 ms
# Max: 120.4 ms
# Avg: 64.34 ms
# P95: 115.2 ms
# P99: 119.5 ms
```

---

### 2. GC Throughput 계산

```
GC Throughput = (Total Time - GC Time) / Total Time

예:
  실행 시간: 100초
  GC 시간: 5초
  
  Throughput = (100 - 5) / 100 = 95%
  
목표:
  > 95% (GC 시간 < 5%)
  
  90% 미만: 문제 (GC 과다)
```

---

### 3. 메모리 사용 패턴

```
힙 사용량 추이:

시간    Before GC    After GC
0s      500MB        100MB
10s     600MB        120MB
20s     700MB        150MB
30s     800MB        200MB  ← 증가 추세
40s     900MB        300MB
50s     OOM

분석:
  After GC가 지속 증가
  → 메모리 누수 의심
  → Heap Dump 분석 필요
```

---

### 4. Promotion Rate

```
Promotion Rate = Old 증가량 / 시간

예:
  10초 동안 Old: 500MB → 800MB
  Promotion Rate = 300MB / 10s = 30MB/s
  
높은 Promotion:
  > 50MB/s → 문제
  
  원인:
  - Young Gen 너무 작음
  - 객체 수명 길어짐
  
  해결:
  Young Gen 크게 (-Xmn 증가)
  MaxTenuringThreshold 증가
```

---

## 🔧 GC 로그 분석 도구

### 1. GCEasy (gceasy.io)

```
사용법:
  1. gc.log 업로드
  2. 자동 분석
  3. 리포트 생성

제공 정보:
  - Pause Time 분포 (그래프)
  - Throughput
  - 메모리 누수 탐지
  - 권장 튜닝
  
장점: 웹 기반, 무료
단점: 업로드 필요 (보안)
```

---

### 2. GCViewer

```
오픈소스 도구:
  https://github.com/chewiebug/GCViewer

사용법:
  java -jar gcviewer.jar gc.log

기능:
  - 시계열 그래프
  - 메모리 사용량 시각화
  - Pause Time 분포
  
장점: 로컬 실행, 무료
단점: UI 다소 복잡
```

---

### 3. JClarity Censum (상용)

```
엔터프라이즈 도구

기능:
  - 실시간 모니터링
  - 머신러닝 기반 분석
  - 자동 튜닝 권장
  
단점: 유료
```

---

## 🚨 문제 패턴과 해결

### 패턴 1: Full GC 빈번

```
로그:
  [gc] GC(10) Pause Full 5.2s
  [gc] GC(15) Pause Full 5.8s
  [gc] GC(20) Pause Full 6.1s
  
  5분마다 Full GC
  
원인:
  Old Generation 부족
  Promotion Rate 높음
  
해결:
  힙 크기 증가 (-Xmx)
  Young Gen 크게 (-Xmn)
  G1/ZGC로 전환
```

---

### 패턴 2: 메모리 누수

```
로그:
  시간    After GC
  0h      500MB
  1h      600MB
  2h      700MB
  3h      800MB
  4h      900MB
  5h      OOM
  
After GC가 계속 증가
→ 메모리 누수

진단:
  jmap -dump:live,format=b,file=heap.hprof <pid>
  
  Eclipse MAT로 분석:
  - Leak Suspects
  - Dominator Tree
  
해결:
  누수 객체 찾기
  코드 수정
```

---

### 패턴 3: Young GC 과다

```
로그:
  1초에 10회 Young GC
  
  [gc] GC(0) Pause Young 50ms
  [gc] GC(1) Pause Young 48ms
  ...
  [gc] GC(10) Pause Young 52ms
  
  (1초 후)
  
원인:
  Eden 너무 작음
  할당률 높음
  
해결:
  Young Gen 크게 (-Xmn 증가)
  힙 전체 크게
  
  효과:
  GC 빈도 감소 (10회 → 2회)
```

---

## 📊 실무 분석 예시

### 시나리오: 웹 서버 응답 느려짐

```
1. GC 로그 확인

[gc] GC(100) Pause Young 200ms  ← 목표 100ms 초과
[gc] GC(105) Pause Full 8.5s    ← Full GC 발생

2. 메트릭 추출

Pause Time:
  Avg: 150ms
  P99: 250ms
  → 목표 초과 (100ms)

Full GC:
  시간당 2회
  → 너무 빈번

3. 원인 분석

Old Generation 사용률:
  95% 이상 유지
  → Old 부족

4. 튜닝

Before:
  -Xmx4g -Xmn1g
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=100

After:
  -Xmx8g -Xmn2g  ← 힙 2배
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=100

5. 결과

Pause Time:
  Avg: 80ms
  P99: 120ms
  → 개선

Full GC:
  시간당 0회
  → 해결
```

---

## 📌 핵심 정리

```
GC 로그 읽기
  타임스탬프, GC 번호, 타입
  힙 사용량 (Before → After)
  Pause Time

주요 메트릭
  Pause Time (Min/Max/Avg/P95/P99)
  Throughput (> 95%)
  GC 빈도
  Promotion Rate

문제 패턴
  Full GC 빈번 → 힙 부족
  After GC 증가 → 메모리 누수
  Young GC 과다 → Eden 작음

분석 도구
  GCEasy (웹)
  GCViewer (로컬)
  jstat, jmap (CLI)

진단 순서
  1. GC 로그 수집
  2. 메트릭 계산
  3. 패턴 찾기
  4. 원인 파악
  5. 튜닝 적용
  6. 효과 측정
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 GC 로그 패턴이 의미하는 것은 무엇인가? 문제를 진단하고 해결 방법을 제시하라.
```
[gc] GC(0)  500M -> 100M (2048M) 50ms
[gc] GC(10) 600M -> 150M (2048M) 55ms
[gc] GC(20) 700M -> 220M (2048M) 60ms
[gc] GC(30) 800M -> 310M (2048M) 65ms
[gc] GC(40) 900M -> 420M (2048M) 70ms
```

**Q2.** GC 로그에서 Promotion Rate를 계산하는 방법을 설명하고, 다음 데이터에서 문제가 있는지 판단하라.
- 10초 동안 Old Generation: 1000MB → 1500MB
- 힙: 4GB, Young: 1GB, Old: 3GB

**Q3.** 웹 서버의 GC 로그 분석 결과 다음과 같다. 어느 부분을 우선 개선해야 하는가?
- Avg Pause: 80ms (목표 100ms)
- P99 Pause: 350ms (목표 100ms 초과)
- Full GC: 시간당 3회

> 💡 **해설**
>
> **Q1.** 진단: After GC (GC 후 힙 사용량)가 지속 증가 (100MB → 420MB) → 메모리 누수 의심. Before GC도 증가 (500MB → 900MB) → 정상 패턴. 하지만 After GC는 줄어야 함 → 살아있는 객체가 계속 누적 중. 해결: ① Heap Dump 생성 (`jmap -dump:live,format=b,file=heap.hprof <pid>`). ② Eclipse MAT로 분석 → Leak Suspects 찾기. ③ Dominator Tree로 큰 객체 확인. ④ Static 컬렉션, Cache, Listener 등 확인. ⑤ 코드 수정 (명시적 제거 또는 WeakReference 사용).
>
> **Q2.** Promotion Rate 계산: (1500MB - 1000MB) / 10s = 50MB/s. 문제 판단: ① 50MB/s는 높은 편 (일반적으로 10~30MB/s). ② Young 1GB에서 50MB/s 승격 → 매 20초마다 1GB 승격 → Young GC 20초마다 발생 추정. ③ Old 3GB → 50MB/s 승격 시 60초에 Old 가득 참 → Full GC 빈번 예상. 문제: Promotion Rate 과다 → Old 압박. 해결: ① Young Gen 증가 (`-Xmn2g`) → 더 많은 객체가 Young에서 죽음 → Promotion 감소. ② MaxTenuringThreshold 증가 (기본 15 → 31) → 더 오래 Young 유지. ③ 힙 증가 (`-Xmx8g`) → Old 여유 확보.
>
> **Q3.** 우선순위: P99 Pause 개선 (350ms). 이유: ① Avg Pause 80ms는 목표 달성 (100ms 이하). ② P99 Pause 350ms는 심각 — 1% 요청이 3.5배 느림 → 사용자 경험 저하. ③ Full GC 시간당 3회도 문제지만 P99가 더 시급. P99 개선 방법: ① GC 로그에서 350ms Pause 찾기 → 어떤 GC인지 확인 (Full GC? Mixed GC?). ② Full GC라면 → 힙 증가, Young 증가로 Full GC 회피. ③ Mixed GC라면 → `MaxGCPauseMillis=80` 낮추기, `G1MixedGCCountTarget` 증가. Full GC 개선: ① 시간당 3회는 과다 (1회 미만 목표). ② Old 압박 → 힙 증가, Promotion 감소. 순서: P99 먼저, Full GC 나중.

---

## 📚 참고 자료

- [GCEasy - GC Log Analyzer](https://gceasy.io/)
- [Understanding GC Logs](https://www.oracle.com/technical-resources/articles/java/gcloganalyzer.html)
- [GCViewer GitHub](https://github.com/chewiebug/GCViewer)

---

<div align="center">

**[⬅️ 이전: GC Tuning Flags](./10-gc-tuning-flags.md)** | **[홈으로 🏠](../README.md)**

</div>

# Tiered Compilation - 계층화 컴파일

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Tiered Compilation은 무엇이며, 왜 필요한가?
- Level 0~4는 각각 무엇을 의미하는가?
- C1과 C2 컴파일러는 어떻게 협력하는가?
- Level 전환은 언제 일어나며, 어떤 조건으로 결정되는가?
- `-XX:-TieredCompilation`을 끄면 어떻게 되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: C1과 C2는 서로 다른 장단점을 가진다

```
C1 (Client Compiler):
  장점: 빠른 컴파일 (수 ms)
  단점: 기본적인 최적화만
  
  적합: 빠른 Warm-up 필요
  부적합: 장시간 실행 (최종 성능 부족)

C2 (Server Compiler):
  장점: 고급 최적화 (25배 빠름)
  단점: 느린 컴파일 (수백 ms)
  
  적합: 장시간 실행 서버
  부적합: 짧은 스크립트 (컴파일 시간 > 실행 시간)

딜레마:
  빠른 시작 vs 최종 성능
  → 둘 다 원한다면?
```

Tiered Compilation은 **C1으로 시작해 C2로 업그레이드**한다.

---

## 📐 내부 구조

### 1. Compilation Level 0~4

```
Level 0: Interpreter
  - 바이트코드 직접 실행
  - Counter 증가 (프로파일링)
  - 가장 느림

Level 1: C1 (Simple)
  - 빠른 컴파일 (간단한 최적화)
  - 프로파일링 없음
  - Interpreter보다 5~10배 빠름

Level 2: C1 (Limited Profiling)
  - Invocation Counter만 수집
  - Backedge Counter는 수집 안 함
  - Level 1보다 약간 느림 (프로파일링 오버헤드)

Level 3: C1 (Full Profiling)
  - Invocation + Backedge Counter 수집
  - Branch Profiling
  - Type Profiling (어떤 타입이 호출되는지)
  - C2를 위한 데이터 수집
  - 가장 많은 정보 수집

Level 4: C2 (Full Optimization)
  - Level 3의 프로파일링 데이터 활용
  - 고급 최적화 (Inlining, Escape Analysis 등)
  - 가장 빠름 (최대 25배)
  - 가장 느린 컴파일 (수백 ms)
```

---

### 2. Level 전환 흐름

```
일반적인 경로:

Level 0 (Interpreter)
  ↓ CompileThreshold 도달
Level 3 (C1 Full Profiling)
  ↓ Tier3CompileThreshold 도달 + 프로파일링 데이터 충분
Level 4 (C2 Full Optimization)

빠른 경로 (Trivial Method):

Level 0
  ↓ 매우 간단한 메서드 (getter/setter)
Level 1 (C1 Simple)
  ↓ 프로파일링 불필요
끝 (Level 4로 안 감)

백오프 경로 (C2 큐 혼잡):

Level 0
  ↓
Level 3
  ↓ C2 컴파일 큐 가득 참
Level 2 (C1 Limited Profiling)
  ↓ 대기
Level 4 (나중에)

실제 예시:

시간  Level  설명
0ms   0      Interpreter 시작
100ms 3      2000회 호출 → C1 Full Profiling
150ms 4      C2 컴파일 시작
200ms 4      C2 완료, Level 3 "made not entrant"
```

---

### 3. Level 전환 조건

```
Level 0 → 3:
  invocationCounter > Tier3CompileThreshold (기본 2000)
  또는
  backedgeCounter > Tier3BackEdgeThreshold

Level 3 → 4:
  invocationCounter > Tier4CompileThreshold (기본 15000)
  그리고
  충분한 프로파일링 데이터 수집됨
  그리고
  메서드가 충분히 Hot함 (계속 호출됨)

Level 0 → 1 (Trivial):
  메서드가 매우 간단함 (바이트코드 < 10)
  프로파일링 불필요
  
Level 3 → 2 (백오프):
  C2 컴파일 큐가 가득 참
  또는
  C2 컴파일러 스레드 모두 바쁨
```

---

### 4. PrintCompilation 출력 해석

```bash
java -XX:+PrintCompilation MyApp

# Level 0 → 3 → 4 예시:
#    100    1       3       MyClass::compute (42 bytes)
#    150    2       4       MyClass::compute (42 bytes)
#    155    1       3       MyClass::compute (42 bytes)   made not entrant

# 설명:
# 100ms: Level 3 (C1) 컴파일 완료
# 150ms: Level 4 (C2) 컴파일 시작
# 155ms: Level 3 버전 비활성화 (Level 4 사용)

# Level 표시:
# 1 = C1 Simple
# 2 = C1 Limited Profiling
# 3 = C1 Full Profiling
# 4 = C2 Full Optimization

# 추가 표시:
# %   = OSR (On-Stack Replacement)
# s   = synchronized
# !   = exception handlers
# b   = blocking (컴파일 완료까지 대기)
# n   = native wrapper
```

---

### 5. C1 vs C2 최적화 비교

```
C1 최적화:
  - Constant Folding (상수 접기)
  - Dead Code Elimination (죽은 코드 제거)
  - Range Check Elimination (간단한 경우만)
  - 기본 Inlining (작은 메서드만)

C2 최적화:
  - Aggressive Inlining (대규모)
  - Escape Analysis (객체 스택 할당)
  - Loop Unrolling (루프 펼치기)
  - Vectorization (SIMD)
  - Global Value Numbering
  - Speculative Optimization (타입 예측)
  - Lock Elision (불필요한 lock 제거)

컴파일 시간:
  C1: 5~50 ms
  C2: 50~500 ms

최종 성능:
  Interpreter: 1x
  C1:          5~10x
  C2:          20~25x
```

---

## 💻 실험으로 확인하기

### 실험 1: Tiered Compilation 활성화/비활성화 비교

```java
public class TieredBenchmark {
    public static long fibonacci(int n) {
        if (n <= 1) return n;
        return fibonacci(n - 1) + fibonacci(n - 2);
    }
    
    public static void main(String[] args) {
        long start = System.nanoTime();
        
        for (int i = 0; i < 20; i++) {
            fibonacci(35);
        }
        
        long elapsed = System.nanoTime() - start;
        System.out.println("Time: " + elapsed / 1_000_000 + " ms");
    }
}
```

```bash
# Tiered Compilation 활성화 (기본)
java -XX:+PrintCompilation TieredBenchmark
# 출력: Time: 2500 ms
# Level 3 → Level 4 전환 확인

# Tiered Compilation 비활성화 (C2만)
java -XX:-TieredCompilation -XX:+PrintCompilation TieredBenchmark
# 출력: Time: 3500 ms (더 느림, 느린 Warm-up)

# C1만 사용
java -XX:TieredStopAtLevel=1 -XX:+PrintCompilation TieredBenchmark
# 출력: Time: 5000 ms (최종 성능 낮음)
```

---

### 실험 2: Level 전환 관찰

```bash
java -XX:+PrintCompilation \
     -XX:+UnlockDiagnosticVMOptions \
     -XX:+LogCompilation \
     -XX:LogFile=compilation.log \
     MyApp

# compilation.log 분석:
# <task compile_id='1' method='MyClass compute' level='3' ...>
#   → Level 3 컴파일
# <task compile_id='2' method='MyClass compute' level='4' ...>
#   → Level 4 컴파일
# <make_not_entrant compile_id='1' ...>
#   → Level 3 비활성화
```

---

### 실험 3: CompileThreshold 조정

```bash
# Level 3 임계값 낮추기
java -XX:Tier3CompileThreshold=500 \
     -XX:+PrintCompilation \
     MyApp
# → 500회만 호출해도 C1 컴파일

# Level 4 임계값 낮추기
java -XX:Tier4CompileThreshold=5000 \
     -XX:+PrintCompilation \
     MyApp
# → 5000회만 호출해도 C2 컴파일
# (기본 15000보다 빠른 C2 전환)
```

---

## ⚡ 실무 임팩트

### Tiered Compilation의 이점

```
시나리오: 웹 서버 시작

Tiered OFF (C2만):
  0s:    시작
  0.5s:  첫 요청 도착
         Interpreter 모드 (느림)
         응답: 100ms
  
  10s:   10,000회 호출 → C2 컴파일
  10.5s: C2 컴파일 완료
         응답: 5ms
  
  문제: 초기 10초간 느린 응답

Tiered ON (기본):
  0s:    시작
  0.5s:  첫 요청 도착
         Interpreter 모드
         응답: 100ms
  
  1s:    2,000회 호출 → C1 컴파일
  1.05s: C1 완료
         응답: 20ms (5배 향상)
  
  10s:   15,000회 호출 → C2 컴파일
  10.5s: C2 완료
         응답: 5ms (최종 성능)
  
  이점: 1초 후부터 빠른 응답 (20ms)
```

### TieredStopAtLevel 활용

```
메모리 제약 환경 (임베디드, 컨테이너):

-XX:TieredStopAtLevel=1
  → C1만 사용, C2 비활성화
  
장점:
  - 메모리 절약 (C2 컴파일러 ~10MB)
  - 빠른 컴파일 (CPU 절약)
  - Code Cache 절약

단점:
  - 최종 성능 낮음 (C2 대비 50% 느림)

사용 케이스:
  - 짧은 실행 배치 작업
  - 메모리 < 512MB 컨테이너
  - 임베디드 디바이스
```

### StartUp 최적화

```
애플리케이션별 권장:

마이크로서비스 (짧은 요청):
  -XX:TieredCompilation (기본)
  -XX:Tier3CompileThreshold=1000
  → 빠른 C1 전환

장시간 실행 서버:
  -XX:TieredCompilation (기본)
  -XX:Tier4CompileThreshold=10000
  → 더 많은 프로파일링 → 더 나은 C2 최적화

배치 작업 (< 1분):
  -XX:TieredStopAtLevel=1
  → C2 컴파일 비용 제거
```

---

## 🚫 흔한 오해

### "Tiered Compilation을 끄면 더 빠르다"

```
❌ 잘못된 이해:
  -XX:-TieredCompilation하면 C2만 써서 빠르다.

✅ 실제:
  대부분의 경우 Tiered ON이 더 빠름
  
이유:
  Tiered OFF: 0 → 4 (직접 C2)
  - Warm-up 느림 (Interpreter → C2)
  - C2 컴파일 전까지 Interpreter
  
  Tiered ON: 0 → 3 → 4
  - Warm-up 빠름 (C1 거쳐감)
  - C1으로 빠르게 성능 향상
  
벤치마크:
  Tiered ON:  초기 5배 빠름 (C1), 최종 25배 (C2)
  Tiered OFF: 초기 1배 (Interpreter), 최종 25배 (C2)
  
  → Tiered ON이 전체적으로 유리
```

### "Level 4가 항상 최종 목표다"

```
❌ 잘못된 이해:
  모든 메서드가 Level 4로 가야 한다.

✅ 실제:
  많은 메서드가 Level 3 또는 Level 1에 머뭄
  
Level 1에서 끝:
  - Getter/Setter (간단함)
  - 1~2회만 호출되는 메서드
  - 프로파일링 불필요
  
Level 3에서 끝:
  - 중간 정도로 자주 호출
  - C2 컴파일 비용 > 성능 이득
  
Level 4로 가는 것:
  - 매우 Hot한 메서드만 (상위 5%)
  - 루프가 많은 계산 집약적 코드
  
통계:
  Level 0~1: 80%
  Level 3:   15%
  Level 4:   5%
```

### "Level은 순차적으로만 올라간다"

```
❌ 잘못된 이해:
  0 → 1 → 2 → 3 → 4 순서로만 간다.

✅ 실제:
  여러 경로 가능
  
가능한 경로:
  0 → 3 → 4 (일반적)
  0 → 1      (Trivial)
  0 → 3 → 2 → 4 (백오프)
  0 → 4      (Tiered OFF)
  
역행도 가능:
  4 → 0 (Deoptimization)
  → Speculative Optimization 실패 시
```

---

## 📌 핵심 정리

```
Tiered Compilation
  Level 0~4로 점진적 최적화
  C1으로 빠른 Warm-up, C2로 최종 성능

Level 구분
  0: Interpreter
  1: C1 Simple
  2: C1 Limited Profiling
  3: C1 Full Profiling
  4: C2 Full Optimization

전환 조건
  0 → 3: Tier3CompileThreshold (2000)
  3 → 4: Tier4CompileThreshold (15000) + 프로파일링 데이터

C1 vs C2
  C1: 빠른 컴파일 (5~50ms), 기본 최적화, 5~10배
  C2: 느린 컴파일 (50~500ms), 고급 최적화, 20~25배

장점
  빠른 시작 (C1)
  높은 최종 성능 (C2)
  효율적 리소스 사용 (필요한 만큼만 C2)

TieredStopAtLevel
  1: C1만 (메모리 절약)
  3: C2 없이 (중간 성능)
  4: 기본 (모든 최적화)

실무 권장
  대부분: 기본 설정 유지
  메모리 제약: TieredStopAtLevel=1
  초고성능: Tier4CompileThreshold 조정
```

---

## 🤔 생각해볼 문제

**Q1.** 메서드가 Level 3에서 Level 4로 전환되는 동안 해당 메서드가 계속 호출된다. 어느 버전이 실행되는가? Level 3 → 4 전환 과정을 설명하라.

**Q2.** `-XX:TieredStopAtLevel=1`과 `-XX:-TieredCompilation` 중 어느 것이 더 빠른 시작 시간을 제공하는가? 각각의 동작 방식과 컴파일 경로를 설명하라.

**Q3.** C2 컴파일러 큐가 가득 차서 Level 3 → 2로 백오프되는 상황은 언제 발생하는가? 이를 방지하려면 어떻게 해야 하는가?

> 💡 **해설**
>
> **Q1.** Level 3 버전이 계속 실행됨. Level 4 컴파일은 백그라운드 스레드에서 비동기로 진행됨. 컴파일 완료 후: ① method->entry_point를 Level 4 코드로 교체. ② 이후 호출부터 Level 4 실행. ③ Level 3 버전을 "made not entrant"로 표시 (새 진입 금지, 실행 중인 건 완료까지 허용). ④ 모든 Level 3 실행 종료 후 "made zombie"로 전환 → 메모리 회수. 따라서 Level 4 컴파일 중에도 성능 저하 없음 (Level 3 계속 사용).
>
> **Q2.** TieredStopAtLevel=1이 약간 빠름. 이유: ① TieredStopAtLevel=1 — 0 → 1 경로. Trivial Method는 즉시 Level 1 컴파일 (수 ms). C2 컴파일러 로드 안 함 → 메모리 절약. ② -TieredCompilation — 0 → 4 직행. CompileThreshold (10000) 도달 전까지 Interpreter. C2 컴파일러 로드 필요 → 추가 메모리. 하지만 차이는 미미 (수백 ms 이내). 최종 성능은 -TieredCompilation이 우수 (C2 최적화).
>
> **Q3.** 발생 조건: ① 많은 Hot Method 동시 발생 (애플리케이션 시작 시 트래픽 급증). ② C2 컴파일러 스레드 부족 (-XX:CICompilerCount 낮음). ③ C2 큐 크기 제한 도달. 백오프 동작: Level 3에서 대기 중 Level 2로 강등 (프로파일링 줄임) → C2 큐 여유 생기면 Level 4로 재컴파일. 방지 방법: ① CICompilerCount 증가 (기본: CPU 코어수 / 3). ② CompileThreshold 증가 (동시 컴파일 요청 감소). ③ Warm-up 분산 (점진적 트래픽 증가). ④ Code Cache 크기 증가 (-XX:ReservedCodeCacheSize).

---

## 📚 참고 자료

- [JEP 243 — Java-Level JVM Compiler Interface](https://openjdk.org/jeps/243)
- [Tiered Compilation in HotSpot](https://www.oracle.com/technical-resources/articles/java/architect-turbo-charging.html)
- [Understanding Tiered Compilation](https://ionutbalosin.com/2023/08/understanding-jvm-tiered-compilation/)

---

<div align="center">

**[⬅️ 이전: JIT Compilation Basics](./02-jit-compilation-basics.md)** | **[다음: JIT Optimizations ➡️](./04-jit-optimizations.md)**

</div>

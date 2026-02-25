# JIT Compilation Basics - JIT 컴파일 기초

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JIT (Just-In-Time) 컴파일러는 무엇이며, 언제 동작하는가?
- HotSpot은 어떻게 Hot Method를 감지하는가?
- Invocation Counter와 Backedge Counter의 역할은?
- `-XX:+PrintCompilation`으로 무엇을 확인할 수 있는가?
- Warm-up이란 무엇이며, 왜 필요한가?

---

## 🔍 왜 이게 존재하는가

### 문제: Interpreter만으로는 느리다

```
고성능 애플리케이션:
  - 웹 서버: 초당 수만 요청 처리
  - 데이터 처리: 수억 건 레코드 연산
  - 게임: 60 FPS (프레임당 16ms)

Interpreter 성능:
  명령어당 ~50ns
  → 1초에 2000만 명령어
  
  문제:
  복잡한 연산 (수백만 명령어)
  → Interpreter로는 느림
  
  예: 1억 번 루프
  Interpreter: 5초
  네이티브 코드: 0.2초
  → 25배 차이
```

JIT 컴파일러는 **자주 실행되는 코드를 네이티브 코드로 변환**한다.

---

## 📐 내부 구조

### 1. JIT 컴파일러 개요

```
HotSpot JVM의 JIT:

C1 (Client Compiler):
  - 빠른 컴파일
  - 기본 최적화
  - Warm-up 단계

C2 (Server Compiler):
  - 느린 컴파일
  - 고급 최적화
  - 최종 성능

실행 흐름:
  Interpreter → C1 (Level 1~3) → C2 (Level 4)
  
  0: Interpreter
  1: C1 (간단한 최적화)
  2: C1 (Invocation + Backedge Counter)
  3: C1 (Full Profiling)
  4: C2 (최종 최적화)
```

---

### 2. Hot Method 감지

```
Invocation Counter:
  메서드 호출 횟수 추적
  
  void myMethod() {
      // ...
  }
  
  myMethod() 호출 시:
  invocationCounter++;
  
  if (invocationCounter > CompileThreshold) {
      enqueue_compilation(myMethod);
  }

Backedge Counter:
  루프 반복 횟수 추적
  
  for (int i = 0; i < 10000; i++) {
      // ...
      // 루프 끝 (backedge)
      backedgeCounter++;
  }
  
  if (backedgeCounter > OnStackReplacePercentage * CompileThreshold / 100) {
      enqueue_osr_compilation(loop);
  }

기본값:
  -XX:CompileThreshold=10000 (C2)
  → 메서드 10,000회 호출 시 컴파일
  
  -XX:Tier3CompileThreshold=2000 (C1)
  → C1은 더 빨리 컴파일
```

---

### 3. 컴파일 큐

```
Compilation Queue:

메서드 호출 → Counter 증가 → Threshold 초과
→ Compilation Queue에 추가
→ 백그라운드 컴파일러 스레드가 처리

┌───────────────────────────────┐
│   Compilation Queue           │
├───────────────────────────────┤
│  1. MyService.process() (C2)  │
│  2. Utils.calculate() (C1)    │
│  3. Handler.handle() (C2)     │
└───────────────────────────────┘
        ↓
┌───────────────────────────────┐
│  Compiler Threads (C1/C2)     │
│  Thread 1: Compiling #1       │
│  Thread 2: Compiling #2       │
└───────────────────────────────┘

컴파일 완료 후:
  method->entry_point = compiled_code;
  
이후 호출:
  Interpreter 대신 compiled_code 실행
```

---

### 4. -XX:+PrintCompilation 출력

```bash
java -XX:+PrintCompilation MyApp

# 출력 예시:
#     68    1       3       java.lang.String::hashCode (55 bytes)
#     80    2       4       java.util.HashMap::getNode (148 bytes)
#    120    3       3       com.example.MyService::process (42 bytes)
#    135    3       4       com.example.MyService::process (42 bytes)
#    150    4  s    3       com.example.Utils::calculate (120 bytes)

# 컬럼 설명:
# 68:     타임스탬프 (ms)
# 1:      Compilation ID
# 3:      Tier (Level)
# s:      synchronized 메서드
# !:      예외 핸들러 있음
# %:      OSR (On-Stack Replacement)
# method: 메서드 이름
# (55 bytes): 바이트코드 크기
```

---

### 5. Warm-up 과정

```
애플리케이션 시작 후 성능 변화:

0ms:    JVM 시작, Interpreter 모드
100ms:  첫 요청 처리
        응답 시간: 50ms (Interpreter)

1000ms: 요청 100개 처리
        Hot Method 감지 → C1 컴파일 시작

1200ms: C1 컴파일 완료
        응답 시간: 10ms (C1 네이티브)

10000ms: 요청 10,000개 처리
         Hot Method 감지 → C2 컴파일 시작

10500ms: C2 컴파일 완료
         응답 시간: 2ms (C2 최적화)

Warm-up 완료: ~10초
정상 상태: 2ms 응답 시간 유지

벤치마크 주의:
  첫 실행: Interpreter (느림)
  10초 후: C2 최적화 (빠름)
  → Warm-up 후 측정해야 정확
```

---

## 💻 실험으로 확인하기

### 실험 1: PrintCompilation으로 컴파일 관찰

```java
public class CompilationDemo {
    public static int compute(int n) {
        int sum = 0;
        for (int i = 0; i < n; i++) {
            sum += i;
        }
        return sum;
    }
    
    public static void main(String[] args) {
        // Warm-up
        for (int i = 0; i < 20000; i++) {
            compute(1000);
        }
        
        System.out.println("Warm-up done");
        
        // 실제 측정
        long start = System.nanoTime();
        for (int i = 0; i < 10000000; i++) {
            compute(1000);
        }
        long elapsed = System.nanoTime() - start;
        
        System.out.println("Time: " + elapsed / 1_000_000 + " ms");
    }
}
```

```bash
java -XX:+PrintCompilation CompilationDemo

# 출력:
#     85    1       3       CompilationDemo::compute (21 bytes)
#     90    2       4       CompilationDemo::compute (21 bytes)
#     95    1       3       CompilationDemo::compute (21 bytes)   made not entrant
# Warm-up done
# Time: 150 ms

# 설명:
# Tier 3 (C1) → Tier 4 (C2)로 업그레이드
# "made not entrant": 이전 버전(C1) 비활성화
```

---

### 실험 2: CompileThreshold 조정

```bash
# 기본값 (10000)
java -XX:+PrintCompilation CompilationDemo

# 낮은 임계값 (1000)
java -XX:+PrintCompilation -XX:CompileThreshold=1000 CompilationDemo
# → 1000회만 호출해도 컴파일
# → 더 빠른 Warm-up

# 높은 임계값 (100000)
java -XX:+PrintCompilation -XX:CompileThreshold=100000 CompilationDemo
# → 100,000회 호출해야 컴파일
# → 느린 Warm-up, 하지만 더 많은 프로파일링 데이터
```

---

### 실험 3: Warm-up 전후 성능 비교

```java
public class WarmupBenchmark {
    public static long fibonacci(int n) {
        if (n <= 1) return n;
        return fibonacci(n - 1) + fibonacci(n - 2);
    }
    
    public static void main(String[] args) {
        System.out.println("=== Cold Start ===");
        long start = System.nanoTime();
        fibonacci(35);
        long cold = System.nanoTime() - start;
        System.out.println("Cold: " + cold / 1_000_000 + " ms");
        
        // Warm-up
        for (int i = 0; i < 10; i++) {
            fibonacci(35);
        }
        
        System.out.println("=== Warm Start ===");
        start = System.nanoTime();
        fibonacci(35);
        long warm = System.nanoTime() - start;
        System.out.println("Warm: " + warm / 1_000_000 + " ms");
        
        System.out.println("Speedup: " + (double)cold / warm + "x");
    }
}
```

```bash
# 출력:
# === Cold Start ===
# Cold: 2500 ms
# === Warm Start ===
# Warm: 100 ms
# Speedup: 25.0x

# → Warm-up 후 25배 빠름
```

---

## ⚡ 실무 임팩트

### 마이크로벤치마크 주의사항

```java
// ❌ 잘못된 벤치마크
public static void main(String[] args) {
    long start = System.nanoTime();
    compute();
    long elapsed = System.nanoTime() - start;
    System.out.println("Time: " + elapsed);
}

// 문제:
// 1. Warm-up 없음 → Interpreter 모드 측정
// 2. 1회 실행 → 통계적 유의성 없음
// 3. JIT 최적화 고려 안 함

// ✅ 올바른 벤치마크 (JMH 사용 권장)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(1)
public class MyBenchmark {
    @Benchmark
    public void testMethod() {
        compute();
    }
}
```

### 서버 시작 시 Warm-up

```java
// Spring Boot 예시
@Component
public class WarmupRunner implements ApplicationRunner {
    @Autowired
    private MyService service;
    
    @Override
    public void run(ApplicationArguments args) {
        // 주요 메서드 미리 호출 (Warm-up)
        for (int i = 0; i < 10000; i++) {
            service.process(mockData);
        }
        log.info("Warm-up completed");
    }
}

// 효과:
// 시작 직후부터 빠른 응답 시간
// 첫 실사용자가 느린 응답 경험 안 함
```

### CompileThreshold 튜닝

```
기본값 (10000):
  대부분의 애플리케이션에 적합
  
낮은 값 (1000-5000):
  장점: 빠른 Warm-up
  단점: 덜 최적화된 코드 (프로파일링 데이터 부족)
  
  사용처: 짧은 실행 시간 배치
         테스트 환경
  
높은 값 (50000-100000):
  장점: 더 많은 프로파일링 → 더 나은 최적화
  단점: 느린 Warm-up
  
  사용처: 장시간 실행 서버
         성능 중시 애플리케이션
```

---

## 🚫 흔한 오해

### "JIT 컴파일은 항상 성능을 향상시킨다"

```
❌ 잘못된 이해:
  JIT 컴파일되면 무조건 빨라진다.

✅ 실제:
  대부분은 빨라지지만, 예외 있음
  
느려지는 경우:
  1. 컴파일 오버헤드
     짧은 메서드를 1~2회만 호출
     → 컴파일 비용 > 실행 절감
  
  2. 잘못된 최적화
     Speculative Optimization 실패
     → Deoptimization 발생
     → 오히려 느려짐
  
  3. 코드 캐시 부족
     -XX:ReservedCodeCacheSize 초과
     → 컴파일 실패 또는 오래된 코드 제거
```

### "Warm-up은 한 번만 필요하다"

```
❌ 잘못된 이해:
  시작 시 Warm-up 하면 영구히 빠르다.

✅ 실제:
  Deoptimization으로 Interpreter 복귀 가능
  
  시나리오:
  1. Warm-up 완료 → C2 최적화
  2. 새로운 타입 등장 (다형성)
     예: List<Dog> → List<Cat> 추가
  3. 기존 최적화 무효화 (Deoptimization)
  4. Interpreter로 복귀
  5. 재컴파일 (새 타입 고려)
  
  → "영구적"이 아님
```

### "PrintCompilation 출력이 많을수록 좋다"

```
❌ 잘못된 이해:
  컴파일이 많이 일어나면 성능이 좋다.

✅ 실제:
  적정량이 중요
  
  너무 적음:
  Hot Method를 놓침
  → 최적화 기회 상실
  
  너무 많음:
  - 불필요한 메서드까지 컴파일
  - Compilation Queue 병목
  - Code Cache 낭비
  - 컴파일 스레드 과부하
  
  적정량:
  주요 Hot Path만 컴파일
  80/20 규칙 (20% 코드가 80% 실행)
```

---

## 📌 핵심 정리

```
JIT 컴파일러
  Just-In-Time: 런타임에 바이트코드 → 네이티브 코드
  HotSpot: C1 (빠른 컴파일) + C2 (고급 최적화)

Hot Method 감지
  Invocation Counter: 메서드 호출 횟수
  Backedge Counter: 루프 반복 횟수
  Threshold 초과 → Compilation Queue

CompileThreshold
  기본값: 10000 (C2), 2000 (C1)
  조정 가능: -XX:CompileThreshold=<n>

Warm-up
  Interpreter → C1 → C2 전환 과정
  ~10초 소요 (애플리케이션 의존)
  벤치마크는 Warm-up 후 측정

PrintCompilation
  -XX:+PrintCompilation
  컴파일 이벤트 실시간 출력
  Tier, 메서드, 바이트코드 크기 확인

성능 향상
  Interpreter: ~50ns/명령어
  C1: ~10ns/명령어
  C2: ~2ns/명령어
  최대 25배 차이

주의사항
  짧은 실행: 컴파일 오버헤드
  Deoptimization: Interpreter 복귀
  Code Cache: 메모리 제한
```

---

## 🤔 생각해볼 문제

**Q1.** 메서드가 9,999회 호출된 상태에서 1회 더 호출하면 JIT 컴파일이 즉시 시작되는가? 컴파일 큐와 백그라운드 스레드의 역할을 고려해 설명하라.

**Q2.** 다음 두 설정 중 어느 것이 더 빠른 Warm-up을 제공하는가? 각각의 장단점을 설명하라.
- 옵션 A: `-XX:CompileThreshold=1000`
- 옵션 B: `-XX:CompileThreshold=50000`

**Q3.** 왜 JVM은 모든 메서드를 JIT 컴파일하지 않고 Hot Method만 컴파일하는가? 메모리, CPU, Code Cache 관점에서 설명하라.

> 💡 **해설**
>
> **Q1.** 10,000회 호출 시 Counter가 Threshold 초과 → Compilation Queue에 추가됨 (즉시 컴파일 아님). 백그라운드 컴파일러 스레드가 큐에서 작업을 가져와 컴파일 시작 (비동기). 컴파일 중에도 Interpreter 버전 계속 실행됨. 컴파일 완료 후 entry point를 네이티브 코드로 교체 → 이후 호출부터 네이티브 코드 사용. 컴파일 시간: 간단한 메서드 수 ms, 복잡한 메서드 수백 ms. 따라서 10,000회 호출 직후가 아니라 수백~수천 회 더 호출된 후 네이티브 코드 실행.
>
> **Q2.** Warm-up 속도: 옵션 A가 빠름 (1000회 후 컴파일 vs 50000회). 장단점: A — 빠른 응답 시작, 하지만 프로파일링 데이터 부족 → 덜 최적화된 코드 → 최종 성능 약간 낮음. B — 느린 Warm-up, 하지만 50,000회 호출 데이터로 더 정확한 프로파일 → 더 나은 최적화 → 최종 성능 높음. 선택: 짧은 실행 (배치, 테스트) → A. 장시간 실행 (서버) → B (기본값 10000이 중간). 실무에서는 기본값 유지 권장.
>
> **Q3.** ① 메모리 — Code Cache 제한 (기본 ~240MB). 모든 메서드 컴파일하면 수 GB 필요 → 불가능. ② CPU — 컴파일은 CPU 집약적. 수천 메서드 동시 컴파일하면 애플리케이션 스레드 CPU 부족 → 전체 처리량 저하. ③ Pareto 원칙 — 20% 코드가 80% 실행. 80% 코드는 거의 안 쓰임 → 컴파일 낭비. ④ 빠른 시작 — 모든 메서드 컴파일하면 시작 시간 수 초~분 증가. Hot Method만 컴파일하면 즉시 시작 + 필요시만 최적화 → 균형.

---

## 📚 참고 자료

- [HotSpot Compilation](https://github.com/openjdk/jdk/tree/master/src/hotspot/share/compiler)
- [JVM JIT Compiler Overview](https://www.oracle.com/technical-resources/articles/java/architect-evans-pt1.html)
- [Understanding JIT Compilation](https://shipilev.net/jvm/anatomy-quarks/)

---

<div align="center">

**[⬅️ 이전: Interpreter Mechanism](./01-interpreter-mechanism.md)** | **[다음: Tiered Compilation ➡️](./03-tiered-compilation.md)**

</div>

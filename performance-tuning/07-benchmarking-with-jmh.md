# Benchmarking with JMH - JMH 벤치마킹

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JMH는 무엇이며, 왜 필요한가?
- System.nanoTime()으로 측정하면 왜 부정확한가?
- Warm-up, Blackhole, @State는 무엇인가?
- 어떻게 정확한 마이크로벤치마크를 작성하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 단순 시간 측정은 부정확하다

```java
// ❌ 부정확한 벤치마크
long start = System.nanoTime();
for (int i = 0; i < 1000; i++) {
    result = compute(i);
}
long time = System.nanoTime() - start;

문제:
  - JIT 컴파일 미반영
  - Dead Code Elimination
  - Constant Folding
  - Loop Unrolling
  → 실제 성능과 다름
```

JMH는 **정확한 마이크로벤치마크 도구**다.

---

## 📐 JMH 기본

### 1. JMH 특징

```
개발:
  OpenJDK 팀 (Oracle)

목적:
  JVM 최적화 고려한 벤치마크

기능:
  - JIT Warm-up 자동
  - Dead Code Elimination 방지
  - Fork 모드 (독립 JVM)
  - 통계 분석

Java 버전:
  Java 8+
```

---

### 2. 프로젝트 설정

```xml
<!-- Maven pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.openjdk.jmh</groupId>
        <artifactId>jmh-core</artifactId>
        <version>1.37</version>
    </dependency>
    <dependency>
        <groupId>org.openjdk.jmh</groupId>
        <artifactId>jmh-generator-annprocess</artifactId>
        <version>1.37</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.2.4</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals><goal>shade</goal></goals>
                    <configuration>
                        <transformers>
                            <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                <mainClass>org.openjdk.jmh.Main</mainClass>
                            </transformer>
                        </transformers>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

---

### 3. 기본 벤치마크

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
public class StringBenchmark {
    
    @Benchmark
    public String stringConcat() {
        String result = "";
        for (int i = 0; i < 10; i++) {
            result += "a";
        }
        return result;
    }
    
    @Benchmark
    public String stringBuilder() {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 10; i++) {
            sb.append("a");
        }
        return sb.toString();
    }
}
```

---

### 4. 실행

```bash
# 빌드
mvn clean package

# 실행
java -jar target/benchmarks.jar

# 출력:
# Benchmark                           Mode  Cnt   Score    Error  Units
# StringBenchmark.stringConcat        avgt   25  245.678 ±  5.123  ns/op
# StringBenchmark.stringBuilder       avgt   25   45.123 ±  1.234  ns/op

# stringBuilder가 5배 빠름
```

---

## 🔧 JMH 핵심 개념

### 1. @BenchmarkMode

```java
@BenchmarkMode(Mode.Throughput)  // 처리량 (ops/s)
@BenchmarkMode(Mode.AverageTime) // 평균 시간
@BenchmarkMode(Mode.SampleTime)  // 샘플링 시간 분포
@BenchmarkMode(Mode.SingleShotTime) // 단일 실행 (Warm-up 없음)
@BenchmarkMode({Mode.Throughput, Mode.AverageTime}) // 복수

예:
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
public void test() {
    // 초당 ops 측정
}
```

---

### 2. @State (상태 공유)

```java
@State(Scope.Thread)  // 스레드별 독립
@State(Scope.Benchmark) // 모든 스레드 공유
@State(Scope.Group)  // 그룹별 공유

예:
@State(Scope.Thread)
public class MyState {
    List<String> list;
    
    @Setup
    public void setup() {
        list = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            list.add("item" + i);
        }
    }
}

@Benchmark
public void testList(MyState state) {
    state.list.get(500);
}
```

---

### 3. @Setup / @TearDown

```java
@State(Scope.Thread)
public class DatabaseBenchmark {
    Connection conn;
    
    @Setup(Level.Trial)  // 전체 벤치마크 1회
    public void setupTrial() {
        // DB 연결
        conn = DriverManager.getConnection(...);
    }
    
    @Setup(Level.Iteration)  // 각 Iteration마다
    public void setupIteration() {
        // 테스트 데이터 준비
    }
    
    @TearDown(Level.Trial)
    public void tearDown() {
        conn.close();
    }
}

Level:
  Trial: 전체 1회
  Iteration: Iteration마다
  Invocation: 메서드 호출마다
```

---

### 4. Blackhole (최적화 방지)

```java
// ❌ Dead Code Elimination
@Benchmark
public void badBenchmark() {
    int result = compute();  // JIT가 제거 가능
}

// ✅ Blackhole 사용
@Benchmark
public void goodBenchmark(Blackhole bh) {
    int result = compute();
    bh.consume(result);  // JIT가 제거 못 함
}

// ✅ 또는 return
@Benchmark
public int goodBenchmark2() {
    return compute();  // return도 효과적
}
```

---

### 5. @Warmup / @Measurement

```java
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS)
public class MyBenchmark {
    @Benchmark
    public void test() {
        // Warm-up 5회 × 1초
        // Measurement 10회 × 1초
    }
}

기본값:
  Warm-up: 5 iterations
  Measurement: 5 iterations
```

---

### 6. @Fork (JVM 독립 실행)

```java
@Fork(value = 3, jvmArgs = {"-Xms2g", "-Xmx2g"})
public class MyBenchmark {
    // 3개 독립 JVM에서 실행
    // 각 JVM: -Xms2g -Xmx2g
}

이유:
  JVM 간 간섭 방지
  프로파일 유도 최적화(PGO) 영향 제거
```

---

## 💻 실전 벤치마크 예시

### 예시 1: HashMap vs ConcurrentHashMap

```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
@Fork(1)
public class MapBenchmark {
    
    Map<String, String> hashMap = new HashMap<>();
    Map<String, String> concurrentMap = new ConcurrentHashMap<>();
    
    @Setup
    public void setup() {
        for (int i = 0; i < 1000; i++) {
            String key = "key" + i;
            hashMap.put(key, "value" + i);
            concurrentMap.put(key, "value" + i);
        }
    }
    
    @Benchmark
    public String testHashMap() {
        return hashMap.get("key500");
    }
    
    @Benchmark
    public String testConcurrentMap() {
        return concurrentMap.get("key500");
    }
}
```

```bash
# 결과:
# Benchmark                            Mode  Cnt    Score    Error   Units
# MapBenchmark.testHashMap            thrpt    5  45000.0 ± 1000.0  ops/ms
# MapBenchmark.testConcurrentMap      thrpt    5  40000.0 ±  800.0  ops/ms

# HashMap이 약간 빠름 (단일 스레드)
```

---

### 예시 2: 파라미터 테스트

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
public class SortBenchmark {
    
    @Param({"10", "100", "1000", "10000"})
    int size;
    
    int[] array;
    
    @Setup
    public void setup() {
        array = new int[size];
        for (int i = 0; i < size; i++) {
            array[i] = ThreadLocalRandom.current().nextInt();
        }
    }
    
    @Benchmark
    public void bubbleSort() {
        bubbleSortImpl(array.clone());
    }
    
    @Benchmark
    public void quickSort() {
        Arrays.sort(array.clone());
    }
}
```

```bash
# 결과:
# Benchmark               (size)  Mode  Cnt    Score
# SortBenchmark.bubbleSort    10  avgt    5    150 ns/op
# SortBenchmark.bubbleSort   100  avgt    5   8000 ns/op
# SortBenchmark.bubbleSort  1000  avgt    5 800000 ns/op
# SortBenchmark.quickSort     10  avgt    5    100 ns/op
# SortBenchmark.quickSort    100  avgt    5   1500 ns/op
# SortBenchmark.quickSort   1000  avgt    5  25000 ns/op

# quickSort 압도적 (O(n log n) vs O(n²))
```

---

## 🚫 흔한 실수

### 실수 1: Warm-up 부족

```java
// ❌ Warm-up 없음
@Warmup(iterations = 0)
@Benchmark
public void test() {
    // JIT 컴파일 전 측정
    // → 부정확
}

// ✅ 충분한 Warm-up
@Warmup(iterations = 5, time = 1)
@Benchmark
public void test() {
    // JIT 최적화 후 측정
}
```

---

### 실수 2: Dead Code Elimination

```java
// ❌ JIT가 제거
@Benchmark
public void bad() {
    compute();  // 결과 사용 안 함 → 제거됨
}

// ✅ Blackhole 또는 return
@Benchmark
public int good() {
    return compute();
}
```

---

### 실수 3: Loop Unrolling

```java
// ❌ 루프가 풀림
@Benchmark
public void bad() {
    for (int i = 0; i < 10; i++) {
        compute();
    }
    // JIT가 10번 인라인
}

// ✅ @OperationsPerInvocation
@Benchmark
@OperationsPerInvocation(10)
public void good() {
    for (int i = 0; i < 10; i++) {
        compute();
    }
}
```

---

## 📌 핵심 정리

```
JMH
  Java Microbenchmark Harness
  OpenJDK 공식 벤치마크 도구

핵심 기능
  JIT Warm-up 자동
  Dead Code 방지 (Blackhole)
  Fork 모드 (독립 JVM)
  통계 분석

주요 어노테이션
  @Benchmark: 측정 메서드
  @BenchmarkMode: 측정 모드
  @State: 상태 공유 범위
  @Setup/@TearDown: 초기화/정리
  @Warmup/@Measurement: 반복 설정
  @Fork: JVM 분리

Blackhole
  결과 소비 (최적화 방지)
  bh.consume(result)

@State Scope
  Thread: 스레드별
  Benchmark: 전체 공유
  Group: 그룹별

@Param
  파라미터 조합 테스트
  @Param({"10", "100"})

Best Practice
  충분한 Warm-up (5+ iterations)
  Blackhole 또는 return 사용
  Fork 모드 활성화
  통계적 유의성 확인
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 벤치마크가 부정확한 이유를 설명하고, 수정하라.

```java
@Benchmark
public void testCompute() {
    int result = 0;
    for (int i = 0; i < 1000; i++) {
        result += i;
    }
}
```

**Q2.** @State(Scope.Thread)와 @State(Scope.Benchmark)의 차이를 설명하고, 각각 언제 사용하는가?

**Q3.** JMH에서 Warm-up이 왜 필요한가? JIT 컴파일과 연결해 설명하라.

> 💡 **해설**
>
> **Q1.** 부정확한 이유: ① Dead Code Elimination — result 변수가 사용 안 됨 (메서드 밖으로 반환 안 됨) → JIT가 전체 루프 제거 가능. ② Loop Unrolling — JIT가 루프를 풀어서 인라인 → 실제 성능과 다름. ③ Constant Folding — i값이 컴파일 시점에 알려짐 → JIT가 result = 499500 (상수)로 최적화. 수정: `@Benchmark public int testCompute() { int result = 0; for (int i = 0; i < 1000; i++) { result += i; } return result; }` 또는 Blackhole 사용. + `@OperationsPerInvocation(1000)` 추가.
>
> **Q2.** Scope 차이: ① Scope.Thread — 각 벤치마크 스레드가 독립 State 인스턴스 사용. 멀티스레드 벤치마크에서 스레드 간 간섭 없음. 사용: 단일 스레드 테스트, 스레드별 독립 데이터. ② Scope.Benchmark — 모든 스레드가 1개 State 인스턴스 공유. 멀티스레드에서 경쟁/동기화 테스트. 사용: ConcurrentHashMap 성능, Lock 경쟁. 예: Thread로 ArrayList (비안전), Benchmark로 ConcurrentHashMap (안전) 테스트.
>
> **Q3.** Warm-up 필요 이유: ① JIT 컴파일 — Java는 인터프리터로 시작 → 느림. 반복 실행 시 JIT가 Hot Spot 탐지 → Native 코드로 컴파일 → 빠름. ② C1 (Client) → C2 (Server) — Tiered Compilation: 처음 C1 (빠른 컴파일), 나중 C2 (최적화). ③ Warm-up 없이 측정 → 인터프리터/C1 성능 → 부정확. Warm-up 후 측정 → C2 최적화 반영 → 실제 성능. ④ 권장: 5+ iterations, 1초씩 → JIT 안정화.

---

## 📚 참고 자료

- [JMH Official Samples](https://github.com/openjdk/jmh/tree/master/jmh-samples/src/main/java/org/openjdk/jmh/samples)
- [JMH Visual Guide](https://www.baeldung.com/java-microbenchmark-harness)

---

<div align="center">

**[⬅️ 이전: Memory Leak Analysis](./06-memory-leak-analysis.md)** | **[홈으로 🏠](../README.md)**

</div>

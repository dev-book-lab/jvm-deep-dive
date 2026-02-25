# JIT Optimizations - JIT 최적화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JIT 컴파일러는 어떤 최적화를 수행하는가?
- Inlining은 왜 가장 중요한 최적화인가?
- Escape Analysis는 무엇이며, 어떻게 객체 할당을 제거하는가?
- Loop Unrolling과 Vectorization은 어떻게 루프를 가속화하는가?
- Dead Code Elimination은 언제 발생하며, 어떤 코드가 제거되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 바이트코드는 일반적이고 추상적이다

```java
int sum(int a, int b) {
    return a + b;
}

int result = sum(10, 20);
```

```
바이트코드:
  iload_1      // a
  iload_2      // b
  iadd         // a + b
  ireturn
  
  aload_0      // this
  bipush 10
  bipush 20
  invokevirtual sum

문제:
  - 메서드 호출 오버헤드 (스택 프레임 생성)
  - 상수인데도 변수처럼 처리 (10, 20)
  - 불필요한 코드 (this 로드)
```

JIT 컴파일러는 이를 **최적화된 기계어**로 변환한다.

---

## 📐 내부 구조

### 1. Inlining — 가장 중요한 최적화

```
Inlining: 메서드 호출을 본문으로 대체

원본:
  int result = sum(10, 20);
  
  int sum(int a, int b) {
      return a + b;
  }

Inlining 후:
  int result = 10 + 20;

효과:
  - 메서드 호출 오버헤드 제거
  - 추가 최적화 기회 (Constant Folding)
  
  최종:
  int result = 30;  // 컴파일 타임에 계산
  
  기계어:
  mov eax, 30  // 단 1개 명령어!

Inlining 조건:
  - 메서드 크기 < 35 bytes (MaxInlineSize)
  - Hot Method (자주 호출)
  - final, private, static 또는 단일 구현체
```

---

### 2. Escape Analysis

```
Escape Analysis: 객체가 메서드 밖으로 탈출하는지 분석

탈출하지 않는 경우:

void process() {
    Point p = new Point(1, 2);
    int sum = p.x + p.y;
    // p는 여기서만 사용, 밖으로 안 나감
}

최적화:
  1. Scalar Replacement
     객체를 스칼라 변수로 분해
     
     int x = 1;
     int y = 2;
     int sum = x + y;
     
     → new Point() 제거!
     → Heap 할당 제거
  
  2. Stack Allocation (대안)
     객체를 스택에 할당
     → GC 압박 감소

탈출하는 경우:

Point createPoint() {
    return new Point(1, 2);  // 메서드 밖으로 반환
}

→ Heap 할당 유지 (최적화 불가)

효과:
  - 객체 할당 제거 → 메모리 절약
  - GC 압박 감소 → 처리량 향상
  - CPU 캐시 효율 증가
```

---

### 3. Constant Folding & Propagation

```
Constant Folding: 컴파일 타임 상수 계산

int x = 2 + 3;
→
int x = 5;

int y = x * 10;
→
int y = 50;

복잡한 예:
  int a = 10;
  int b = 20;
  int c = a + b;
  int d = c * 2;
  
  최적화 후:
  int d = 60;

Constant Propagation:
  final int SIZE = 100;
  for (int i = 0; i < SIZE; i++) { ... }
  
  →
  for (int i = 0; i < 100; i++) { ... }
  → Loop Unrolling 가능
```

---

### 4. Dead Code Elimination

```
Dead Code: 실행되지 않거나 결과가 사용되지 않는 코드

if (false) {
    System.out.println("Never executed");
}
→ 완전히 제거

int x = compute();  // 결과 사용 안 함
→ compute() 부작용 없으면 제거

try {
    return result;
} finally {
    cleanup();
}

→ finally는 항상 실행되므로 유지

예:
  int x = 10;
  x = 20;  // 첫 번째 할당은 dead
  return x;
  
  최적화:
  int x = 20;
  return x;
  
  더 최적화:
  return 20;
```

---

### 5. Loop Unrolling

```
Loop Unrolling: 루프를 펼쳐서 반복 오버헤드 감소

원본:
  for (int i = 0; i < 4; i++) {
      sum += arr[i];
  }

Unrolling:
  sum += arr[0];
  sum += arr[1];
  sum += arr[2];
  sum += arr[3];

효과:
  - 루프 카운터 증가/비교 제거
  - 분기 예측 실패 감소
  - CPU 파이프라인 효율 증가

Partial Unrolling:
  for (int i = 0; i < 100; i++) {
      sum += arr[i];
  }
  
  →
  for (int i = 0; i < 100; i += 4) {
      sum += arr[i];
      sum += arr[i+1];
      sum += arr[i+2];
      sum += arr[i+3];
  }
```

---

### 6. Vectorization (SIMD)

```
Vectorization: 단일 명령어로 여러 데이터 처리

원본:
  for (int i = 0; i < 8; i++) {
      c[i] = a[i] + b[i];
  }

SIMD (AVX2):
  // 한 번에 8개 int 처리
  __m256i va = _mm256_loadu_si256((__m256i*)a);
  __m256i vb = _mm256_loadu_si256((__m256i*)b);
  __m256i vc = _mm256_add_epi32(va, vb);
  _mm256_storeu_si256((__m256i*)c, vc);

효과:
  8번 덧셈 → 1번 SIMD 덧셈
  → 8배 빠름 (이론상)

조건:
  - 루프가 단순해야 함
  - 배열 접근이 순차적
  - 의존성 없음
  - CPU가 SIMD 지원 (SSE, AVX)
```

---

### 7. Range Check Elimination

```
Range Check: 배열 경계 검사

arr[i]
→
if (i < 0 || i >= arr.length) throw ArrayIndexOutOfBoundsException;
value = arr[i];

최적화 (루프 내):

for (int i = 0; i < arr.length; i++) {
    sum += arr[i];  // i는 항상 범위 내
}

→ Range Check 제거
→ 수백만 번 루프에서 큰 효과

효과:
  - 분기 제거
  - 명령어 수 감소
  - 5~10% 성능 향상
```

---

## 💻 실험으로 확인하기

### 실험 1: Inlining 확인

```java
public class InliningDemo {
    private int add(int a, int b) {
        return a + b;
    }
    
    public void test() {
        int result = 0;
        for (int i = 0; i < 1_000_000; i++) {
            result = add(result, i);
        }
        System.out.println(result);
    }
}
```

```bash
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintInlining \
     InliningDemo

# 출력:
#   @ 5   InliningDemo::add (4 bytes)   inline
# → add() 메서드가 인라인됨
```

---

### 실험 2: Escape Analysis 확인

```java
public class EscapeAnalysisDemo {
    static class Point {
        int x, y;
        Point(int x, int y) { this.x = x; this.y = y; }
    }
    
    public int compute() {
        Point p = new Point(1, 2);
        return p.x + p.y;
    }
    
    public static void main(String[] args) {
        EscapeAnalysisDemo demo = new EscapeAnalysisDemo();
        for (int i = 0; i < 1_000_000; i++) {
            demo.compute();
        }
    }
}
```

```bash
java -XX:+DoEscapeAnalysis \
     -XX:+PrintEscapeAnalysis \
     -XX:+UnlockDiagnosticVMOptions \
     EscapeAnalysisDemo

# 출력:
# NoEscape(NoEscape) Point
# → Point 객체가 탈출하지 않음
# → Scalar Replacement 적용 가능
```

---

### 실험 3: 최적화 전후 성능 비교

```java
public class OptimizationBenchmark {
    public static void main(String[] args) {
        long start, elapsed;
        int sum;
        
        // Warm-up
        for (int i = 0; i < 10000; i++) {
            compute(new int[1000]);
        }
        
        // 측정
        start = System.nanoTime();
        sum = 0;
        for (int i = 0; i < 100000; i++) {
            sum += compute(new int[1000]);
        }
        elapsed = System.nanoTime() - start;
        
        System.out.println("Time: " + elapsed / 1_000_000 + " ms");
        System.out.println("Sum: " + sum);
    }
    
    static int compute(int[] arr) {
        int sum = 0;
        for (int i = 0; i < arr.length; i++) {
            sum += arr[i];
        }
        return sum;
    }
}
```

```bash
# 모든 최적화 활성화 (기본)
java OptimizationBenchmark
# Time: 50 ms

# Inlining 비활성화
java -XX:-Inline OptimizationBenchmark
# Time: 200 ms (4배 느림)

# Escape Analysis 비활성화
java -XX:-DoEscapeAnalysis OptimizationBenchmark
# Time: 80 ms (더 느림)
```

---

## ⚡ 실무 임팩트

### Inlining을 위한 코드 작성

```java
// ✅ Inlining 가능 (작은 메서드)
private int getX() {
    return x;
}

// ❌ Inlining 어려움 (큰 메서드)
public void process() {
    // 100 lines of code
    // → MaxInlineSize (35 bytes) 초과
}

// ✅ 메서드 분리
public void process() {
    step1();  // 인라인 가능
    step2();  // 인라인 가능
    step3();  // 인라인 가능
}

private void step1() { ... }  // 작은 메서드
private void step2() { ... }
private void step3() { ... }
```

### Escape Analysis를 위한 패턴

```java
// ❌ 탈출함 (최적화 불가)
public Point createPoint() {
    return new Point(1, 2);  // 반환 → 탈출
}

// ✅ 탈출 안 함 (최적화 가능)
public int compute() {
    Point p = new Point(1, 2);  // 지역 사용만
    return p.x + p.y;
}

// ✅ 재사용 패턴
private final Point temp = new Point(0, 0);  // 재사용

public int computeMany(int[][] coords) {
    int sum = 0;
    for (int[] coord : coords) {
        temp.x = coord[0];
        temp.y = coord[1];
        sum += temp.x + temp.y;
    }
    return sum;
}
// → 객체 할당 1회만
```

### Loop Unrolling을 위한 힌트

```java
// JIT가 Unrolling하기 쉬운 루프
for (int i = 0; i < 1000; i++) {
    sum += arr[i];  // 단순, 의존성 없음
}

// Unrolling 어려운 루프
for (int i = 0; i < n; i++) {  // n이 변수
    if (condition) {  // 복잡한 분기
        arr[i] = compute(arr[i-1]);  // 의존성
    }
}

// 수동 Unrolling (필요시)
for (int i = 0; i < 1000; i += 4) {
    sum += arr[i];
    sum += arr[i+1];
    sum += arr[i+2];
    sum += arr[i+3];
}
```

---

## 🚫 흔한 오해

### "항상 모든 최적화가 적용된다"

```
❌ 잘못된 이해:
  JIT 컴파일되면 모든 최적화가 자동으로 된다.

✅ 실제:
  조건에 따라 선택적 적용
  
Inlining:
  - 메서드 크기 제한 (35 bytes)
  - Polymorphic Call Site면 어려움
  
Escape Analysis:
  - 객체가 탈출하면 불가
  - 복잡한 제어 흐름에서 보수적
  
Loop Unrolling:
  - 루프 카운트 불명확하면 제한
  - 복잡한 루프 본문이면 제한
  
Vectorization:
  - 루프가 복잡하면 불가
  - CPU가 SIMD 미지원하면 불가
```

### "수동 최적화가 항상 JIT보다 낫다"

```
❌ 잘못된 이해:
  수동으로 루프 언롤링하면 더 빠르다.

✅ 실제:
  JIT가 더 잘하는 경우 많음
  
이유:
  - JIT는 런타임 정보 활용 (타입, 값)
  - CPU별 최적화 (SSE, AVX)
  - 프로파일링 기반 최적화
  
예:
  수동 Unrolling: 항상 4배수
  JIT Unrolling: 실제 루프 카운트 고려
                CPU 캐시 크기 고려
                
권장:
  간결하게 작성 → JIT에게 맡김
  복잡한 수동 최적화는 오히려 방해
```

### "final을 쓰면 무조건 빨라진다"

```
❌ 잘못된 이해:
  final 키워드가 성능을 향상시킨다.

✅ 실제:
  final은 의미론적 제약일 뿐
  
성능 영향:
  - final 필드: JIT가 이미 최적화 (거의 차이 없음)
  - final 메서드: Inlining 쉬워짐 (약간 유리)
  - final 지역변수: 성능 영향 전혀 없음
  
예:
  final int x = 10;
  int y = 10;
  
  → JIT 컴파일 후 기계어는 동일
  
사용 이유:
  성능보다 불변성 보장, 코드 의도 명확화
```

---

## 📌 핵심 정리

```
주요 최적화

Inlining
  메서드 호출 → 본문 대체
  가장 중요한 최적화
  MaxInlineSize=35 bytes

Escape Analysis
  객체가 메서드 밖으로 안 나가면
  → Scalar Replacement (객체 할당 제거)
  → Stack Allocation

Constant Folding
  컴파일 타임 상수 계산
  2 + 3 → 5

Dead Code Elimination
  사용되지 않는 코드 제거
  if (false) { ... } → 제거

Loop Unrolling
  루프 펼치기 (반복 오버헤드 감소)
  for 4회 → 4줄 코드

Vectorization (SIMD)
  한 명령어로 여러 데이터 처리
  8배 이론 성능 (AVX2)

Range Check Elimination
  배열 경계 검사 제거
  루프 내에서 자동 감지

성능 향상
  Inlining: 2~5배
  Escape Analysis: 1.5~2배
  Loop Unrolling: 1.2~1.5배
  Vectorization: 2~8배

최적화 활성화
  대부분 기본 활성화
  -XX:-Inline: 비활성화 (디버깅용)
  -XX:-DoEscapeAnalysis: EA 비활성화
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 Escape Analysis가 적용 가능한가? 이유를 설명하라.

```java
public List<Point> process() {
    List<Point> result = new ArrayList<>();
    for (int i = 0; i < 100; i++) {
        Point p = new Point(i, i);
        result.add(p);
    }
    return result;
}
```

**Q2.** Inlining이 다른 최적화의 "게이트웨이"라고 불리는 이유를 Constant Folding과 연결해 설명하라.

**Q3.** 다음 두 루프 중 어느 것이 JIT 최적화(Loop Unrolling, Vectorization)에 더 유리한가? 이유를 설명하라.

```java
// 루프 1
for (int i = 0; i < arr.length; i++) {
    sum += arr[i];
}

// 루프 2
int i = 0;
while (i < arr.length) {
    sum += arr[i++];
}
```

> 💡 **해설**
>
> **Q1.** Escape Analysis 불가. Point 객체들이 ArrayList에 추가되고, 그 ArrayList가 메서드 밖으로 반환됨 → 모든 Point 객체가 탈출함. 따라서 Scalar Replacement 불가, Heap 할당 유지. 개선: Point를 int[] 배열로 저장 (x, y를 두 배열에 분리) 또는 Point를 재사용 (pooling).
>
> **Q2.** Inlining이 게이트웨이인 이유: Inlining 전에는 메서드 경계를 넘어 최적화 불가. 예: `int result = add(2, 3);` — Inlining 전: add() 호출, 반환값 사용. Inlining 후: `int result = 2 + 3;` → Constant Folding 가능 → `int result = 5;`. 즉, Inlining이 선행되어야 메서드 내부 상수를 볼 수 있고, 이후 Constant Folding, Dead Code Elimination 등 추가 최적화 연쇄 발생. Inlining이 없으면 대부분의 고급 최적화 불가.
>
> **Q3.** 루프 1이 더 유리. 이유: ① Canonical Form — for 루프는 JIT가 인식하기 쉬운 표준 형태. 초기값, 종료 조건, 증가량이 명확 → Loop Unrolling, Vectorization 적용 쉬움. ② 증가 연산 분리 — `i++`가 배열 접근과 분리되어 있어 의존성 분석 쉬움. 루프 2: while은 for보다 분석 어려움 (초기값, 증가량이 명시적이지 않음). `arr[i++]`는 부작용 (i 증가)과 배열 접근이 합쳐져 의존성 복잡. 결론: for 루프 + 단순한 증가 패턴이 최적.

---

## 📚 참고 자료

- [HotSpot JIT Compiler Optimizations](https://docs.oracle.com/en/java/javase/17/vm/java-hotspot-virtual-machine-performance-enhancements.html)
- [Escape Analysis in HotSpot](https://wiki.openjdk.org/display/HotSpot/EscapeAnalysis)
- [Loop Optimizations](https://cr.openjdk.java.net/~vlivanov/talks/2017_Vectorization_in_HotSpot_JVM.pdf)

---

<div align="center">

**[⬅️ 이전: Tiered Compilation](./03-tiered-compilation.md)** | **[다음: On-Stack Replacement (OSR) ➡️](./05-on-stack-replacement.md)**

</div>

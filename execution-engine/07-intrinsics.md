# JVM Intrinsics - JVM 내장 함수

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JVM Intrinsics는 무엇이며, 왜 필요한가?
- 어떤 메서드가 Intrinsic으로 구현되어 있는가?
- Intrinsic은 어떻게 CPU 명령어로 직접 대체되는가?
- `Math.sqrt()`가 일반 메서드보다 빠른 이유는 무엇인가?
- Intrinsic을 비활성화하면 어떻게 되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 일부 메서드는 최적화에 한계가 있다

```java
double result = Math.sqrt(x);
```

```
일반 메서드 호출:
  1. sqrt() 메서드 진입
  2. 바이트코드 실행 (또는 JIT 코드)
  3. 제곱근 알고리즘 실행
  4. 반환
  
  문제:
  - 메서드 호출 오버헤드
  - 알고리즘 구현의 한계
  - CPU에는 sqrt 명령어가 있는데 안 씀

CPU 명령어:
  sqrtsd xmm0, xmm1  // 단 1개 명령어로 제곱근
  → 훨씬 빠름 (수십 배)
```

JVM Intrinsics는 **특정 메서드를 CPU 명령어로 직접 대체**한다.

---

## 📐 내부 구조

### 1. Intrinsic 개요

```
Intrinsic: JVM이 특별히 인식하는 메서드
  - 바이트코드 무시
  - CPU 명령어로 직접 대체
  - 또는 최적화된 네이티브 코드 삽입

예시:

Math.sqrt(x)
→ sqrtsd xmm0, [x]  // x86 SSE 명령어

System.arraycopy(src, 0, dest, 0, len)
→ rep movsq  // x86 고속 메모리 복사

String.equals(other)
→ SIMD 비교 (AVX2)

Object.hashCode()
→ 특수 알고리즘 (Identity Hash)
```

---

### 2. 주요 Intrinsic 메서드

```
java.lang.Math:
  abs, min, max
  sqrt, cbrt
  sin, cos, tan
  exp, log, log10, pow
  
java.lang.System:
  arraycopy
  currentTimeMillis
  nanoTime
  identityHashCode
  
java.lang.Object:
  getClass
  hashCode
  clone
  
java.lang.String:
  equals
  compareTo
  indexOf
  charAt (일부 조건)
  
java.lang.Integer (Long, Float, Double):
  bitCount
  numberOfLeadingZeros
  numberOfTrailingZeros
  reverseBytes
  
java.util.Arrays:
  copyOf
  copyOfRange
  fill
  sort (일부)
  
sun.misc.Unsafe:
  대부분의 메서드 (메모리 직접 접근)
  compareAndSwapInt
  getAndAddLong
  park, unpark
```

---

### 3. Intrinsic 코드 생성 과정

```
JIT 컴파일 중:

1. 메서드 호출 발견
   invokestatic Math.sqrt (D)D

2. Intrinsic 확인
   if (method == Math.sqrt && CPU.supports_SSE2) {
       generate_intrinsic();
   } else {
       generate_call();
   }

3. CPU 명령어 생성
   movsd xmm0, [rbp-8]  // x 로드
   sqrtsd xmm0, xmm0    // 제곱근 계산
   // 결과는 xmm0에 (반환값)

일반 호출 (Intrinsic 아님):
   push x
   call Math.sqrt
   add rsp, 8
   → 훨씬 느림
```

---

### 4. System.arraycopy Intrinsic

```
일반 구현:

for (int i = 0; i < len; i++) {
    dest[i] = src[i];
}
→ len번 반복, 경계 체크 등

Intrinsic 구현:

x86:
  mov rcx, len       // 복사할 요소 수
  mov rsi, src_addr  // 소스 주소
  mov rdi, dest_addr // 대상 주소
  rep movsq          // 고속 블록 복사
  
ARM:
  NEON SIMD 명령어 사용
  
효과:
  일반 루프 대비 5~10배 빠름
```

---

### 5. String.equals Intrinsic

```
일반 구현:

public boolean equals(Object obj) {
    if (obj == this) return true;
    if (!(obj instanceof String)) return false;
    String other = (String) obj;
    if (value.length != other.value.length) return false;
    for (int i = 0; i < value.length; i++) {
        if (value[i] != other.value[i]) return false;
    }
    return true;
}

Intrinsic 구현 (AVX2):

// 한 번에 16개 char 비교
__m256i v1 = _mm256_loadu_si256((__m256i*)str1);
__m256i v2 = _mm256_loadu_si256((__m256i*)str2);
__m256i cmp = _mm256_cmpeq_epi16(v1, v2);
int mask = _mm256_movemask_epi8(cmp);
if (mask != 0xFFFFFFFF) return false;

효과:
  16배 빠름 (이론상)
  실제로는 5~8배 빠름
```

---

## 💻 실험으로 확인하기

### 실험 1: Math.sqrt Intrinsic 확인

```java
public class IntrinsicDemo {
    public static void main(String[] args) {
        double sum = 0;
        
        // Warm-up
        for (int i = 0; i < 100000; i++) {
            sum += Math.sqrt(i);
        }
        
        // 측정
        long start = System.nanoTime();
        sum = 0;
        for (int i = 0; i < 10_000_000; i++) {
            sum += Math.sqrt(i);
        }
        long elapsed = System.nanoTime() - start;
        
        System.out.println("Time: " + elapsed / 1_000_000 + " ms");
        System.out.println("Sum: " + sum);
    }
}
```

```bash
# Intrinsic 활성화 (기본)
java IntrinsicDemo
# Time: 50 ms

# Intrinsic 비활성화
java -XX:+UnlockDiagnosticVMOptions \
     -XX:DisableIntrinsic=_dsqrt \
     IntrinsicDemo
# Time: 500 ms (10배 느림)
```

---

### 실험 2: System.arraycopy 성능

```java
public class ArrayCopyBenchmark {
    public static void main(String[] args) {
        int[] src = new int[1_000_000];
        int[] dest = new int[1_000_000];
        
        for (int i = 0; i < src.length; i++) {
            src[i] = i;
        }
        
        // arraycopy (Intrinsic)
        long start = System.nanoTime();
        for (int i = 0; i < 100; i++) {
            System.arraycopy(src, 0, dest, 0, src.length);
        }
        long intrinsic = System.nanoTime() - start;
        
        // 수동 복사
        start = System.nanoTime();
        for (int i = 0; i < 100; i++) {
            for (int j = 0; j < src.length; j++) {
                dest[j] = src[j];
            }
        }
        long manual = System.nanoTime() - start;
        
        System.out.println("arraycopy: " + intrinsic / 1_000_000 + " ms");
        System.out.println("Manual: " + manual / 1_000_000 + " ms");
        System.out.println("Speedup: " + (double)manual / intrinsic + "x");
    }
}
```

```bash
# 출력:
# arraycopy: 50 ms
# Manual: 300 ms
# Speedup: 6.0x
```

---

### 실험 3: Intrinsic 목록 확인

```bash
# PrintInlining으로 Intrinsic 확인
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintInlining \
     IntrinsicDemo

# 출력:
#   @ 5   java.lang.Math::sqrt (implicit intrinsic)
#          ↑ Intrinsic으로 처리됨 (인라인 아님)

# 모든 Intrinsic 목록
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintIntrinsics \
     -version

# 출력:
#   Intrinsic: _dsqrt
#   Intrinsic: _dsin
#   Intrinsic: _dcos
#   Intrinsic: _arraycopy
#   Intrinsic: _hashCode
#   ...
```

---

## ⚡ 실무 임팩트

### Math 메서드 사용 권장

```java
// ✅ Intrinsic 사용 (빠름)
double result = Math.sqrt(x);
double angle = Math.sin(radians);
int min = Math.min(a, b);

// ❌ 직접 구현 (느림)
double result = customSqrt(x);  // 뉴턴-랩슨 방법
double angle = taylorSin(radians);  // 테일러 급수
int min = (a < b) ? a : b;  // Math.min과 동일하게 컴파일됨
```

### System.arraycopy vs Arrays.copyOf

```java
// 둘 다 Intrinsic
int[] dest1 = new int[len];
System.arraycopy(src, 0, dest1, 0, len);

int[] dest2 = Arrays.copyOf(src, len);
// 내부적으로 System.arraycopy 호출

// 성능은 동일
// Arrays.copyOf가 편리함 (새 배열 생성 자동)
```

### Unsafe 사용 (고급)

```java
import sun.misc.Unsafe;

// Unsafe의 대부분 메서드가 Intrinsic
Unsafe unsafe = getUnsafe();

// CAS (Compare-And-Swap) - Intrinsic
boolean success = unsafe.compareAndSwapInt(obj, offset, expected, newValue);
// → x86: lock cmpxchg 명령어 (atomic)

// 메모리 직접 접근 - Intrinsic
long value = unsafe.getLong(address);
unsafe.putLong(address, value);
// → 직접 메모리 읽기/쓰기 (매우 빠름)

// AtomicInteger 내부 구현이 Unsafe CAS 사용
```

---

## 🚫 흔한 오해

### "Intrinsic은 항상 사용된다"

```
❌ 잘못된 이해:
  Math.sqrt()를 쓰면 무조건 Intrinsic이다.

✅ 실제:
  조건이 맞아야 함
  
조건:
  1. CPU 지원
     sqrt → SSE2 명령어 필요
     AVX2 SIMD → AVX2 지원 CPU
     
  2. JIT 컴파일됨
     Interpreter 모드에서는 일반 호출
     
  3. 명시적으로 비활성화 안 됨
     -XX:DisableIntrinsic=_dsqrt
     
  4. JDK 버전
     일부 Intrinsic은 최신 JDK만 지원

확인 방법:
  -XX:+PrintInlining
  "implicit intrinsic" 표시 확인
```

### "Intrinsic은 Java 코드를 우회한다"

```
❌ 잘못된 이해:
  Intrinsic은 Java 구현을 완전히 무시한다.

✅ 실제:
  Fallback 메커니즘 존재
  
흐름:
  1. JIT 컴파일 시도
  2. Intrinsic 조건 확인
     CPU 지원? → YES → Intrinsic 생성
               → NO  → Java 코드 컴파일
  
예:
  Math.sqrt() on CPU without SSE2
  → Java 구현 (또는 JNI 호출)
  
이유:
  플랫폼 독립성 유지
  모든 CPU에서 동작 보장
```

### "Intrinsic을 직접 만들 수 있다"

```
❌ 잘못된 이해:
  어노테이션으로 Intrinsic 등록 가능하다.

✅ 실제:
  JVM 내부에 하드코딩됨
  
Intrinsic 추가 과정:
  1. JVM C++ 코드 수정
     src/hotspot/share/opto/library_call.cpp
  
  2. 각 CPU 아키텍처별 구현
     x86: src/hotspot/cpu/x86/...
     ARM: src/hotspot/cpu/aarch64/...
  
  3. JVM 재컴파일
  
  → 일반 개발자는 불가능
  → OpenJDK 컨트리뷰션으로만 가능

대안:
  JNI로 네이티브 코드 호출
  (하지만 Intrinsic만큼 빠르지 않음)
```

---

## 📌 핵심 정리

```
JVM Intrinsics
  특정 메서드를 CPU 명령어로 직접 대체
  바이트코드 우회
  매우 빠름 (5~20배)

주요 Intrinsic
  Math: sqrt, sin, cos, exp, log, abs, min, max
  System: arraycopy, currentTimeMillis, nanoTime
  String: equals, compareTo, indexOf
  Integer: bitCount, numberOfLeadingZeros
  Unsafe: CAS, 메모리 직접 접근

동작 원리
  JIT 컴파일 시 메서드 확인
  → Intrinsic 조건 충족 시 CPU 명령어 생성
  → 메서드 호출 오버헤드 제거

성능 향상
  Math.sqrt: 10배
  System.arraycopy: 5~10배
  String.equals: 5~8배 (SIMD)

조건
  CPU 지원 (SSE2, AVX2 등)
  JIT 컴파일됨
  명시적 비활성화 안 됨

비활성화
  -XX:DisableIntrinsic=_dsqrt
  (디버깅, 성능 비교용)

확인 방법
  -XX:+PrintInlining
  "implicit intrinsic" 표시

실무 권장
  Math 메서드 적극 사용
  System.arraycopy 사용
  직접 구현보다 표준 API
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 두 코드의 성능 차이를 Intrinsic 관점에서 설명하라.

```java
// 코드 1
double sum = 0;
for (int i = 0; i < 1_000_000; i++) {
    sum += Math.sqrt(i);
}

// 코드 2
double sum = 0;
for (int i = 0; i < 1_000_000; i++) {
    sum += customSqrt(i);  // 뉴턴-랩슨 구현
}
```

**Q2.** `System.arraycopy()`가 Intrinsic이 아니었다면 어떻게 구현될까? 일반 루프 구현과 비교해 성능 차이를 설명하라.

**Q3.** Interpreter 모드(`-Xint`)에서 `Math.sqrt()`를 호출하면 Intrinsic이 사용되는가? 그 이유를 설명하라.

> 💡 **해설**
>
> **Q1.** 코드 1 (Math.sqrt): JIT 컴파일 후 `sqrtsd` 명령어로 직접 대체 (SSE2) → 1개 명령어, ~3 사이클. 메서드 호출 없음, 인라인됨. 코드 2 (customSqrt): 뉴턴-랩슨 방법은 반복 알고리즘 (3~5회 반복) → 수십 개 명령어, ~50 사이클. 메서드 호출 오버헤드 (인라인되어도 복잡). 성능 차이: 10~20배. Math.sqrt가 압도적으로 빠름. 권장: 항상 Math.sqrt 사용, 재발명 금지.
>
> **Q2.** Intrinsic 없다면: Java 구현 (for 루프) 또는 JNI 호출. 루프 구현: `for (int i = 0; i < len; i++) dest[i] = src[i];` → len번 반복, 매 반복마다 경계 체크 (ArrayIndexOutOfBoundsException), 메모리 접근 (비효율적 패턴). Intrinsic 구현: x86 `rep movsq` → 블록 복사 (한 번에 8바이트씩), CPU 최적화 (캐시 프리페칭, 파이프라인 활용), 경계 체크 1회만. 성능: 루프 100ms, Intrinsic 10ms → 10배 차이. 대용량 배열일수록 차이 커짐.
>
> **Q3.** 사용 안 됨. 이유: Intrinsic은 JIT 컴파일 과정에서 생성됨. Interpreter는 바이트코드를 직접 실행 → JIT 컴파일 없음 → Intrinsic 생성 기회 없음. Interpreter 모드에서는 Math.sqrt()의 Java 구현 (또는 JNI) 호출 → 일반 메서드처럼 동작. 따라서 성능: Interpreter 500ms, JIT+Intrinsic 50ms → 10배 차이. Intrinsic의 이점은 JIT 컴파일 시에만 발휘됨.

---

## 📚 참고 자료

- [HotSpot Intrinsics](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/opto/library_call.cpp)
- [JVM Intrinsics List](https://shipilev.net/jvm/anatomy-quarks/16-megamorphic-virtual-calls/)
- [Adding New Intrinsics to HotSpot](https://cr.openjdk.java.net/~vlivanov/talks/2015_JIT_Intrinsics.pdf)

---

<div align="center">

**[⬅️ 이전: Deoptimization](./06-deoptimization.md)** | **[홈으로 🏠](../README.md)**

</div>

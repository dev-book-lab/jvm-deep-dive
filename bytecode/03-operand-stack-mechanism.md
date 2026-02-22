# Operand Stack Mechanism - 피연산자 스택 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Operand Stack은 무엇이며, 바이트코드 실행 시 어떤 역할을 하는가?
- 스택 기반 VM과 레지스터 기반 VM의 차이는 무엇인가?
- `max_stack`은 어떻게 결정되며, 왜 컴파일 타임에 알 수 있는가?
- 바이트코드 명령어는 스택을 어떻게 조작하는가?
- 스택 기반 설계가 플랫폼 독립성에 어떻게 기여하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 바이트코드 연산의 중간 값을 어디에 저장할 것인가

```java
int result = (a + b) * (c - d);
```

```
기계어 방식 (레지스터 기반, x86):
  mov eax, [a]
  add eax, [b]      // eax = a + b
  mov ebx, [c]
  sub ebx, [d]      // ebx = c - d
  imul eax, ebx     // eax = eax * ebx
  mov [result], eax
  
  → CPU 레지스터(eax, ebx) 사용
  → 플랫폼 종속적

JVM 방식 (스택 기반):
  iload_1           // a → Stack
  iload_2           // b → Stack
  iadd              // Stack: [a+b]
  iload_3           // c → Stack
  iload 4           // d → Stack
  isub              // Stack: [a+b, c-d]
  imul              // Stack: [(a+b)*(c-d)]
  istore 5          // result 저장
  
  → 가상 스택 사용
  → 플랫폼 독립적
```

Operand Stack은 **추상화된 작업 공간**이다.

---

## 📐 내부 구조

### 1. Stack Frame과 Operand Stack

```
하나의 메서드 실행 시 Stack Frame:

┌─────────────────────────────────────┐
│          Stack Frame                │
├─────────────────────────────────────┤
│  Local Variable Array (LVA)         │
│  [this] [arg1] [arg2] [local1] ...  │
├─────────────────────────────────────┤
│  Operand Stack                      │
│  ┌──────┐                           │
│  │  15  │ ← top                     │
│  ├──────┤                           │
│  │  10  │                           │
│  ├──────┤                           │
│  │   5  │                           │
│  └──────┘                           │
│  (max_stack=3)                      │
├─────────────────────────────────────┤
│  Frame Data                         │
│  (return address, constant pool...) │
└─────────────────────────────────────┘

Operand Stack 특징:
  - LIFO (Last In, First Out)
  - 타입별 슬롯 크기:
    int, float, reference = 1 slot
    long, double = 2 slots
  - max_stack: 컴파일 타임 결정
```

---

### 2. 스택 기반 vs 레지스터 기반

```
스택 기반 (JVM, Python Bytecode):

장점:
  ✓ 플랫폼 독립적 (레지스터 개수 무관)
  ✓ 바이트코드 간결 (피연산자 위치 암시적)
  ✓ 구현 단순

단점:
  ✗ 명령어 수 증가 (push/pop 빈번)
  ✗ 메모리 접근 많음

예 (a + b):
  iload_1    // a → Stack
  iload_2    // b → Stack
  iadd       // a + b
  → 3개 명령어

레지스터 기반 (Dalvik, Lua):

장점:
  ✓ 명령어 수 적음
  ✓ 레지스터 접근 빠름

단점:
  ✗ 플랫폼 종속 가능성
  ✗ 레지스터 할당 복잡
  ✗ 바이트코드 길어짐 (명시적 주소)

예 (a + b):
  add r1, r2, r3  // r1 = r2 + r3
  → 1개 명령어 (하지만 3바이트)

JVM 선택: 스택 기반
  이유: 플랫폼 독립성 우선
```

---

### 3. max_stack 결정 과정

```
컴파일러(javac)가 max_stack 계산:

void example() {
    int a = 1 + 2;
    int b = a * 3;
}

바이트코드:
  0: iconst_1      // Stack: [1]         depth=1
  1: iconst_2      // Stack: [1, 2]      depth=2  ← 최대
  2: iadd          // Stack: [3]         depth=1
  3: istore_1      // Stack: []          depth=0
  4: iload_1       // Stack: [3]         depth=1
  5: iconst_3      // Stack: [3, 3]      depth=2  ← 최대
  6: imul          // Stack: [9]         depth=1
  7: istore_2      // Stack: []          depth=0
  
  max_stack = 2

복잡한 경우:
  int x = (a + b) * (c + d);
  
  iload_1    // [a]           depth=1
  iload_2    // [a, b]        depth=2
  iadd       // [a+b]         depth=1
  iload_3    // [a+b, c]      depth=2
  iload 4    // [a+b, c, d]   depth=3  ← 최대
  iadd       // [a+b, c+d]    depth=2
  imul       // [(a+b)*(c+d)] depth=1
  
  max_stack = 3

컴파일러 알고리즘:
  모든 실행 경로 탐색
  각 명령어에서 스택 깊이 계산
  최대값을 max_stack으로 설정
```

---

### 4. 스택 조작 명령어

```
기본 조작:

pop     → Stack에서 1개 제거
pop2    → 2개 제거 (또는 long/double 1개)

dup     → top을 복제
  Stack: [a]
  dup 후: [a, a]

dup_x1  → top을 1개 아래에 복제
  Stack: [a, b]
  dup_x1 후: [b, a, b]

dup_x2  → top을 2개 아래에 복제
  Stack: [a, b, c]
  dup_x2 후: [c, a, b, c]

dup2    → top 2개 복제 (또는 long/double 1개)
  Stack: [a, b]
  dup2 후: [a, b, a, b]

swap    → top 2개 교환
  Stack: [a, b]
  swap 후: [b, a]

사용 예 (a = a + 1):
  iload_1      // [a]
  iconst_1     // [a, 1]
  iadd         // [a+1]
  dup          // [a+1, a+1]  ← 복제
  istore_1     // [a+1]       ← 하나는 저장
  // 스택에 a+1 남음 (다음 연산에 사용 가능)
```

---

### 5. 바이트코드 실행 시뮬레이션

```java
int add(int a, int b) {
    return a + b;
}
```

```
초기 상태:
  LVA: [this, a=5, b=10]
  Operand Stack: []

실행:
  0: iload_1
     LVA[1](5) → Stack
     Stack: [5]
  
  1: iload_2
     LVA[2](10) → Stack
     Stack: [5, 10]
  
  2: iadd
     Stack에서 5, 10 pop
     5 + 10 = 15 계산
     15 → Stack
     Stack: [15]
  
  3: ireturn
     Stack에서 15 pop
     caller에게 15 반환
     Stack: []
```

```java
int complex(int x, int y) {
    int a = x + y;
    int b = a * 2;
    return b;
}
```

```
LVA 초기: [this, x=3, y=4]
Stack: []

0: iload_1      // Stack: [3]
1: iload_2      // Stack: [3, 4]
2: iadd         // Stack: [7]        (x+y)
3: istore_3     // Stack: []         a=7 저장
4: iload_3      // Stack: [7]
5: iconst_2     // Stack: [7, 2]
6: imul         // Stack: [14]       (a*2)
7: istore 4     // Stack: []         b=14 저장
8: iload 4      // Stack: [14]
9: ireturn      // Stack: []         return 14
```

---

## 💻 실험으로 확인하기

### 실험 1: max_stack 확인

```java
public class StackDepthDemo {
    public int simple(int a, int b) {
        return a + b;
    }
    
    public int complex(int a, int b, int c, int d) {
        return (a + b) * (c + d);
    }
}
```

```bash
javap -v StackDepthDemo.class

# simple 메서드:
# Code:
#   stack=2, locals=3
#   0: iload_1
#   1: iload_2
#   2: iadd
#   3: ireturn

# complex 메서드:
# Code:
#   stack=3, locals=5
#   0: iload_1
#   1: iload_2
#   2: iadd
#   3: iload_3
#   4: iload 4
#   6: iadd
#   7: imul
#   8: ireturn

# complex에서 stack=3인 이유:
# 4번 명령어 (iload 4) 시점에 스택에 [a+b, c, d] 3개
```

---

### 실험 2: 스택 조작 명령어 동작

```java
public class StackManipulation {
    public int duplicate(int x) {
        int y = x;
        return x + y;
    }
}
```

```bash
javap -c StackManipulation.class

# 출력:
# public int duplicate(int);
#   Code:
#      0: iload_1      // x → Stack: [x]
#      1: dup          // Stack: [x, x]
#      2: istore_2     // y = x, Stack: [x]
#      3: iload_2      // Stack: [x, y]
#      4: iadd         // Stack: [x+y]
#      5: ireturn

# dup 사용으로 iload_1을 한 번만 실행
```

---

### 실험 3: 스택 깊이 시각화 도구

```java
import org.objectweb.asm.*;

public class StackTracer extends ClassVisitor {
    public StackTracer() {
        super(Opcodes.ASM9);
    }
    
    @Override
    public MethodVisitor visitMethod(int access, String name, 
                                     String descriptor, String signature, String[] exceptions) {
        return new MethodVisitor(Opcodes.ASM9) {
            int depth = 0;
            int maxDepth = 0;
            
            @Override
            public void visitInsn(int opcode) {
                // 각 명령어의 스택 변화 추적
                switch (opcode) {
                    case Opcodes.IADD:
                    case Opcodes.ISUB:
                        depth -= 1;  // 2 pop, 1 push → net -1
                        break;
                    case Opcodes.DUP:
                        depth += 1;
                        break;
                    // ...
                }
                maxDepth = Math.max(maxDepth, depth);
                System.out.printf("Opcode: %d, Depth: %d%n", opcode, depth);
            }
            
            @Override
            public void visitEnd() {
                System.out.println("Max Depth: " + maxDepth);
            }
        };
    }
}
```

---

## ⚡ 실무 임팩트

### JIT 컴파일러의 스택 → 레지스터 변환

```
바이트코드 (스택 기반):
  iload_1
  iload_2
  iadd
  istore_3

JIT 컴파일 후 (레지스터 기반, x86):
  mov eax, [rbp-8]   // a
  add eax, [rbp-12]  // a + b
  mov [rbp-16], eax  // c = a + b

변환 과정:
  1. 스택 시뮬레이션
  2. 각 스택 슬롯을 가상 레지스터로 매핑
  3. 레지스터 할당 최적화
  4. 네이티브 코드 생성

이점:
  - 스택 push/pop 제거
  - 레지스터 직접 사용 (빠름)
  - 중간 값을 메모리에 저장 안 함
```

### VerifyError와 스택 불일치

```java
// 바이트코드 수동 조작 시 흔한 실수

// 잘못된 바이트코드:
0: iconst_1     // Stack: [1]
1: iadd         // ERROR! Stack Underflow (2개 필요, 1개만 있음)

→ java.lang.VerifyError: Operand stack underflow

// 타입 불일치:
0: iload_1      // Stack: [int]
1: fadd         // ERROR! int에 fadd 불가

→ java.lang.VerifyError: Bad type on operand stack

Bytecode Verifier:
  클래스 로딩 시 스택 깊이와 타입 검증
  모든 경로에서 max_stack 초과 여부 확인
  타입 안전성 보장
```

### 스택 깊이와 메서드 복잡도

```
max_stack이 큰 메서드 = 복잡한 표현식

예:
  max_stack = 2
  → 간단한 연산 (a + b)
  
  max_stack = 10
  → 복잡한 중첩 연산
  → ((a + b) * (c + d)) / ((e - f) + (g * h))

코드 리뷰 지표:
  max_stack > 10
  → 표현식 분리 권장
  → 가독성 향상
  
  int x = ((a + b) * (c + d)) / ((e - f) + (g * h));
  
  →
  
  int sum1 = a + b;
  int sum2 = c + d;
  int diff = e - f;
  int prod = g * h;
  int numerator = sum1 * sum2;
  int denominator = diff + prod;
  int x = numerator / denominator;
  
  → max_stack 감소, 가독성 증가
```

---

## 🚫 흔한 오해

### "Operand Stack은 메서드 간 공유된다"

```
❌ 잘못된 이해:
  여러 메서드가 하나의 Operand Stack을 공유한다.

✅ 실제:
  각 Stack Frame마다 독립적인 Operand Stack
  
  main() 호출 → foo() 호출:
  
  ┌─────────────────┐
  │ main Frame      │
  │ Operand Stack: []│ ← main의 스택
  └─────────────────┘
  ┌─────────────────┐
  │ foo Frame       │
  │ Operand Stack: []│ ← foo의 스택 (별도)
  └─────────────────┘
  
  foo() 종료 시:
  foo의 Operand Stack pop
  결과를 main의 Operand Stack에 push
```

### "max_stack은 런타임에 동적으로 증가한다"

```
❌ 잘못된 이해:
  스택이 부족하면 자동으로 max_stack이 증가한다.

✅ 실제:
  max_stack은 컴파일 타임에 고정
  런타임에 변경 불가
  
  이유:
  - JVM이 Stack Frame 크기를 미리 할당
  - max_stack 초과 시 VerifyError (로딩 단계)
  - 실행 중 스택 오버플로우 없음 (올바른 바이트코드면)
  
  cf. Thread Stack (-Xss):
  스택 프레임들이 쌓이는 공간 (다름)
  → StackOverflowError 발생 가능
```

### "스택 기반이라 레지스터 기반보다 느리다"

```
❌ 잘못된 이해:
  JVM은 스택 기반이라 Dalvik(레지스터)보다 느리다.

✅ 실제:
  Interpreter 단계: 스택 기반이 약간 느림
  JIT 컴파일 후: 거의 동일
  
  이유:
  JIT 컴파일러가 스택 → 레지스터 변환
  → 최종 기계어는 레지스터 사용
  → 성능 차이 미미
  
  벤치마크 (HotSpot vs Dalvik):
  Warm-up 후 성능 거의 동일
  차이는 GC, JIT 알고리즘에서 발생
```

---

## 📌 핵심 정리

```
Operand Stack
  메서드 실행 시 임시 데이터 저장
  각 Stack Frame마다 독립적
  LIFO 구조

스택 기반 vs 레지스터 기반
  스택: 플랫폼 독립적, 명령어 간결, 구현 단순
  레지스터: 명령어 수 적음, 빠름 (JIT 전)
  JVM 선택: 스택 (플랫폼 독립성)

max_stack
  컴파일 타임 결정
  모든 실행 경로의 최대 스택 깊이
  런타임에 변경 불가

스택 조작 명령어
  pop, dup, swap 등
  중간 값 재사용, 교환 가능

바이트코드 실행
  Load → Stack push
  연산 → Stack pop → 계산 → push
  Store → Stack pop → LVA 저장

JIT 최적화
  스택 → 레지스터 변환
  push/pop 제거
  최종 성능: 레지스터 기반과 동등

Bytecode Verifier
  스택 깊이 검증 (max_stack 초과 여부)
  타입 안전성 검증
  VerifyError 발생 조건
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 Java 코드의 max_stack을 계산하라. 각 바이트코드 실행 시점의 스택 깊이를 추적하라.

```java
int calculate(int a, int b, int c) {
    return (a + b) * c + (a - b);
}
```

**Q2.** `dup` 명령어가 없다면 `a = a + 1` 같은 증가 연산을 어떻게 구현해야 하는가? 바이트코드 명령어 수와 스택 깊이를 비교하라.

**Q3.** 왜 JVM은 레지스터 기반이 아닌 스택 기반을 선택했는가? Android의 Dalvik/ART가 레지스터 기반을 선택한 이유와 비교해 설명하라.

> 💡 **해설**
>
> **Q1.** 바이트코드: `iload_1` [a] depth=1, `iload_2` [a,b] depth=2, `iadd` [a+b] depth=1, `iload_3` [a+b,c] depth=2, `imul` [(a+b)*c] depth=1, `iload_1` [(a+b)*c,a] depth=2, `iload_2` [(a+b)*c,a,b] depth=3 ← 최대, `isub` [(a+b)*c,a-b] depth=2, `iadd` [최종결과] depth=1, `ireturn`. max_stack=3. 핵심: `iload_2` 시점에 스택에 [(a+b)*c, a, b] 3개 동시 존재.
>
> **Q2.** dup 없이: `iload_1` (a), `iconst_1` (1), `iadd` (a+1), `istore_1` (저장), `iload_1` (다시 로드) → 5개 명령어. dup 사용: `iload_1` (a), `iconst_1` (1), `iadd` (a+1), `dup` (복제), `istore_1` → 5개 명령어 (동일). 차이: dup 버전은 스택에 값이 남아 다음 연산에 즉시 사용 가능 (iload 불필요). 예: `b = (a = a + 1) * 2`에서 dup 사용 시 `iload` 1회 절약. 스택 깊이는 둘 다 최대 2.
>
> **Q3.** JVM 스택 선택 이유: ① 플랫폼 독립성 — 레지스터 개수가 CPU마다 다름 (x86=8~16개, ARM=16개), 스택은 가상이라 무관. ② 바이트코드 간결 — 피연산자 위치가 암시적 (항상 스택 top), 레지스터는 명시적 주소 필요. ③ 구현 단순 — Interpreter 구현이 쉬움. Dalvik/ART 레지스터 선택 이유: ① 모바일 최적화 — 배터리/성능 중시, JIT 없던 초기 Android에서 Interpreter 성능 중요. ② Ahead-of-Time(AOT) 컴파일 — ART는 설치 시 AOT 컴파일, 레지스터 기반이 최적화에 유리. ③ 단일 플랫폼 — ARM으로 제한되어 플랫폼 독립성 불필요. 현재는 JVM도 JIT로 레지스터 변환하므로 성능 차이 미미.

---

## 📚 참고 자료

- [JVMS §2.6.2 — Operand Stacks](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html#jvms-2.6.2)
- [Stack-Based vs Register-Based VMs](https://markfaction.wordpress.com/2012/07/15/stack-based-vs-register-based-virtual-machine-architecture/)
- [HotSpot Interpreter Implementation](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/interpreter/bytecodeInterpreter.cpp)

---

<div align="center">

**[⬅️ 이전: Bytecode Instruction Set](./02-bytecode-instruction-set.md)** | **[다음: Method Invocation Instructions ➡️](./04-method-invocation-instructions.md)**

</div>

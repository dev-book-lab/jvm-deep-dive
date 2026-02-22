# Bytecode Instruction Set - 바이트코드 명령어 집합

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JVM 바이트코드는 총 몇 개의 명령어로 구성되며, 어떻게 분류되는가?
- `iload`, `fload`, `aload`처럼 타입별로 명령어가 분리된 이유는 무엇인가?
- `iconst_1`과 `bipush 1`의 차이는 무엇이며, 언제 어느 것을 사용하는가?
- 바이트코드는 왜 스택 기반이며, 피연산자는 어디서 오는가?
- `javap -c`로 본 바이트코드와 실제 바이너리의 관계는?

---

## 🔍 왜 이게 존재하는가

### 문제: Java 소스 코드를 어떤 중간 언어로 변환할 것인가

```java
int sum = a + b;
```

```
기계어로 직접 컴파일 (C/C++ 방식):
  mov eax, [a]
  add eax, [b]
  mov [sum], eax
  → CPU 종속적 (x86, ARM마다 다름)

JVM의 접근: 플랫폼 독립적 중간 언어
  iload_1      // a를 스택에 push
  iload_2      // b를 스택에 push
  iadd         // 두 값을 pop, 더한 후 push
  istore_3     // 결과를 sum에 저장
  
  → 어떤 CPU든 JVM만 구현하면 실행 가능
```

바이트코드는 **추상화된 기계어**다.

---

## 📐 내부 구조

### 1. 바이트코드 명령어 개요

```
총 명령어 수: 약 200개 (opcode 0x00 ~ 0xC9)

명령어 형식:
  opcode [operand1] [operand2] ...
  
  예:
  0x10 0x2A        → bipush 42 (opcode=0x10, operand=0x2A=42)
  0x59             → dup (opcode=0x59, operand 없음)

opcode 크기: 1 byte
  → 최대 256개 명령어 가능
  → 실제 사용: ~200개
```

---

### 2. 명령어 분류 (카테고리별)

```
1. Load & Store (LVA ↔ Operand Stack)
   iload, istore, aload, astore, ...
   
2. Arithmetic (산술 연산)
   iadd, isub, imul, idiv, irem, ...
   
3. Type Conversion (타입 변환)
   i2l, l2i, f2d, d2i, ...
   
4. Stack Manipulation (스택 조작)
   pop, dup, swap, ...
   
5. Control Flow (제어 흐름)
   if_icmpeq, goto, ifeq, ifne, ...
   
6. Method Invocation (메서드 호출)
   invokevirtual, invokestatic, invokespecial, ...
   
7. Object Operations (객체 연산)
   new, newarray, getfield, putfield, ...
   
8. Return
   ireturn, areturn, return, ...
```

---

### 3. 타입별 명령어 분리

```
왜 iload, fload, aload가 따로 존재하는가?

JVM은 타입 안전성을 바이트코드 수준에서 보장

int 연산:
  iload   → int load
  istore  → int store
  iadd    → int add
  
float 연산:
  fload   → float load
  fstore  → float store
  fadd    → float add

reference 연산:
  aload   → reference load (array, object)
  astore  → reference store

타입 접두사:
  i = int
  l = long
  f = float
  d = double
  a = reference (address)
  b = byte
  c = char
  s = short

이점:
  1. Bytecode Verifier가 타입 오류 검출
  2. JIT 컴파일러의 최적화 힌트
  3. 명령어만 보고 피연산자 타입 파악 가능
```

---

### 4. 상수 로드 명령어 최적화

```
int 상수를 로드하는 여러 방법:

iconst_m1    → -1
iconst_0     → 0
iconst_1     → 1
iconst_2     → 2
iconst_3     → 3
iconst_4     → 4
iconst_5     → 5

bipush <n>   → -128 ~ 127
sipush <n>   → -32768 ~ 32767
ldc <index>  → Constant Pool[index] (int, float, String, ...)

예:
  int a = 0;      → iconst_0 (1 byte)
  int b = 100;    → bipush 100 (2 bytes)
  int c = 10000;  → sipush 10000 (3 bytes)
  int d = 1000000; → ldc #N (2 bytes, Constant Pool 사용)

최적화:
  javac는 상수값에 따라 가장 짧은 명령어 선택
  → 클래스 파일 크기 최소화
```

---

### 5. Load & Store 명령어

```
Local Variable Array (LVA) ↔ Operand Stack

Load (LVA → Stack):
  iload_0, iload_1, iload_2, iload_3  → LVA[0~3]의 int 값
  iload <n>                           → LVA[n]의 int 값
  
  fload_0, fload_1, ...
  aload_0, aload_1, ...  (a = reference)
  lload_0, lload_1, ...  (long은 2슬롯 차지)

Store (Stack → LVA):
  istore_0, istore_1, istore_2, istore_3
  istore <n>
  
  fstore_0, fstore_1, ...
  astore_0, astore_1, ...

예:
  void foo(int a, int b) {
      int c = a + b;
  }
  
  바이트코드:
  0: iload_1      // LVA[1](a) → Stack
  1: iload_2      // LVA[2](b) → Stack
  2: iadd         // Stack에서 두 값 pop, 더한 후 push
  3: istore_3     // Stack → LVA[3](c)
```

---

### 6. 산술 연산 명령어

```
int 연산:
  iadd, isub, imul, idiv, irem
  ineg (부호 반전)
  iand, ior, ixor (비트 연산)
  ishl, ishr, iushr (시프트)

long 연산:
  ladd, lsub, lmul, ldiv, lrem
  lneg
  land, lor, lxor
  lshl, lshr, lushr

float 연산:
  fadd, fsub, fmul, fdiv, frem
  fneg

double 연산:
  dadd, dsub, dmul, ddiv, drem
  dneg

스택 동작:
  iadd:
    Stack 전: [a, b]  (b가 top)
    Stack 후: [a+b]
```

---

### 7. 타입 변환 명령어

```
Widening (자동 확장, 정보 손실 없음):
  i2l  → int to long
  i2f  → int to float
  i2d  → int to double
  l2f  → long to float
  l2d  → long to double
  f2d  → float to double

Narrowing (축소, 정보 손실 가능):
  l2i  → long to int
  f2i  → float to int
  d2i  → double to int
  d2l  → double to long
  d2f  → double to float
  i2b  → int to byte
  i2c  → int to char
  i2s  → int to short

예:
  long l = 10;
  int i = (int) l;
  
  바이트코드:
  ldc2_w 10   // long 10을 스택에
  l2i         // long → int 변환
  istore_1    // i에 저장
```

---

### 8. 제어 흐름 명령어

```
조건 분기 (if):
  ifeq  → 0이면 점프
  ifne  → 0이 아니면 점프
  iflt  → < 0 이면 점프
  ifle  → <= 0 이면 점프
  ifgt  → > 0 이면 점프
  ifge  → >= 0 이면 점프

비교 후 분기 (if_cmp):
  if_icmpeq  → 두 int 값이 같으면 점프
  if_icmpne  → 두 int 값이 다르면 점프
  if_icmplt  → v1 < v2 이면 점프
  if_icmple  → v1 <= v2 이면 점프
  if_icmpgt  → v1 > v2 이면 점프
  if_icmpge  → v1 >= v2 이면 점프

무조건 점프:
  goto <offset>

예:
  if (a > b) {
      ...
  }
  
  바이트코드:
  0: iload_1        // a
  1: iload_2        // b
  2: if_icmple 8    // a <= b 이면 8번으로 점프
  5: ...            // a > b일 때 실행
  8: ...            // 다음 코드
```

---

## 💻 실험으로 확인하기

### 실험 1: javap로 바이트코드 확인

```java
public class BytecodeDemo {
    public int add(int a, int b) {
        return a + b;
    }
    
    public int multiply(int x, int y) {
        int result = x * y;
        return result;
    }
}
```

```bash
javac BytecodeDemo.java
javap -c BytecodeDemo.class

# 출력:
# public int add(int, int);
#   Code:
#      0: iload_1       // a
#      1: iload_2       // b
#      2: iadd          // a + b
#      3: ireturn       // 반환

# public int multiply(int, int);
#   Code:
#      0: iload_1       // x
#      1: iload_2       // y
#      2: imul          // x * y
#      3: istore_3      // result에 저장
#      4: iload_3       // result
#      5: ireturn       // 반환
```

---

### 실험 2: 상수 로드 최적화 관찰

```java
public class ConstantDemo {
    public void constants() {
        int a = 0;
        int b = 5;
        int c = 100;
        int d = 10000;
        int e = 1000000;
    }
}
```

```bash
javap -c ConstantDemo.class

# 출력:
# public void constants();
#   Code:
#      0: iconst_0         // a = 0 (1 byte)
#      1: istore_1
#      2: iconst_5         // b = 5 (1 byte)
#      3: istore_2
#      4: bipush 100       // c = 100 (2 bytes)
#      6: istore_3
#      7: sipush 10000     // d = 10000 (3 bytes)
#     10: istore 4
#     12: ldc #2           // e = 1000000 (2 bytes + Constant Pool)
#     14: istore 5
```

---

### 실험 3: 제어 흐름 바이트코드

```java
public class ControlFlow {
    public int max(int a, int b) {
        if (a > b) {
            return a;
        } else {
            return b;
        }
    }
}
```

```bash
javap -c ControlFlow.class

# 출력:
# public int max(int, int);
#   Code:
#      0: iload_1         // a
#      1: iload_2         // b
#      2: if_icmple 7     // a <= b 이면 7번으로 점프
#      5: iload_1         // a
#      6: ireturn         // return a
#      7: iload_2         // b (else 블록)
#      8: ireturn         // return b
```

---

### 실험 4: 바이너리 opcode 확인

```bash
hexdump -C BytecodeDemo.class | grep -A5 "Code:"

# add 메서드 바이트코드:
# 1B 1C 60 AC
# ^^iload_1  ^^iload_2  ^^iadd  ^^ireturn

# Opcode 매핑:
# 0x1B = iload_1 (27)
# 0x1C = iload_2 (28)
# 0x60 = iadd (96)
# 0xAC = ireturn (172)
```

---

## ⚡ 실무 임팩트

### JIT 컴파일러와 바이트코드

```
바이트코드는 JIT 컴파일러의 입력:

Interpreter 모드 (Warm-up):
  바이트코드를 하나씩 해석 실행
  느리지만 즉시 시작 가능

JIT 컴파일 (Hot Method):
  바이트코드 → 네이티브 기계어
  빠르지만 컴파일 비용 발생
  
C1 (Client Compiler):
  빠른 컴파일, 기본 최적화
  iadd → add eax, ebx
  
C2 (Server Compiler):
  느린 컴파일, 고급 최적화
  - Inlining
  - Escape Analysis
  - Loop Unrolling
  
바이트코드가 간결할수록:
  - 파싱 빠름
  - JIT 컴파일 빠름
  - 최적화 기회 많음
```

### 바이트코드 크기와 메서드 인라이닝

```
JIT 인라이닝 임계값:
  -XX:MaxInlineSize=35 (기본)
  → 바이트코드 35바이트 이하 메서드만 인라인

작은 메서드 예:
  public int getX() { return x; }
  → 바이트코드: aload_0, getfield, ireturn (6 bytes)
  → 인라인 가능

큰 메서드:
  복잡한 로직 (100+ bytes)
  → 인라인 불가
  → 메서드 호출 오버헤드 발생

최적화 팁:
  자주 호출되는 메서드는 작게 유지
  → 인라인 가능 → 성능 향상
```

### 바이트코드 검증과 보안

```
Bytecode Verifier:
  클래스 로딩 시 바이트코드 검증
  
검증 항목:
  1. 스택 오버플로우/언더플로우
     max_stack 범위 내 연산
     
  2. 타입 안전성
     iload 후 iadd (OK)
     iload 후 fadd (NG → VerifyError)
     
  3. 접근 제어
     private 메서드를 외부에서 호출 (NG)
     
  4. Control Flow
     모든 경로에서 return
     goto가 메서드 밖으로 점프 안 함

악의적 바이트코드 차단:
  수동 조작된 .class 파일
  → Verifier가 거부
  → ClassFormatError / VerifyError
```

---

## 🚫 흔한 오해

### "바이트코드 명령어는 256개다"

```
❌ 잘못된 이해:
  opcode가 1 byte이므로 명령어가 256개다.

✅ 실제:
  정의된 명령어: 약 200개
  예약됨: 0xBA (breakpoint), 0xFE, 0xFF (내부 사용)
  미사용: 56개 정도
  
  이유:
  - 타입별 분리로 명령어 수 증가
  - 하지만 모든 조합이 필요한 건 아님
  - 예: lload_4, lload_5 같은 건 없음 (lload <n> 사용)
```

### "iload_0은 항상 this를 로드한다"

```
❌ 잘못된 이해:
  iload_0은 this를 의미한다.

✅ 실제:
  iload_0 → LVA[0]의 int 값
  aload_0 → LVA[0]의 reference 값
  
  인스턴스 메서드:
  LVA[0] = this (aload_0으로 접근)
  LVA[1~] = 매개변수 + 지역 변수
  
  static 메서드:
  LVA[0] = 첫 번째 매개변수
  (this 없음)

예:
  void foo(int x) {  // 인스턴스 메서드
      int y = x + 1;
  }
  // LVA[0]=this, LVA[1]=x, LVA[2]=y
  // x 접근: iload_1 (not iload_0)
```

### "모든 타입에 _0, _1, _2, _3 버전이 있다"

```
❌ 잘못된 이해:
  lload_0, lload_1, lload_2, lload_3이 모두 존재한다.

✅ 실제:
  long, double은 2슬롯 차지
  → lload_0, lload_1, lload_2, lload_3 존재
  → lload_4, lload_5는 없음 (lload <n> 사용)
  
  int, float, reference:
  iload_0~3, fload_0~3, aload_0~3
  istore_0~3, fstore_0~3, astore_0~3
  
  이유:
  LVA[0~3]이 가장 빈번하게 사용
  → 전용 명령어로 1바이트 절약
```

---

## 📌 핵심 정리

```
바이트코드 기본
  Opcode (1 byte) + Operands
  약 200개 명령어
  타입별 분리 (i, l, f, d, a)

명령어 카테고리
  Load/Store: LVA ↔ Stack
  Arithmetic: add, sub, mul, div
  Control Flow: if, goto
  Method Call: invoke*
  Object: new, getfield, putfield

타입별 명령어 분리
  iload, fload, aload (타입 안전성)
  Bytecode Verifier가 타입 검증
  JIT 최적화 힌트

상수 로드 최적화
  iconst_0~5 (1 byte)
  bipush -128~127 (2 bytes)
  sipush -32768~32767 (3 bytes)
  ldc (Constant Pool)

Load/Store
  iload_0~3: LVA[0~3] → Stack
  istore_0~3: Stack → LVA[0~3]
  long/double은 2슬롯

제어 흐름
  if_icmpeq, if_icmplt, ... (비교 후 분기)
  ifeq, ifne, iflt, ... (0과 비교 후 분기)
  goto (무조건 점프)

실무 임팩트
  바이트코드 크기 < 35 bytes → JIT 인라인 가능
  Bytecode Verifier가 타입/보안 검증
  JIT 컴파일러의 최적화 입력
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 Java 코드의 바이트코드를 작성하라. 각 명령어가 스택에 미치는 영향을 설명하라.

```java
int calculate(int x) {
    int y = x * 2;
    return y + 1;
}
```

**Q2.** `int a = 127;`과 `int b = 128;`의 바이트코드가 왜 다른가? 어떤 명령어를 사용하며 클래스 파일 크기에 어떤 영향을 주는가?

**Q3.** 다음 두 메서드 중 JIT 인라이닝에 더 유리한 것은? 바이트코드 크기와 연결해 설명하라.

```java
// 방법 1
int getArea() {
    return width * height;
}

// 방법 2
int getArea() {
    int w = getWidth();
    int h = getHeight();
    return w * h;
}
```

> 💡 **해설**
>
> **Q1.** 바이트코드: `0: iload_1` (LVA[1]=x → Stack), `1: iconst_2` (2 → Stack), `2: imul` (Stack에서 x, 2 pop → x*2 push), `3: istore_2` (Stack → LVA[2]=y), `4: iload_2` (y → Stack), `5: iconst_1` (1 → Stack), `6: iadd` (y, 1 pop → y+1 push), `7: ireturn` (Stack top 반환). 스택 변화: [] → [x] → [x,2] → [x*2] → [] → [y] → [y,1] → [y+1] → [].
>
> **Q2.** `int a = 127;` → `bipush 127` (2 bytes: opcode 0x10 + operand 0x7F). `int b = 128;` → `sipush 128` (3 bytes: opcode 0x11 + operand 0x00 0x80). 이유: `bipush`는 -128~127만 지원 (1 byte signed). 128은 범위 밖이라 `sipush` 사용 (2 bytes signed). 클래스 파일 크기 1바이트 증가. 수백 개 상수가 있다면 누적 영향 커짐.
>
> **Q3.** 방법 1이 유리. 이유: 방법 1 바이트코드 ~10 bytes (`aload_0`, `getfield width`, `getfield height`, `imul`, `ireturn`). 방법 2는 `getWidth()`, `getHeight()` 호출 추가로 ~20 bytes 이상. JIT 인라이닝 임계값 35 bytes 기준으로 방법 1은 여유 있고, 방법 2는 메서드 크기가 커져 인라인 가능성 낮아짐. 또한 방법 1은 메서드 호출 오버헤드 없음. 단, `getWidth()`/`getHeight()`도 작으면 재귀 인라인 가능하지만 보장 안 됨.

---

## 📚 참고 자료

- [JVMS Chapter 6 — The Java Virtual Machine Instruction Set](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-6.html)
- [JVM Opcodes Reference](https://en.wikipedia.org/wiki/Java_bytecode_instruction_listings)
- [OpenJDK Bytecode Interpreter](https://github.com/openjdk/jdk/tree/master/src/hotspot/share/interpreter)

---

<div align="center">

**[⬅️ 이전: Class File Format](./01-class-file-format.md)** | **[다음: Operand Stack Mechanism ➡️](./03-operand-stack-mechanism.md)**

</div>

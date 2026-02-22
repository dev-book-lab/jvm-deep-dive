# Exception Handling Bytecode - 예외 처리 바이트코드

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `try-catch-finally`는 바이트코드로 어떻게 변환되는가?
- Exception Table은 무엇이며, JVM은 어떻게 예외를 catch하는가?
- `athrow` 명령어는 어떻게 동작하며, 스택 상태는 어떻게 변하는가?
- `finally` 블록은 왜 바이트코드에서 중복되어 나타나는가?
- `try-with-resources`는 바이트코드에서 어떻게 구현되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 예외 처리를 어떻게 구조화할 것인가

```java
void readFile() {
    try {
        FileReader fr = new FileReader("file.txt");
        // ...
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        // cleanup
    }
}
```

```
전통적 C 방식 (goto):
  FILE* f = fopen("file.txt", "r");
  if (f == NULL) goto error;
  // ...
  error:
    // error handling
  cleanup:
    // cleanup code

문제:
  - goto는 스파게티 코드 유발
  - 에러 처리 누락 쉬움
  - 구조화 어려움

JVM 방식 (Exception Table):
  try 블록 범위 기록
  예외 발생 시 Table 탐색
  → 구조적, 명확
```

바이트코드는 **Exception Table**로 예외를 처리한다.

---

## 📐 내부 구조

### 1. Exception Table 구조

```
Code 속성 내 Exception Table:

exception_table_length: 2
exception_table:
  start_pc  end_pc  handler_pc  catch_type
  0         10      13          #5 (IOException)
  0         10      20          #0 (any)

의미:
  start_pc ~ end_pc: try 블록 범위 (바이트코드 오프셋)
  handler_pc: catch 블록 시작 위치
  catch_type: 잡을 예외 타입 (Constant Pool 인덱스)
                #0이면 모든 예외 (finally 구현용)

예외 발생 시 JVM 동작:
  1. 현재 PC(Program Counter)가 어느 범위에 있는지 확인
  2. Exception Table에서 매칭되는 엔트리 탐색
  3. 예외 타입 확인 (instanceof)
  4. handler_pc로 점프
  5. 예외 객체를 Operand Stack에 push
```

---

### 2. try-catch 바이트코드

```java
void example() {
    try {
        riskyMethod();
    } catch (IOException e) {
        System.out.println("IO Error");
    }
}
```

```
바이트코드:

   0: aload_0
   1: invokevirtual #2    // riskyMethod
   4: goto 15              // try 블록 끝, catch 건너뛰기

   // catch 블록 시작
   7: astore_1             // 예외 객체를 LVA[1](e)에 저장
   8: getstatic #3         // System.out
  11: ldc #4               // "IO Error"
  13: invokevirtual #5    // println
  14: return

  15: return               // 정상 종료

Exception Table:
  start_pc  end_pc  handler_pc  catch_type
  0         4       7           #6 (IOException)

흐름:
  정상: 0 → 1 → 4 (goto 15) → 15 (return)
  예외: 0 → 1 (예외 발생) → Table 탐색 → 7 (catch) → ... → 14 (return)
```

---

### 3. try-catch-finally 바이트코드

```java
void example() {
    try {
        riskyMethod();
    } catch (IOException e) {
        handleError(e);
    } finally {
        cleanup();
    }
}
```

```
바이트코드 (finally 중복):

   // try 블록
   0: aload_0
   1: invokevirtual #2    // riskyMethod
   4: aload_0
   5: invokevirtual #3    // cleanup (finally - 정상 경로)
   8: goto 30

   // catch 블록
  11: astore_1             // IOException e
  12: aload_0
  13: aload_1
  14: invokevirtual #4    // handleError(e)
  17: aload_0
  18: invokevirtual #3    // cleanup (finally - catch 경로)
  21: goto 30

   // finally 블록 (예외 재throw용)
  24: astore_2             // any exception
  25: aload_0
  26: invokevirtual #3    // cleanup (finally - 재throw 경로)
  29: aload_2
  30: athrow               // 예외 재throw

  33: return

Exception Table:
  start_pc  end_pc  handler_pc  catch_type
  0         4       11          #5 (IOException)
  0         4       24          #0 (any - finally)
  11        17      24          #0 (any - finally)

특징:
  cleanup() 호출이 3번 중복됨
  → 정상, catch, 재throw 경로 각각
  → 바이트코드 크기 증가
```

---

### 4. athrow 명령어

```
athrow (Throw Exception):

동작:
  1. Operand Stack에서 예외 객체 pop
  2. 현재 메서드의 Exception Table 탐색
  3. 매칭되는 handler 없으면 메서드 종료
  4. caller의 Exception Table 탐색
  5. 최종적으로 처리 안 되면 스레드 종료

스택 변화:
  throw 전: [..., exception_obj]
  athrow 실행
  → 현재 스택 클리어
  → handler_pc로 점프
  → 새 스택: [exception_obj]

바이트코드:
  new #2              // new IOException
  dup
  ldc #3              // "Error message"
  invokespecial #4   // IOException.<init>(String)
  athrow              // throw
```

---

### 5. try-with-resources 바이트코드

```java
void example() {
    try (FileReader fr = new FileReader("file.txt")) {
        // use fr
    }
}
```

```
바이트코드 (매우 복잡):

   0: new #2              // FileReader
   3: dup
   4: ldc #3              // "file.txt"
   6: invokespecial #4   // FileReader.<init>
   9: astore_1            // fr 저장

  10: aload_1
  11: invokevirtual #5   // use fr (생략)

  // 정상 종료 시 close
  14: aload_1
  15: ifnull 43
  18: aload_1
  19: invokevirtual #6   // fr.close()
  22: goto 43

  // try 블록에서 예외 발생
  25: astore_2            // 예외 저장
  26: aload_1
  27: ifnull 40
  30: aload_1
  31: invokevirtual #6   // fr.close()
  34: goto 40
  37: invokevirtual #7   // addSuppressed (close 실패 시)
  40: aload_2
  41: athrow              // 원래 예외 재throw

  // close()에서 예외 발생
  42: ...

Exception Table:
  start_pc  end_pc  handler_pc  catch_type
  10        14      25          #0 (any)
  30        34      37          #8 (Throwable - addSuppressed용)
  ...

특징:
  - close() 호출이 여러 번 중복
  - close() 실패 시 addSuppressed() 처리
  - 원본 예외 우선, close 예외는 suppressed
  → 바이트코드 매우 복잡 (수십 줄)
```

---

## 💻 실험으로 확인하기

### 실험 1: Exception Table 확인

```java
public class ExceptionDemo {
    public void simple() {
        try {
            riskyMethod();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (Exception e) {
            System.out.println("General error");
        }
    }
    
    void riskyMethod() throws IOException { }
}
```

```bash
javap -c -v ExceptionDemo.class

# simple 메서드:
# Exception table:
#    from    to  target type
#        0     4     7   Class java/io/IOException
#        0     4    17   Class java/lang/Exception
#
# Code:
#    0: aload_0
#    1: invokevirtual #2   // riskyMethod
#    4: goto 27
#    7: astore_1            // IOException
#    8: aload_1
#    9: invokevirtual #4   // printStackTrace
#   12: goto 27
#   17: astore_1            // Exception
#   18: getstatic #5        // System.out
#   21: ldc #6              // "General error"
#   23: invokevirtual #7   // println
#   26: return
#   27: return
```

---

### 실험 2: finally 블록 중복 확인

```java
public class FinallyDemo {
    public void test() {
        try {
            System.out.println("try");
        } finally {
            System.out.println("finally");
        }
    }
}
```

```bash
javap -c FinallyDemo.class

# Code:
#    0: getstatic #2    // System.out
#    3: ldc #3          // "try"
#    5: invokevirtual #4
#    8: getstatic #2    // System.out (finally - 정상 경로)
#   11: ldc #5          // "finally"
#   13: invokevirtual #4
#   16: goto 30
#   19: astore_1        // any exception
#   20: getstatic #2    // System.out (finally - 예외 경로)
#   23: ldc #5          // "finally"
#   25: invokevirtual #4
#   28: aload_1
#   29: athrow
#   30: return

# "finally" 문자열과 println 호출이 2번 반복됨
```

---

### 실험 3: try-with-resources 복잡도

```java
public class TryWithResourcesDemo {
    public void simple() {
        try (FileReader fr = new FileReader("test.txt")) {
            int c = fr.read();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

```bash
javap -c TryWithResourcesDemo.class

# Code 길이: ~100 lines
# Exception Table: 5~7개 엔트리
# close() 호출: 여러 경로에서 중복
# addSuppressed() 처리: 별도 예외 처리 블록

# → 단순해 보이지만 바이트코드는 매우 복잡
```

---

## ⚡ 실무 임팩트

### try-catch 성능 오버헤드

```
예외가 발생하지 않을 때:
  try 블록 진입: 거의 오버헤드 없음
  Exception Table은 메타데이터일 뿐
  → 성능 영향 미미

예외가 발생할 때:
  1. Exception Table 탐색
  2. Stack Unwinding (메서드 역순 탐색)
  3. 예외 객체 생성 (+ Stack Trace)
  → 매우 느림 (정상 실행 대비 100~1000배)

권장:
  예외는 예외적인 경우에만 사용
  정상 흐름 제어에 사용 금지
  
// ❌ 안티패턴
try {
    return map.get(key);
} catch (NullPointerException e) {
    return null;
}

// ✅ 정상 패턴
return map.getOrDefault(key, null);
```

### finally vs try-with-resources

```
finally 문제:
  - 코드 중복 (바이트코드)
  - close() 누락 가능
  - 복잡한 예외 처리 필요

try-with-resources 장점:
  - 자동 close() 보장
  - Suppressed Exception 자동 처리
  - 코드 간결

바이트코드 크기:
  finally: ~30 lines
  try-with-resources: ~100 lines
  → 컴파일러가 안전하게 확장
  
권장:
  AutoCloseable 구현 시 항상 try-with-resources 사용
```

### Exception Table과 JIT 최적화

```
JIT 컴파일러 관점:

try 블록이 많은 메서드:
  - Exception Table이 복잡
  - 최적화 범위 제한
  - 인라이닝 어려움

성능 최적화:
  - Hot Path에 try-catch 최소화
  - 예외를 메서드 외부로 위임
  
// ❌ 루프 내 try
for (int i = 0; i < 1000000; i++) {
    try {
        process(i);
    } catch (Exception e) { }
}

// ✅ try 밖으로 이동
try {
    for (int i = 0; i < 1000000; i++) {
        process(i);
    }
} catch (Exception e) { }
```

---

## 🚫 흔한 오해

### "finally는 항상 실행된다"

```java
❌ 잘못된 이해:
  finally 블록은 어떤 경우에도 항상 실행된다.

✅ 실제:
  대부분 실행되지만 예외 있음
  
실행 안 되는 경우:
  1. System.exit()
     try { ... } 
     finally { ... }  // 실행 안 됨
     System.exit(0);
  
  2. JVM 크래시
     infinite loop, native crash
  
  3. 스레드 강제 종료
     thread.stop() (deprecated)
  
  4. finally 블록 도달 전 무한 루프
     try {
         while(true) { }
     } finally { }  // 도달 불가

바이트코드 관점:
  finally는 "정상/예외 경로 각각에 복사"
  경로에 도달하지 못하면 실행 안 됨
```

### "try-catch는 예외 발생 여부와 무관하게 느리다"

```
❌ 잘못된 이해:
  try 블록이 있으면 무조건 성능 저하된다.

✅ 실제:
  예외 발생 안 하면 거의 오버헤드 없음
  
벤치마크:
  // try 없음
  int sum = 0;
  for (int i = 0; i < 1000000; i++) {
      sum += i;
  }
  // 시간: 1ms
  
  // try 있음 (예외 안 남)
  try {
      int sum = 0;
      for (int i = 0; i < 1000000; i++) {
          sum += i;
      }
  } catch (Exception e) { }
  // 시간: 1ms (동일)
  
  // 예외 발생
  for (int i = 0; i < 1000000; i++) {
      try {
          throw new Exception();
      } catch (Exception e) { }
  }
  // 시간: 1000ms (1000배 느림)

결론:
  try-catch 자체는 빠름
  예외 객체 생성 + Stack Unwinding이 느림
```

### "catch (Exception e)는 모든 예외를 잡는다"

```
❌ 잘못된 이해:
  Exception을 잡으면 모든 예외를 처리한다.

✅ 실제:
  Error는 잡지 못함
  
Throwable 계층:
  Throwable
  ├─ Error (잡지 말아야 함)
  │  ├─ OutOfMemoryError
  │  ├─ StackOverflowError
  │  └─ VirtualMachineError
  └─ Exception (잡아도 됨)
     ├─ RuntimeException
     └─ IOException

catch (Exception e):
  Exception과 그 하위만 catch
  Error는 통과 → JVM이 처리

catch (Throwable t):
  모든 예외 catch
  → 권장 안 함 (Error도 잡힘)
  
바이트코드:
  Exception Table의 catch_type이 타입 결정
  #0 (any)는 모든 Throwable
```

---

## 📌 핵심 정리

```
Exception Table
  try 블록 범위: start_pc ~ end_pc
  catch 블록 위치: handler_pc
  예외 타입: catch_type (Constant Pool 인덱스)

예외 발생 시 흐름
  1. PC가 Exception Table 범위 내?
  2. 예외 타입 매칭? (instanceof)
  3. handler_pc로 점프
  4. 예외 객체 → Operand Stack

athrow
  스택에서 예외 객체 pop
  Exception Table 탐색
  매칭 없으면 caller로 전파
  스택 클리어 후 handler로 점프

finally 구현
  정상/예외/재throw 경로에 각각 중복
  바이트코드 크기 증가
  catch_type=#0 (any) 사용

try-with-resources
  AutoCloseable.close() 자동 호출
  Suppressed Exception 처리
  바이트코드 매우 복잡 (~100 lines)

성능
  try 진입: 오버헤드 거의 없음
  예외 발생: 매우 느림 (100~1000배)
  → 예외는 예외적인 경우에만

실무 팁
  Hot Path에 try-catch 최소화
  try-with-resources 권장 (finally 대신)
  루프 밖으로 try 이동
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드의 Exception Table을 작성하라. start_pc, end_pc, handler_pc, catch_type를 명시하라.

```java
void test() {
    try {
        methodA();
        methodB();
    } catch (IOException e) {
        handleIO(e);
    } catch (Exception e) {
        handleGeneral(e);
    } finally {
        cleanup();
    }
}
```

**Q2.** finally 블록이 바이트코드에서 3번 중복되는 이유를 예외 발생 경로와 연결해 설명하라. 중복을 제거할 수 있는가?

**Q3.** 다음 두 코드의 바이트코드 크기와 성능을 비교하라.

```java
// 방법 1
for (int i = 0; i < 1000; i++) {
    try {
        process(i);
    } catch (Exception e) {
        handle(e);
    }
}

// 방법 2
try {
    for (int i = 0; i < 1000; i++) {
        process(i);
    }
} catch (Exception e) {
    handle(e);
}
```

> 💡 **해설**
>
> **Q1.** Exception Table (개념적):
> ```
> start_pc  end_pc  handler_pc  catch_type
> 0         10      13          IOException
> 0         10      20          Exception
> 0         10      30          any (finally - 재throw)
> 13        17      30          any (finally - IOException catch 후)
> 20        24      30          any (finally - Exception catch 후)
> ```
> cleanup()은 정상 경로(10번 이후), IOException catch 후(17번 이후), Exception catch 후(24번 이후), 재throw 경로(30번)에서 각각 호출됨. 총 4번 중복.
>
> **Q2.** finally 중복 이유: ① 정상 종료 경로 (try 끝) — cleanup() 호출 후 return. ② catch 블록 경로 — catch 처리 후 cleanup() 호출. ③ 처리 안 된 예외 경로 — cleanup() 호출 후 athrow로 재throw. JVM은 goto로만 제어 흐름 구현 → finally를 서브루틴으로 호출 불가 (jsr/ret는 deprecated) → 각 경로에 코드 복사. 중복 제거 불가: 바이트코드는 구조적 점프만 지원, 서브루틴 메커니즘 없음 (Java 6부터 jsr/ret 제거).
>
> **Q3.** 바이트코드 크기: 방법 1은 루프 내 try-catch → Exception Table이 루프 안쪽 범위만 커버 → 작음 (~20 lines). 방법 2는 루프 전체를 try로 감쌈 → Exception Table이 전체 루프 커버 → 크기 비슷 또는 약간 큼. 성능: 방법 1은 예외 발생 안 하면 동일하지만, 예외 발생 시 매번 Exception Table 탐색 (미미). 방법 2가 약간 유리 (Exception Table 탐색 1회). 단, 예외 발생 빈도가 낮으면 차이 무시 가능. JIT 최적화: 방법 2가 유리 — try 블록이 큰 단위로 있어 최적화 범위 넓음. 권장: 방법 2 (try를 루프 밖으로).

---

## 📚 참고 자료

- [JVMS §2.10 — Exceptions](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html#jvms-2.10)
- [JVMS §3.12 — Throwing and Handling Exceptions](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-3.html#jvms-3.12)
- [JEP 213 — Milling Project Coin (try-with-resources)](https://openjdk.org/jeps/213)

---

<div align="center">

**[⬅️ 이전: Method Invocation Instructions](./04-method-invocation-instructions.md)** | **[다음: Lambda & InvokeDynamic ➡️](./06-lambda-and-invokedynamic.md)**

</div>

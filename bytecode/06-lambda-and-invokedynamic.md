# Lambda & InvokeDynamic - 람다와 인보크다이나믹

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 람다 표현식은 바이트코드로 어떻게 변환되는가?
- 왜 람다는 내부 클래스가 아닌 `invokedynamic`을 사용하는가?
- `LambdaMetafactory`는 무엇이며, 어떻게 람다 인스턴스를 생성하는가?
- `invokedynamic`의 Bootstrap Method는 언제 실행되며, CallSite는 무엇인가?
- 람다의 캡처(capture)는 바이트코드에서 어떻게 처리되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 함수형 프로그래밍을 어떻게 JVM에서 구현할 것인가

```java
// Java 7 이전 (익명 클래스)
Runnable r = new Runnable() {
    public void run() {
        System.out.println("Hello");
    }
};

// Java 8+ (람다)
Runnable r = () -> System.out.println("Hello");
```

```
익명 클래스 문제:
  - 클래스 파일 증가 (Outer$1.class, Outer$2.class...)
  - 메모리 오버헤드 (매번 새 객체 생성)
  - 바이트코드 고정 (최적화 여지 없음)

람다 요구사항:
  - 간결한 문법
  - 효율적인 메모리 사용
  - 미래 최적화 가능성 확보
```

JVM은 이를 **invokedynamic**으로 해결했다.

---

## 📐 내부 구조

### 1. 람다의 바이트코드 변환

```java
public class LambdaDemo {
    public void test() {
        Runnable r = () -> System.out.println("Hello");
        r.run();
    }
}
```

```bash
javap -c -v LambdaDemo.class

# test 메서드:
#    0: invokedynamic #2, 0  // InvokeDynamic #0:run:()Ljava/lang/Runnable;
#    5: astore_1
#    6: aload_1
#    7: invokeinterface #3, 1  // Runnable.run:()V
#   12: return

# BootstrapMethods:
#   0: #27 REF_invokeStatic java/lang/invoke/LambdaMetafactory.metafactory:
#       (Ljava/lang/invoke/MethodHandles$Lookup;
#        Ljava/lang/String;
#        Ljava/lang/invoke/MethodType;
#        Ljava/lang/invoke/MethodType;
#        Ljava/lang/invoke/MethodHandle;
#        Ljava/lang/invoke/MethodType;)
#        Ljava/lang/invoke/CallSite;
#     Method arguments:
#       #28 ()V
#       #29 REF_invokeStatic LambdaDemo.lambda$test$0:()V
#       #28 ()V

# 람다 본문 (컴파일러 생성 메서드):
# private static void lambda$test$0();
#   Code:
#      0: getstatic #4   // System.out
#      3: ldc #5         // "Hello"
#      5: invokevirtual #6  // println
#      8: return
```

---

### 2. invokedynamic 구조

```
invokedynamic의 3가지 구성 요소:

1. Bootstrap Method (BSM)
   - 클래스 로딩 시 한 번만 실행
   - CallSite 생성
   - 람다의 경우: LambdaMetafactory.metafactory()

2. CallSite
   - 실제 호출 대상 (MethodHandle)
   - 캐싱됨 (첫 호출 후 재사용)
   
3. MethodHandle
   - 메서드 참조 (함수 포인터)
   - 람다 본문을 가리킴

실행 흐름:

첫 호출:
  invokedynamic #2
  → BSM(LambdaMetafactory.metafactory) 호출
  → 람다 구현 클래스 동적 생성 (예: LambdaDemo$$Lambda$1)
  → CallSite 생성 (람다 인스턴스 반환)
  → CallSite 캐싱

이후 호출:
  invokedynamic #2
  → 캐시된 CallSite 사용
  → 람다 인스턴스 즉시 반환
```

---

### 3. LambdaMetafactory 동작 원리

```
LambdaMetafactory.metafactory() 파라미터:

1. caller (MethodHandles.Lookup)
   - 호출자 컨텍스트 (접근 권한)

2. invokedName (String)
   - 인터페이스 메서드명 (예: "run")

3. invokedType (MethodType)
   - 람다 팩토리 메서드 시그니처
   - 예: ()Runnable (인자 없음, Runnable 반환)

4. samMethodType (MethodType)
   - 함수형 인터페이스 메서드 시그니처
   - 예: ()V (Runnable.run)

5. implMethod (MethodHandle)
   - 람다 본문 (컴파일러 생성 메서드)
   - 예: LambdaDemo.lambda$test$0

6. instantiatedMethodType (MethodType)
   - 실제 메서드 타입 (제네릭 erasure 후)

동작:
  1. 파라미터 검증
  2. 내부 클래스 동적 생성 (ASM 사용)
     예: LambdaDemo$$Lambda$1 implements Runnable
  3. run() 메서드가 lambda$test$0 호출하도록 구현
  4. 인스턴스 생성 후 반환
```

---

### 4. 람다 캡처 (Capture)

```java
int x = 10;
Runnable r = () -> System.out.println(x);
```

```
캡처 변수:
  람다 외부 변수 (x)를 람다 내부에서 사용
  → effectively final이어야 함

바이트코드:

# test 메서드:
#    0: bipush 10
#    2: istore_1         // x = 10
#    3: iload_1
#    4: invokedynamic #2, 0  // InvokeDynamic #0:run:(I)Ljava/lang/Runnable;
#                             ↑ x를 파라미터로 전달
#    9: astore_2
#   10: aload_2
#   11: invokeinterface #3, 1
#   16: return

# BootstrapMethods:
#   Method arguments:
#     ()V
#     REF_invokeStatic LambdaDemo.lambda$test$0:(I)V
#                                               ↑ int 파라미터 추가
#     ()V

# 람다 본문:
# private static void lambda$test$0(int);
#   Code:
#      0: getstatic #4   // System.out
#      3: iload_0        // 캡처된 x
#      4: invokevirtual #5  // println(int)
#      7: return

메커니즘:
  1. 캡처 변수를 invokedynamic 파라미터로 전달
  2. 람다 메서드에 파라미터 추가
  3. 생성된 람다 인스턴스의 필드로 저장
```

---

### 5. 람다 vs 익명 클래스

```
바이트코드 비교:

익명 클래스:
  class Outer {
      void test() {
          Runnable r = new Runnable() {
              public void run() { ... }
          };
      }
  }
  
  → Outer$1.class 파일 생성
  → 컴파일 타임에 구조 고정

람다:
  class Outer {
      void test() {
          Runnable r = () -> { ... };
      }
  }
  
  → 별도 .class 파일 없음
  → 런타임에 동적 생성 (Outer$$Lambda$1)
  → 클래스 로더에만 존재 (파일 시스템에 없음)

메모리:
  익명 클래스:
  매번 new → 새 인스턴스
  
  람다 (캡처 없음):
  싱글톤처럼 재사용 가능
  
  Runnable r1 = () -> System.out.println("Hello");
  Runnable r2 = () -> System.out.println("Hello");
  // r1 == r2 일 수도 있음 (구현 의존적)
```

---

## 💻 실험으로 확인하기

### 실험 1: 람다 바이트코드 확인

```java
import java.util.function.*;

public class LambdaTypes {
    public void test() {
        // 캡처 없음
        Runnable r1 = () -> System.out.println("No capture");
        
        // 캡처 있음
        int x = 42;
        Runnable r2 = () -> System.out.println(x);
        
        // 메서드 참조
        Runnable r3 = System.out::println;
    }
}
```

```bash
javap -c -v LambdaTypes.class

# r1 (캡처 없음):
# invokedynamic #2, 0  // run:()Ljava/lang/Runnable;
# Method arguments:
#   ()V
#   REF_invokeStatic lambda$test$0:()V
#   ()V

# r2 (캡처 있음):
# invokedynamic #4, 0  // run:(I)Ljava/lang/Runnable;
#                       ↑ int 파라미터 (캡처된 x)
# Method arguments:
#   ()V
#   REF_invokeStatic lambda$test$1:(I)V
#   ()V

# r3 (메서드 참조):
# getstatic System.out
# invokedynamic #5, 0  // run:(Ljava/io/PrintStream;)Ljava/lang/Runnable;
# Method arguments:
#   ()V
#   REF_invokeVirtual PrintStream.println:()V
#   ()V
```

---

### 실험 2: 런타임 생성 클래스 확인

```java
public class LambdaClassDemo {
    public static void main(String[] args) {
        Runnable r = () -> System.out.println("Hello");
        
        System.out.println("Lambda class: " + r.getClass().getName());
        System.out.println("Declared methods:");
        for (var m : r.getClass().getDeclaredMethods()) {
            System.out.println("  " + m);
        }
    }
}
```

```bash
# 출력:
# Lambda class: LambdaClassDemo$$Lambda$1/0x0000000800c00400
#               ↑ 동적 생성된 클래스 (파일 없음)
# Declared methods:
#   public void LambdaClassDemo$$Lambda$1.run()
```

---

### 실험 3: 람다 재사용 확인

```java
public class LambdaSingleton {
    public static void main(String[] args) {
        // 캡처 없는 람다
        Runnable r1 = () -> System.out.println("A");
        Runnable r2 = () -> System.out.println("A");
        System.out.println("r1 == r2: " + (r1 == r2));  // true 가능
        
        // 캡처 있는 람다
        int x = 10;
        Runnable r3 = () -> System.out.println(x);
        Runnable r4 = () -> System.out.println(x);
        System.out.println("r3 == r4: " + (r3 == r4));  // false (새 인스턴스)
    }
}
```

---

### 실험 4: -Djdk.internal.lambda.dumpProxyClasses

```bash
# 람다 클래스를 파일로 덤프
java -Djdk.internal.lambda.dumpProxyClasses=. LambdaDemo

# 생성된 파일:
# LambdaDemo$$Lambda$1.class

# 디컴파일:
javap -c LambdaDemo\$\$Lambda\$1.class

# 출력:
# final class LambdaDemo$$Lambda$1 implements Runnable {
#   public void run();
#     Code:
#        0: invokestatic LambdaDemo.lambda$test$0:()V
#        3: return
# }
```

---

## ⚡ 실무 임팩트

### 람다 성능 최적화

```
캡처 없는 람다:
  Supplier<String> s1 = () -> "Hello";
  Supplier<String> s2 = () -> "Hello";
  
  → JVM이 싱글톤으로 최적화 가능
  → 메모리 효율적

캡처 있는 람다:
  int x = 10;
  Supplier<Integer> s = () -> x;
  
  → 매번 새 인스턴스 생성
  → 캡처 변수를 필드로 저장
  
루프 내 람다 (캡처 있음):
  for (int i = 0; i < 1000; i++) {
      int x = i;
      list.add(() -> x);  // 1000개 인스턴스 생성
  }
  
  → 메모리 압박
  
최적화:
  가능하면 캡처 없는 람다 사용
  또는 메서드 참조로 대체
```

### invokedynamic의 미래 확장성

```
JDK 버전별 개선:

Java 8:
  LambdaMetafactory 기본 구현
  → 내부 클래스 동적 생성

Java 9+:
  String Concatenation도 invokedynamic 사용
  "Hello " + name → invokedynamic
  
Java 17+:
  Switch 표현식 최적화
  Pattern Matching

장점:
  바이트코드는 invokedynamic만 유지
  → JVM 업그레이드 시 자동으로 최적화 적용
  → 재컴파일 불필요
```

### MethodHandle vs Reflection

```
Reflection:
  Method m = String.class.getMethod("length");
  int len = (int) m.invoke("hello");
  
  → 느림 (타입 체크, boxing)
  → 보안 검사

MethodHandle:
  MethodHandles.Lookup lookup = MethodHandles.lookup();
  MethodHandle mh = lookup.findVirtual(String.class, "length", 
                                       MethodType.methodType(int.class));
  int len = (int) mh.invokeExact("hello");
  
  → 빠름 (JIT 인라인 가능)
  → invokedynamic의 기반

성능:
  Reflection: 100 ns
  MethodHandle: 1 ns (JIT 후)
  → 100배 차이
```

---

## 🚫 흔한 오해

### "람다는 익명 클래스의 문법 설탕이다"

```
❌ 잘못된 이해:
  람다는 익명 클래스를 간결하게 쓴 것일 뿐이다.

✅ 실제:
  완전히 다른 메커니즘
  
익명 클래스:
  - 컴파일 타임에 클래스 파일 생성
  - 항상 새 객체 (new)
  - this는 익명 클래스 인스턴스

람다:
  - 런타임에 동적 생성 (invokedynamic)
  - 재사용 가능 (캡처 없는 경우)
  - this는 외부 클래스 (enclosing class)

예:
  class Outer {
      void test() {
          // 익명 클래스
          Runnable r1 = new Runnable() {
              public void run() {
                  System.out.println(this.getClass());  // Outer$1
              }
          };
          
          // 람다
          Runnable r2 = () -> {
              System.out.println(this.getClass());  // Outer
          };
      }
  }
```

### "람다는 항상 익명 클래스보다 빠르다"

```
❌ 잘못된 이해:
  람다를 쓰면 무조건 성능이 좋아진다.

✅ 실제:
  Warm-up 후에는 거의 동일
  
벤치마크:
  익명 클래스: 1.0 ns (JIT 인라인 후)
  람다:       1.0 ns (JIT 인라인 후)
  
차이점:
  - 클래스 로딩 시간: 람다가 빠름 (동적 생성)
  - 메모리: 람다가 적음 (재사용 가능)
  - 코드 크기: 람다가 작음 (.class 파일 없음)
  
  실행 속도는 거의 동일
```

### "invokedynamic은 항상 느리다"

```
❌ 잘못된 이해:
  동적 호출이라 느릴 것이다.

✅ 실제:
  첫 호출만 느림, 이후는 빠름
  
비용:
  첫 호출:
  - Bootstrap Method 실행 (~100 us)
  - 클래스 동적 생성
  - CallSite 생성
  → 느림
  
  이후 호출:
  - CallSite 캐시 사용
  - JIT 인라인 가능
  → invokevirtual과 동일하거나 더 빠름

권장:
  Hot Path에서 람다 사용 OK
  단, 루프 밖으로 이동 (재사용)
```

---

## 📌 핵심 정리

```
람다 바이트코드
  invokedynamic 명령어 사용
  BootstrapMethod: LambdaMetafactory.metafactory
  람다 본문: 컴파일러 생성 static 메서드

invokedynamic 구조
  Bootstrap Method (첫 호출 시 실행)
  → CallSite 생성 (캐싱)
  → MethodHandle (실제 호출 대상)

LambdaMetafactory
  런타임에 람다 구현 클래스 동적 생성
  예: Outer$$Lambda$1 implements Runnable
  ASM으로 바이트코드 생성

캡처 (Capture)
  외부 변수를 람다 내부에서 사용
  → invokedynamic 파라미터로 전달
  → 람다 인스턴스 필드로 저장
  → effectively final 필수

람다 vs 익명 클래스
  람다: 동적 생성, 재사용 가능, this=외부
  익명: 컴파일 타임 생성, 항상 새 객체, this=자신

성능
  첫 호출: Bootstrap Method 비용
  이후: 캐시 사용, JIT 인라인 가능
  Warm-up 후: 익명 클래스와 동일

미래 확장성
  바이트코드는 invokedynamic만
  JVM 업그레이드 시 자동 최적화
  재컴파일 불필요
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 두 람다의 바이트코드 차이를 설명하라. invokedynamic 파라미터와 람다 메서드 시그니처에 초점을 맞춰라.

```java
// 람다 1
Runnable r1 = () -> System.out.println("Hello");

// 람다 2
int x = 42;
Runnable r2 = () -> System.out.println(x);
```

**Q2.** 왜 람다는 `invokedynamic`을 사용하는가? 컴파일 타임에 익명 클래스로 변환하지 않는 이유를 3가지 이상 설명하라.

**Q3.** 다음 코드의 성능 문제를 지적하고 개선 방법을 제시하라.

```java
for (int i = 0; i < 10000; i++) {
    int x = i;
    executor.submit(() -> process(x));
}
```

> 💡 **해설**
>
> **Q1.** 람다 1 (캡처 없음): `invokedynamic #2, 0 // run:()Ljava/lang/Runnable;` — 파라미터 없음. 람다 메서드: `lambda$test$0:()V` — 파라미터 없음. 싱글톤처럼 재사용 가능. 람다 2 (캡처 있음): `invokedynamic #4, 0 // run:(I)Ljava/lang/Runnable;` — int 파라미터 (x=42). 람다 메서드: `lambda$test$1:(I)V` — int 파라미터 받음. 매번 새 인스턴스 생성 (x 값을 필드로 저장). 핵심: 캡처 변수는 invokedynamic 파라미터로 전달 → 람다 인스턴스 생성 시 필드로 저장.
>
> **Q2.** ① 미래 확장성 — JVM이 람다 구현 방식을 자유롭게 변경 가능. Java 8은 내부 클래스 생성, 미래에는 더 나은 방법 적용 가능 (바이트코드 재컴파일 불필요). ② 메모리 효율 — 캡처 없는 람다는 싱글톤처럼 재사용 가능. 익명 클래스는 매번 new로 새 객체. ③ 클래스 파일 최소화 — 익명 클래스는 Outer$1.class 파일 생성. 람다는 런타임 동적 생성 (파일 없음). ④ 최적화 기회 — invokedynamic의 Bootstrap Method가 최적화된 구현 선택 (예: 상태 없으면 싱글톤, 있으면 새 인스턴스). ⑤ 타입 추론 — 람다는 타입 소거 후에도 정보 유지 가능 (MethodType).
>
> **Q3.** 문제: 루프 10,000회마다 캡처 있는 람다 생성 → 10,000개 람다 인스턴스 + 10,000번 invokedynamic 첫 호출 오버헤드 (각 i마다 다른 CallSite). 개선: ① 람다를 루프 밖으로 이동 (불가능, x가 루프 변수). ② 익명 클래스 또는 별도 클래스 사용 — `executor.submit(new ProcessTask(i))` where `ProcessTask implements Runnable`. ③ 메서드 참조로 변경 (가능하다면). ④ 배치 처리 — 10,000개를 모아서 한 번에 처리. 핵심: 캡처 있는 람다를 대량 생성하면 메모리와 GC 압박. 가능하면 재사용 가능한 구조로 변경.

---

## 📚 참고 자료

- [JEP 126 — Lambda Expressions & Virtual Extension Methods](https://openjdk.org/jeps/126)
- [Translation of Lambda Expressions (Brian Goetz)](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html)
- [JVMS §4.7.23 — BootstrapMethods Attribute](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html#jvms-4.7.23)

---

<div align="center">

**[⬅️ 이전: Exception Handling Bytecode](./05-exception-handling-bytecode.md)** | **[다음: Bytecode Manipulation (ASM) ➡️](./07-bytecode-manipulation-asm.md)**

</div>

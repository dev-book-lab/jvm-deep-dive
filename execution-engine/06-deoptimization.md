# Deoptimization - 역최적화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Deoptimization은 무엇이며, 왜 발생하는가?
- Speculative Optimization은 무엇이며, 어떻게 실패하는가?
- JIT 코드에서 Interpreter로 복귀하는 과정은 어떻게 진행되는가?
- "made not entrant"와 "made zombie"의 차이는 무엇인가?
- Deoptimization을 최소화하려면 어떻게 해야 하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 최적화는 가정에 기반한다

```java
interface Shape {
    double area();
}

class Circle implements Shape {
    double radius;
    public double area() { return Math.PI * radius * radius; }
}

void process(Shape shape) {
    double a = shape.area();
}
```

```
초기 상황 (Circle만 사용):
  shape.area() 호출
  → JIT가 프로파일링: 항상 Circle
  → Speculative Optimization:
     "shape는 항상 Circle이다"
     → Virtual Call 제거
     → Circle.area() 직접 호출 (빠름)

나중에 Square 추가:
  class Square implements Shape { ... }
  process(new Square());
  
  문제:
  최적화된 코드는 Circle만 가정
  → Square가 오면 잘못된 결과
  
  해결:
  Deoptimization → Interpreter 복귀
  → 재컴파일 (Circle + Square 모두 고려)
```

Deoptimization은 **잘못된 가정을 수정**한다.

---

## 📐 내부 구조

### 1. Speculative Optimization

```
JIT가 하는 가정들:

1. Type Speculation (타입 예측)
   void process(List<Animal> list) {
       for (Animal a : list) {
           a.sound();
       }
   }
   
   프로파일링: 항상 Dog
   → Monomorphic Call Site
   → Dog.sound() 직접 호출 (Virtual Call 제거)
   
   나중에 Cat 등장:
   → Deoptimization

2. Null Check Elimination
   if (obj != null) {
       obj.method();
   }
   
   프로파일링: obj는 항상 non-null
   → null 체크 제거
   
   나중에 null 등장:
   → Deoptimization

3. Range Check Elimination
   for (int i = 0; i < arr.length; i++) {
       sum += arr[i];
   }
   
   프로파일링: i는 항상 범위 내
   → 경계 체크 제거
   
   나중에 범위 벗어남 (버그):
   → Deoptimization

4. Uncommon Trap
   if (unlikely_condition) {
       // 거의 실행 안 됨
   }
   
   → 이 분기를 컴파일 안 함
   → 실행되면 Deoptimization
```

---

### 2. Deoptimization 과정

```
실행 흐름:

1. JIT 코드 실행 중
   0x7f3a2b10000: mov rax, [rsi]
   0x7f3a2b10003: cmp [rax], Dog.class
   0x7f3a2b10007: jne uncommon_trap  ← 가정 실패 감지
   0x7f3a2b10009: call Dog.sound()

2. Uncommon Trap 발생
   jne uncommon_trap
   → Cat이 등장
   → uncommon_trap 루틴 실행

3. Stack Frame 복구
   JIT Frame → Interpreter Frame 변환
   - 지역 변수 복사
   - Operand Stack 재구성
   - PC 설정 (바이트코드 위치)

4. JIT 코드 무효화
   "made not entrant"
   → 새 진입 금지
   → 실행 중인 건 완료까지 허용

5. Interpreter로 재실행
   현재 바이트코드부터 Interpreter 실행

6. 재컴파일 (선택적)
   새로운 프로파일링 데이터로
   → Polymorphic Call Site 반영
   → 재최적화
```

---

### 3. Uncommon Trap

```
Uncommon Trap: Deoptimization 트리거 지점

코드 예시:
  JIT 코드:
  if (shape.getClass() != Circle.class) {
      uncommon_trap();  ← Deoptimization 트리거
  }
  // Circle.area() 직접 호출
  
Uncommon Trap 종류:

1. class_check
   타입 가정 실패
   
2. null_check
   Null 가정 실패
   
3. range_check
   배열 범위 가정 실패
   
4. div0_check
   0으로 나누기
   
5. intrinsic_or_type_checked_inlining
   인라인 가정 실패

발생 빈도:
  Uncommon Trap은 "uncommon"해야 함
  → 자주 발생하면 성능 저하
  → 10회 이상 발생 시 재컴파일 비활성화
```

---

### 4. PrintCompilation에서 Deoptimization 확인

```bash
java -XX:+PrintCompilation MyApp

# Deoptimization 표시:
#    150    2       4       MyClass::process (42 bytes)
#    200    2       4       MyClass::process (42 bytes)   made not entrant
#                                                        ↑ Deoptimization
#    250    3       4       MyClass::process (42 bytes)
#         ↑ 재컴파일 (새 Compilation ID)

# 추가 정보:
# -XX:+PrintDeoptimizationDetails
#    Uncommon trap: reason=class_check action=reinterpret
#    @ bci=15 MyClass::process
```

---

### 5. Made Not Entrant vs Made Zombie

```
JIT 코드 생명주기:

1. Active
   정상 실행 중
   새 호출 가능

2. Made Not Entrant (비활성)
   Deoptimization 발생
   → 새 진입 금지
   → 실행 중인 Stack Frame은 완료까지 유지
   
   예:
   Thread 1: JIT 코드 실행 중 (완료까지 허용)
   Thread 2: 새 호출 → Interpreter 또는 새 JIT 버전

3. Made Zombie (좀비)
   모든 Stack Frame 종료됨
   → 메모리 회수 대기
   → Code Cache에서 제거 예정

4. Reclaimed (회수)
   메모리 완전히 제거
   
타임라인:
  0ms:   Active
  100ms: Deoptimization → Made Not Entrant
  150ms: 마지막 Frame 종료 → Made Zombie
  200ms: GC → Reclaimed
```

---

## 💻 실험으로 확인하기

### 실험 1: Deoptimization 발생시키기

```java
interface Shape {
    double area();
}

class Circle implements Shape {
    double radius = 1.0;
    public double area() { return Math.PI * radius * radius; }
}

class Square implements Shape {
    double side = 1.0;
    public double area() { return side * side; }
}

public class DeoptDemo {
    public static void main(String[] args) {
        Shape shape;
        
        // Warm-up: Circle만 사용
        for (int i = 0; i < 20000; i++) {
            shape = new Circle();
            process(shape);
        }
        
        System.out.println("=== Circle 최적화 완료 ===");
        
        // Square 도입 → Deoptimization
        for (int i = 0; i < 10; i++) {
            shape = new Square();
            process(shape);
        }
        
        System.out.println("=== Deoptimization 발생 ===");
    }
    
    static double process(Shape shape) {
        return shape.area();
    }
}
```

```bash
java -XX:+PrintCompilation \
     -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintDeoptimization \
     DeoptDemo

# 출력:
#     80    1       3       DeoptDemo::process (5 bytes)
#     90    2       4       DeoptDemo::process (5 bytes)
# === Circle 최적화 완료 ===
#     Uncommon trap: reason=class_check action=maybe_recompile
#     95    2       4       DeoptDemo::process (5 bytes)   made not entrant
# === Deoptimization 발생 ===
#    100    3       4       DeoptDemo::process (5 bytes)
```

---

### 실험 2: Uncommon Trap 빈도 확인

```bash
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintDeoptimizationDetails \
     -XX:+LogCompilation \
     -XX:LogFile=deopt.log \
     MyApp

# deopt.log 분석:
# <uncommon_trap thread='...' reason='class_check' action='reinterpret'>
#   <jvms bci='15' method='MyClass process' .../>
# </uncommon_trap>
```

---

### 실험 3: 성능 영향 측정

```java
public class DeoptBenchmark {
    interface Worker {
        int work(int x);
    }
    
    static class TypeA implements Worker {
        public int work(int x) { return x * 2; }
    }
    
    static class TypeB implements Worker {
        public int work(int x) { return x * 3; }
    }
    
    public static void main(String[] args) {
        // Monomorphic (단일 타입)
        long start = System.nanoTime();
        Worker w = new TypeA();
        int sum = 0;
        for (int i = 0; i < 10_000_000; i++) {
            sum += w.work(i);
        }
        long mono = System.nanoTime() - start;
        
        // Polymorphic (여러 타입)
        start = System.nanoTime();
        sum = 0;
        for (int i = 0; i < 10_000_000; i++) {
            w = (i % 2 == 0) ? new TypeA() : new TypeB();
            sum += w.work(i);
        }
        long poly = System.nanoTime() - start;
        
        System.out.println("Monomorphic: " + mono / 1_000_000 + " ms");
        System.out.println("Polymorphic: " + poly / 1_000_000 + " ms");
        System.out.println("Slowdown: " + (double)poly / mono + "x");
    }
}
```

```bash
# 출력:
# Monomorphic: 20 ms (최적화됨)
# Polymorphic: 100 ms (Deoptimization + Virtual Call)
# Slowdown: 5.0x
```

---

## ⚡ 실무 임팩트

### Deoptimization 회피 패턴

```java
// ❌ Polymorphic Call Site (Deoptimization 유발)
void process(List<Animal> animals) {
    for (Animal a : animals) {
        a.sound();  // Dog, Cat, Bird 섞임
    }
}

// ✅ Monomorphic (타입별 분리)
void processDogs(List<Dog> dogs) {
    for (Dog d : dogs) {
        d.sound();  // Dog만
    }
}

void processCats(List<Cat> cats) {
    for (Cat c : cats) {
        c.sound();  // Cat만
    }
}

// 또는 타입별 분류 후 처리
Map<Class<?>, List<Animal>> byType = animals.stream()
    .collect(Collectors.groupingBy(Animal::getClass));

byType.forEach((type, list) -> {
    for (Animal a : list) {
        a.sound();  // 각 루프는 Monomorphic
    }
});
```

### Null 처리

```java
// ❌ Null이 가끔 등장 (Deoptimization)
void process(User user) {
    if (user != null) {
        user.getName();  // JIT: null 없다 가정 → 실패
    }
}

// ✅ Null을 명확히 처리
void process(User user) {
    Objects.requireNonNull(user);  // 명시적 체크
    user.getName();  // JIT: non-null 확신
}

// 또는
Optional<User> user = ...;
user.ifPresent(u -> u.getName());  // Null 분리
```

### Megamorphic Call Site 회피

```
Call Site 분류:

Monomorphic (1개 타입):
  최적화 가능 (Virtual Call 제거)
  
Polymorphic (2~3개 타입):
  조건부 최적화 (타입 체크 + 인라인)
  
Megamorphic (4개 이상 타입):
  최적화 불가 (Virtual Call 유지)

권장:
  타입을 3개 이하로 제한
  또는 타입별로 코드 분리
```

---

## 🚫 흔한 오해

### "Deoptimization은 버그다"

```
❌ 잘못된 이해:
  Deoptimization이 발생하면 코드에 문제가 있다.

✅ 실제:
  정상적인 JVM 동작
  
  Speculative Optimization:
  "현재까지 관찰한 패턴"으로 최적화
  → 패턴 변화 시 조정
  
  예:
  초기: Dog만 등장 → Dog에 최적화
  나중: Cat 등장 → 재최적화
  
  이는 JIT의 adaptive 특성
  버그 아님, 설계 의도
```

### "Deoptimization은 항상 느리다"

```
❌ 잘못된 이해:
  Deoptimization이 발생하면 성능이 떨어진다.

✅ 실제:
  일시적 영향, 재최적화 후 회복
  
흐름:
  1. Deoptimization 발생 (수백 us)
  2. Interpreter 실행 (느림)
  3. 재컴파일 (백그라운드)
  4. 새 최적화 코드 (빠름)
  
총 영향:
  수백 ms 정도만 느림
  이후 정상 성능 회복
  
문제가 되는 경우:
  Deoptimization이 반복됨 (Thrashing)
  → 타입이 계속 바뀜
  → 최적화 포기
```

### "Made Not Entrant는 즉시 메모리 해제"

```
❌ 잘못된 이해:
  "made not entrant" 표시 후 즉시 메모리 회수된다.

✅ 실제:
  여러 단계를 거침
  
  1. Made Not Entrant
     실행 중인 Frame 유지 (수 ms ~ 수 s)
  
  2. Made Zombie
     모든 Frame 종료 후 (수 s ~ 수 분)
  
  3. Reclaimed
     GC 시점에 회수 (불규칙)
  
  이유:
  - 안전성 (실행 중 코드 제거 안 함)
  - 메모리는 Code Cache에서 관리
    GC가 아닌 별도 메커니즘
```

---

## 📌 핵심 정리

```
Deoptimization
  JIT 코드 → Interpreter 복귀
  잘못된 Speculative Optimization 수정

발생 원인
  Type Speculation 실패 (다형성)
  Null Check 가정 실패
  Range Check 가정 실패
  Uncommon Trap 도달

Uncommon Trap
  Deoptimization 트리거 지점
  가정 검증 실패 시 발동
  종류: class_check, null_check, range_check 등

과정
  1. Uncommon Trap 감지
  2. Stack Frame 복구 (JIT → Interpreter)
  3. JIT 코드 무효화 (made not entrant)
  4. Interpreter 재실행
  5. 재컴파일 (선택적)

생명주기
  Active → Made Not Entrant → Made Zombie → Reclaimed

성능 영향
  Monomorphic: 최적화 (빠름)
  Polymorphic: 조건부 최적화 (보통)
  Megamorphic: 최적화 불가 (느림)

회피 방법
  타입 분리 (Monomorphic 유지)
  Null 명시적 처리
  타입 수 제한 (3개 이하)

PrintCompilation
  "made not entrant" = Deoptimization
  재컴파일 시 새 Compilation ID
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 Deoptimization이 발생할 가능성이 있는가? 어떤 Uncommon Trap이 트리거될 수 있는가?

```java
void process(List<Shape> shapes) {
    for (Shape s : shapes) {
        if (s != null) {
            double a = s.area();
            System.out.println(a);
        }
    }
}
```

**Q2.** "made not entrant"와 "made zombie"의 차이를 설명하라. 왜 즉시 메모리를 회수하지 않는가?

**Q3.** Monomorphic, Polymorphic, Megamorphic Call Site의 성능 차이를 Deoptimization과 연결해 설명하라.

> 💡 **해설**
>
> **Q1.** 가능한 Deoptimization: ① class_check — shapes에 Circle만 있다가 Square 추가 → Type Speculation 실패. JIT가 "s는 항상 Circle"로 가정했다가 실패 → Deoptimization. ② null_check — s가 항상 non-null이다 가정 → null 등장 시 실패. 하지만 명시적 null 체크가 있어 가능성 낮음 (JIT가 체크 인식). 빈도: class_check가 더 흔함 (타입 변화). null_check는 드뭄 (이미 체크 있음). 최적화: 타입별로 분리 (`processCircles`, `processSquares`) → Monomorphic 유지.
>
> **Q2.** "made not entrant" — JIT 코드가 무효화됨. 새 호출은 금지되지만, 현재 실행 중인 Stack Frame은 완료까지 허용됨 (안전성). 예: Thread 1이 JIT 코드 실행 중이면 종료까지 실행. "made zombie" — 모든 Stack Frame이 종료됨. 아무도 코드를 사용하지 않음. 메모리 회수 대기 상태. 즉시 회수 안 하는 이유: ① 실행 중 코드 제거는 위험 (크래시). ② Code Cache 관리는 별도 메커니즘 (GC 아님). ③ 회수 비용 (메타데이터 정리 등) → 일괄 처리가 효율적.
>
> **Q3.** Monomorphic (1개 타입) — JIT가 Virtual Call 제거 → 직접 호출 → 인라인 가능 → 매우 빠름. Deoptimization 없음 (타입 변화 없음). Polymorphic (2~3개) — JIT가 타입 체크 + 조건부 인라인. 예: `if (type == Circle) inlined_circle() else if (type == Square) inlined_square()`. 약간 느림 (분기). Deoptimization 가능 (새 타입 추가 시). Megamorphic (4개 이상) — 최적화 포기 → Virtual Call 유지 → 인라인 불가 → 느림. Deoptimization 빈번 (타입 계속 변함) → JIT가 컴파일 포기. 성능: Mono 1x, Poly 2~3x, Mega 5~10x 느림.

---

## 📚 참고 자료

- [HotSpot Deoptimization](https://wiki.openjdk.org/display/HotSpot/Deoptimization)
- [Uncommon Traps in HotSpot](https://shipilev.net/blog/2014/exceptional-performance/)
- [Understanding Deoptimization](https://krzysztofslusarski.github.io/2022/12/12/async-manual.html#deoptimization)

---

<div align="center">

**[⬅️ 이전: On-Stack Replacement (OSR)](./05-on-stack-replacement.md)** | **[다음: JVM Intrinsics ➡️](./07-intrinsics.md)**

</div>

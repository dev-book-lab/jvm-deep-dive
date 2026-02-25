# On-Stack Replacement (OSR) - 온스택 교체

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- OSR (On-Stack Replacement)은 무엇이며, 왜 필요한가?
- 실행 중인 메서드를 어떻게 Interpreter에서 JIT 버전으로 교체하는가?
- Backedge Counter는 OSR과 어떤 관계인가?
- OSR은 일반 JIT 컴파일과 어떻게 다른가?
- `-XX:+PrintCompilation`에서 `%` 표시는 무엇을 의미하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 긴 루프는 언제 최적화되는가

```java
void longLoop() {
    for (int i = 0; i < 1_000_000; i++) {
        compute(i);
    }
}

longLoop();
```

```
일반 JIT 컴파일:
  메서드 진입 시 Invocation Counter 증가
  → 10,000회 호출 시 컴파일
  
  문제:
  longLoop()는 1회만 호출됨
  → Invocation Counter = 1
  → JIT 컴파일 안 됨
  → 100만 번 루프를 모두 Interpreter로 실행
  → 매우 느림

해결책:
  루프 백점프(backedge) 추적
  → 루프 반복 횟수로 컴파일 결정
  → 루프 도중에 JIT 버전으로 교체
```

OSR은 **실행 중인 메서드를 JIT 버전으로 교체**한다.

---

## 📐 내부 구조

### 1. OSR 동작 원리

```
일반 컴파일 vs OSR:

일반 컴파일:
  메서드 진입 → Counter 증가 → Threshold → 컴파일
  → 다음 호출부터 JIT 버전 사용

OSR (On-Stack Replacement):
  루프 실행 중 → Backedge Counter 증가 → Threshold
  → 컴파일 (루프만)
  → 실행 중인 Stack Frame을 JIT 버전으로 교체
  → 루프 도중부터 JIT 실행

예시:
  for (int i = 0; i < 100000; i++) {
      compute(i);
  }
  
  i=0~9999:   Interpreter 실행
  i=10000:    Backedge Counter Threshold 초과
              → OSR 컴파일 시작 (백그라운드)
  i=10050:    OSR 컴파일 완료
              → Stack Frame 교체
  i=10051~:   JIT 버전 실행 (빠름)
```

---

### 2. Backedge Counter

```
Backedge: 루프의 끝에서 시작으로 점프

for (int i = 0; i < n; i++) {
    // loop body
    // ← backedge: 여기서 i++ → for 시작으로 점프
}

바이트코드:
   0: iconst_0
   1: istore_1      // i = 0
   2: iload_1
   3: iload_2
   4: if_icmpge 15  // i >= n이면 15로 점프
   7: ...           // loop body
  12: iinc 1, 1     // i++
  15: goto 2        ← Backedge! (Counter 증가)
  18: return

Backedge Counter:
  매 루프 반복마다 증가
  Threshold (OnStackReplacePercentage * CompileThreshold / 100)
  기본: 140 * 10000 / 100 = 14000
  
  → 14,000회 반복 시 OSR 컴파일
```

---

### 3. OSR Entry Point

```
OSR 컴파일 결과:

일반 메서드 컴파일:
  0x7f3a2b10000: method entry point
  → 메서드 시작부터 실행

OSR 컴파일:
  0x7f3a2b20000: method entry point (사용 안 함)
  0x7f3a2b20100: OSR entry point
  → 루프 도중부터 실행 (i=10000 시점)

OSR Entry에서 해야 할 일:
  1. Stack Frame 복사
     Interpreter Frame → JIT Frame
     
  2. 지역 변수 복사
     i=10000 (현재 값)
     sum=... (누적 값)
     
  3. Operand Stack 복사
     
  4. PC 설정
     → 루프 본문 시작으로
```

---

### 4. OSR 컴파일 트리거

```
조건:

1. Backedge Counter > Threshold
   기본: 14,000 (OnStackReplacePercentage * CompileThreshold / 100)

2. 메서드가 현재 실행 중
   Stack Frame이 활성 상태

3. 루프가 충분히 Hot
   최근 백점프가 빈번함

OSR 컴파일 대상:
  메서드 전체가 아닌 루프 본문만
  → 컴파일 시간 단축
  → 빠른 전환
```

---

### 5. PrintCompilation에서 OSR 확인

```bash
java -XX:+PrintCompilation MyApp

# OSR 표시 (%):
#    120    1    %  3       MyClass::longLoop @ 15 (50 bytes)
#                ↑ OSR 표시
#                   ↑ Level 3 (C1)
#                      ↑ @ 15: 바이트코드 오프셋 (backedge 위치)

# 일반 컴파일 (% 없음):
#    150    2       3       MyClass::shortMethod (20 bytes)

설명:
  % = OSR 컴파일
  @ 15 = 루프 백점프 바이트코드 위치
  (50 bytes) = 메서드 전체 크기
```

---

## 💻 실험으로 확인하기

### 실험 1: OSR 발생 관찰

```java
public class OSRDemo {
    public static void main(String[] args) {
        longLoop();
    }
    
    static void longLoop() {
        int sum = 0;
        for (int i = 0; i < 100000; i++) {
            sum += i;
            
            // 진행 상황 출력
            if (i % 10000 == 0) {
                System.out.println("i = " + i);
            }
        }
        System.out.println("Sum: " + sum);
    }
}
```

```bash
java -XX:+PrintCompilation OSRDemo

# 출력:
# i = 0
# i = 10000
#    85    1    %  3       OSRDemo::longLoop @ 15 (60 bytes)
#         ↑ OSR 발생 (i=10000 이후)
# i = 20000
#    90    2    %  4       OSRDemo::longLoop @ 15 (60 bytes)
#         ↑ C2로 재컴파일
# i = 30000
# ...
```

---

### 실험 2: OSR Threshold 조정

```bash
# 기본 Threshold (14000)
java -XX:+PrintCompilation OSRDemo

# 낮은 Threshold (1000)
java -XX:OnStackReplacePercentage=10 \
     -XX:+PrintCompilation \
     OSRDemo
# → 1000회만 반복해도 OSR

# 높은 Threshold (100000)
java -XX:OnStackReplacePercentage=1000 \
     -XX:+PrintCompilation \
     OSRDemo
# → 100,000회 반복해야 OSR
```

---

### 실험 3: OSR 성능 영향 측정

```java
public class OSRBenchmark {
    public static void main(String[] args) {
        // OSR 활성화 (기본)
        long start = System.nanoTime();
        longLoop();
        long withOSR = System.nanoTime() - start;
        
        System.out.println("With OSR: " + withOSR / 1_000_000 + " ms");
    }
    
    static void longLoop() {
        long sum = 0;
        for (int i = 0; i < 10_000_000; i++) {
            sum += i;
        }
        System.out.println("Sum: " + sum);
    }
}
```

```bash
# OSR 활성화 (기본)
java OSRBenchmark
# With OSR: 50 ms

# OSR 비활성화 (Interpreter만)
java -XX:-UseOnStackReplacement OSRBenchmark
# With OSR: 500 ms (10배 느림)
```

---

## ⚡ 실무 임팩트

### 초기화 루프 최적화

```java
// 애플리케이션 시작 시 데이터 로딩
void loadData() {
    for (int i = 0; i < 1_000_000; i++) {
        data.add(processRecord(i));
    }
}

// OSR 덕분에:
// i=0~14000: Interpreter (느림)
// i=14000~: JIT (빠름)
// → 초기 로딩 시간 단축
```

### 배치 처리

```java
void processBatch(List<Record> records) {
    for (Record r : records) {  // 수백만 건
        process(r);
    }
}

// 1회 호출 (Invocation Counter = 1)
// → 일반 JIT 컴파일 안 됨
// → OSR로 루프 최적화
```

### OSR의 한계

```
OSR 컴파일은 루프만 최적화:
  - 루프 이전 코드는 Interpreter
  - 루프 이후 코드도 Interpreter
  
예:
  void method() {
      setup();           // Interpreter
      for (...) {        // OSR → JIT
          compute();
      }
      cleanup();         // Interpreter
  }

→ 메서드 전체 최적화는 다음 호출부터
```

---

## 🚫 흔한 오해

### "OSR은 메서드 전체를 컴파일한다"

```
❌ 잘못된 이해:
  OSR이 발생하면 메서드 전체가 JIT 컴파일된다.

✅ 실제:
  루프 본문만 컴파일
  
  void method() {
      int x = init();     // Interpreter
      for (int i = 0; i < 100000; i++) {
          compute(i);     // OSR → JIT
      }
      cleanup();          // Interpreter
  }
  
  OSR Entry Point:
  루프 시작 지점부터만 JIT 코드
  
  이유:
  - 빠른 컴파일 (루프만)
  - 빠른 전환 (루프 도중부터)
  - Stack Frame 복사 최소화
```

### "OSR은 항상 성능 향상을 보장한다"

```
❌ 잘못된 이해:
  OSR이 발생하면 무조건 빨라진다.

✅ 실제:
  상황에 따라 다름
  
빨라지는 경우:
  - 긴 루프 (수만~수백만 반복)
  - 계산 집약적 루프 본문
  
느려질 수 있는 경우:
  - OSR 전환 오버헤드
    (Stack Frame 복사 등)
  - 짧은 루프 (수천 반복)
    → 전환 비용 > 성능 이득
  
벤치마크:
  100 반복: OSR 없음 (너무 짧음)
  10,000 반복: OSR 손익분기점
  1,000,000 반복: OSR 큰 이득
```

### "OSR은 Deoptimization과 같다"

```
❌ 잘못된 이해:
  OSR = JIT → Interpreter 복귀

✅ 실제:
  OSR: Interpreter → JIT (최적화)
  Deoptimization: JIT → Interpreter (최적화 취소)
  
  정반대 방향:
  
  OSR:
  Interpreter 실행 중 → JIT로 전환
  성능 향상 목적
  
  Deoptimization:
  JIT 실행 중 → Interpreter로 복귀
  잘못된 Speculative Optimization 수정
```

---

## 📌 핵심 정리

```
OSR (On-Stack Replacement)
  실행 중인 메서드를 Interpreter → JIT 교체
  긴 루프 최적화 목적

Backedge Counter
  루프 반복 횟수 추적
  Threshold 초과 시 OSR 컴파일
  기본: 14,000 (OnStackReplacePercentage * CompileThreshold / 100)

OSR 동작
  1. 루프 반복 → Backedge Counter 증가
  2. Threshold 초과 → OSR 컴파일 (루프만)
  3. Stack Frame 복사
  4. JIT 버전으로 교체
  5. 루프 도중부터 JIT 실행

일반 컴파일과 차이
  일반: 메서드 진입 시 전환 (다음 호출부터)
  OSR: 루프 도중 전환 (현재 실행 중)

PrintCompilation
  % 표시 = OSR 컴파일
  @ N = 바이트코드 오프셋 (backedge 위치)

성능 영향
  긴 루프: 10~20배 성능 향상
  짧은 루프: 전환 오버헤드 존재

실무 활용
  초기화 루프 (데이터 로딩)
  배치 처리 (1회 호출, 긴 루프)
  스트리밍 처리

비활성화
  -XX:-UseOnStackReplacement
  (디버깅, 성능 비교용)
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 OSR이 발생하는가? 발생한다면 어느 시점에 발생하며, 발생하지 않는다면 그 이유는?

```java
void processData() {
    for (int i = 0; i < 100; i++) {
        heavyComputation(i);
    }
}

// 이 메서드를 100,000번 호출
for (int j = 0; j < 100000; j++) {
    processData();
}
```

**Q2.** OSR 컴파일 시 Stack Frame을 어떻게 복사하는가? 지역 변수 `i`, `sum`, Operand Stack의 처리를 설명하라.

**Q3.** 다음 두 코드 중 어느 것이 더 빠른가? OSR 관점에서 설명하라.

```java
// 코드 1: 1개의 긴 루프
for (int i = 0; i < 100000; i++) {
    compute(i);
}

// 코드 2: 100개의 짧은 루프
for (int j = 0; j < 100; j++) {
    for (int i = 0; i < 1000; i++) {
        compute(i * 1000 + j);
    }
}
```

> 💡 **해설**
>
> **Q1.** OSR 발생 안 함. 이유: processData() 메서드는 Invocation Counter가 100,000까지 증가 → CompileThreshold (10,000) 초과 → 일반 JIT 컴파일됨. 이후 호출부터 JIT 버전 실행 (루프 포함). 내부 루프는 100회만 반복 → Backedge Counter가 14,000에 도달 안 함 → OSR 불필요. OSR은 "긴 루프 + 적은 호출"에서만 발생. 여기서는 "짧은 루프 + 많은 호출" → 일반 컴파일로 충분.
>
> **Q2.** OSR Entry Point 진입 시: ① Interpreter Frame에서 지역 변수 읽기 — `i` 현재 값 (예: 14000), `sum` 누적 값 복사. ② JIT Frame 생성 — 새 Stack Frame 할당, 복사한 지역 변수 저장. ③ Operand Stack 복사 — Interpreter Stack의 현재 상태를 JIT Stack으로 복사 (대부분 비어있음, 루프 시작 지점이므로). ④ PC 설정 — JIT 코드의 루프 본문 시작 지점으로 설정. ⑤ 실행 시작 — 루프 본문부터 JIT 버전 실행. 핵심: Interpreter와 JIT Frame 레이아웃이 달라도 값만 올바르게 전달하면 됨.
>
> **Q3.** 코드 1이 더 빠름. 이유: 코드 1 — 단일 루프 100,000회 → Backedge Counter 14,000 초과 → OSR 발생 → i=14,000부터 JIT 실행 → 나머지 86,000회가 빠름. 코드 2 — 외부 루프 100회 (OSR 안 됨), 내부 루프 1,000회씩 → 각 내부 루프는 1,000회라 OSR 안 됨 → 전체 100,000회를 Interpreter로 실행. 또한 외부 루프가 100회라 일반 JIT 컴파일도 안 됨 (Invocation Counter 부족). 결과: 코드 1은 OSR로 부분 최적화, 코드 2는 전혀 최적화 안 됨 → 코드 1이 10배 이상 빠름.

---

## 📚 참고 자료

- [HotSpot On-Stack Replacement](https://wiki.openjdk.org/display/HotSpot/OnStackReplacement)
- [JVM JIT OSR Optimization](https://shipilev.net/jvm/anatomy-quarks/12-on-stack-replacement/)
- [Understanding OSR in HotSpot](https://bugs.openjdk.org/browse/JDK-8046699)

---

<div align="center">

**[⬅️ 이전: JIT Optimizations](./04-jit-optimizations.md)** | **[다음: Deoptimization ➡️](./06-deoptimization.md)**

</div>

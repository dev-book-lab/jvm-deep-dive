# Reflection & Performance - Reflection과 성능

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Reflection 호출 경로는 어떻게 되는가?
- 15회 임계값은 무엇이며, 왜 존재하는가?
- Reflection이 JIT 최적화를 방해하는 이유는?
- Reflection 성능을 개선하는 방법은?

---

## 🔍 왜 이게 존재하는가

### 문제: Reflection은 느리다

```java
// 일반 호출 (~1ns)
obj.method();

// Reflection (~500ns)
Method m = clazz.getDeclaredMethod("method");
m.invoke(obj);

500배 차이!
```

Reflection은 **동적 호출의 대가**를 치른다.

---

## 📐 Reflection 호출 경로

### 1. Method.invoke() 내부

```java
// Method.invoke()
public Object invoke(Object obj, Object... args) {
    if (!override) {
        // 접근 권한 체크
        checkAccess();
    }
    
    // MethodAccessor 위임
    return methodAccessor.invoke(obj, args);
}

MethodAccessor 종류:
  1. NativeMethodAccessorImpl (초기)
  2. GeneratedMethodAccessor (15회 후)
```

---

### 2. NativeMethodAccessorImpl

```java
// 초기 구현 (0~14회)
class NativeMethodAccessorImpl {
    public Object invoke(Object obj, Object[] args) {
        // JNI 호출
        return invoke0(method, obj, args);
    }
    
    private native Object invoke0(Method m, Object obj, Object[] args);
}

특징:
  - JNI 경계 넘김
  - 비용 높음 (~500ns)
  - 유연함 (코드 생성 불필요)
```

---

### 3. GeneratedMethodAccessor (15회 후)

```java
// 동적 생성 (15회 임계값 후)
class GeneratedMethodAccessor1 {
    public Object invoke(Object obj, Object[] args) {
        // 직접 바이트코드 생성
        return ((MyClass) obj).method((Integer) args[0]);
    }
}

특징:
  - JNI 없음
  - 직접 호출 (~50ns)
  - 10배 빠름
```

---

### 4. Inflation 메커니즘

```
호출 횟수별 동작:

0~14회: NativeMethodAccessorImpl
  - JNI 호출
  - 느림 (~500ns)
  - 코드 생성 비용 없음

15회: 임계값 도달
  - GeneratedMethodAccessor 생성
  - ASM으로 바이트코드 생성
  - MethodAccessor 교체

15회~: GeneratedMethodAccessor
  - 직접 호출
  - 빠름 (~50ns)
  - JIT 최적화 가능

임계값 조정:
  -Dsun.reflect.inflationThreshold=15
```

---

## 💻 실험으로 확인하기

### 실험 1: Reflection 성능

```java
import java.lang.reflect.Method;

public class ReflectionBenchmark {
    static class Target {
        public int add(int a, int b) {
            return a + b;
        }
    }
    
    public static void main(String[] args) throws Exception {
        Target target = new Target();
        Method method = Target.class.getMethod("add", int.class, int.class);
        
        // Warm-up
        for (int i = 0; i < 20; i++) {
            method.invoke(target, 1, 2);
        }
        
        // 직접 호출 측정
        long start = System.nanoTime();
        for (int i = 0; i < 1_000_000; i++) {
            target.add(1, 2);
        }
        long directTime = System.nanoTime() - start;
        
        // Reflection 측정
        start = System.nanoTime();
        for (int i = 0; i < 1_000_000; i++) {
            method.invoke(target, 1, 2);
        }
        long reflectionTime = System.nanoTime() - start;
        
        System.out.println("Direct: " + directTime / 1_000_000 + "ms");
        System.out.println("Reflection: " + reflectionTime / 1_000_000 + "ms");
        System.out.println("Overhead: " + (reflectionTime / directTime) + "x");
    }
}
```

```bash
# 출력:
# Direct: 2ms
# Reflection: 100ms
# Overhead: 50x
```

---

### 실험 2: Inflation 임계값 확인

```java
public class InflationTest {
    static class Target {
        public void method() {}
    }
    
    public static void main(String[] args) throws Exception {
        Target target = new Target();
        Method method = Target.class.getMethod("method");
        
        for (int i = 0; i < 20; i++) {
            long start = System.nanoTime();
            method.invoke(target);
            long time = System.nanoTime() - start;
            
            System.out.println("Call " + i + ": " + time + "ns");
        }
    }
}
```

```bash
# 출력:
# Call 0: 5000ns  ← Native
# Call 1: 4800ns
# ...
# Call 14: 4500ns
# Call 15: 500ns  ← Generated (10배 빠름)
# Call 16: 450ns
```

---

## ⚡ 실무 임팩트

### Framework의 Reflection 사용

```java
// Spring, Hibernate 등
@Autowired
private MyService service;

// 내부 동작 (간략화)
Field field = clazz.getDeclaredField("service");
field.setAccessible(true);
field.set(instance, serviceInstance);

특징:
  - 초기화 시 1회 실행
  - 성능 영향 미미
  - 편의성 > 성능
```

---

### JSON 직렬화/역직렬화

```java
// Jackson, Gson
String json = objectMapper.writeValueAsString(obj);

// 내부: Reflection으로 필드 읽기
for (Field field : clazz.getDeclaredFields()) {
    field.setAccessible(true);
    Object value = field.get(obj);
    // JSON 변환
}

최적화:
  - 첫 호출 시 메타데이터 캐싱
  - Generated Code 사용 (Jackson)
  - 반복 호출 시 빠름
```

---

## 🔧 성능 개선 방법

### 1. setAccessible(true)

```java
// ❌ 매번 체크
Method method = clazz.getDeclaredMethod("method");
for (int i = 0; i < 1000; i++) {
    method.invoke(obj);  // 접근 권한 체크
}

// ✅ 한 번만 체크
Method method = clazz.getDeclaredMethod("method");
method.setAccessible(true);  // 권한 체크 생략
for (int i = 0; i < 1000; i++) {
    method.invoke(obj);
}

개선: 10~20% 빠름
```

---

### 2. MethodHandle (Java 7+)

```java
// Reflection
Method method = clazz.getMethod("add", int.class, int.class);
method.invoke(obj, 1, 2);  // ~50ns

// MethodHandle
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodHandle mh = lookup.findVirtual(clazz, "add", 
    MethodType.methodType(int.class, int.class, int.class));
int result = (int) mh.invoke(obj, 1, 2);  // ~10ns

장점:
  - JIT 최적화 가능
  - 인라인 가능
  - 5배 빠름
```

---

### 3. LambdaMetafactory (Java 8+)

```java
// Reflection 대신 Lambda 생성
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodHandle mh = lookup.findVirtual(...);

BiFunction<Target, Integer, Integer> lambda = 
    (BiFunction) LambdaMetafactory.metafactory(
        lookup, "apply",
        MethodType.methodType(BiFunction.class),
        MethodType.methodType(Object.class, Object.class, Object.class),
        mh,
        mh.type()
    ).getTarget().invokeExact();

// 사용
int result = lambda.apply(target, 1, 2);  // ~5ns

장점:
  - 직접 호출과 유사한 성능
  - JIT 완전 최적화
```

---

## 🚫 흔한 오해

### "Reflection은 항상 느리다"

```
❌ 잘못된 이해:
  모든 Reflection이 느림

✅ 실제:
  반복 호출 시 개선
  
  초기 (0~14회): ~500ns
  이후 (15회~): ~50ns
  MethodHandle: ~10ns
  
  한 번만 호출: 느림 (500ns)
  반복 호출: 괜찮음 (50ns)
```

---

## 📌 핵심 정리

```
Reflection 호출
  Method.invoke()
  → MethodAccessor 위임

NativeMethodAccessor
  0~14회 호출
  JNI 경계 넘김
  ~500ns

GeneratedMethodAccessor
  15회 임계값 후 생성
  바이트코드 직접 호출
  ~50ns (10배 빠름)

Inflation 메커니즘
  초기: Native (느림, 유연)
  이후: Generated (빠름)

성능 개선
  setAccessible(true): 20% 개선
  MethodHandle: 5배 빠름
  LambdaMetafactory: 10배 빠름

실무
  Framework: 초기화 시 1회
  JSON: 메타데이터 캐싱
  반복 호출 시 괜찮음
```

---

## 🤔 생각해볼 문제

**Q1.** Reflection의 15회 임계값은 왜 15인가? 너무 작거나 크면 어떤 문제가 있는가?

**Q2.** setAccessible(true)가 성능을 개선하는 이유를 설명하라.

**Q3.** MethodHandle이 Reflection보다 빠른 이유를 JIT 컴파일 관점에서 설명하라.

> 💡 **해설**
>
> **Q1.** 15회 임계값 이유: ① 트레이드오프 — 너무 작으면 (예: 1) 코드 생성 비용이 낭비 (1회만 호출 시), 너무 크면 (예: 100) 오래 느린 Native 사용. ② 15회 = 균형점 — 대부분 메서드는 15회 이상 호출되거나 15회 미만. ③ 코드 생성 비용 ~ 10회 Native 호출 비용 → 15회면 이득. ④ Tuning: -Dsun.reflect.inflationThreshold 조정 가능.
>
> **Q2.** setAccessible(true) 개선: ① 기본 동작 — 매 invoke() 시 접근 권한 체크 (private 접근 가능?). ② 체크 비용 ~ 10~20ns. ③ setAccessible(true) → 권한 체크 생략 → 비용 제거. ④ 총 시간 50ns → 40ns (20% 개선). ⑤ 보안: private 우회 가능 → 신중히 사용.
>
> **Q3.** MethodHandle이 빠른 이유: ① Reflection — invoke()는 Object[] 가변 인자 → Boxing, 배열 생성 → JIT 최적화 어려움. ② MethodHandle — 정확한 시그니처 (MethodType) → JIT가 타입 파악 → 인라인 가능. ③ invokeExact() — 타입 체크 없음 → 직접 호출과 유사. ④ JIT 최적화 — 호출 경로 단순 → C2가 완전 인라인 → 5~10배 빠름.

---

## 📚 참고 자료

- [Method Handles](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/invoke/MethodHandles.html)
- [Reflection Performance](https://www.baeldung.com/java-reflection-performance)

---

<div align="center">

**[⬅️ 이전: Unsafe API](./04-unsafe-api.md)** | **[다음: Instrumentation & Java Agent ➡️](./06-instrumentation-and-agent.md)**

</div>

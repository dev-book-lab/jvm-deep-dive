# Instrumentation & Java Agent - 계측과 Java Agent

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `-javaagent`는 어떻게 동작하는가?
- `ClassFileTransformer`로 클래스를 어떻게 변환하는가?
- APM (Application Performance Monitoring) 도구는 어떻게 구현되는가?
- Java Agent의 활용 사례는?

---

## 🔍 왜 이게 존재하는가

### 문제: 코드 수정 없이 동작 변경

```
요구사항:
  - 모든 메서드 호출 시간 측정
  - 코드 수정 불가
  - 재컴파일 불가

해결:
  Java Agent
  → 클래스 로딩 시 바이트코드 변환
```

Java Agent는 **런타임 바이트코드 조작 도구**다.

---

## 📐 Java Agent 구조

### 1. Agent 구현

```java
// MyAgent.java
import java.lang.instrument.Instrumentation;

public class MyAgent {
    // JVM이 호출하는 진입점
    public static void premain(String agentArgs, Instrumentation inst) {
        System.out.println("Agent starting...");
        
        // ClassFileTransformer 등록
        inst.addTransformer(new MyTransformer());
    }
}
```

---

### 2. ClassFileTransformer

```java
import java.lang.instrument.ClassFileTransformer;
import java.security.ProtectionDomain;

public class MyTransformer implements ClassFileTransformer {
    @Override
    public byte[] transform(
            ClassLoader loader,
            String className,
            Class<?> classBeingRedefined,
            ProtectionDomain protectionDomain,
            byte[] classfileBuffer) {
        
        // 특정 클래스만 변환
        if (className.startsWith("com/myapp")) {
            System.out.println("Transforming: " + className);
            return transformClass(classfileBuffer);
        }
        
        return null;  // 변환 안 함
    }
    
    private byte[] transformClass(byte[] original) {
        // ASM, ByteBuddy, Javassist 등 사용
        // 바이트코드 조작
        return modified;
    }
}
```

---

### 3. MANIFEST.MF

```
Manifest-Version: 1.0
Premain-Class: com.myapp.MyAgent
Can-Retransform-Classes: true
Can-Redefine-Classes: true
```

---

### 4. 실행

```bash
# 빌드
javac MyAgent.java
jar cvfm myagent.jar MANIFEST.MF MyAgent.class

# 실행
java -javaagent:myagent.jar MyApp
```

---

## 💻 실습: 메서드 호출 시간 측정

### ASM으로 바이트코드 변환

```java
import org.objectweb.asm.*;

public class TimingTransformer extends ClassVisitor {
    public TimingTransformer(ClassVisitor cv) {
        super(Opcodes.ASM9, cv);
    }
    
    @Override
    public MethodVisitor visitMethod(int access, String name, 
                                     String descriptor, String signature, 
                                     String[] exceptions) {
        MethodVisitor mv = super.visitMethod(access, name, descriptor, 
                                            signature, exceptions);
        return new TimingMethodVisitor(mv, name);
    }
}

class TimingMethodVisitor extends MethodVisitor {
    private String methodName;
    
    @Override
    public void visitCode() {
        // 메서드 시작: long start = System.nanoTime();
        mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", 
                          "nanoTime", "()J", false);
        mv.visitVarInsn(LSTORE, 1);  // start 변수
        super.visitCode();
    }
    
    @Override
    public void visitInsn(int opcode) {
        if (opcode >= IRETURN && opcode <= RETURN) {
            // 메서드 종료 전: 
            // long time = System.nanoTime() - start;
            // System.out.println(methodName + ": " + time + "ns");
            
            mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", 
                              "nanoTime", "()J", false);
            mv.visitVarInsn(LLOAD, 1);
            mv.visitInsn(LSUB);
            // 출력 코드...
        }
        super.visitInsn(opcode);
    }
}
```

---

### ByteBuddy로 간단히

```java
import net.bytebuddy.agent.builder.AgentBuilder;
import net.bytebuddy.asm.Advice;

public class MyAgent {
    public static void premain(String agentArgs, Instrumentation inst) {
        new AgentBuilder.Default()
            .type(nameStartsWith("com.myapp"))
            .transform((builder, type, classLoader, module) ->
                builder.visit(Advice.to(TimingAdvice.class)
                             .on(any())))
            .installOn(inst);
    }
}

class TimingAdvice {
    @Advice.OnMethodEnter
    static long enter(@Advice.Origin String method) {
        return System.nanoTime();
    }
    
    @Advice.OnMethodExit
    static void exit(@Advice.Origin String method, 
                     @Advice.Enter long start) {
        long time = System.nanoTime() - start;
        System.out.println(method + ": " + time + "ns");
    }
}
```

---

## ⚡ 실무 활용 사례

### APM (Application Performance Monitoring)

```
New Relic, DataDog, Pinpoint:

1. Java Agent로 시작
   -javaagent:newrelic.jar

2. 모든 메서드 계측
   - 호출 시간 측정
   - 스택 추적
   - 예외 캡처

3. 데이터 수집
   - 메트릭 수집
   - 트랜잭션 추적
   - 서버로 전송

4. 대시보드 표시
   - 성능 분석
   - 병목 지점
   - 에러율
```

---

### 분산 추적 (Distributed Tracing)

```java
// Zipkin, Jaeger 등
public class TracingAgent {
    public static void premain(String args, Instrumentation inst) {
        inst.addTransformer((loader, className, classBeingRedefined, 
                            protectionDomain, classfileBuffer) -> {
            // HTTP 클라이언트 메서드에 추적 코드 삽입
            if (className.equals("org/apache/http/client/HttpClient")) {
                // Trace ID 생성
                // HTTP 헤더에 추가
                // Span 생성
            }
            return classfileBuffer;
        });
    }
}
```

---

### 보안 모니터링

```java
// SQL Injection 탐지
public class SecurityAgent {
    public static void premain(String args, Instrumentation inst) {
        inst.addTransformer((loader, className, ...) -> {
            if (className.contains("Statement")) {
                // executeQuery() 메서드에 검사 코드 삽입
                // SQL 쿼리 분석
                // 의심스러운 패턴 탐지
            }
            return null;
        });
    }
}
```

---

## 🔧 고급 기능

### Retransform (런타임 재변환)

```java
// 이미 로드된 클래스 재변환
public static void agentmain(String args, Instrumentation inst) {
    inst.addTransformer(new MyTransformer(), true);
    
    // 특정 클래스 재변환
    for (Class<?> clazz : inst.getAllLoadedClasses()) {
        if (clazz.getName().startsWith("com.myapp")) {
            try {
                inst.retransformClasses(clazz);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}

// Attach API로 런타임 연결
VirtualMachine vm = VirtualMachine.attach(pid);
vm.loadAgent("/path/to/agent.jar");
vm.detach();
```

---

## 🚫 주의사항

### 성능 오버헤드

```
측정:
  계측 없음: 100ms
  계측 있음: 110ms
  → 10% 오버헤드

최소화:
  - 필요한 클래스만 변환
  - 가벼운 계측 코드
  - 샘플링 (모든 호출이 아닌 일부만)
```

---

## 📌 핵심 정리

```
Java Agent
  -javaagent:agent.jar
  premain() 진입점
  클래스 로딩 시 바이트코드 변환

ClassFileTransformer
  transform() 메서드
  byte[] → byte[] 변환
  ASM, ByteBuddy, Javassist

활용 사례
  APM: New Relic, DataDog
  추적: Zipkin, Jaeger
  보안: SQL Injection 탐지
  디버깅: 메서드 호출 로깅

고급 기능
  Retransform: 런타임 재변환
  Attach API: 동적 연결

성능
  5~15% 오버헤드
  프로덕션 사용 가능
  필요한 것만 계측
```

---

## 🤔 생각해볼 문제

**Q1.** Java Agent가 모든 메서드 호출 시간을 측정하면 성능에 어떤 영향을 미치는가? 최소화 방법은?

**Q2.** premain()과 agentmain()의 차이를 설명하라.

**Q3.** APM 도구가 분산 시스템에서 트랜잭션을 추적하는 원리를 설명하라.

> 💡 **해설**
>
> **Q1.** 성능 영향: ① 모든 메서드 계측 → 매 호출마다 System.nanoTime() 2회 + 출력. ② 오버헤드: 메서드당 ~100ns 추가. ③ 대량 호출 시 (초당 100만 회) → 100ms 추가 → 10% 성능 저하. 최소화: ① 샘플링 — 1/100 호출만 측정. ② 필터링 — 핵심 클래스만 (com.myapp.service.*). ③ 비동기 수집 — 별도 스레드에서 집계. ④ 조건부 — 느린 메서드만 (> 10ms).
>
> **Q2.** premain() vs agentmain(): ① premain() — JVM 시작 시 호출 (-javaagent), 모든 클래스 로딩 전. 사용: APM, 전역 계측. ② agentmain() — 런타임 중 Attach API로 연결 시 호출. 이미 로드된 클래스에 retransform 필요. 사용: 동적 디버깅, 프로파일링. ③ MANIFEST: Premain-Class vs Agent-Class.
>
> **Q3.** 분산 추적 원리: ① Agent가 HTTP 요청 메서드 계측 (HttpClient.execute()). ② Trace ID 생성 (UUID) + Span ID. ③ HTTP 헤더에 추가 (X-Trace-Id, X-Span-Id). ④ 서버 측 Agent가 헤더 읽기 → 부모 Span과 연결. ⑤ 메서드 진입/종료 시 Span 생성 → 타임스탬프 기록. ⑥ 백엔드로 전송 → 전체 트랜잭션 트리 구성.

---

## 📚 참고 자료

- [Java Instrumentation](https://docs.oracle.com/en/java/javase/17/docs/api/java.instrument/java/lang/instrument/package-summary.html)
- [ByteBuddy](https://bytebuddy.net/)

---

<div align="center">

**[⬅️ 이전: Reflection & Performance](./05-reflection-and-performance.md)** | **[다음: JNI Internals ➡️](./07-jni-internals.md)**

</div>

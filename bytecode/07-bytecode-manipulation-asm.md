# Bytecode Manipulation (ASM) - 바이트코드 조작

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- ASM 라이브러리는 무엇이며, 왜 바이트코드를 직접 조작하는가?
- ClassVisitor, MethodVisitor, FieldVisitor의 역할과 동작 원리는?
- 바이트코드 조작으로 AOP, 로깅, 프로파일링을 어떻게 구현하는가?
- ASM Core API와 Tree API의 차이는 무엇인가?
- 실무에서 바이트코드 조작이 어떻게 활용되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 소스 코드 수정 없이 기능을 추가하고 싶다

```java
// 원본 코드
public class UserService {
    public void createUser(String name) {
        // business logic
    }
}

// 요구사항:
// 1. 모든 메서드 호출 시 로깅
// 2. 메서드 실행 시간 측정
// 3. 트랜잭션 처리
// 4. 보안 체크

// 하지만 소스 코드 수정은 불가능 (라이브러리, 레거시 등)
```

```
해결 방법:

1. Reflection Proxy (런타임):
   Proxy.newProxyInstance(...)
   → 인터페이스만 가능
   → 성능 오버헤드

2. Code Generation (컴파일 타임):
   어노테이션 프로세서
   → 새 클래스만 생성 가능
   → 기존 클래스 수정 불가

3. Bytecode Manipulation (로드 타임/런타임):
   ASM, Javassist, ByteBuddy
   → 모든 클래스 수정 가능
   → 인터페이스 불필요
   → 빠름
```

ASM은 **바이트코드를 직접 조작**하는 라이브러리다.

---

## 📐 내부 구조

### 1. ASM 개요

```
ASM (ObjectWeb ASM):
  - Java 바이트코드 조작 라이브러리
  - 매우 빠름 (바이트코드 직접 생성)
  - 작음 (~50KB)
  - Visitor 패턴 기반

사용처:
  - Spring AOP (CGLIB)
  - Hibernate (Lazy Loading)
  - JaCoCo (코드 커버리지)
  - Mockito (모킹)
  - Kotlin 컴파일러

의존성:
  <dependency>
      <groupId>org.ow2.asm</groupId>
      <artifactId>asm</artifactId>
      <version>9.5</version>
  </dependency>
```

---

### 2. Visitor 패턴

```
ASM의 핵심: Visitor 패턴

ClassReader (입력)
  → ClassVisitor (변환)
    → ClassWriter (출력)

ClassVisitor:
  visit()           // 클래스 헤더
  visitField()      // 필드
  visitMethod()     // 메서드
    → MethodVisitor
      visitCode()   // 메서드 바이트코드 시작
      visitInsn()   // 명령어
      visitEnd()    // 메서드 끝
  visitEnd()        // 클래스 끝

흐름:
  ClassReader.accept(ClassVisitor)
  → Visitor가 각 요소를 순회하며 visitXXX() 호출
  → Visitor가 바이트코드 변환 (추가/삭제/수정)
  → ClassWriter가 새 바이트코드 생성
```

---

### 3. 기본 사용 예시 — 클래스 읽기

```java
import org.objectweb.asm.*;

public class ClassPrinter extends ClassVisitor {
    public ClassPrinter() {
        super(Opcodes.ASM9);
    }
    
    @Override
    public void visit(int version, int access, String name, 
                     String signature, String superName, String[] interfaces) {
        System.out.println("Class: " + name);
        System.out.println("Super: " + superName);
    }
    
    @Override
    public MethodVisitor visitMethod(int access, String name, 
                                     String descriptor, String signature, String[] exceptions) {
        System.out.println("  Method: " + name + descriptor);
        return null;
    }
}

// 사용:
ClassReader reader = new ClassReader("com.example.MyClass");
ClassVisitor visitor = new ClassPrinter();
reader.accept(visitor, 0);
```

---

### 4. 바이트코드 변환 — 메서드 시작/끝에 로깅 추가

```java
public class LoggingClassAdapter extends ClassVisitor {
    private String className;
    
    public LoggingClassAdapter(ClassVisitor cv) {
        super(Opcodes.ASM9, cv);
    }
    
    @Override
    public void visit(int version, int access, String name, 
                     String signature, String superName, String[] interfaces) {
        this.className = name;
        super.visit(version, access, name, signature, superName, interfaces);
    }
    
    @Override
    public MethodVisitor visitMethod(int access, String name, 
                                     String descriptor, String signature, String[] exceptions) {
        MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
        
        // 로깅 추가 (생성자와 static 블록 제외)
        if (!name.equals("<init>") && !name.equals("<clinit>")) {
            return new LoggingMethodAdapter(mv, className, name);
        }
        return mv;
    }
}

class LoggingMethodAdapter extends MethodVisitor {
    private String className;
    private String methodName;
    
    public LoggingMethodAdapter(MethodVisitor mv, String className, String methodName) {
        super(Opcodes.ASM9, mv);
        this.className = className;
        this.methodName = methodName;
    }
    
    @Override
    public void visitCode() {
        super.visitCode();
        
        // 메서드 시작 시 로깅
        // System.out.println("Enter: " + className + "." + methodName);
        mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        mv.visitLdcInsn("Enter: " + className + "." + methodName);
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", 
                          "(Ljava/lang/String;)V", false);
    }
    
    @Override
    public void visitInsn(int opcode) {
        // return 명령어 전에 로깅
        if ((opcode >= Opcodes.IRETURN && opcode <= Opcodes.RETURN) || opcode == Opcodes.ATHROW) {
            mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
            mv.visitLdcInsn("Exit: " + className + "." + methodName);
            mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", 
                              "(Ljava/lang/String;)V", false);
        }
        super.visitInsn(opcode);
    }
}

// 사용:
ClassReader reader = new ClassReader("com.example.MyClass");
ClassWriter writer = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
ClassVisitor adapter = new LoggingClassAdapter(writer);
reader.accept(adapter, 0);
byte[] modifiedBytecode = writer.toByteArray();
```

---

### 5. 메서드 실행 시간 측정

```java
public class TimingMethodAdapter extends MethodVisitor {
    private String methodName;
    
    public TimingMethodAdapter(MethodVisitor mv, String methodName) {
        super(Opcodes.ASM9, mv);
        this.methodName = methodName;
    }
    
    @Override
    public void visitCode() {
        super.visitCode();
        
        // long startTime = System.nanoTime();
        mv.visitMethodInsn(Opcodes.INVOKESTATIC, "java/lang/System", "nanoTime", "()J", false);
        mv.visitVarInsn(Opcodes.LSTORE, 1);  // LVA[1] = startTime (long, 2 slots)
    }
    
    @Override
    public void visitInsn(int opcode) {
        if ((opcode >= Opcodes.IRETURN && opcode <= Opcodes.RETURN)) {
            // long elapsed = System.nanoTime() - startTime;
            mv.visitMethodInsn(Opcodes.INVOKESTATIC, "java/lang/System", "nanoTime", "()J", false);
            mv.visitVarInsn(Opcodes.LLOAD, 1);
            mv.visitInsn(Opcodes.LSUB);
            
            // System.out.println(methodName + " took " + elapsed + " ns");
            mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
            mv.visitInsn(Opcodes.SWAP);  // stack: [out, elapsed] → [elapsed, out]
            mv.visitLdcInsn(methodName + " took ");
            mv.visitInsn(Opcodes.SWAP);  // stack: [string, elapsed] → [elapsed, string]
            // ... (복잡한 문자열 연결 생략)
        }
        super.visitInsn(opcode);
    }
}
```

---

### 6. Java Agent로 런타임 변환

```java
import java.lang.instrument.*;

public class LoggingAgent {
    public static void premain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new ClassFileTransformer() {
            @Override
            public byte[] transform(ClassLoader loader, String className, 
                                   Class<?> classBeingRedefined, 
                                   ProtectionDomain protectionDomain, 
                                   byte[] classfileBuffer) {
                if (className.startsWith("com/example")) {
                    try {
                        ClassReader reader = new ClassReader(classfileBuffer);
                        ClassWriter writer = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
                        ClassVisitor adapter = new LoggingClassAdapter(writer);
                        reader.accept(adapter, 0);
                        return writer.toByteArray();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
                return null;  // 변환 안 함
            }
        });
    }
}
```

```
MANIFEST.MF:
  Premain-Class: com.example.LoggingAgent

실행:
  java -javaagent:agent.jar -jar myapp.jar
  
  → JVM이 클래스 로딩 시 LoggingAgent 호출
  → com.example 패키지 클래스만 변환
  → 소스 코드 수정 없이 로깅 추가
```

---

## 💻 실험으로 확인하기

### 실험 1: 간단한 클래스 생성

```java
import org.objectweb.asm.*;

public class ClassGenerator {
    public static byte[] generateHelloWorld() {
        ClassWriter cw = new ClassWriter(0);
        
        // public class HelloWorld
        cw.visit(Opcodes.V11, Opcodes.ACC_PUBLIC, "HelloWorld", 
                null, "java/lang/Object", null);
        
        // public static void main(String[] args)
        MethodVisitor mv = cw.visitMethod(
            Opcodes.ACC_PUBLIC | Opcodes.ACC_STATIC,
            "main",
            "([Ljava/lang/String;)V",
            null, null);
        
        mv.visitCode();
        
        // System.out.println("Hello, World!");
        mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        mv.visitLdcInsn("Hello, World!");
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", 
                          "(Ljava/lang/String;)V", false);
        
        // return
        mv.visitInsn(Opcodes.RETURN);
        
        mv.visitMaxs(2, 1);  // max_stack=2, max_locals=1
        mv.visitEnd();
        
        cw.visitEnd();
        return cw.toByteArray();
    }
    
    public static void main(String[] args) throws Exception {
        byte[] bytecode = generateHelloWorld();
        
        // 클래스 로딩
        ClassLoader loader = new ClassLoader() {
            @Override
            protected Class<?> findClass(String name) {
                return defineClass(name, bytecode, 0, bytecode.length);
            }
        };
        
        Class<?> clazz = loader.loadClass("HelloWorld");
        clazz.getMethod("main", String[].class).invoke(null, (Object) new String[0]);
        // 출력: Hello, World!
    }
}
```

---

### 실험 2: 기존 메서드 수정

```java
public class MethodModifier {
    public static void main(String[] args) throws Exception {
        // 원본 클래스
        class Original {
            public int add(int a, int b) {
                return a + b;
            }
        }
        
        // ASM으로 수정
        ClassReader reader = new ClassReader(Original.class.getName());
        ClassWriter writer = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
        
        ClassVisitor cv = new ClassVisitor(Opcodes.ASM9, writer) {
            @Override
            public MethodVisitor visitMethod(int access, String name, 
                                            String descriptor, String signature, String[] exceptions) {
                MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
                
                if (name.equals("add")) {
                    return new MethodVisitor(Opcodes.ASM9, mv) {
                        @Override
                        public void visitCode() {
                            super.visitCode();
                            // 메서드 시작 시 System.out.println("add called");
                            mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", 
                                            "Ljava/io/PrintStream;");
                            mv.visitLdcInsn("add called");
                            mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", 
                                             "println", "(Ljava/lang/String;)V", false);
                        }
                    };
                }
                return mv;
            }
        };
        
        reader.accept(cv, 0);
        byte[] modified = writer.toByteArray();
        
        // 수정된 클래스 로딩
        ClassLoader loader = new ClassLoader() {
            protected Class<?> findClass(String name) {
                return defineClass(name, modified, 0, modified.length);
            }
        };
        
        Class<?> modifiedClass = loader.loadClass(Original.class.getName());
        Object instance = modifiedClass.getDeclaredConstructor().newInstance();
        modifiedClass.getMethod("add", int.class, int.class).invoke(instance, 1, 2);
        // 출력: add called
        // 반환: 3
    }
}
```

---

## ⚡ 실무 임팩트

### Spring AOP와 CGLIB

```
Spring AOP:
  @Transactional, @Cacheable 등의 어노테이션
  → 런타임에 프록시 생성
  
인터페이스 있음:
  JDK Dynamic Proxy 사용
  
인터페이스 없음:
  CGLIB (ASM 기반) 사용
  → 클래스 상속 + 메서드 오버라이드
  
CGLIB 동작:
  1. ASM으로 원본 클래스 상속한 서브클래스 생성
  2. 메서드를 오버라이드해 인터셉터 호출
  3. super.method() 로 원본 호출
  
예:
  @Service
  public class UserService {
      @Transactional
      public void createUser(String name) { ... }
  }
  
  → UserService$$EnhancerByCGLIB$$12345 클래스 생성
  → createUser() 오버라이드:
      transactionManager.begin();
      super.createUser(name);
      transactionManager.commit();
```

### JaCoCo 코드 커버리지

```
JaCoCo (Java Code Coverage):
  테스트 실행 시 어느 라인이 실행됐는지 추적
  
원리:
  1. Java Agent로 클래스 로딩 인터셉트
  2. ASM으로 각 라인에 카운터 추가
  
  원본:
    if (x > 0) {
        return 1;
    }
    return 0;
  
  변환 후:
    counter[0]++;  // line 1
    if (x > 0) {
        counter[1]++;  // line 2
        return 1;
    }
    counter[2]++;  // line 4
    return 0;
  
  3. 테스트 종료 시 counter 배열 분석
  4. 리포트 생성 (어느 라인이 실행 안 됐는지)
```

### Mockito 모킹

```
Mockito:
  mock(UserService.class)
  → UserService의 모든 메서드가 기본값 반환
  
내부 동작 (ByteBuddy 사용, ASM 기반):
  1. UserService 상속한 프록시 클래스 생성
  2. 모든 메서드 오버라이드:
     - when(...).thenReturn(...) 설정 확인
     - 설정 있으면 반환, 없으면 기본값
  
  예:
    UserService mock = mock(UserService.class);
    when(mock.getUser(1)).thenReturn(new User("Alice"));
    
    → getUser(1) 호출 시:
      if (stubbing.hasStubFor(1)) {
          return stubbing.get(1);  // new User("Alice")
      }
      return null;  // 기본값
```

---

## 🚫 흔한 오해

### "ASM은 너무 어려워서 실무에서 못 쓴다"

```
❌ 잘못된 이해:
  ASM은 저수준이라 사용하기 어렵다.

✅ 실제:
  직접 쓸 일은 드물지만, 내부에서 많이 사용됨
  
직접 사용:
  - 커스텀 AOP
  - 프로파일링 도구
  - 코드 생성기
  
간접 사용 (ASM 기반 라이브러리):
  - Spring CGLIB
  - Hibernate ByteBuddy
  - Mockito
  - JaCoCo
  
권장:
  일반 개발: ByteBuddy (ASM wrapper, 쉬움)
  고급 최적화: ASM 직접 사용
```

### "바이트코드 조작은 항상 느리다"

```
❌ 잘못된 이해:
  런타임에 클래스를 변환하니 느릴 것이다.

✅ 실제:
  변환 자체는 빠름 (밀리초 단위)
  변환 후 실행은 거의 동일
  
벤치마크:
  원본 메서드: 1.0 ns
  ASM 변환 후: 1.1 ns (10% 증가)
  
  변환 비용:
  클래스당 1~10 ms (클래스 크기에 따라)
  
결론:
  변환 비용 < 1초 (수백 클래스)
  실행 성능은 거의 동일 (JIT 후)
```

### "ASM으로 모든 것을 할 수 있다"

```
❌ 잘못된 이해:
  ASM이면 어떤 바이트코드든 생성 가능하다.

✅ 실제:
  JVM 제약 사항은 여전히 존재
  
불가능한 것:
  - final 클래스 상속 (String 등)
  - final 메서드 오버라이드
  - private 필드/메서드 접근 (같은 패키지 아니면)
  - Bytecode Verifier 우회 (잘못된 스택 조작)
  
가능한 것:
  - 새 클래스 생성
  - 기존 클래스 수정 (상속)
  - 메서드 추가/수정
  - 필드 추가
```

---

## 📌 핵심 정리

```
ASM 라이브러리
  Java 바이트코드 조작
  빠름, 작음 (~50KB)
  Visitor 패턴 기반

주요 클래스
  ClassReader: 바이트코드 읽기
  ClassVisitor: 바이트코드 방문/변환
  ClassWriter: 바이트코드 쓰기
  MethodVisitor: 메서드 바이트코드 조작

사용 흐름
  ClassReader → ClassVisitor → ClassWriter
  visitXXX() 메서드로 요소 순회
  바이트코드 추가/삭제/수정

실무 활용
  Spring AOP (CGLIB)
  Hibernate (Lazy Loading)
  JaCoCo (코드 커버리지)
  Mockito (모킹)

Java Agent
  premain()으로 클래스 로딩 인터셉트
  ClassFileTransformer로 변환
  -javaagent 플래그로 실행

장점
  소스 코드 수정 불필요
  모든 클래스 변환 가능
  빠른 실행 속도

주의사항
  Bytecode Verifier 통과해야 함
  final 클래스/메서드는 제한
  스택 조작 실수 시 VerifyError
```

---

## 🤔 생각해볼 문제

**Q1.** ASM으로 모든 public 메서드에 실행 시간 측정 코드를 추가하려고 한다. visitCode()와 visitInsn()에서 각각 어떤 바이트코드를 삽입해야 하는가? max_stack과 max_locals는 어떻게 변하는가?

**Q2.** Spring의 `@Transactional`이 어떻게 동작하는지 ASM/CGLIB 관점에서 설명하라. 왜 인터페이스가 있을 때와 없을 때 다르게 동작하는가?

**Q3.** 다음 코드에 ASM으로 NullPointerException 방어 코드를 추가하려고 한다. 어떻게 구현할 것인가?

```java
public String process(User user) {
    return user.getName().toUpperCase();
}

// 변환 후:
// if (user == null) throw new IllegalArgumentException("user is null");
// if (user.getName() == null) throw new IllegalArgumentException("name is null");
```

> 💡 **해설**
>
> **Q1.** visitCode()에서: `System.nanoTime()` 호출 후 LVA에 저장 → `mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "nanoTime", "()J", false); mv.visitVarInsn(LSTORE, N);` (N은 첫 번째 사용 가능한 로컬 변수 인덱스). visitInsn()에서 return 명령어 전: 현재 시간 - 저장된 시간 계산 → 출력 → `mv.visitMethodInsn(INVOKESTATIC, ..., "nanoTime", ...); mv.visitVarInsn(LLOAD, N); mv.visitInsn(LSUB); ...`. max_stack: +3 (System.out, elapsed, 문자열 연결). max_locals: +2 (long은 2슬롯).
>
> **Q2.** 인터페이스 있음: JDK Dynamic Proxy 사용 → `Proxy.newProxyInstance()` → InvocationHandler로 트랜잭션 래핑 → 메서드 호출 시 `invoke()` 가로채기. 인터페이스 없음: CGLIB (ASM 기반) 사용 → 원본 클래스 상속한 서브클래스 생성 (예: `UserService$$EnhancerByCGLIB`) → 메서드 오버라이드해 트랜잭션 로직 추가 → `super.method()` 호출. 차이 이유: JDK Proxy는 인터페이스 기반으로만 동작 (Reflection Proxy 제약), CGLIB는 상속 기반이라 구체 클래스도 가능. 단, final 클래스는 CGLIB도 불가.
>
> **Q3.** MethodVisitor에서 각 `ALOAD` 명령어 후 null 체크 삽입. ① user null 체크: `visitVarInsn(ALOAD, 1)` 후 → `visitInsn(DUP); visitJumpInsn(IFNONNULL, label); throwException("user is null"); visitLabel(label);`. ② getName() 결과 null 체크: `invokevirtual getName` 후 → `visitInsn(DUP); visitJumpInsn(IFNONNULL, label2); throwException("name is null"); visitLabel(label2);`. throwException 헬퍼: `new IllegalArgumentException` 생성 → message 전달 → athrow. 복잡도: 스택 관리 주의 (DUP로 값 복제, 체크 후 원본 유지). max_stack 증가 필요.

---

## 📚 참고 자료

- [ASM Official Guide](https://asm.ow2.io/asm4-guide.pdf)
- [ASM GitHub Repository](https://github.com/llbit/ow2-asm)
- [ByteBuddy (ASM Wrapper)](https://bytebuddy.net/)

---

<div align="center">

**[⬅️ 이전: Lambda & InvokeDynamic](./06-lambda-and-invokedynamic.md)** | **[홈으로 🏠](../README.md)**

</div>

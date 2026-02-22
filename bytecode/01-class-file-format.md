# Class File Format - 클래스 파일 형식

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `.class` 파일은 정확히 어떤 바이너리 구조를 가지는가?
- Magic Number `CAFEBABE`는 왜 필요하며, 어떻게 검증하는가?
- Constant Pool은 왜 클래스 파일의 가장 앞에 위치하며, 어떻게 인덱싱되는가?
- `javap -v` 출력을 보고 실제 바이너리와 연결할 수 있는가?
- 클래스 파일의 버전 번호는 어떻게 결정되며, JVM 호환성과 어떤 관계인가?

---

## 🔍 왜 이게 존재하는가

### 문제: Java 소스 코드를 어떻게 플랫폼 독립적으로 실행하는가

```
전통적 컴파일 언어 (C/C++):
  source.c → gcc → machine code (x86, ARM, ...)
  → 플랫폼 종속적
  → Windows .exe는 Linux에서 실행 불가

Java의 접근:
  source.java → javac → bytecode (.class)
  → JVM이 bytecode 해석
  → 어떤 플랫폼이든 JVM만 있으면 실행
  
  "Write Once, Run Anywhere"
```

`.class` 파일은 JVM이 읽을 수 있는 **표준화된 바이너리 형식**이다.

---

## 📐 내부 구조

### 1. 클래스 파일 전체 구조

```
ClassFile {
    u4             magic;              // 0xCAFEBABE
    u2             minor_version;      // 부 버전
    u2             major_version;      // 주 버전
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;       // public, final, abstract 등
    u2             this_class;         // Constant Pool 인덱스
    u2             super_class;        // 부모 클래스
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}

u1 = unsigned 1 byte
u2 = unsigned 2 bytes
u4 = unsigned 4 bytes
```

---

### 2. Magic Number와 버전

```
첫 8바이트:

Offset   Bytes         의미
0x00     CA FE BA BE   Magic Number (클래스 파일 식별)
0x04     00 00         Minor Version
0x06     00 34         Major Version (0x34 = 52 = Java 8)

버전 매핑:
  45 (0x2D) = Java 1.1
  46 (0x2E) = Java 1.2
  47 (0x2F) = Java 1.3
  48 (0x30) = Java 1.4
  49 (0x31) = Java 5
  50 (0x32) = Java 6
  51 (0x33) = Java 7
  52 (0x34) = Java 8
  53 (0x35) = Java 9
  ...
  61 (0x3D) = Java 17
  65 (0x41) = Java 21

호환성:
  JVM은 자신의 버전 이하 클래스만 실행 가능
  Java 17 JVM: 버전 61 이하 클래스 실행 OK
  Java 8 JVM:  버전 53 이상 클래스 실행 불가
  → java.lang.UnsupportedClassVersionError
```

---

### 3. Constant Pool — 모든 심볼릭 참조의 저장소

```
Constant Pool 구조:

constant_pool_count: 3바이트 정수 (예: 30)
→ 실제 엔트리는 constant_pool[1] ~ constant_pool[29]
→ 인덱스 0은 null 의미로 예약

Constant Pool Entry 타입 (일부):

CONSTANT_Utf8 (tag=1)
  - 문자열 데이터 (클래스명, 필드명, 메서드명 등)
  
CONSTANT_Integer (tag=3)
  - int 리터럴

CONSTANT_Float (tag=4)
  - float 리터럴

CONSTANT_Long (tag=5)
  - long 리터럴 (2개 슬롯 차지)

CONSTANT_Double (tag=6)
  - double 리터럴 (2개 슬롯 차지)

CONSTANT_Class (tag=7)
  - 클래스/인터페이스 참조
  - name_index → CONSTANT_Utf8

CONSTANT_String (tag=8)
  - String 리터럴
  - string_index → CONSTANT_Utf8

CONSTANT_Fieldref (tag=9)
  - 필드 참조
  - class_index → CONSTANT_Class
  - name_and_type_index → CONSTANT_NameAndType

CONSTANT_Methodref (tag=10)
  - 메서드 참조
  
CONSTANT_InterfaceMethodref (tag=11)
  - 인터페이스 메서드 참조

CONSTANT_NameAndType (tag=12)
  - 이름과 타입 디스크립터
  - name_index → CONSTANT_Utf8
  - descriptor_index → CONSTANT_Utf8
```

#### Constant Pool 예시

```java
public class Example {
    public void hello() {
        System.out.println("Hello");
    }
}
```

```
Constant Pool:
   #1 = Methodref    #6.#15   // java/lang/Object."<init>":()V
   #2 = Fieldref     #16.#17  // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String       #18      // Hello
   #4 = Methodref    #19.#20  // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class        #21      // Example
   #6 = Class        #22      // java/lang/Object
   ...
  #15 = NameAndType  #23:#24  // "<init>":()V
  #16 = Class        #25      // java/lang/System
  #17 = NameAndType  #26:#27  // out:Ljava/io/PrintStream;
  #18 = Utf8         Hello
  #19 = Class        #28      // java/io/PrintStream
  #20 = NameAndType  #29:#30  // println:(Ljava/lang/String;)V
  #21 = Utf8         Example
  #22 = Utf8         java/lang/Object
  #23 = Utf8         <init>
  #24 = Utf8         ()V
  #25 = Utf8         java/lang/System
  #26 = Utf8         out
  #27 = Utf8         Ljava/io/PrintStream;
  #28 = Utf8         java/io/PrintStream
  #29 = Utf8         println
  #30 = Utf8         (Ljava/lang/String;)V

인덱스 체인:
  #4 (Methodref) → #19 (Class) + #20 (NameAndType)
    → #19 → #28 (Utf8 "java/io/PrintStream")
    → #20 → #29 (Utf8 "println") + #30 (Utf8 "(Ljava/lang/String;)V")
```

---

### 4. Access Flags

```
클래스 Access Flags (비트 마스크):

ACC_PUBLIC     = 0x0001   // public
ACC_FINAL      = 0x0010   // final
ACC_SUPER      = 0x0020   // invokespecial 의미론 (항상 설정)
ACC_INTERFACE  = 0x0200   // interface
ACC_ABSTRACT   = 0x0400   // abstract
ACC_SYNTHETIC  = 0x1000   // 컴파일러 생성
ACC_ANNOTATION = 0x2000   // @interface
ACC_ENUM       = 0x4000   // enum

예:
  public class Foo            → 0x0021 (PUBLIC | SUPER)
  public final class Bar      → 0x0031 (PUBLIC | FINAL | SUPER)
  public interface Baz        → 0x0601 (PUBLIC | INTERFACE | ABSTRACT)
  public enum Status          → 0x4031 (PUBLIC | FINAL | SUPER | ENUM)
```

---

### 5. Fields와 Methods

```
field_info {
    u2             access_flags;    // public, static, final 등
    u2             name_index;      // Constant Pool → 필드명
    u2             descriptor_index; // 타입 디스크립터
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}

method_info {
    u2             access_flags;
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count]; // Code, Exceptions 등
}

디스크립터 (Type Descriptor):
  B = byte
  C = char
  D = double
  F = float
  I = int
  J = long
  S = short
  Z = boolean
  L<classname>; = 참조 타입
  [<type> = 배열

예:
  int x;                 → descriptor: I
  String name;           → descriptor: Ljava/lang/String;
  int[] arr;             → descriptor: [I
  void foo(int, String)  → descriptor: (ILjava/lang/String;)V
  Object bar(int[][])    → descriptor: ([[I)Ljava/lang/Object;
```

---

### 6. Code Attribute — 메서드 바이트코드 저장

```
Code_attribute {
    u2 attribute_name_index;  // "Code"
    u4 attribute_length;
    u2 max_stack;             // Operand Stack 최대 깊이
    u2 max_locals;            // Local Variable Array 크기
    u4 code_length;
    u1 code[code_length];     // 실제 바이트코드
    u2 exception_table_length;
    {   u2 start_pc;
        u2 end_pc;
        u2 handler_pc;
        u2 catch_type;
    } exception_table[exception_table_length];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}

예:
  public int add(int a, int b) {
      return a + b;
  }

Code:
  max_stack=2, max_locals=3 (this, a, b)
  0: iload_1      // LVA[1](a)을 스택에 push
  1: iload_2      // LVA[2](b)을 스택에 push
  2: iadd         // 두 값을 pop, 더한 후 push
  3: ireturn      // 스택 top 값을 반환
```

---

## 💻 실험으로 확인하기

### 실험 1: 바이너리 직접 분석 (hexdump)

```java
public class Simple {
    public static void main(String[] args) {
        System.out.println("Hello");
    }
}
```

```bash
javac Simple.java
hexdump -C Simple.class | head -20

# 출력:
# 00000000  ca fe ba be 00 00 00 34  00 1d 0a 00 06 00 0f 09  |.......4........|
#           ^^^^^^^^^^^ Magic       ^^^^^ Minor  ^^^^^ Major (52 = Java 8)
#                                         ^^^^^ Constant Pool Count (29)
# 00000010  00 10 00 11 08 00 12 0a  00 13 00 14 07 00 15 07  |................|
# ...
```

---

### 실험 2: javap로 디스어셈블

```bash
javap -v Simple.class

# 출력:
# public class Simple
#   minor version: 0
#   major version: 52
#   flags: ACC_PUBLIC, ACC_SUPER
# Constant pool:
#    #1 = Methodref          #6.#15
#    #2 = Fieldref           #16.#17
#    #3 = String             #18
#    #4 = Methodref          #19.#20
#    #5 = Class              #21
#    ...
#   #18 = Utf8               Hello
#   ...
# {
#   public static void main(java.lang.String[]);
#     descriptor: ([Ljava/lang/String;)V
#     flags: ACC_PUBLIC, ACC_STATIC
#     Code:
#       stack=2, locals=1, args_size=1
#          0: getstatic     #2    // Field java/lang/System.out:Ljava/io/PrintStream;
#          3: ldc           #3    // String Hello
#          5: invokevirtual #4    // Method java/io/PrintStream.println:(Ljava/lang/String;)V
#          8: return
# }
```

---

### 실험 3: 클래스 파일 리더 구현

```java
import java.io.*;

public class ClassFileReader {
    public static void main(String[] args) throws Exception {
        DataInputStream in = new DataInputStream(
            new FileInputStream("Simple.class"));
        
        // Magic Number
        int magic = in.readInt();
        System.out.printf("Magic: 0x%X%n", magic);
        if (magic != 0xCAFEBABE) {
            throw new IOException("Invalid class file");
        }
        
        // Version
        int minor = in.readUnsignedShort();
        int major = in.readUnsignedShort();
        System.out.printf("Version: %d.%d (Java %s)%n", 
            major, minor, getJavaVersion(major));
        
        // Constant Pool Count
        int cpCount = in.readUnsignedShort();
        System.out.printf("Constant Pool Count: %d%n", cpCount);
        
        // Constant Pool Entries
        for (int i = 1; i < cpCount; i++) {
            int tag = in.readUnsignedByte();
            System.out.printf("#%d = tag %d%n", i, tag);
            
            switch (tag) {
                case 1: // CONSTANT_Utf8
                    int len = in.readUnsignedShort();
                    byte[] bytes = new byte[len];
                    in.readFully(bytes);
                    System.out.printf("     Utf8: %s%n", new String(bytes));
                    break;
                case 7: // CONSTANT_Class
                    int nameIndex = in.readUnsignedShort();
                    System.out.printf("     Class: #%d%n", nameIndex);
                    break;
                // ... 다른 타입 처리
                default:
                    throw new IOException("Unknown tag: " + tag);
            }
        }
        
        in.close();
    }
    
    static String getJavaVersion(int major) {
        return major >= 45 ? "1." + (major - 44) : "Unknown";
    }
}
```

---

### 실험 4: ASM으로 클래스 파일 읽기

```java
// ASM 라이브러리 사용 (gradle/maven 의존성 필요)
import org.objectweb.asm.*;
import java.io.FileInputStream;

public class ASMReader {
    public static void main(String[] args) throws Exception {
        ClassReader reader = new ClassReader(
            new FileInputStream("Simple.class"));
        
        reader.accept(new ClassVisitor(Opcodes.ASM9) {
            @Override
            public void visit(int version, int access, String name, 
                              String signature, String superName, String[] interfaces) {
                System.out.println("Class: " + name);
                System.out.println("Super: " + superName);
                System.out.println("Version: " + version);
            }
            
            @Override
            public MethodVisitor visitMethod(int access, String name, 
                                             String descriptor, String signature, String[] exceptions) {
                System.out.println("Method: " + name + descriptor);
                return null;
            }
            
            @Override
            public FieldVisitor visitField(int access, String name, 
                                          String descriptor, String signature, Object value) {
                System.out.println("Field: " + name + ":" + descriptor);
                return null;
            }
        }, 0);
    }
}
```

---

## ⚡ 실무 임팩트

### 클래스 파일 버전 호환성 문제 해결

```
문제:
  개발: Java 17
  운영: Java 8
  → java.lang.UnsupportedClassVersionError: ... (class file version 61.0)

해결책 1: --release 플래그
  javac --release 8 MyClass.java
  → Java 8 호환 바이트코드 생성 (major version 52)

해결책 2: target 옵션
  javac -source 8 -target 8 MyClass.java
  (Deprecated in Java 9+)

Gradle:
  compileJava {
      options.release = 8
  }

Maven:
  <maven.compiler.release>8</maven.compiler.release>
```

### 난독화와 클래스 파일 검증

```
난독화 도구 (ProGuard, R8):
  - 클래스명, 메서드명, 필드명을 짧게 변경
  - Constant Pool의 Utf8 엔트리 수정
  - 사용하지 않는 코드 제거
  
검증:
  JVM은 클래스 로딩 시 Bytecode Verifier 실행
  - Constant Pool 무결성 확인
  - 타입 안전성 검증
  - 스택/로컬 변수 접근 범위 확인
  
  잘못된 난독화 → VerifyError
```

### JAR 서명과 클래스 파일 무결성

```
JAR 서명:
  jarsigner -keystore mystore.jks myapp.jar mykey

서명 검증 과정:
  1. META-INF/MANIFEST.MF 읽기
     각 .class 파일의 SHA-256 해시 저장
  
  2. 클래스 로딩 시 해시 재계산
     실제 파일 내용과 MANIFEST 비교
  
  3. 불일치 시 SecurityException
     → 변조 감지

클래스 파일 구조 지식이 필요한 이유:
  수동 검증, 커스텀 ClassLoader 구현 시 필수
```

---

## 🚫 흔한 오해

### "클래스 파일 버전은 JDK 버전과 항상 동일하다"

```
❌ 잘못된 이해:
  Java 8로 컴파일하면 클래스 파일 버전은 항상 52다.

✅ 실제:
  javac --release 옵션으로 하위 버전 생성 가능
  
  javac --release 7 MyClass.java
  → major version 51 (Java 7)
  
  Java 8 JDK로 Java 7 호환 클래스 생성
  → 하위 호환성 보장
  
  단, 새 문법(람다 등)은 사용 불가
```

### "Constant Pool 인덱스는 0부터 시작한다"

```
❌ 잘못된 이해:
  constant_pool[0]이 첫 번째 엔트리다.

✅ 실제:
  constant_pool[1]이 첫 번째 엔트리
  
  이유:
  - 인덱스 0은 "no reference" 의미로 예약
  - super_class가 없는 java.lang.Object는 super_class=0
  
  constant_pool_count = 30이면
  → 실제 엔트리: constant_pool[1] ~ constant_pool[29]
  → 총 29개 엔트리
```

### "클래스 파일은 텍스트 에디터로 읽을 수 있다"

```
❌ 잘못된 이해:
  .class 파일을 텍스트 에디터로 열면 코드를 볼 수 있다.

✅ 실제:
  .class 파일은 바이너리 형식
  → 텍스트 에디터로 열면 깨진 문자 (binary garbage)
  
  분석 도구:
  - javap (JDK 기본 제공)
  - hexdump, xxd (바이너리 뷰어)
  - JD-GUI, CFR (디컴파일러)
  - ASM, Javassist (라이브러리)
```

---

## 📌 핵심 정리

```
클래스 파일 구조
  Magic Number (CA FE BA BE)
  Version (Minor + Major)
  Constant Pool (심볼릭 참조 저장소)
  Access Flags (public, final 등)
  This/Super Class (클래스 정보)
  Interfaces (구현 인터페이스)
  Fields (필드 목록)
  Methods (메서드 목록 + Code 속성)
  Attributes (추가 정보)

Constant Pool
  인덱스 1부터 시작 (0은 예약)
  Utf8, Class, String, Methodref 등
  모든 심볼릭 참조의 중앙 저장소

버전 호환성
  Major Version: JDK 버전 결정 (52 = Java 8)
  JVM은 자신의 버전 이하 클래스만 실행
  --release 옵션으로 하위 호환 바이트코드 생성

디스크립터
  타입 표현: I (int), J (long), L...; (객체)
  메서드: (매개변수)반환타입

Code Attribute
  max_stack: Operand Stack 최대 깊이
  max_locals: Local Variable Array 크기
  code[]: 실제 바이트코드

분석 도구
  javap -v: 디스어셈블 (가장 기본)
  hexdump: 바이너리 직접 확인
  ASM: 프로그래밍 방식 분석/조작
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 메서드의 디스크립터를 작성하라. `List<String> process(int[] arr, Map<String, Integer> map)`

**Q2.** Constant Pool이 왜 클래스 파일의 앞쪽에 위치하는가? Fields, Methods 다음에 있으면 안 되는 이유를 설명하라.

**Q3.** Java 17로 컴파일한 클래스를 Java 8 JVM에서 실행하려면 어떻게 해야 하는가? 가능한 방법과 제약사항을 설명하라.

> 💡 **해설**
>
> **Q1.** `([ILjava/util/Map;)Ljava/util/List;`. 설명: 매개변수 부분 `[I` (int 배열) + `Ljava/util/Map;` (Map 객체, 제네릭은 erasure로 소거). 반환 타입 `Ljava/util/List;` (List 객체). 제네릭 타입 정보 `<String>`, `<String, Integer>`는 바이트코드에서 사라지고 Signature 속성에만 남는다 (디스크립터에는 없음).
>
> **Q2.** Constant Pool이 앞에 있는 이유: Fields, Methods, Attributes가 모두 Constant Pool의 인덱스를 참조하기 때문이다. 파일을 순차 읽기할 때 Constant Pool을 먼저 메모리에 로드해야 이후 데이터(필드명, 메서드명, 타입)를 해석할 수 있다. 만약 뒤에 있다면 파일을 두 번 읽거나 랜덤 액세스가 필요해 성능 저하. 또한 Constant Pool Count가 있어야 나머지 데이터를 파싱할 오프셋을 계산할 수 있다.
>
> **Q3.** 방법 1: `javac --release 8 MyClass.java`로 Java 8 호환 바이트코드 (major version 52) 생성. 제약: Java 9+ 신규 문법/API 사용 불가 (람다, 스트림 OK, 모듈은 X). 방법 2: Retrolambda 같은 도구로 바이트코드를 다운그레이드. 방법 3: 이미 컴파일된 클래스라면 불가능 — JVM 버전을 올리거나 재컴파일 필요. 핵심: 클래스 파일 버전은 컴파일 시점에 결정되고, 런타임에 변경 불가.

---

## 📚 참고 자료

- [JVMS Chapter 4 — The class File Format](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html)
- [ASM Bytecode Manipulation Framework](https://asm.ow2.io/)
- [JVM Class File Disassembler (javap)](https://docs.oracle.com/en/java/javase/17/docs/specs/man/javap.html)

---

<div align="center">

**[⬅️ 이전: Off-Heap & Direct Memory](../runtime-data-areas/07-off-heap-direct-memory.md)** | **[다음: Bytecode Instruction Set ➡️](./02-bytecode-instruction-set.md)**

</div>

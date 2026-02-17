# Loading → Linking → Initializing - 클래스 로딩 3단계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `static` 초기화 블록은 정확히 언제 실행되는가? 클래스가 "로드"될 때인가?
- `Class.forName()`과 `ClassLoader.loadClass()`는 어떻게 다른가?
- 순환 참조하는 두 클래스 A, B를 동시에 초기화하면 무슨 일이 생기는가?
- `final static int MAX = 100`은 다른 static 필드와 초기화 시점이 왜 다른가?
- JVM은 검증(Verification) 단계에서 무엇을 확인하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 클래스 사용과 클래스 준비는 다른 시점에 이루어진다

```java
public class Config {
    static {
        System.out.println("Config 초기화");
        loadFromFile(); // 파일 I/O, DB 연결 등 비용이 큰 작업
    }
    public static String DB_URL = "jdbc:mysql://...";
}

// 질문: 아래 중 어느 시점에 "Config 초기화"가 출력되는가?
// 1. Config.class 파일이 classpath에 존재하는 것이 확인될 때
// 2. ClassLoader가 Config.class 바이트를 읽을 때
// 3. Config.DB_URL 필드에 처음 접근할 때
// 4. new Config()를 처음 호출할 때
```

JVM이 로딩을 단일 단계로 처리했다면 세 가지 문제가 생긴다.

```
문제 1: 성능
  모든 클래스를 JVM 시작 시 한꺼번에 초기화하면
  사용하지도 않는 클래스의 static 블록이 전부 실행됨
  → 기동 시간 폭증

문제 2: 보안
  초기화 전에 바이트코드 검증이 끝나지 않은 클래스를
  실행하면 JVM 내부를 오염시킬 수 있음

문제 3: 의존성
  A를 초기화하려면 B가 필요하고
  B를 초기화하려면 A가 필요한 경우를 처리해야 함
```

JVM은 이를 세 단계로 분리해 각 단계의 책임을 명확히 했다.

```
Loading      → .class 파일을 읽어 메모리에 올리기
Linking      → 검증, 준비, 심볼릭 참조 해결
Initializing → static 초기화 코드 실행
```

---

## 📐 내부 구조

### 1. Loading (로딩)

ClassLoader가 `.class` 파일의 바이트를 읽어 `Class` 객체를 생성하는 단계.

```
.class 파일 (디스크 / 네트워크 / 메모리)
       │
       │ ClassLoader.findClass()
       ▼
  바이트 배열 (byte[])
       │
       │ defineClass()
       ▼
  java.lang.Class 객체 생성
  (Method Area / Metaspace에 메타데이터 적재)
       │
       │ 클래스 캐시에 등록
       ▼
  로딩 완료 → Linking 단계로
```

로딩은 **지연(lazy)** 방식으로 동작한다. 클래스가 실제로 필요한 시점까지 미룬다.

```java
// 이 코드만으로는 SomeClass가 로드되지 않는다
SomeClass obj; // 변수 선언만

// 이 시점에 로드됨 (처음 사용)
obj = new SomeClass(); // → Loading 시작
```

---

### 2. Linking (링킹)

링킹은 세 단계로 나뉜다.

```
Verification (검증)
       │
Preparation (준비)
       │
Resolution (해결) ← 선택적, 지연 가능
```

#### 2-1. Verification (검증)

바이트코드가 JVM 명세를 위반하지 않는지 검사한다.  
악의적으로 조작된 `.class` 파일이 JVM을 오염시키는 것을 막는다.

```
검증 항목:
  ① 파일 형식 검사
     - magic number가 0xCAFEBABE 인가?
     - major/minor version이 현재 JVM이 지원하는 범위인가?

  ② 바이트코드 검사
     - 유효하지 않은 opcodes가 없는가?
     - operand stack 언더플로우/오버플로우가 없는가?
     - 지역 변수 인덱스가 범위를 벗어나지 않는가?

  ③ 타입 검사
     - 메서드 호출 시 인자 타입이 맞는가?
     - 필드 접근 타입이 일치하는가?

  ④ 구조 검사
     - final 클래스를 상속하지 않는가?
     - final 메서드를 오버라이드하지 않는가?
     - Object 이외의 클래스가 super class가 없지 않은가?
```

```java
// 검증이 실패하는 예시 (바이트코드를 직접 조작한 경우)
// java.lang.VerifyError: Bad type on operand stack
// java.lang.VerifyError: Cannot inherit from final class
```

#### 2-2. Preparation (준비)

static 필드의 메모리를 확보하고 **기본값(default value)으로 초기화**한다.  
개발자가 지정한 값이 아닌 타입의 기본값으로.

```
타입별 기본값:
  int, short, byte, char  → 0
  long                    → 0L
  float                   → 0.0f
  double                  → 0.0d
  boolean                 → false
  참조 타입 (Object, 배열) → null
```

```java
public class Example {
    static int count = 100;         // Preparation: count = 0  (기본값)
    static String name = "hello";   // Preparation: name = null (기본값)
    static boolean flag = true;     // Preparation: flag = false (기본값)
    // ↑ 개발자가 지정한 값(100, "hello", true)은 Initializing 단계에서 설정됨
}
```

**예외: Compile-time constant**

```java
public class Example {
    // 컴파일 타임 상수 → Preparation 단계에서 이미 실제 값 설정
    static final int MAX = 100;          // Preparation: MAX = 100 (즉시!)
    static final String PREFIX = "JVM";  // Preparation: PREFIX = "JVM" (즉시!)

    // 컴파일 타임 상수가 아님 → Initializing에서 설정
    static final int COMPUTED = Integer.parseInt("100"); // Preparation: COMPUTED = 0
    static final List<String> LIST = new ArrayList<>();  // Preparation: LIST = null
}
```

이것이 `final static int MAX = 100`을 참조하는 클래스가 **Example 클래스를 초기화하지 않아도** 값을 얻을 수 있는 이유다. 컴파일러가 상수를 호출 지점에 직접 인라인하기 때문이다.

#### 2-3. Resolution (해결)

ConstantPool의 심볼릭 참조(symbolic reference)를 직접 참조(direct reference)로 변환한다.

```
ConstantPool의 심볼릭 참조:
  "com/example/Service.process(Ljava/lang/String;)V"
  (문자열로 된 메서드 서명)
       │
       │ Resolution
       ▼
  직접 참조:
  실제 메모리 주소 또는 오프셋
  (포인터로 된 실제 위치)
```

Resolution은 **지연(lazy) 실행**이 가능하다. 즉, 해당 심볼릭 참조가 처음 실제로 사용될 때 해결해도 된다. JVM 구현마다 전략이 다르다.

---

### 3. Initializing (초기화)

`static` 초기화 코드를 실행하는 단계. 이 단계에서 비로소 개발자가 작성한 값이 적용된다.

JVM은 클래스 초기화를 위해 **`<clinit>` 메서드**를 자동으로 생성한다.

```java
// 개발자가 작성한 코드
public class Service {
    static int count = 0;
    static String name;
    static final Map<String, String> CACHE;

    static {
        name = "ServiceA";
        CACHE = new HashMap<>();
        CACHE.put("key", "value");
        System.out.println("Service 초기화 완료");
    }
}

// JVM이 생성하는 <clinit> (javap로 확인 가능)
// static {};
//   Code:
//     iconst_0          // 0 push
//     putstatic count   // count = 0
//     ldc "ServiceA"    // "ServiceA" push
//     putstatic name    // name = "ServiceA"
//     new HashMap       // new HashMap()
//     invokespecial <init>
//     putstatic CACHE   // CACHE = new HashMap()
//     ... (put 호출)
//     getstatic System.out
//     ldc "Service 초기화 완료"
//     invokevirtual println
//     return
```

#### 초기화 트리거 — 정확히 언제 실행되는가

JVM 명세가 규정하는 **6가지 active use** 시점에만 초기화가 실행된다.

```
1. new 키워드로 인스턴스 생성
   new Service();  → Service 초기화

2. static 필드 읽기/쓰기 (컴파일 타임 상수 제외)
   Service.count = 1;  → Service 초기화
   int x = Service.count;  → Service 초기화

3. static 메서드 호출
   Service.process();  → Service 초기화

4. Class.forName() 호출 (initialize=true가 기본)
   Class.forName("Service");  → Service 초기화

5. 리플렉션 API 사용
   Service.class.getDeclaredMethods();  → Service 초기화

6. 서브클래스 초기화 시 부모 클래스 먼저 초기화
   class Child extends Service {}
   new Child();  → Service 초기화 → Child 초기화
```

**초기화를 트리거하지 않는 것들 (passive use)**

```java
// ① 참조만 하는 경우
Service s;  // 초기화 안 됨

// ② 배열 타입 생성
Service[] arr = new Service[10];  // Service 초기화 안 됨

// ③ 컴파일 타임 상수 접근
System.out.println(Service.MAX_SIZE);  // MAX_SIZE가 final+원시타입+리터럴이면 초기화 안 됨

// ④ ClassLoader.loadClass() (Class.forName과 다름!)
ClassLoader.getSystemClassLoader().loadClass("Service");
// → Loading + Linking까지만. Initializing 안 됨

// ⑤ 서브클래스를 통한 부모 static 필드 접근
Child.count;  // count가 Service의 것이면 Service만 초기화, Child는 아님
```

---

### 4. 초기화 순서와 순환 의존

JVM은 클래스 초기화의 **스레드 안전성을 보장**한다.

```java
// 단일 스레드 초기화 순서
public class Main {
    public static void main(String[] args) {
        System.out.println(Child.value);
    }
}

class Parent {
    static int x = 10;
    static { System.out.println("Parent 초기화, x=" + x); }
}

class Child extends Parent {
    static int value = x * 2;  // 부모의 x 사용
    static { System.out.println("Child 초기화, value=" + value); }
}

// 출력:
// Parent 초기화, x=10    ← 부모 먼저
// Child 초기화, value=20 ← 자식 다음
// 20
```

**순환 초기화 문제**

```java
class A {
    static int val = B.val + 1;  // B에 의존
    static { System.out.println("A.val = " + val); }
}

class B {
    static int val = A.val + 1;  // A에 의존
    static { System.out.println("B.val = " + val); }
}

class Main {
    public static void main(String[] args) {
        System.out.println("A.val=" + A.val + ", B.val=" + B.val);
    }
}

// 실행 결과:
// B.val = 1    ← A.val이 아직 0(기본값)일 때 B 초기화됨
// A.val = 2    ← B.val=1을 사용해 A 초기화됨
// A.val=2, B.val=1

// 왜 이런가?
// A 초기화 시작 → B.val 필요 → B 초기화 시작
// B 초기화 중 A.val 필요 → A는 이미 초기화 중(진행 중 상태)
// → A.val의 현재 값(0, 아직 기본값)을 그대로 사용
// → B.val = 0 + 1 = 1 로 초기화 완료
// → A 초기화 재개: A.val = 1 + 1 = 2
```

JVM은 순환 참조를 **에러로 처리하지 않는다**. 진행 중인 클래스의 현재(미완성) 상태를 그대로 사용한다. 이는 의도치 않은 버그로 이어질 수 있다.

---

## 💻 실험으로 확인하기

### 실험 1: 초기화 트리거 시점 확인

```java
public class InitTriggerTest {

    static class Lazy {
        static final int CONSTANT = 42;        // compile-time constant
        static int mutable = 100;              // 일반 static 필드

        static {
            System.out.println("[Lazy 초기화됨]");
        }
    }

    public static void main(String[] args) throws Exception {
        System.out.println("=== 1. CONSTANT 접근 ===");
        System.out.println(Lazy.CONSTANT);  // 초기화 안 됨 (컴파일 타임 상수)

        System.out.println("\n=== 2. mutable 접근 ===");
        System.out.println(Lazy.mutable);   // 여기서 초기화 발생

        System.out.println("\n=== 3. ClassLoader.loadClass() ===");
        ClassLoader.getSystemClassLoader().loadClass("InitTriggerTest$Lazy2");
        // Lazy2는 로드되지만 초기화 안 됨

        System.out.println("\n=== 4. Class.forName() ===");
        Class.forName("InitTriggerTest$Lazy2");
        // 여기서 Lazy2 초기화 발생
    }

    static class Lazy2 {
        static { System.out.println("[Lazy2 초기화됨]"); }
    }
}
```

예상 출력:

```
=== 1. CONSTANT 접근 ===
42                          ← [Lazy 초기화됨] 출력 없음

=== 2. mutable 접근 ===
[Lazy 초기화됨]             ← 여기서 처음 초기화
100

=== 3. ClassLoader.loadClass() ===
                            ← [Lazy2 초기화됨] 출력 없음

=== 4. Class.forName() ===
[Lazy2 초기화됨]            ← 여기서 초기화
```

---

### 실험 2: Preparation vs Initializing 관찰

```java
public class PreparationTest {

    static class Target {
        static int a = 10;
        static String b = "hello";
        static final int CONST = 99;          // compile-time constant
        static final String COMPUTED;         // compile-time constant 아님

        static {
            COMPUTED = "computed_" + System.currentTimeMillis();
            System.out.println("초기화: a=" + a + ", b=" + b);
        }
    }

    public static void main(String[] args) throws Exception {
        // Preparation 단계 직접 확인이 어렵지만
        // 초기화 순서를 통해 간접 확인 가능

        // CONST는 컴파일러가 호출 지점에 인라인 → Target 초기화 안 됨
        System.out.println("CONST = " + Target.CONST);  // 초기화 없이 99

        // a 접근 → 초기화 트리거
        System.out.println("a = " + Target.a);
    }
}
```

javap로 바이트코드 확인:

```bash
javap -c -verbose PreparationTest\$Target.class

# <clinit> 메서드에서:
# Preparation: a=0, b=null, COMPUTED=null (기본값)
# Initializing: a=10, b="hello", COMPUTED="computed_..."
```

---

### 실험 3: 순환 초기화 탐지

```java
public class CircularInitTest {
    public static void main(String[] args) {
        System.out.println("A.val = " + CircA.val);
        System.out.println("B.val = " + CircB.val);
    }
}

class CircA {
    static int val = CircB.val + 1;
    static { System.out.println("CircA 초기화 완료: val=" + val); }
}

class CircB {
    static int val = CircA.val + 1;
    static { System.out.println("CircB 초기화 완료: val=" + val); }
}
```

```bash
javac CircularInitTest.java
java -Xlog:class+init=debug CircularInitTest
# 초기화 순서와 순환 감지 로그 확인 가능
```

---

## ⚡ 실무 임팩트

### Lazy Initialization 패턴 선택 기준

```java
// ❌ 잘못된 Lazy Initialization
public class ResourceManager {
    private static Connection connection;

    public static Connection getConnection() {
        if (connection == null) {           // 스레드 안전하지 않음
            connection = createConnection();
        }
        return connection;
    }
}

// ✅ Initialization-on-demand Holder 패턴
// JVM의 클래스 초기화 보장을 활용
public class ResourceManager {

    private static class Holder {
        // Holder 클래스는 getConnection() 첫 호출 시에만 초기화됨
        // JVM이 <clinit> 실행을 스레드 안전하게 보장
        static final Connection CONNECTION = createConnection();
    }

    public static Connection getConnection() {
        return Holder.CONNECTION;  // 이 시점에 Holder 초기화 (최초 1회)
    }
}

// 왜 안전한가?
// JVM 명세: 클래스 초기화는 동기화됨
// Holder.<clinit>은 단 한 번만, 스레드 안전하게 실행됨
// synchronized 키워드 없이 Lazy + Thread-safe 달성
```

### Class.forName vs ClassLoader.loadClass 선택

```java
// Class.forName() → Loading + Linking + Initializing
// static 블록 실행이 필요한 경우 (JDBC 드라이버 등록 등)
Class.forName("com.mysql.cj.jdbc.Driver");
// → Driver의 static 블록에서 DriverManager.registerDriver() 실행됨

// ClassLoader.loadClass() → Loading + Linking만
// 클래스 존재 여부 확인, 리플렉션 준비 등 초기화 없이 메타데이터만 필요한 경우
boolean exists = true;
try {
    ClassLoader.getSystemClassLoader().loadClass("com.optional.Feature");
} catch (ClassNotFoundException e) {
    exists = false;
}
// static 블록 실행 없이 클래스 존재 여부만 확인
```

### static 필드 초기화 실패 처리

```java
public class Config {
    static final Properties PROPS;

    static {
        try {
            PROPS = new Properties();
            PROPS.load(new FileInputStream("config.properties"));
        } catch (IOException e) {
            // static 블록에서 checked exception을 던지려면 ExceptionInInitializerError로 감싸야 함
            throw new ExceptionInInitializerError(e);
        }
    }
}

// 호출하는 쪽에서:
try {
    Config.PROPS.getProperty("key");
} catch (ExceptionInInitializerError e) {
    // 초기화 실패
    Throwable cause = e.getCause(); // 원래 IOException
} catch (NoClassDefFoundError e) {
    // 초기화가 한 번 실패한 클래스를 다시 사용하려 하면 이 에러
    // JVM은 실패한 초기화를 재시도하지 않음
}
```

---

## 🚫 흔한 오해

### "클래스가 로드되면 바로 초기화된다"

```
❌ 잘못된 이해:
  ClassLoader가 .class 파일을 읽는 순간 static 블록이 실행된다.

✅ 실제:
  Loading → Linking → Initializing 는 별개의 단계.
  초기화는 6가지 active use 조건 중 하나를 만족할 때만 실행.
  클래스가 로드되어 있어도 오랫동안 초기화되지 않을 수 있다.
  ClassLoader.loadClass()는 초기화를 트리거하지 않는 대표적인 예.
```

### "static final 필드는 항상 컴파일 타임 상수다"

```java
// 컴파일 타임 상수 O → Preparation 단계에서 즉시 값 할당
static final int A = 42;
static final String B = "hello";
static final boolean C = true;

// 컴파일 타임 상수 X → Initializing 단계에서 값 할당
static final int D = Integer.parseInt("42");  // 메서드 호출 포함
static final String E = System.getProperty("key");
static final List<String> F = new ArrayList<>();
static final int G;  // 선언과 초기화 분리 (static 블록에서 할당)

// 구분법: 우변이 리터럴 + 기본 연산만인가?
// 그렇다면 컴파일 타임 상수 → 클래스 초기화 없이 접근 가능
```

### "초기화 실패 후 재시도하면 된다"

```java
// ❌ 초기화에 실패한 클래스는 이후 모든 접근에서 NoClassDefFoundError
class BadInit {
    static {
        if (true) throw new RuntimeException("초기화 실패");
    }
}

// 첫 번째 접근
try { new BadInit(); }
catch (ExceptionInInitializerError e) { System.out.println("첫 실패"); }

// 두 번째 접근
try { new BadInit(); }
catch (NoClassDefFoundError e) {
    System.out.println("두 번째는 NoClassDefFoundError");
    // JVM이 이 클래스를 "실패한 상태"로 영구 마킹
    // 재초기화 시도 없이 즉시 에러 반환
}
```

---

## 📌 핵심 정리

```
3단계 요약
  Loading      .class 바이트를 읽어 Class 객체 생성
  Linking      검증(Verify) → 준비(Prepare) → 해결(Resolve)
  Initializing static 초기화 코드(<clinit>) 실행

Preparation vs Initializing
  Preparation  → 타입 기본값(0, null, false)으로 세팅
  Initializing → 개발자가 지정한 값으로 세팅

컴파일 타임 상수 예외
  static final int X = 100 처럼 리터럴만 있는 경우
  Preparation에서 즉시 실제 값 설정
  클래스 초기화 없이 접근 가능

초기화 트리거 6가지
  new, static 필드 접근, static 메서드 호출,
  Class.forName(), 리플렉션, 서브클래스 초기화

초기화 안 되는 것들
  ClassLoader.loadClass(), 배열 생성, 컴파일 타임 상수 접근

순환 초기화
  JVM은 에러로 처리하지 않음
  진행 중인 클래스의 현재(미완성) 값을 그대로 사용
  → 의도치 않은 0 또는 null이 사용될 수 있음

스레드 안전성
  JVM이 <clinit> 실행을 동기화함
  Initialization-on-demand Holder 패턴의 근거
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 코드의 실행 결과를 예측하고, 그 이유를 3단계(Loading/Linking/Initializing)로 설명하라.

```java
class Foo {
    static final int X = Bar.Y + 1;
    static { System.out.println("Foo init, X=" + X); }
}
class Bar {
    static final int Y = Foo.X + 1;
    static { System.out.println("Bar init, Y=" + Y); }
}
public class Main {
    public static void main(String[] args) {
        System.out.println(Foo.X);
    }
}
```

**Q2.** JDBC 드라이버 등록에 `Class.forName("com.mysql.cj.jdbc.Driver")`를 쓰는 이유는 무엇인가? `ClassLoader.loadClass()`로 대체하면 어떻게 되는가?

**Q3.** Initialization-on-demand Holder 패턴이 `synchronized` 키워드 없이도 스레드 안전한 이유를 JVM 초기화 보장과 연결해 설명하라.

> 💡 **해설**
>
> **Q1.** Main이 `Foo.X`를 접근 → Foo 초기화 시작 → `Bar.Y` 필요 → Bar 초기화 시작 → `Foo.X` 필요 → Foo는 이미 초기화 중 → Foo.X의 현재값(0, 아직 기본값)을 사용 → `Bar.Y = 0 + 1 = 1` Bar 초기화 완료 → Foo 초기화 재개: `Foo.X = 1 + 1 = 2` → 출력: `Bar init, Y=1` → `Foo init, X=2` → `2`. 순환 초기화로 인해 직관과 다른 결과.
>
> **Q2.** `Class.forName()`은 Initializing까지 실행한다. MySQL Driver의 `static` 블록 안에 `DriverManager.registerDriver(new Driver())`가 있어 이 블록이 실행돼야 드라이버가 등록된다. `ClassLoader.loadClass()`로 대체하면 Loading + Linking만 되고 `static` 블록이 실행되지 않아 드라이버가 등록되지 않는다. Java 6+ 이후 ServiceLoader 메커니즘으로 자동 등록되므로 현재는 생략 가능하지만, 원리는 동일하다.
>
> **Q3.** JVM 명세는 클래스 초기화(`<clinit>` 실행)가 동기화됨을 보장한다. 여러 스레드가 동시에 Holder 클래스를 처음 사용하더라도 `<clinit>`는 단 한 번만, 원자적으로 실행된다. 나머지 스레드는 초기화가 완료될 때까지 블록된 후 완성된 값을 받는다. `synchronized`는 매 호출마다 잠금 비용이 발생하지만, 클래스 초기화 잠금은 초기화 완료 이후 완전히 사라진다.

---

## 📚 참고 자료

- [JVMS §5.4 — Linking](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html#jvms-5.4)
- [JVMS §5.5 — Initialization](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html#jvms-5.5)
- [JLS §12.4 — Initialization of Classes and Interfaces](https://docs.oracle.com/javase/specs/jls/se21/html/jls-12.html#jls-12.4)

---

<div align="center">

**[⬅️ 이전: ClassLoader Hierarchy](./01-classloader-hierarchy.md)** | **[다음: Bytecode Verification ➡️](./03-bytecode-verification.md)**

</div>

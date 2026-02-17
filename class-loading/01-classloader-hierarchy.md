# ClassLoader Hierarchy - 클래스로더 계층 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `String.class.getClassLoader()`는 왜 `null`을 반환하는가?
- JVM은 어떻게 같은 이름의 클래스가 두 번 로드되는 것을 막는가?
- Tomcat은 어떻게 두 웹앱이 서로 다른 Jackson 버전을 동시에 사용하게 하는가?
- ClassLoader 누수가 왜 `OutOfMemoryError: Metaspace`로 이어지는가?
- 내가 만든 `java.lang.String`을 classpath에 심으면 어떻게 되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 클래스를 "누가" 로드하느냐가 중요하다

단순히 생각하면 클래스 로딩은 `.class` 파일을 읽어 메모리에 올리는 과정이다.  
그런데 JVM은 왜 이걸 단일 메커니즘으로 처리하지 않고 **계층 구조**를 만들었을까?

세 가지 문제를 동시에 해결해야 했기 때문이다.

```
문제 1: 보안
  악의적인 코드가 classpath에 java/lang/String.class를 심으면?
  진짜 String을 가짜로 교체해 모든 문자열 처리를 가로챌 수 있다.

문제 2: 일관성
  모든 클래스의 최상위 부모인 Object가
  각자 다른 버전으로 로드된다면?
  instanceof, 상속, 캐스팅이 전부 무너진다.

문제 3: 격리
  A 라이브러리는 jackson 2.12, B 라이브러리는 jackson 2.15가 필요할 때
  하나의 JVM에서 두 버전을 동시에 쓰려면?
```

이 세 문제를 해결하는 설계가 **Parent Delegation Model을 가진 ClassLoader 계층 구조**다.

---

## 📐 내부 구조

### 1. 계층 구조 전체 그림

```
┌─────────────────────────────────────────────────┐
│         Bootstrap ClassLoader                   │
│   (JVM 내장, C++로 구현, Java 객체 아님 → null)      │
│   로드 대상: rt.jar (Java 8) / jdk.* 모듈 (9+)     │
│   java.lang.*, java.util.*, java.io.* ...       │
└──────────────────────┬──────────────────────────┘
                       │ parent
┌──────────────────────▼──────────────────────────┐
│     Extension ClassLoader (Java 8)              │
│     Platform ClassLoader  (Java 11+)            │
│   로드 대상: $JAVA_HOME/lib/ext/*.jar             │
│   javax.*, com.sun.* ...                        │
└──────────────────────┬──────────────────────────┘
                       │ parent
┌──────────────────────▼──────────────────────────┐
│         Application ClassLoader                 │
│         (= System ClassLoader)                  │
│   로드 대상: -classpath 또는 CLASSPATH 환경변수       │
│   내가 만든 클래스, 3rd party jar 전부               │
└──────────────────────┬──────────────────────────┘
                       │ parent
┌──────────────────────▼──────────────────────────┐
│           Custom ClassLoader                    │
│   Tomcat WebApp ClassLoader, OSGi Bundle,       │
│   Spring DevTools Restart ClassLoader 등        │
└─────────────────────────────────────────────────┘
```

```java
// 실제 코드로 확인
System.out.println(String.class.getClassLoader());
// null  → Bootstrap이 로드 (Java 객체가 아니라 null로 표현)

System.out.println(ClassLoader.getSystemClassLoader().getParent());
// jdk.internal.loader.ClassLoaders$PlatformClassLoader@...

System.out.println(MyClass.class.getClassLoader());
// jdk.internal.loader.ClassLoaders$AppClassLoader@...
```

---

### 2. Parent Delegation Model — 핵심 동작 원리

클래스 로드 요청이 들어오면 아래 순서로 동작한다.

```
클래스 로드 요청: "com.example.MyClass"

  Application ClassLoader
       │
       │  1. 캐시 확인: findLoadedClass("com.example.MyClass")
       │     → 없음. 부모에게 위임
       │
       ▼  위임 (delegation)
  Platform ClassLoader
       │
       │  2. 캐시 확인 → 없음. 부모에게 위임
       │
       ▼  위임
  Bootstrap ClassLoader
       │
       │  3. rt.jar / 모듈에서 탐색
       │     → com.example.MyClass 없음
       │
       ▼  실패 반환
  Platform ClassLoader
       │  4. lib/ext에서 탐색 → 없음
       ▼  실패 반환
  Application ClassLoader
       │  5. classpath에서 탐색 → 발견!
       ▼
     로드 완료 ✅
```

이 흐름이 `ClassLoader.loadClass()`의 실제 구현이다.

```java
// java.lang.ClassLoader 핵심 로직 (단순화)
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException {

    synchronized (getClassLoadingLock(name)) {
        // Step 1: 이미 로드된 클래스인지 확인 (캐시)
        Class<?> c = findLoadedClass(name);

        if (c == null) {
            try {
                // Step 2: 부모에게 먼저 위임 (재귀)
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    // parent == null → Bootstrap에게 위임
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // 부모가 못 찾은 것 → 정상 흐름, 내가 찾아야 함
            }

            if (c == null) {
                // Step 3: 부모가 모두 실패 → 내가 직접 로드
                c = findClass(name);  // ← 실제 파일 I/O 발생
            }
        }

        if (resolve) resolveClass(c);
        return c;
    }
}
```

**Parent Delegation이 보안 문제 1을 해결하는 방식:**

```
공격 시도:
  classpath에 java/lang/String.class (악성 버전) 배치

Parent Delegation 동작:
  Application ClassLoader → Platform → Bootstrap
  Bootstrap이 rt.jar에서 진짜 String을 먼저 발견
  → 악성 String은 로드될 기회조차 없음

Parent Delegation 없이 Application이 먼저 탐색했다면:
  classpath의 악성 String이 로드됨
  모든 문자열 처리가 공격자 코드로 흐름
```

---

### 3. 클래스의 정체성: (ClassLoader + 클래스명)

JVM에서 클래스를 식별하는 조건은 이름만이 아니다.

```
클래스 정체성 = ClassLoader 인스턴스 + 패키지명.클래스명

ClassLoader A가 로드한 com.example.Foo
ClassLoader B가 로드한 com.example.Foo
→ JVM이 보기에 완전히 다른 두 클래스
```

```java
URLClassLoader loaderA = new URLClassLoader(urls, null); // parent 없음
URLClassLoader loaderB = new URLClassLoader(urls, null);

Class<?> fromA = loaderA.loadClass("com.example.Foo");
Class<?> fromB = loaderB.loadClass("com.example.Foo");

System.out.println(fromA == fromB);             // false
System.out.println(fromA.equals(fromB));        // false

Object instanceA = fromA.getDeclaredConstructor().newInstance();
System.out.println(fromB.isInstance(instanceA)); // false
// → ClassCastException도 발생
```

이것이 문제 3(격리)을 해결하는 원리다.

---

### 4. Tomcat의 ClassLoader 설계 — 격리 구현

```
Bootstrap ClassLoader
       │
System ClassLoader
       │
Tomcat Common ClassLoader  ← Tomcat 공용 라이브러리
       │
  ┌────┴────┐
  │         │
WebApp A   WebApp B   ← 각 웹앱에 독립적인 ClassLoader
  │         │
Jackson   Jackson
2.12.0    2.15.0      ← 서로 다른 버전이 충돌 없이 공존
```

Tomcat WebApp ClassLoader는 Parent Delegation을 **의도적으로 역전**한다.

```java
// Tomcat WebApp ClassLoader의 핵심 로직 (단순화)
@Override
public Class<?> loadClass(String name) throws ClassNotFoundException {

    // java.lang.*, javax.* 등 핵심 클래스는 항상 부모 우선 (보안 유지)
    if (isJavaCoreClass(name)) {
        return getParent().loadClass(name);
    }

    // ★ 역전: 웹앱 자체 클래스(WEB-INF/classes, WEB-INF/lib)를 먼저 탐색
    try {
        Class<?> c = findClass(name);
        if (c != null) return c;
    } catch (ClassNotFoundException ignored) {}

    // 웹앱에 없으면 그제서야 부모(Common ClassLoader)에게
    return getParent().loadClass(name);
}
```

Parent Delegation은 강제 사항이 아닌 **권장 사항**이다.  
`loadClass()`를 오버라이드해 순서를 바꿀 수 있다.  
단, `java.lang.*`까지 역전하면 JVM 안정성이 파괴되므로 핵심 클래스는 항상 예외 처리한다.

---

### 5. ClassLoader 누수 메커니즘

```
ClassLoader가 GC되려면 ClassLoader 자신에 대한
모든 강한 참조(Strong Reference)가 사라져야 한다.

문제는 이 참조 체인이 생각보다 길다는 것:

static 필드
    │ 참조
    ▼
  인스턴스
    │ 내부 참조 (instance.getClass())
    ▼
  Class 객체
    │ 내부 참조 (clazz.getClassLoader())
    ▼
  ClassLoader    ← 이게 GC되지 않는다
    │ 로드한 모든 클래스 참조
    ▼
  Metaspace에 적재된 모든 Class 메타데이터들
                 ← 전부 해제되지 않는다
```

```java
// ❌ 누수가 발생하는 패턴
public class PluginManager {
    // static 필드 → 인스턴스 → Class → ClassLoader 참조 체인 형성
    private static Object pluginInstance;

    public void loadPlugin() throws Exception {
        URLClassLoader loader = new URLClassLoader(urls);
        Class<?> clazz = loader.loadClass("com.example.Plugin");
        pluginInstance = clazz.getDeclaredConstructor().newInstance();

        loader.close(); // File 핸들만 닫힘. ClassLoader는 GC 안 됨
    }
}

// ✅ 올바른 패턴
public class PluginManager {
    private static WeakReference<Object> pluginRef;

    public void loadPlugin() throws Exception {
        URLClassLoader loader = new URLClassLoader(urls);
        Class<?> clazz = loader.loadClass("com.example.Plugin");
        Object instance = clazz.getDeclaredConstructor().newInstance();
        pluginRef = new WeakReference<>(instance); // 약한 참조 → GC 허용
        loader.close();
    }

    public void unloadPlugin() {
        pluginRef = null;
        // 이제 instance → clazz → loader 체인의 강한 참조가 없어짐
        // GC 대상이 됨
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: ClassLoader 계층 직접 탐색

```java
public class ExploreHierarchy {
    public static void main(String[] args) {
        printHierarchy("내 클래스", ExploreHierarchy.class.getClassLoader());

        System.out.println("\n--- 주요 클래스들의 ClassLoader ---");
        System.out.printf("%-12s → %s%n", "String",    String.class.getClassLoader());
        System.out.printf("%-12s → %s%n", "ArrayList", java.util.ArrayList.class.getClassLoader());
        System.out.printf("%-12s → %s%n", "MyClass",   ExploreHierarchy.class.getClassLoader());
    }

    static void printHierarchy(String label, ClassLoader cl) {
        System.out.println("[" + label + "] ClassLoader 계층:");
        int level = 0;
        while (cl != null) {
            System.out.println("  " + "  ".repeat(level) + "↳ " + cl.getClass().getName());
            cl = cl.getParent();
            level++;
        }
        System.out.println("  " + "  ".repeat(level) + "↳ null (Bootstrap)");
    }
}
```

예상 출력 (Java 17):

```
[내 클래스] ClassLoader 계층:
  ↳ jdk.internal.loader.ClassLoaders$AppClassLoader
    ↳ jdk.internal.loader.ClassLoaders$PlatformClassLoader
      ↳ null (Bootstrap)

--- 주요 클래스들의 ClassLoader ---
String       → null         ← Bootstrap이 로드
ArrayList    → null         ← Bootstrap이 로드
MyClass      → jdk.internal.loader.ClassLoaders$AppClassLoader
```

---

### 실험 2: 두 ClassLoader → 다른 타입 증명

```java
import java.io.File;
import java.net.*;

public class ClassIdentityDemo {
    public static void main(String[] args) throws Exception {
        URL[] urls = {new File("./build/classes").toURI().toURL()};

        // parent를 null로 설정 → Application ClassLoader를 완전히 우회
        URLClassLoader loaderA = new URLClassLoader(urls, null);
        URLClassLoader loaderB = new URLClassLoader(urls, null);

        Class<?> fromA = loaderA.loadClass("SampleClass");
        Class<?> fromB = loaderB.loadClass("SampleClass");

        System.out.println("fromA == fromB          : " + (fromA == fromB));        // false
        System.out.println("이름이 같은가            : " + fromA.getName().equals(fromB.getName())); // true
        System.out.println("fromA.isInstance(B obj) : "
            + fromA.isInstance(fromB.getDeclaredConstructor().newInstance()));       // false

        loaderA.close();
        loaderB.close();
    }
}
```

실행:

```bash
# SampleClass.java 를 build/classes 에 컴파일
javac -d ./build/classes SampleClass.java
javac ClassIdentityDemo.java
java ClassIdentityDemo
```

---

### 실험 3: ClassLoader 누수 탐지

```java
import java.io.File;
import java.lang.ref.*;
import java.net.*;

public class LeakDetectionDemo {
    public static void main(String[] args) throws Exception {
        URL[] urls = {new File("./build/classes").toURI().toURL()};

        WeakReference<ClassLoader> ref;

        {
            URLClassLoader loader = new URLClassLoader(urls);
            loader.loadClass("SampleClass");
            ref = new WeakReference<>(loader);
            // 블록 종료 → loader 지역 변수 소멸
        }

        System.out.println("GC 전: " + (ref.get() != null ? "살아있음" : "수거됨"));

        System.gc();
        Thread.sleep(200);
        System.gc();

        System.out.println("GC 후: " + (ref.get() != null ? "살아있음 ← 누수!" : "수거됨 ✅"));
    }
}
```

Metaspace 증가 추적:

```bash
# 프로세스 ID 확인
jps

# 로드된 클래스 수 1초마다 출력
jstat -class <pid> 1000

# 출력:
# Loaded  Bytes   Unloaded  Bytes     Time
#   8453  16234          0     0.0    1.23
#   8453  16234          0     0.0    1.24  ← Unloaded가 계속 0 → 누수 의심
```

---

## ⚡ 실무 임팩트

### Spring Boot DevTools의 빠른 재시작 원리

```
Spring Boot DevTools가 두 개의 ClassLoader를 쓰는 이유:

Base ClassLoader:
  → 변경될 일 없는 3rd party 라이브러리 로드
  → 재시작 시 재사용

Restart ClassLoader:
  → 개발자가 작성한 클래스만 로드
  → 파일 변경 감지 시 이것만 교체

결과:
  전체 재시작: 수십 초
  Restart ClassLoader만 교체: 1~2초

이것이 가능한 이유가 ClassLoader 계층 구조 덕분
```

### Metaspace OOM 장애 대응

```
증상:
  java.lang.OutOfMemoryError: Metaspace

1차 진단 — 로드된 클래스 수 추이 확인:
  jstat -class <pid> 2000 30
  
  Loaded 숫자가 계속 증가하고 Unloaded가 0에 가깝다면
  ClassLoader 누수 가능성이 높다.

2차 진단 — Heap Dump 분석:
  jcmd <pid> GC.heap_dump /tmp/heap.hprof
  
  Eclipse MAT 또는 IntelliJ에서 열기
  "ClassLoader" 검색 후 어떤 ClassLoader가
  얼마나 많은 클래스를 붙들고 있는지 확인

3차 — GC Root 역추적:
  MAT의 "Path to GC Roots" 기능 사용
  ClassLoader를 참조하는 최상위 GC Root 찾기
  대부분 static 필드 또는 ThreadLocal에서 발견됨
```

### 클래스 충돌 디버깅 (ClassCastException이 이상한 위치에서 날 때)

```java
// "분명히 같은 타입인데 ClassCastException?"
// → 두 클래스로더가 같은 클래스를 각각 로드한 것

// 진단 코드
Object obj = someFactory.create();
System.out.println("객체 타입: " + obj.getClass().getName());
System.out.println("객체 ClassLoader: " + obj.getClass().getClassLoader());
System.out.println("기대 타입 ClassLoader: " + MyInterface.class.getClassLoader());
// 두 ClassLoader가 다르다면 → 그것이 원인
```

---

## 🚫 흔한 오해

### "loader.close()를 호출하면 ClassLoader가 GC된다"

```java
// ❌ 잘못된 이해
URLClassLoader loader = new URLClassLoader(urls);
Class<?> clazz = loader.loadClass("Plugin");
Object instance = clazz.newInstance();

loader.close(); // 이것만으로는 GC 안 됨

// close()가 하는 일:
//   열어둔 jar 파일의 FileDescriptor만 반납
//   ClassLoader 객체 자체의 생명주기와 무관

// ClassLoader가 GC되려면:
//   instance, clazz, loader 에 대한 모든 강한 참조가 사라져야 함
```

### "Bootstrap ClassLoader는 null이니 존재하지 않는다"

```
String.class.getClassLoader() == null
→ Bootstrap ClassLoader가 없다는 뜻이 아님

null인 이유:
  Bootstrap은 C++로 구현된 JVM 내부 컴포넌트
  Java 힙 위에 있지 않아 Java 레벨에서 참조 불가
  존재하지만 Java 객체로 표현할 수 없어 null 반환

실제로는 Bootstrap이 수천 개의 클래스를 로드하고
모든 ClassLoader 계층의 최상위로서 항상 동작 중
```

### "Parent Delegation은 항상 지켜져야 한다"

```
Parent Delegation은 강제 규칙이 아닌 권장 패턴이다.
loadClass()를 오버라이드하면 얼마든지 변경 가능.

실제로 역전하는 사례:
  Tomcat WebApp ClassLoader  → 웹앱 클래스 우선 탐색
  OSGi Framework             → 번들별 독립적 탐색 순서
  Spring DevTools            → 개발 클래스 우선

역전 시 지켜야 할 한 가지:
  java.lang.*, java.util.* 등 JDK 핵심 클래스는
  항상 Bootstrap에 위임해야 한다.
  이를 어기면 Object, String, Thread가 여러 버전 존재하게 돼
  JVM 전체가 붕괴된다.
```

---

## 📌 핵심 정리

```
ClassLoader 계층 (Java 17 기준)
  Bootstrap (null)
    ↑ parent
  PlatformClassLoader (구 Extension)
    ↑ parent
  AppClassLoader (= SystemClassLoader)
    ↑ parent
  Custom ClassLoader (Tomcat, OSGi 등)

Parent Delegation 흐름
  요청 → 자식 → 부모 → 부모 → Bootstrap
  결과 ← 자식 ← 부모 ← 처음 찾은 곳에서 로드

클래스 정체성
  (ClassLoader 인스턴스) + (FQCN) = 유일한 클래스
  다른 ClassLoader가 로드한 같은 이름 ≠ 같은 클래스

Parent Delegation이 해결하는 문제
  보안   → 핵심 클래스 교체 공격 차단
  일관성 → Object는 항상 Bootstrap이 로드한 하나
  격리   → 다른 ClassLoader = 다른 클래스 공간

ClassLoader 누수 경로
  static 필드 → 인스턴스 → Class → ClassLoader → Metaspace
  체인 어딘가에 강한 참조가 살아있으면 Metaspace OOM
  진단: jstat -class, MAT heap dump 분석
```

---

## 🤔 생각해볼 문제

**Q1.** Parent Delegation 모델이 없다면 JVM에 어떤 보안 취약점이 생길까?

**Q2.** 같은 JVM에서 두 라이브러리가 jackson 2.12와 2.15를 각각 필요로 한다.  
일반 Spring Boot 애플리케이션(ClassLoader가 하나)에서는 어떻게 해결해야 할까?  
Tomcat 같은 환경과 어떻게 다른가?

**Q3.** `jstat -class` 로 모니터링 중 Loaded가 계속 증가하고 Unloaded가 0이다.  
이 상황에서 어떤 순서로 원인을 추적하겠는가?

> 💡 **해설**
>
> **Q1.** classpath에 `java/lang/String.class`를 심으면 Application ClassLoader가 이를 먼저 로드한다. 모든 문자열 처리가 공격자 코드를 통하게 되어 비밀번호, 토큰 등 민감 정보를 탈취할 수 있다. Parent Delegation 덕분에 Bootstrap이 항상 rt.jar의 String을 먼저 찾아 이 공격을 원천 차단한다.
>
> **Q2.** 일반 Spring Boot는 단일 AppClassLoader를 공유하므로 두 버전 중 하나만 클래스패스에 올라간다. 해결 방법은 두 라이브러리를 직접 격리하는 것 — OSGi 사용, 별도 프로세스 분리, 또는 아예 MSA로 분리. Tomcat이 다른 이유는 웹앱마다 독립적인 ClassLoader를 부여하기 때문. 단일 JVM에서 ClassLoader 격리를 직접 구현하지 않는 이상 공유 환경에서 두 버전 공존은 불가능하다.
>
> **Q3.** ① `jstat -class <pid>`로 증가 속도와 패턴 확인 (배포 후 증가? 요청마다 증가?) → ② `jcmd <pid> GC.heap_dump` 로 heap dump 수집 → ③ Eclipse MAT에서 ClassLoader 인스턴스 수와 각각이 붙들고 있는 클래스 수 확인 → ④ 가장 많은 클래스를 들고 있는 ClassLoader에 대해 "Path to GC Roots" 실행 → ⑤ 최상위 GC Root(static 필드 또는 ThreadLocal)를 코드에서 찾아 참조 해제.

---

## 📚 참고 자료

- [JVMS §5 — Loading, Linking, Initializing](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html)
- [OpenJDK ClassLoader.java 소스](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/lang/ClassLoader.java)
- [JEP 261: Module System](https://openjdk.org/jeps/261)

---

<div align="center">

**[⬅️ 목차로 돌아가기](../README.md)** | **[다음: Loading → Linking → Initializing ➡️](./02-loading-linking-initializing.md)**

</div>

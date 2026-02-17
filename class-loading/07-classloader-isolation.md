# ClassLoader Isolation - 클래스로더 격리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 같은 `.class` 파일을 두 ClassLoader로 로드했을 때 `==` 비교가 `false`인 이유는?
- 격리된 두 ClassLoader 사이에서 객체를 어떻게 안전하게 주고받는가?
- OSGi가 하나의 JVM에서 라이브러리 버전 충돌 없이 수백 개의 번들을 운영하는 원리는?
- ClassLoader 격리가 보안에 어떻게 활용되는가?
- 격리 설계 시 공유 영역과 격리 영역을 어떻게 나누어야 하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 하나의 JVM에서 서로 다른 세계가 공존해야 한다

```
시나리오 1: 라이브러리 버전 충돌
  플러그인 A: jackson 2.12 사용
  플러그인 B: jackson 2.15 사용
  
  단일 ClassLoader: 두 버전 중 하나만 존재
  → 나중에 로드된 버전이 먼저 로드된 버전을 무시하거나 충돌

시나리오 2: Tomcat 멀티 웹앱
  WebApp A: Spring 5.3 기반
  WebApp B: Spring 6.1 기반
  같은 Tomcat 위에서 동시에 실행
  → Spring 클래스가 두 버전 공존해야 함

시나리오 3: 플러그인 보안 격리
  외부에서 받은 플러그인이 핵심 서비스에 접근하면 안 됨
  → 플러그인을 제한된 ClassLoader에서 실행

시나리오 4: 테스트 격리
  테스트 A의 static 상태가 테스트 B에 영향을 주면 안 됨
  → 각 테스트를 독립된 ClassLoader에서 실행
```

이 모든 시나리오의 해결책이 **ClassLoader 격리**다.  
서로 다른 ClassLoader = 서로 다른 클래스 공간 = 상호 영향 없음.

---

## 📐 내부 구조

### 1. 격리의 기반 — 클래스 정체성 재확인

```
JVM의 클래스 정체성 규칙:
  클래스 C1 == 클래스 C2
  ↔ (C1.getClassLoader() == C2.getClassLoader())
     AND (C1.getName().equals(C2.getName()))

즉:
  같은 .class 파일 → 다른 ClassLoader로 로드
  → 완전히 다른 두 개의 Class 객체
  → instanceof false, 캐스팅 불가, == false
```

```java
URLClassLoader loaderA = new URLClassLoader(urls, null);
URLClassLoader loaderB = new URLClassLoader(urls, null);

Class<?> classA = loaderA.loadClass("com.example.Foo");
Class<?> classB = loaderB.loadClass("com.example.Foo");

classA == classB              // false
classA.equals(classB)         // false
classA.isInstance(new Foo())  // 상황에 따라 다름
classA.isInstance(classB.newInstance())  // false → 격리됨
```

---

### 2. 격리 구조 설계 패턴

#### 패턴 1: 완전 격리 (Sibling ClassLoaders)

```
         공통 부모 ClassLoader
         (공유 인터페이스, API 계층)
                │
       ┌────────┴────────┐
       │                 │
  ClassLoader A     ClassLoader B
  (플러그인 A)       (플러그인 B)
  자체 jackson       자체 jackson
  자체 구현체         자체 구현체
  
특징:
  A와 B는 서로의 클래스를 볼 수 없음
  공통 부모의 클래스만 공유
  
사용처: OSGi 번들, 플러그인 시스템, Tomcat WebApp
```

#### 패턴 2: 계층 격리 (Hierarchical ClassLoaders)

```
  Bootstrap ClassLoader
         │
  Platform ClassLoader
         │
  Application ClassLoader (공통 라이브러리)
         │
  Module ClassLoader (모듈별 격리)
         │
  Plugin ClassLoader (플러그인별 격리)

특징:
  아래로 갈수록 더 세밀한 격리
  상위 ClassLoader의 클래스는 모든 하위에서 공유
  
사용처: Java EE 애플리케이션 서버, 복잡한 플러그인 계층
```

#### 패턴 3: 역전 격리 (Parent-Last)

```
  부모 ClassLoader (공통 API)
         │
  자식 ClassLoader (자신을 먼저 탐색)

특징:
  일반적인 Parent Delegation을 역전
  자식이 부모보다 자신의 클래스를 먼저 사용
  부모와 자식이 같은 라이브러리의 다른 버전을 독립적으로 사용 가능
  
사용처: Tomcat WebApp ClassLoader
```

---

### 3. 격리된 ClassLoader 간 통신 — 인터페이스 브리지 패턴

격리된 두 ClassLoader 사이에서 객체를 직접 주고받으면 `ClassCastException`이 발생한다.  
해결책은 **공유 ClassLoader에 인터페이스를 두는 것**이다.

```
              공통 부모 ClassLoader
              ┌────────────────────┐
              │  interface Plugin  │  ← 여기에 인터페이스 정의
              │  interface Service │
              └────────────────────┘
                       │ parent
              ┌────────┴────────┐
              │                 │
    Plugin A ClassLoader   Plugin B ClassLoader
    ┌──────────────────┐   ┌──────────────────┐
    │ PluginAImpl      │   │ PluginBImpl      │
    │ implements Plugin│   │ implements Plugin│
    │ (jackson 2.12)   │   │ (jackson 2.15)   │
    └──────────────────┘   └──────────────────┘

통신 방법:
  Plugin a = (Plugin) loaderA.loadClass("PluginAImpl").newInstance();
  Plugin b = (Plugin) loaderB.loadClass("PluginBImpl").newInstance();
  // Plugin 인터페이스는 공통 ClassLoader에 있으므로 캐스팅 가능
  
  a.execute();  // PluginAImpl의 구현 실행 (내부에서 jackson 2.12 사용)
  b.execute();  // PluginBImpl의 구현 실행 (내부에서 jackson 2.15 사용)
```

---

### 4. OSGi — 격리의 극한

OSGi는 ClassLoader 격리를 가장 정교하게 구현한 프레임워크다.

```
OSGi 번들(Bundle) 구조:
  각 번들 = 독립적인 ClassLoader
  번들 간 의존성은 Import-Package / Export-Package로 명시
  
MANIFEST.MF 예시:
  Bundle-Name: My Plugin
  Export-Package: com.example.api;version="1.0.0"     ← 이 패키지를 외부에 공개
  Import-Package: org.slf4j;version="[1.7,2.0)"       ← 이 패키지가 필요
  Private-Package: com.example.internal               ← 외부에 비공개

OSGi ClassLoader 탐색 순서:
  1. java.* 패키지 → Bootstrap
  2. Import-Package에 명시된 패키지 → 제공하는 번들의 ClassLoader
  3. 번들 자체의 클래스 → 자신의 ClassLoader
  4. Export-Package에 없는 클래스 → 다른 번들에서 접근 불가
```

```
OSGi가 해결하는 문제:

번들 A: com.example.api 1.0 사용
번들 B: com.example.api 2.0 사용
번들 C: com.example.api 제공 (1.0과 2.0 모두)

OSGi ClassLoader 연결:
  번들 A의 ClassLoader → api 1.0 조회 시 → 번들 C의 1.0 ClassLoader 참조
  번들 B의 ClassLoader → api 2.0 조회 시 → 번들 C의 2.0 ClassLoader 참조

  같은 JVM, 완전히 다른 버전 공존 ✅
```

---

### 5. 보안 격리 — Sandbox

ClassLoader 격리를 보안 경계로 활용할 수 있다.

```
Java Applet 시대의 SecurityManager + ClassLoader 조합:

신뢰할 수 없는 코드를 별도 ClassLoader로 로드
SecurityManager로 파일 I/O, 네트워크, 리플렉션 제한
→ 코드가 실행돼도 시스템에 접근 불가

현재 (Java 17+):
  SecurityManager deprecated
  더 강력한 방법: GraalVM Polyglot Sandbox, 별도 JVM 프로세스
  또는 모듈 시스템으로 접근 제한
```

```java
// 제한된 ClassLoader 예시 (개념)
public class SandboxClassLoader extends ClassLoader {

    // 허용된 클래스 패키지 목록
    private static final Set<String> ALLOWED_PACKAGES = Set.of(
        "java.lang.", "java.util.", "com.example.sandbox.api."
    );

    @Override
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        // 허용되지 않은 패키지 접근 차단
        boolean allowed = ALLOWED_PACKAGES.stream()
            .anyMatch(name::startsWith);

        if (!allowed) {
            throw new ClassNotFoundException(
                "Sandbox에서 접근 불가: " + name);
        }

        return super.loadClass(name);
    }
}
```

---

### 6. 격리와 성능 — 공유 영역 결정

모든 것을 격리하면 성능 문제가 생긴다.

```
격리 비용:
  같은 라이브러리가 여러 ClassLoader에 중복 로드
  → Metaspace 사용량 증가
  → 클래스 로딩 시간 증가
  → JIT 컴파일 캐시가 ClassLoader별로 별도 관리

공유 가능한 것:
  인터페이스, 추상 클래스 (API 계층)
  변경 없는 공통 유틸리티 라이브러리
  JDK 클래스 (항상 Bootstrap이 공유)

격리해야 하는 것:
  라이브러리 구현체 (버전 충돌 가능)
  상태(static 필드)를 갖는 클래스
  서로 다른 설정이 필요한 클래스
```

---

## 💻 실험으로 확인하기

### 실험 1: 격리 증명 — 같은 이름, 다른 타입

```java
import java.io.File;
import java.net.*;

public class IsolationProof {
    public static void main(String[] args) throws Exception {
        URL[] urls = {new File("./build/classes").toURI().toURL()};

        // 두 독립 ClassLoader (부모 없음 → 완전 격리)
        URLClassLoader loaderA = new URLClassLoader(urls, null);
        URLClassLoader loaderB = new URLClassLoader(urls, null);

        Class<?> classFromA = loaderA.loadClass("com.example.Config");
        Class<?> classFromB = loaderB.loadClass("com.example.Config");

        System.out.println("=== 클래스 정체성 ===");
        System.out.println("classFromA == classFromB    : " + (classFromA == classFromB));
        System.out.println("이름 동일                    : " + classFromA.getName().equals(classFromB.getName()));
        System.out.println("ClassLoader A               : " + classFromA.getClassLoader());
        System.out.println("ClassLoader B               : " + classFromB.getClassLoader());

        System.out.println("\n=== 인스턴스 호환성 ===");
        Object instanceA = classFromA.getDeclaredConstructor().newInstance();
        Object instanceB = classFromB.getDeclaredConstructor().newInstance();

        System.out.println("A instanceof classFromA     : " + classFromA.isInstance(instanceA));
        System.out.println("B instanceof classFromA     : " + classFromA.isInstance(instanceB));  // false!

        System.out.println("\n=== 캐스팅 시도 ===");
        try {
            Object cast = classFromA.cast(instanceB);
        } catch (ClassCastException e) {
            System.out.println("ClassCastException 발생: " + e.getMessage());
        }

        loaderA.close();
        loaderB.close();
    }
}
```

---

### 실험 2: 인터페이스 브리지 패턴 구현

```java
// 공통 ClassLoader에 위치하는 인터페이스
// Greeter.java (Application ClassLoader가 로드)
public interface Greeter {
    String greet(String name);
}

// 플러그인 구현체 (별도 ClassLoader가 로드)
// EnglishGreeter.java
public class EnglishGreeter implements Greeter {
    public String greet(String name) { return "Hello, " + name + "!"; }
}

// KoreanGreeter.java
public class KoreanGreeter implements Greeter {
    public String greet(String name) { return "안녕하세요, " + name + "님!"; }
}
```

```java
import java.io.File;
import java.net.*;

public class BridgePatternDemo {
    public static void main(String[] args) throws Exception {

        // 인터페이스는 Application ClassLoader가 이미 로드
        // 구현체만 별도 ClassLoader로 로드

        URL[] pluginUrlA = {new File("./plugins/english").toURI().toURL()};
        URL[] pluginUrlB = {new File("./plugins/korean").toURI().toURL()};

        // 부모 = Application ClassLoader → Greeter 인터페이스 공유
        ClassLoader appCL = ClassLoader.getSystemClassLoader();

        URLClassLoader loaderA = new URLClassLoader(pluginUrlA, appCL);
        URLClassLoader loaderB = new URLClassLoader(pluginUrlB, appCL);

        // Greeter 인터페이스로 캐스팅 가능 (공통 ClassLoader에 있으므로)
        Greeter english = (Greeter) loaderA.loadClass("EnglishGreeter")
                                           .getDeclaredConstructor()
                                           .newInstance();

        Greeter korean  = (Greeter) loaderB.loadClass("KoreanGreeter")
                                           .getDeclaredConstructor()
                                           .newInstance();

        System.out.println(english.greet("World"));   // Hello, World!
        System.out.println(korean.greet("세계"));      // 안녕하세요, 세계님!

        // 두 ClassLoader 격리 확인
        Class<?> engClass = english.getClass();
        Class<?> korClass = korean.getClass();
        System.out.println("구현체 ClassLoader 동일: " +
            (engClass.getClassLoader() == korClass.getClassLoader()));  // false
        System.out.println("인터페이스 ClassLoader 동일: " +
            (engClass.getInterfaces()[0].getClassLoader() ==
             korClass.getInterfaces()[0].getClassLoader()));  // true (Application CL)

        loaderA.close();
        loaderB.close();
    }
}
```

---

### 실험 3: static 상태 격리 확인

```java
// Counter.java - static 상태를 가진 클래스
public class Counter {
    private static int count = 0;
    public static void increment() { count++; }
    public static int getCount()   { return count; }
}
```

```java
public class StaticIsolationDemo {
    public static void main(String[] args) throws Exception {
        URL[] urls = {new File("./build/classes").toURI().toURL()};

        URLClassLoader loaderA = new URLClassLoader(urls, null);
        URLClassLoader loaderB = new URLClassLoader(urls, null);

        Class<?> counterA = loaderA.loadClass("Counter");
        Class<?> counterB = loaderB.loadClass("Counter");

        // A에서 10번 increment
        for (int i = 0; i < 10; i++) {
            counterA.getMethod("increment").invoke(null);
        }

        // B에서 3번 increment
        for (int i = 0; i < 3; i++) {
            counterB.getMethod("increment").invoke(null);
        }

        int countA = (int) counterA.getMethod("getCount").invoke(null);
        int countB = (int) counterB.getMethod("getCount").invoke(null);

        System.out.println("Counter A: " + countA);  // 10 (B의 변경 영향 없음)
        System.out.println("Counter B: " + countB);  // 3  (A의 변경 영향 없음)
        // → static 상태가 완전히 격리됨

        loaderA.close();
        loaderB.close();
    }
}
```

---

### 실험 4: Metaspace 중복 로딩 비용 측정

```bash
# 10개 ClassLoader가 같은 라이브러리를 각각 로드할 때 Metaspace 비용 측정
java -Xlog:class+load=info \
     -XX:+PrintGCDetails \
     -Xlog:gc*::time \
     IsolationCostDemo

# jstat으로 Metaspace 사용량 확인
jstat -gc <pid> 1000

# MC (Metaspace Capacity) 변화 추이로
# ClassLoader 수에 비례하는 Metaspace 증가 확인
```

---

## ⚡ 실무 임팩트

### Gradle / Maven의 ClassLoader 격리

```
빌드 도구의 ClassLoader 전략:

Gradle:
  BuildScript ClassLoader: build.gradle에서 사용하는 플러그인
  Project ClassLoader:     프로젝트 클래스
  Test ClassLoader:        테스트 클래스 (포크된 JVM)
  
  각 서브모듈은 독립적인 ClassLoader
  → 모듈 A의 의존성이 모듈 B에 영향 없음

Maven Surefire (테스트 실행):
  기본: 같은 JVM, 다른 ClassLoader → 테스트 격리
  forkCount > 0: 완전히 별도 JVM 프로세스 → 완벽한 격리
  
  격리가 중요한 경우 (static 상태, 포트 충돌 등):
  <configuration>
    <forkCount>1</forkCount>
    <reuseForks>false</reuseForks>
  </configuration>
```

### 마이크로서비스 전환 시 ClassLoader 격리 활용

```
Monolith → Modular Monolith → MSA 전환 중간 단계:

각 도메인을 별도 ClassLoader로 격리
  Order ClassLoader     → 주문 관련 클래스
  Payment ClassLoader   → 결제 관련 클래스
  Inventory ClassLoader → 재고 관련 클래스

  도메인 간 통신: 인터페이스 브리지 패턴
  → 실제 MSA로 분리하기 전에 의존성 파악
  → ClassLoader 경계가 곧 서비스 경계 후보

장점:
  한 JVM에서 실행 → 낮은 운영 복잡도
  격리 경계 명확 → MSA 전환 준비 가능
  ClassCastException이 나면 경계 위반 신호
```

### 격리 수준 결정 기준

```
완전 격리 (별도 JVM 프로세스):
  → 완전히 신뢰할 수 없는 코드
  → 메모리 leak이 허용되지 않는 경우
  → 크래시가 다른 서비스에 영향 주면 안 되는 경우

ClassLoader 격리 (같은 JVM):
  → 버전 충돌 해결이 필요한 플러그인
  → Hot Reload가 필요한 개발 환경
  → 서로 다른 설정이 필요한 모듈들

공유 ClassLoader:
  → 같은 라이브러리를 공유해도 충돌 없는 경우
  → 성능이 최우선인 경우
  → 설정이 동일한 모듈들
```

---

## 🚫 흔한 오해

### "ClassLoader 격리만으로 보안 경계를 만들 수 있다"

```
❌ 잘못된 이해:
  별도 ClassLoader로 플러그인을 로드하면 보안이 보장된다.

✅ 실제:
  ClassLoader 격리는 클래스 공간 분리이지 실행 권한 제한이 아님.
  
  격리된 ClassLoader의 코드도:
  - 리플렉션으로 다른 ClassLoader의 클래스에 접근 가능
  - 파일 시스템, 네트워크 자유롭게 접근
  - System.exit()으로 JVM 종료 가능
  
  진정한 보안 격리를 위해서:
  - Java 17+: Module System으로 접근 제한
  - 별도 JVM 프로세스 + OS 권한 제한
  - GraalVM Polyglot Sandbox
  - 컨테이너 격리 (Docker + seccomp)
```

### "격리된 ClassLoader 사이에서는 데이터를 주고받을 수 없다"

```
❌ 잘못된 이해:
  완전히 격리된 두 ClassLoader는 서로 소통이 불가능하다.

✅ 실제:
  직접 객체 전달은 ClassCastException이 발생하지만
  다음 방법으로 우회 가능:
  
  방법 1: 인터페이스 브리지 패턴
    공통 부모 ClassLoader에 인터페이스 정의
    → 타입 안전하게 소통
  
  방법 2: 직렬화/역직렬화
    JSON, 바이트 배열로 변환 후 전달
    → 타입 정보 소실, 성능 비용
  
  방법 3: 리플렉션
    타입 캐스팅 없이 메서드 호출
    → 타입 안전성 포기, 유지보수 어려움
  
  실무에서 권장: 방법 1 (인터페이스 브리지)
```

### "URLClassLoader의 URL 순서는 상관없다"

```java
// ❌ 잘못된 이해
URL[] urls = {
    new File("./lib-v2/jackson.jar").toURI().toURL(),
    new File("./lib-v1/jackson.jar").toURI().toURL()
};
URLClassLoader loader = new URLClassLoader(urls);
// "버전이 알아서 결정되겠지"

// ✅ 실제:
// URL 배열의 앞쪽이 먼저 탐색됨
// 위 코드는 항상 v2가 우선 사용됨
// URL 순서가 버전 우선순위를 결정

// 격리가 필요하다면 각 버전을 별도 URLClassLoader로 분리:
URLClassLoader v1Loader = new URLClassLoader(
    new URL[]{new File("./lib-v1/jackson.jar").toURI().toURL()}, null);
URLClassLoader v2Loader = new URLClassLoader(
    new URL[]{new File("./lib-v2/jackson.jar").toURI().toURL()}, null);
```

---

## 📌 핵심 정리

```
격리의 기반
  (ClassLoader 인스턴스 + FQCN) = 클래스 정체성
  다른 ClassLoader = 다른 클래스 공간 = 격리

격리 구조 패턴
  완전 격리 (Sibling):   공통 부모 + 독립 자식들
  계층 격리 (Hierarchy): 상위일수록 공유, 하위일수록 격리
  역전 격리 (Parent-Last): 자신을 먼저 탐색 (Tomcat 방식)

격리 간 통신
  직접 전달 → ClassCastException
  인터페이스 브리지 → 공통 부모에 인터페이스 정의 (권장)
  직렬화 → 타입 소실, 성능 비용

static 상태 격리
  같은 클래스라도 다른 ClassLoader → 별도 static 공간
  테스트 격리, 멀티테넌트 등에 활용

OSGi의 정교한 격리
  번들별 ClassLoader
  Import/Export-Package로 명시적 공유 제어
  같은 라이브러리 여러 버전 공존 가능

격리의 한계
  ClassLoader 격리 ≠ 보안 격리
  실행 권한 제한은 별도 메커니즘 필요 (Module System, 별도 프로세스)

격리 비용
  같은 라이브러리 중복 로드 → Metaspace 증가
  공유 가능한 것은 공통 부모에 두어 비용 절감
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 `ClassCastException`이 발생하는 이유를 ClassLoader 관점에서 설명하고, 수정 방법을 제시하라.

```java
URLClassLoader loaderA = new URLClassLoader(urls, null);
URLClassLoader loaderB = new URLClassLoader(urls, null);

Object instanceFromB = loaderB.loadClass("com.example.Service")
                               .getDeclaredConstructor().newInstance();

// 이 줄에서 ClassCastException
com.example.Service service = (com.example.Service) instanceFromB;
```

**Q2.** 하나의 JVM에서 같은 라이브러리의 v1과 v2를 동시에 사용하는 시스템을 설계하라. 어떤 클래스를 공유하고, 어떤 클래스를 격리하며, 두 영역 사이에서 데이터는 어떻게 교환하는가?

**Q3.** ClassLoader 격리로 테스트 간 `static` 상태 오염을 막으려 한다. JUnit 5에서 이를 구현하는 방법과, `@TestClassOrder`, `@Isolated` 등 기존 도구와 비교하여 ClassLoader 격리가 언제 더 적합한지 설명하라.

> 💡 **해설**
>
> **Q1.** `loaderA`와 `loaderB`의 부모가 `null`(Bootstrap)이므로 Application ClassLoader를 우회한다. `com.example.Service`는 Application ClassLoader가 아닌 각 URLClassLoader가 로드한다. 캐스팅 시 좌변의 `com.example.Service`는 Application ClassLoader가 로드한 것이고, 우변 인스턴스의 실제 클래스는 `loaderB`가 로드한 것이다. 서로 다른 ClassLoader → 다른 타입 → `ClassCastException`. 수정 방법: (1) 부모를 Application ClassLoader로 설정 — `new URLClassLoader(urls, ClassLoader.getSystemClassLoader())`. 그러면 `com.example.Service`를 Application ClassLoader가 로드하고, loaderA/B는 해당 클래스를 부모에서 찾아 같은 Class 객체 공유. (2) 인터페이스를 Application ClassLoader에 두고 구현체만 loaderB에서 로드. 인터페이스 타입으로 캐스팅.
>
> **Q2.** 설계: 공유 영역 — 두 버전 모두 지켜야 하는 공통 인터페이스를 정의한 `api` 모듈을 Application ClassLoader에 배치. 격리 영역 — v1 구현체는 `v1ClassLoader`, v2 구현체는 `v2ClassLoader`로 각각 로드. 두 로더의 부모는 Application ClassLoader(공통 인터페이스 공유). 데이터 교환 — 인터페이스 브리지 패턴: `ApiInterface v1Client = (ApiInterface) v1ClassLoader.loadClass("ImplV1").newInstance()`. 내부적으로 v1/v2 라이브러리를 쓰지만 외부에서는 같은 인터페이스로 접근. 복잡한 데이터는 공통 DTO 클래스(Application ClassLoader)를 통해 전달.
>
> **Q3.** JUnit의 `@Isolated`는 다른 테스트와 병렬 실행을 막는 것이지 ClassLoader를 분리하지 않는다. `static` 상태가 진짜 문제라면 ClassLoader 격리가 더 강력하다. 구현: JUnit 5 Extension(`BeforeEachCallback`)에서 테스트 클래스를 새 URLClassLoader로 로드하고, 리플렉션으로 테스트 메서드를 실행. `AfterEachCallback`에서 ClassLoader를 닫아 GC 대상으로 만든다. 이 방식이 적합한 경우: static 싱글톤 상태를 초기화할 수 없는 레거시 코드, 클래스 로딩 자체를 테스트하는 경우, 특정 라이브러리가 JVM당 한 번만 초기화되도록 설계된 경우. 단점: 각 테스트마다 클래스 재로딩 비용 발생, 테스트 속도 저하 가능.

---

## 📚 참고 자료

- [JVMS §5.3 — Creation and Loading](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html#jvms-5.3)
- [OSGi Alliance — Core Specification](https://docs.osgi.org/specification/osgi.core/)
- [Tomcat ClassLoader HOW-TO](https://tomcat.apache.org/tomcat-10.1-doc/class-loader-howto.html)
- [Java Module System — JEP 261](https://openjdk.org/jeps/261)

---

<div align="center">

**[⬅️ 이전: Custom ClassLoader](./06-custom-classloader.md)** | **[홈으로 🏠](../README.md)**

</div>

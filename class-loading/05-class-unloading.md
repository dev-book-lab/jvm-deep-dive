# Class Unloading - 클래스 언로딩

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 클래스는 언제 JVM 메모리에서 제거되는가? GC가 객체를 수거하듯이 자동으로 되는가?
- `Class` 객체와 ClassLoader의 관계가 클래스 언로딩에 어떤 영향을 주는가?
- Tomcat에서 웹앱을 Reload할 때 메모리가 줄지 않는 이유는 무엇인가?
- Metaspace는 클래스 언로딩 없이 어떻게 관리되는가?
- `jmap`, `jcmd`로 언로딩이 실제로 발생했는지 어떻게 확인하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 클래스도 메모리를 차지한다

```
일반적인 착각:
  GC는 더 이상 참조되지 않는 객체를 수거한다.
  클래스도 마찬가지 아닌가?

실제:
  클래스 메타데이터(메서드, 필드, 바이트코드...)는
  일반 힙이 아닌 Metaspace에 저장된다.
  일반 GC 알고리즘은 Metaspace에 직접 접근하지 않는다.
  클래스를 제거하려면 별도의 조건과 메커니즘이 필요하다.
```

클래스 언로딩이 중요한 두 가지 이유:

```
이유 1: 동적 클래스 생성 시스템
  리플렉션, 람다, 동적 프록시, 바이트코드 조작 도구들은
  런타임에 클래스를 생성한다.
  이 클래스들이 영구히 남으면 Metaspace 고갈.

이유 2: 플러그인/Hot Reload 시스템
  Tomcat 웹앱 Reload, OSGi 번들 교체, Spring DevTools 등은
  기존 클래스를 새 버전으로 교체해야 한다.
  언로딩이 없으면 구 버전 클래스가 계속 메모리에 남음.
```

---

## 📐 내부 구조

### 1. 클래스 언로딩 조건 — 3가지 모두 충족해야 한다

클래스가 언로딩되려면 다음 세 가지가 **동시에** 만족해야 한다.

```
조건 1: 해당 클래스의 인스턴스가 존재하지 않는다
  힙에 MyClass의 인스턴스가 하나도 없어야 함
  (일반 GC로 모든 인스턴스가 수거된 상태)

조건 2: 해당 Class 객체에 대한 참조가 없다
  Class<MyClass> clazz = MyClass.class;  ← 이 참조가 사라져야 함
  리플렉션으로 캐시된 Class 객체 참조도 포함

조건 3: 해당 클래스를 로드한 ClassLoader가 GC 대상이다
  ClassLoader 자체에 대한 강한 참조가 모두 사라져야 함
  ← 이것이 실제로 가장 어려운 조건
```

```
왜 조건 3이 핵심인가?

ClassLoader ←── 참조 ──── ClassLoader를 참조하는 곳
     │
     └─ 로드한 Class들 (참조 보유)
              │
              └─ 각 Class의 static 필드 → 인스턴스들

ClassLoader가 GC되어야 그것이 로드한 모든 클래스가
함께 Metaspace에서 제거될 수 있다.

반대로 말하면:
  ClassLoader에 대한 강한 참조가 하나라도 남아있으면
  그 ClassLoader가 로드한 수백 개의 클래스 전체가
  Metaspace에 남는다.
```

---

### 2. Bootstrap, Platform, App ClassLoader는 언로딩하지 않는다

```
Bootstrap ClassLoader  → JVM 종료까지 언로딩 없음
Platform ClassLoader   → JVM 종료까지 언로딩 없음
Application ClassLoader → JVM 종료까지 언로딩 없음

이유:
  이 세 ClassLoader는 JVM이 직접 참조를 보유
  JVM이 동작하는 한 GC 대상이 될 수 없음

결론:
  java.lang.String, java.util.ArrayList 등
  Bootstrap이 로드한 JDK 클래스는 절대 언로딩되지 않음
  
  내가 만든 클래스도 Application ClassLoader가 로드하면
  JVM이 종료될 때까지 Metaspace에 남음

언로딩이 가능한 클래스:
  사용자 정의 ClassLoader가 로드한 클래스에 한해서만 가능
  (Tomcat WebApp ClassLoader, OSGi Bundle ClassLoader 등)
```

---

### 3. 언로딩 과정

```
언로딩 트리거:
  Full GC 또는 Metaspace GC 발생 시
  JVM이 언로딩 가능한 ClassLoader를 탐색

탐색 기준:
  ClassLoader에 대한 모든 강한 참조 소멸 여부 확인
  (앞의 3가지 조건 체크)

언로딩 실행:
  1. Metaspace에서 해당 ClassLoader의 클래스 메타데이터 제거
  2. 해당 클래스들의 바이트코드, 메서드 데이터, vtable 제거
  3. ClassLoader 인스턴스 자체도 힙에서 GC

주의:
  언로딩은 항상 ClassLoader 단위로 일어남
  특정 클래스 하나만 선택적으로 언로딩 불가
  ClassLoader 전체가 함께 언로딩됨
```

---

### 4. Metaspace와 클래스 언로딩

```
Metaspace 구조 (Java 8+):

  Metaspace (네이티브 메모리, -XX:MaxMetaspaceSize 로 제한)
  ┌─────────────────────────────────────────────────┐
  │  ClassLoader A의 클래스들                          │
  │  ┌──────────┐ ┌──────────┐ ┌──────────┐         │
  │  │ Foo.class│ │ Bar.class│ │ Baz.class│ ...     │
  │  └──────────┘ └──────────┘ └──────────┘         │
  ├─────────────────────────────────────────────────┤
  │  ClassLoader B의 클래스들                          │
  │  ┌──────────┐ ┌──────────┐                      │
  │  │ Foo.class│ │ Qux.class│ ...                  │
  │  └──────────┘ └──────────┘                      │
  ├─────────────────────────────────────────────────┤
  │  Bootstrap ClassLoader의 클래스들 (언로딩 불가)       │
  │  String, Object, ArrayList ...                  │
  └─────────────────────────────────────────────────┘

언로딩 발생 시:
  ClassLoader A가 GC → 그 영역 전체 해제
  ClassLoader B는 유지
```

```
Metaspace 압박 시나리오:

정상:
  클래스 로드 → Metaspace 증가
  ClassLoader GC → Metaspace 감소 ↔ 반복

누수:
  ClassLoader가 GC되지 않음
  → Metaspace 계속 증가
  → -XX:MaxMetaspaceSize 도달
  → java.lang.OutOfMemoryError: Metaspace
  → JVM 크래시
```

---

### 5. 실제 언로딩이 잘 발생하지 않는 이유

```
함정 1: ThreadLocal에 클래스 인스턴스 보관

  // ❌ ThreadLocal이 ClassLoader 참조 체인을 유지
  ThreadLocal<MyService> tl = new ThreadLocal<>();
  tl.set(new MyService());  // MyService → MyClass → ClassLoader

  // 스레드가 종료되지 않으면 (Thread Pool!) ThreadLocal 엔트리 유지
  // → ClassLoader GC 불가

함정 2: static 필드에 다른 ClassLoader의 클래스 인스턴스 보관

  // ClassLoader A가 로드한 클래스의 static 필드에
  // ClassLoader B가 로드한 객체를 저장하면
  // → B의 ClassLoader → B의 Class → ClassLoader B 참조 체인
  // → ClassLoader B가 GC 되지 않음

함정 3: 리플렉션 캐시

  // 리플렉션 API는 내부적으로 Method, Field 등을 캐시
  // 이 캐시가 Class 객체 참조를 보유 → ClassLoader 참조 유지

함정 4: JDBC 드라이버 등록

  // DriverManager는 static 필드에 Driver 목록 유지
  // WebApp ClassLoader가 로드한 Driver가 여기 등록되면
  // → DriverManager(Bootstrap 공간)가 WebApp ClassLoader 참조
  // → WebApp Reload 시 구 ClassLoader 언로딩 불가
  // → 이것이 Tomcat에서 자주 발생하는 ClassLoader 누수
```

---

## 💻 실험으로 확인하기

### 실험 1: 클래스 언로딩 로그 활성화

```bash
# Java 11+: 클래스 로드/언로드 상세 로그
java -Xlog:class+unload=debug MyApp

# 출력 예시 (언로딩 발생 시):
# [0.843s][debug][class,unload] unloading class com.example.Plugin 0x00000007c0060828
# [0.843s][debug][class,unload] unloading class com.example.PluginImpl 0x00000007c0060a10

# Java 8:
java -verbose:class MyApp 2>&1 | grep -i unload
```

---

### 실험 2: ClassLoader GC 여부 확인

```java
import java.io.File;
import java.lang.ref.WeakReference;
import java.net.URL;
import java.net.URLClassLoader;

public class UnloadingDemo {
    public static void main(String[] args) throws Exception {
        WeakReference<ClassLoader> ref = loadAndRelease();

        System.out.println("GC 전 ClassLoader 생존: " + (ref.get() != null));

        // GC 유도 (보장은 아님, 실험용)
        for (int i = 0; i < 5; i++) {
            System.gc();
            Thread.sleep(100);
        }

        System.out.println("GC 후 ClassLoader 생존: " + (ref.get() != null));
        // → false 이면 언로딩 성공
    }

    static WeakReference<ClassLoader> loadAndRelease() throws Exception {
        URL[] urls = {new File("./build").toURI().toURL()};
        URLClassLoader loader = new URLClassLoader(urls, null);

        Class<?> clazz = loader.loadClass("SampleClass");
        // clazz, loader 모두 이 메서드 스코프에서만 존재
        // 반환 후 강한 참조 소멸

        loader.close();
        return new WeakReference<>(loader);
        // 메서드 종료 시 clazz, loader 지역 변수 소멸 → GC 대상
    }
}
```

---

### 실험 3: ThreadLocal로 인한 언로딩 방해 확인

```java
import java.lang.ref.WeakReference;
import java.net.URLClassLoader;

public class ThreadLocalLeakDemo {

    static ThreadLocal<Object> threadLocal = new ThreadLocal<>();

    public static void main(String[] args) throws Exception {

        WeakReference<ClassLoader> withLeak    = simulateWithLeak();
        WeakReference<ClassLoader> withoutLeak = simulateWithoutLeak();

        // GC 유도
        for (int i = 0; i < 5; i++) {
            System.gc();
            Thread.sleep(100);
        }

        System.out.println("ThreadLocal 누수 있음 - 생존: " + (withLeak.get() != null));    // true (누수)
        System.out.println("ThreadLocal 누수 없음 - 생존: " + (withoutLeak.get() != null)); // false (정상)
    }

    static WeakReference<ClassLoader> simulateWithLeak() throws Exception {
        URLClassLoader loader = new URLClassLoader(new java.net.URL[]{}, null);
        Class<?> clazz = loader.loadClass("java.lang.Object"); // 예시용

        // ❌ ThreadLocal에 로더가 로드한 클래스의 인스턴스 저장
        threadLocal.set(clazz.getDeclaredConstructor().newInstance());
        // threadLocal → instance → clazz → loader 참조 체인 유지

        return new WeakReference<>(loader);
    }

    static WeakReference<ClassLoader> simulateWithoutLeak() throws Exception {
        URLClassLoader loader = new URLClassLoader(new java.net.URL[]{}, null);
        // ThreadLocal 사용 안 함 → 참조 체인 없음
        return new WeakReference<>(loader);
    }
}
```

---

### 실험 4: Metaspace 사용량 모니터링

```bash
# 실행 중인 JVM의 Metaspace 현황
jcmd <pid> VM.metaspace

# 출력:
# Total: reserved=1065984KB, committed=65536KB
# Class space: reserved=1048576KB, committed=6144KB
# Non-class space: reserved=17408KB, committed=59392KB

# jstat으로 Metaspace 변화 추이 (1초 간격)
jstat -gc <pid> 1000

# 출력 컬럼 중 Metaspace 관련:
# MC(Metaspace Capacity), MU(Metaspace Used)
# CCSC(Compressed Class Space Capacity), CCSU(Used)

# Heap Dump로 ClassLoader별 점유 확인
jmap -dump:format=b,file=/tmp/heap.hprof <pid>
# → Eclipse MAT에서 "ClassLoader" 검색 후 분석
```

---

## ⚡ 실무 임팩트

### Tomcat 웹앱 Reload 시 메모리 누수 방지

```
Tomcat Reload 시 발생하는 전형적인 누수:

1. JDBC 드라이버 등록 누수
   웹앱이 DriverManager.registerDriver()를 호출
   → DriverManager(Bootstrap)가 WebApp ClassLoader의 Driver를 참조
   → Reload 후 구 ClassLoader가 GC되지 않음
   
   해결: web.xml에 ServletContextListener 등록
   contextDestroyed()에서 DriverManager.deregisterDriver() 호출

2. 로깅 라이브러리 누수 (Log4j, Logback)
   로깅 프레임워크가 내부적으로 ClassLoader 참조 보유
   
   해결: Tomcat이 제공하는 logging library 사용
   또는 contextDestroyed()에서 LogManager.shutdown()

3. ThreadLocal 누수
   Thread Pool의 스레드는 재사용됨
   Reload 후에도 ThreadLocal에 구 ClassLoader의 객체 잔존
   
   해결: contextDestroyed()에서 ThreadLocal.remove() 호출
   또는 InheritableThreadLocal 대신 명시적 컨텍스트 관리

진단 도구:
  Tomcat의 memory leak detection 기능
  JVM 옵션: -Xlog:class+unload=debug
  Eclipse MAT의 "Find Leaking ClassLoaders" 기능
```

### OSGi 번들 교체 시 안전한 언로딩

```
OSGi는 번들(Bundle)마다 독립적인 ClassLoader를 사용.
번들 업데이트 시 구 ClassLoader가 언로딩되어야 메모리가 확보됨.

안전한 언로딩 체크리스트:
  □ 번들이 등록한 서비스를 ServiceRegistry에서 해제
  □ 다른 번들이 이 번들의 클래스를 참조하지 않는지 확인
  □ 번들이 등록한 EventListener 해제
  □ 번들이 시작한 스레드 종료
  □ 번들의 ThreadLocal 정리
  □ 공유 메모리(static) 참조 정리
```

### GraalVM Native Image에서 클래스 언로딩

```
Native Image는 AOT(Ahead-of-Time) 컴파일을 사용.
빌드 타임에 closed-world assumption:
  "런타임에 새로운 클래스가 로드되지 않는다"

→ 클래스 언로딩 개념 자체가 없음
→ Metaspace도 없음
→ 클래스 메타데이터는 바이너리에 정적으로 포함됨

동적 클래스 로딩이 필요한 경우:
  --initialize-at-run-time 옵션으로 특정 클래스만 런타임 초기화
  Reflection 사용 시 reflect-config.json에 명시적 등록 필요
```

---

## 🚫 흔한 오해

### "System.gc() 호출하면 클래스가 언로딩된다"

```
❌ 잘못된 이해:
  System.gc()를 호출하면 사용하지 않는 클래스가 바로 언로딩된다.

✅ 실제:
  System.gc()는 GC를 요청하는 것이지 보장이 아님 (힌트)
  JVM이 무시할 수 있음 (-XX:+DisableExplicitGC 옵션 존재)
  
  클래스 언로딩은 Full GC 또는 Metaspace GC 압박 시에만 발생
  그리고 언로딩 3가지 조건이 충족되어야 함
  
  System.gc()만으로 ClassLoader 언로딩이 보장되지 않음
  실험에서 사용하는 것과 프로덕션에서 기대하는 것은 다름
```

### "모든 클래스는 GC 대상이다"

```
❌ 잘못된 이해:
  더 이상 쓰이지 않는 클래스는 GC가 자동으로 수거한다.

✅ 실제:
  Bootstrap / Platform / Application ClassLoader가 로드한 클래스는
  JVM 종료까지 절대 언로딩되지 않음.
  
  내가 작성한 비즈니스 클래스 전부가 이 범주에 포함.
  Application ClassLoader는 JVM이 직접 참조를 보유하기 때문.
  
  언로딩 가능한 클래스:
  → 사용자 정의 ClassLoader (URLClassLoader 등)가 로드한 클래스만
  → 플러그인 시스템, 동적 코드 생성, Hot Reload 환경
```

### "ClassLoader를 null로 설정하면 바로 언로딩된다"

```java
// ❌ 잘못된 이해
URLClassLoader loader = new URLClassLoader(urls);
Class<?> clazz = loader.loadClass("Plugin");
Object instance = clazz.newInstance();

loader = null;  // 이것만으로는 언로딩 안 됨!

// ✅ 실제로 필요한 것:
// 1. loader = null
// 2. clazz = null   ← Class 객체 참조도 제거
// 3. instance = null ← 인스턴스 참조도 제거
// 4. 위 참조를 가진 모든 다른 변수/필드도 null 처리
// 5. ThreadLocal 정리
// 6. GC 발생 (언제 발생할지는 JVM이 결정)
// 7. 그제서야 언로딩 가능 상태
```

---

## 📌 핵심 정리

```
클래스 언로딩 3가지 조건 (모두 충족 필요)
  1. 해당 클래스의 인스턴스가 모두 GC됨
  2. Class 객체에 대한 참조가 없음
  3. 해당 클래스를 로드한 ClassLoader가 GC 대상임

언로딩 불가능한 클래스
  Bootstrap / Platform / Application ClassLoader 로드 클래스
  → 내가 작성한 비즈니스 클래스 전부 포함
  → JVM 종료 시까지 Metaspace에 존재

언로딩 가능한 조건
  사용자 정의 ClassLoader (URLClassLoader 등)가 로드한 경우에만
  → 플러그인 시스템, 동적 코드 생성, Tomcat 웹앱 등

ClassLoader 누수의 주범
  ThreadLocal에 보관된 인스턴스
  static 필드에 보관된 다른 ClassLoader의 인스턴스
  리플렉션 캐시
  DriverManager 등 프레임워크 레벨 정적 등록

Metaspace OOM 진단 순서
  jstat -gc 로 Metaspace 증가 추이 확인
  jcmd VM.metaspace 로 상세 현황 파악
  heap dump → MAT로 "누수 ClassLoader" 탐색
  Path to GC Roots로 참조 체인 추적

언로딩 발생 시점
  Full GC 또는 Metaspace 압박 시 JVM이 탐색
  System.gc()는 힌트일 뿐, 즉시 언로딩 보장 안 함
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 `loader`가 GC되어 클래스 언로딩이 발생하려면 무엇을 추가로 정리해야 하는가? 각 항목이 왜 필요한지 참조 체인으로 설명하라.

```java
URLClassLoader loader = new URLClassLoader(urls);
Class<?> clazz = loader.loadClass("Plugin");
Object instance = clazz.newInstance();
cache.put("plugin", instance);  // static ConcurrentHashMap cache
threadLocal.set(instance);      // static ThreadLocal threadLocal
loader.close();
loader = null;
```

**Q2.** Tomcat 웹앱을 100번 Reload했더니 Metaspace 사용량이 계속 증가한다. 구체적인 원인 3가지와 각각의 해결책을 설명하라.

**Q3.** 플러그인 시스템을 설계한다. 플러그인을 교체할 때 구 버전 클래스가 완전히 언로딩되도록 보장하는 설계 원칙을 제시하고, 이를 검증하는 방법을 설명하라.

> 💡 **해설**
>
> **Q1.** 다음 세 가지가 모두 필요하다. ① `cache.remove("plugin")` 또는 `cache = null` — `static ConcurrentHashMap`은 Bootstrap ClassLoader의 Class에 속하므로 GC되지 않는다. cache가 instance를 참조하면 `instance → clazz → loader` 체인이 유지된다. ② `threadLocal.remove()` — Thread Pool의 스레드는 재사용되므로 ThreadLocal 엔트리가 살아있는 한 `instance → clazz → loader` 참조가 끊어지지 않는다. ③ `clazz = null`, `instance = null` — 지역 변수라면 스코프 종료 시 자동 소멸이지만, 필드라면 명시적으로 null 처리 필요. 이 모든 것이 정리된 후 GC가 발생해야 loader가 수거되고 언로딩이 가능해진다.
>
> **Q2.** ① JDBC 드라이버 누수: 웹앱의 ClassLoader가 로드한 Driver가 `DriverManager`(Bootstrap 영역 static)에 등록되어 참조 체인 유지. 해결: `contextDestroyed()`에서 `Enumeration<Driver> drivers = DriverManager.getDrivers(); while(drivers.hasMoreElements()) DriverManager.deregisterDriver(drivers.nextElement());` 실행. ② ThreadLocal 누수: Tomcat의 Worker Thread가 Thread Pool로 재사용되어 이전 웹앱의 ThreadLocal 엔트리가 남아있음. 해결: `contextDestroyed()`에서 애플리케이션이 사용한 모든 ThreadLocal에 `remove()` 호출. ③ 로깅 프레임워크 누수: Log4j/Logback이 내부 스레드나 static 참조로 ClassLoader를 보유. 해결: `contextDestroyed()`에서 `LogManager.shutdown()` 호출, 또는 Tomcat의 JUL(Java Util Logging)만 사용.
>
> **Q3.** 설계 원칙: ① 플러그인 인터페이스는 반드시 부모 ClassLoader(공유 영역)에 배치, 구현체만 플러그인 ClassLoader에 배치. ② 플러그인 레지스트리는 WeakReference로 플러그인 인스턴스 보관. ③ 플러그인 교체 시 `unload()` 메서드 호출 → 이벤트 리스너 해제, ThreadLocal 정리, 캐시 제거 순서로 진행. ④ 플러그인 ClassLoader를 `close()` 후 null 처리. 검증 방법: `WeakReference<ClassLoader>`로 교체 후 GC 발생 여부 확인 (앞의 실험 2 코드 활용). CI에서 플러그인 교체 후 `jstat -gc`로 Metaspace가 감소하는지 자동화 테스트. `-Xlog:class+unload=debug`로 실제 언로딩 로그 확인.

---

## 📚 참고 자료

- [JVMS §5.3 — Creation and Loading](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html#jvms-5.3)
- [JDK-6316710 — Class unloading and GC](https://bugs.openjdk.org/browse/JDK-6316710)
- [Tomcat ClassLoader Documentation](https://tomcat.apache.org/tomcat-10.1-doc/class-loader-howto.html)
- [Eclipse MAT — Find Leaking ClassLoaders](https://help.eclipse.org/latest/index.jsp?topic=/org.eclipse.mat.ui.help/reference/findingmemoryleak.html)

---

<div align="center">

**[⬅️ 이전: Symbolic Reference Resolution](./04-symbolic-reference-resolution.md)** | **[다음: Custom ClassLoader ➡️](./06-custom-classloader.md)**

</div>

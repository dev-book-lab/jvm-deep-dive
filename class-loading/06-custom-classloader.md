# Custom ClassLoader - 커스텀 클래스로더

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `findClass()`와 `loadClass()`를 오버라이드하는 것은 어떻게 다른가?
- 암호화된 `.class` 파일을 런타임에 복호화해 로드하려면 어떻게 구현하는가?
- 네트워크에서 클래스를 다운로드해 실행하는 시스템은 어떻게 만드는가?
- Hot Reload(재시작 없이 클래스 교체)는 내부에서 어떻게 동작하는가?
- 커스텀 ClassLoader를 만들 때 반드시 지켜야 할 제약은 무엇인가?

---

## 🔍 왜 이게 존재하는가

### 문제: 기본 ClassLoader는 파일시스템의 .class 파일만 로드한다

Application ClassLoader는 `-classpath`에 지정된 경로의 `.class` 파일과 `.jar`만 로드할 수 있다. 다음과 같은 요구사항은 기본 ClassLoader로 처리할 수 없다.

```
기본 ClassLoader로 불가능한 것들:

① 암호화된 .class 파일 로드
   보안 목적으로 바이트코드를 암호화 배포
   → 로드 시점에 복호화 필요

② 네트워크에서 클래스 다운로드
   원격 서버의 jar에서 클래스를 가져와 실행
   → URL이나 HTTP로 바이트 배열 획득 후 로드

③ 재시작 없이 클래스 교체 (Hot Reload)
   파일이 변경되면 새 버전의 클래스를 로드
   → 새 ClassLoader 인스턴스로 재로드

④ 클래스별 접근 제어
   특정 클래스는 sandbox 환경에서만 실행
   → 별도 ClassLoader로 격리 후 제한된 권한 부여

⑤ 데이터베이스나 다른 저장소에서 클래스 로드
   DB에 저장된 바이트코드를 꺼내 실행
```

커스텀 ClassLoader는 이 모든 시나리오의 공통 해결책이다. 클래스 바이트를 어떻게 구하느냐만 바꾸면 된다.

---

## 📐 내부 구조

### 1. 오버라이드할 메서드 선택 — findClass vs loadClass

가장 먼저 결정해야 하는 것이 어느 메서드를 오버라이드하느냐다.

```
ClassLoader의 핵심 메서드 계층:

loadClass(name)                ← Parent Delegation 전체 흐름 담당
    │
    ├── findLoadedClass(name)  ← 캐시 확인
    ├── parent.loadClass(name) ← 부모에게 위임
    └── findClass(name)        ← 직접 탐색 (여기만 오버라이드 권장)
            │
            └── defineClass(name, bytes, offset, length)  ← 실제 클래스 생성
```

```
findClass() 오버라이드 (권장):
  Parent Delegation 유지
  부모가 못 찾을 때만 내 탐색 로직 실행
  java.lang.* 등 핵심 클래스는 항상 Bootstrap이 로드
  → 보안 유지

loadClass() 오버라이드 (신중하게):
  Parent Delegation 자체를 변경
  전체 로딩 흐름을 직접 제어
  잘못 구현하면 java.lang.Object도 내가 로드하려 해서 JVM 붕괴
  Tomcat WebApp ClassLoader처럼 명확한 이유가 있을 때만
```

---

### 2. 커스텀 ClassLoader 기본 구조

```java
public class MyClassLoader extends ClassLoader {

    public MyClassLoader(ClassLoader parent) {
        super(parent);  // 부모 ClassLoader 명시적 설정
    }

    // ★ findClass만 오버라이드 → Parent Delegation 유지
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classBytes = loadClassBytes(name);  // 커스텀 로딩 로직

        if (classBytes == null) {
            throw new ClassNotFoundException(name);
        }

        // bytes → Class 객체 생성
        return defineClass(name, classBytes, 0, classBytes.length);
    }

    // 이 메서드만 구현에 따라 달라짐
    protected byte[] loadClassBytes(String name) throws ClassNotFoundException {
        // 구현 1: 파일에서 읽기
        // 구현 2: 복호화 후 반환
        // 구현 3: 네트워크에서 다운로드
        // 구현 4: DB에서 조회
        return null;
    }
}
```

---

### 3. 구현 패턴 1 — 암호화된 클래스 로더

```java
import javax.crypto.Cipher;
import javax.crypto.spec.SecretKeySpec;
import java.io.*;
import java.nio.file.*;
import java.security.Key;

public class EncryptedClassLoader extends ClassLoader {

    private final Path classDir;
    private final Key decryptKey;

    public EncryptedClassLoader(Path classDir, byte[] keyBytes, ClassLoader parent) {
        super(parent);
        this.classDir = classDir;
        this.decryptKey = new SecretKeySpec(keyBytes, "AES");
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        // 패키지명 → 파일 경로 변환
        String filePath = name.replace('.', File.separatorChar) + ".enc";
        Path classFile = classDir.resolve(filePath);

        if (!Files.exists(classFile)) {
            throw new ClassNotFoundException(name);
        }

        try {
            byte[] encrypted = Files.readAllBytes(classFile);
            byte[] decrypted = decrypt(encrypted);
            return defineClass(name, decrypted, 0, decrypted.length);

        } catch (Exception e) {
            throw new ClassNotFoundException("복호화 실패: " + name, e);
        }
    }

    private byte[] decrypt(byte[] data) throws Exception {
        Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
        cipher.init(Cipher.DECRYPT_MODE, decryptKey);
        return cipher.doFinal(data);
    }
}

// 클래스 파일 암호화 유틸리티 (배포 전 실행)
class ClassEncryptor {
    public static void encrypt(Path inputDir, Path outputDir, byte[] keyBytes) throws Exception {
        Key key = new SecretKeySpec(keyBytes, "AES");
        Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
        cipher.init(Cipher.ENCRYPT_MODE, key);

        Files.walk(inputDir)
             .filter(p -> p.toString().endsWith(".class"))
             .forEach(p -> {
                 try {
                     byte[] classBytes = Files.readAllBytes(p);
                     byte[] encrypted  = cipher.doFinal(classBytes);

                     Path relative = inputDir.relativize(p);
                     Path outPath  = outputDir.resolve(
                         relative.toString().replace(".class", ".enc"));

                     Files.createDirectories(outPath.getParent());
                     Files.write(outPath, encrypted);
                 } catch (Exception e) {
                     throw new RuntimeException(e);
                 }
             });
    }
}
```

사용:

```java
byte[] key = "0123456789abcdef".getBytes(); // 16바이트 AES 키

// 배포 전: 클래스 암호화
ClassEncryptor.encrypt(
    Path.of("./build/classes"),
    Path.of("./dist/encrypted"),
    key
);

// 런타임: 암호화된 클래스 로드
EncryptedClassLoader loader = new EncryptedClassLoader(
    Path.of("./dist/encrypted"), key,
    ClassLoader.getSystemClassLoader()
);

Class<?> clazz = loader.loadClass("com.example.SecretService");
Object instance = clazz.getDeclaredConstructor().newInstance();
```

---

### 4. 구현 패턴 2 — Hot Reload ClassLoader

```java
import java.io.*;
import java.nio.file.*;
import java.nio.file.attribute.FileTime;
import java.util.HashMap;
import java.util.Map;

public class HotReloadClassLoader extends ClassLoader {

    private final Path classDir;
    // 클래스명 → 마지막 로드 시점의 파일 수정 시간
    private final Map<String, FileTime> loadedAt = new HashMap<>();

    public HotReloadClassLoader(Path classDir, ClassLoader parent) {
        super(parent);
        this.classDir = classDir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        Path classFile = getClassFile(name);

        if (!Files.exists(classFile)) {
            throw new ClassNotFoundException(name);
        }

        try {
            byte[] bytes = Files.readAllBytes(classFile);
            loadedAt.put(name, Files.getLastModifiedTime(classFile));
            return defineClass(name, bytes, 0, bytes.length);

        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }

    // 파일이 변경됐는지 확인
    public boolean isModified(String name) {
        Path classFile = getClassFile(name);
        try {
            if (!Files.exists(classFile)) return false;
            FileTime lastModified = Files.getLastModifiedTime(classFile);
            FileTime loadTime     = loadedAt.get(name);
            return loadTime == null || lastModified.compareTo(loadTime) > 0;
        } catch (IOException e) {
            return false;
        }
    }

    private Path getClassFile(String name) {
        return classDir.resolve(name.replace('.', '/') + ".class");
    }
}

// Hot Reload 관리자
public class HotReloadManager {

    private final Path classDir;
    private final ClassLoader parent;
    private HotReloadClassLoader currentLoader;

    public HotReloadManager(Path classDir) {
        this.classDir = classDir;
        this.parent   = ClassLoader.getSystemClassLoader();
        this.currentLoader = new HotReloadClassLoader(classDir, parent);
    }

    // 변경된 클래스가 있으면 새 ClassLoader로 교체
    public Class<?> getClass(String name) throws ClassNotFoundException {

        if (currentLoader.isModified(name)) {
            System.out.println("[HotReload] " + name + " 변경 감지 → 재로드");

            // ★ 핵심: 새 ClassLoader 인스턴스 생성
            // 기존 ClassLoader의 캐시와 무관하게 새로 로드
            currentLoader = new HotReloadClassLoader(classDir, parent);
        }

        return currentLoader.loadClass(name);
    }
}
```

```
Hot Reload 원리:

기존 ClassLoader는 findLoadedClass() 캐시를 가짐
한 번 로드한 클래스는 같은 ClassLoader에서 다시 로드 불가

해결책:
  새 ClassLoader 인스턴스 생성 → 캐시 없음 → 파일을 새로 읽음
  구 ClassLoader는 참조가 끊어지면 GC 대상

주의사항:
  구 ClassLoader의 클래스로 만든 인스턴스는
  새 ClassLoader의 클래스와 호환되지 않음
  (같은 이름이지만 다른 Class 객체 → ClassCastException)
  
  인터페이스를 부모 ClassLoader에 두고
  구현체만 재로드하는 방식으로 해결
```

---

### 5. 구현 패턴 3 — 네트워크 ClassLoader

```java
import java.io.*;
import java.net.*;

public class NetworkClassLoader extends ClassLoader {

    private final String baseUrl;

    public NetworkClassLoader(String baseUrl, ClassLoader parent) {
        super(parent);
        // 예: "http://plugin-server.example.com/classes/"
        this.baseUrl = baseUrl.endsWith("/") ? baseUrl : baseUrl + "/";
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classBytes = downloadClass(name);
        return defineClass(name, classBytes, 0, classBytes.length);
    }

    private byte[] downloadClass(String name) throws ClassNotFoundException {
        String url = baseUrl + name.replace('.', '/') + ".class";

        try (InputStream in = new URL(url).openStream();
             ByteArrayOutputStream out = new ByteArrayOutputStream()) {

            byte[] buf = new byte[4096];
            int read;
            while ((read = in.read(buf)) != -1) {
                out.write(buf, 0, read);
            }
            return out.toByteArray();

        } catch (IOException e) {
            throw new ClassNotFoundException(
                "네트워크에서 클래스를 가져올 수 없음: " + name, e);
        }
    }
}
```

---

### 6. defineClass의 내부 동작

```
defineClass(name, bytes, offset, length) 호출 시:

1. 바이트 배열 유효성 확인
   bytes[offset]부터 length 바이트가 유효한지

2. Bytecode Verification (Pass 1, 2, 3)
   03-bytecode-verification에서 다룬 검증 수행
   실패 시 ClassFormatError 또는 VerifyError

3. Class 객체 생성
   Metaspace에 클래스 메타데이터 적재
   java.lang.Class 인스턴스 생성

4. ClassLoader 연결
   생성된 Class 객체를 이 ClassLoader에 연결
   clazz.getClassLoader() == this 가 됨

5. 캐시 등록
   이후 같은 이름으로 findLoadedClass() 호출 시 반환

주의: defineClass는 protected final
  하위 클래스에서 오버라이드 불가
  직접 바이트를 제공하고 JVM이 나머지를 처리
```

---

## 💻 실험으로 확인하기

### 실험 1: 가장 단순한 커스텀 ClassLoader

```java
import java.io.*;
import java.nio.file.*;

public class SimpleFileClassLoader extends ClassLoader {

    private final Path dir;

    public SimpleFileClassLoader(Path dir) {
        // 부모를 null로 → Bootstrap만 부모
        // Application ClassLoader를 우회해 완전히 독립적으로 동작
        super(null);
        this.dir = dir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        Path file = dir.resolve(name.replace('.', '/') + ".class");

        if (!Files.exists(file)) {
            throw new ClassNotFoundException(name);
        }

        try {
            byte[] bytes = Files.readAllBytes(file);
            System.out.println("[SimpleFileClassLoader] 로드: " + name
                + " (" + bytes.length + " bytes)");
            return defineClass(name, bytes, 0, bytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
}

// 테스트
public class CustomClassLoaderTest {
    public static void main(String[] args) throws Exception {
        SimpleFileClassLoader loader = new SimpleFileClassLoader(
            Path.of("./build/classes")
        );

        // 로드
        Class<?> clazz = loader.loadClass("com.example.Greeter");
        System.out.println("로드 완료: " + clazz);
        System.out.println("ClassLoader: " + clazz.getClassLoader());

        // 인스턴스 생성 및 메서드 호출 (리플렉션 사용)
        Object instance = clazz.getDeclaredConstructor().newInstance();
        clazz.getMethod("greet", String.class).invoke(instance, "JVM");
    }
}
```

실행:

```bash
# Greeter.class를 build/classes 디렉토리에 컴파일
javac -d ./build/classes Greeter.java

# 테스트 실행
javac CustomClassLoaderTest.java SimpleFileClassLoader.java
java CustomClassLoaderTest

# 출력:
# [SimpleFileClassLoader] 로드: com.example.Greeter (342 bytes)
# 로드 완료: class com.example.Greeter
# ClassLoader: SimpleFileClassLoader@7852e922
# Hello, JVM!
```

---

### 실험 2: Hot Reload 동작 확인

```java
public class HotReloadTest {
    public static void main(String[] args) throws Exception {
        Path classDir = Path.of("./build/classes");
        HotReloadManager manager = new HotReloadManager(classDir);

        // 1차 로드
        Class<?> v1 = manager.getClass("com.example.Plugin");
        Object instance1 = v1.getDeclaredConstructor().newInstance();
        v1.getMethod("run").invoke(instance1);

        System.out.println("파일을 수정한 후 Enter를 누르세요...");
        System.in.read();  // 이 시점에 Plugin.java를 수정하고 재컴파일

        // 2차 로드 (파일 변경 감지 → 새 ClassLoader로 재로드)
        Class<?> v2 = manager.getClass("com.example.Plugin");
        Object instance2 = v2.getDeclaredConstructor().newInstance();
        v2.getMethod("run").invoke(instance2);

        // 두 Class 객체는 다른 ClassLoader에서 로드됨
        System.out.println("같은 Class 객체인가? " + (v1 == v2));  // false
    }
}
```

---

### 실험 3: 부모 설정에 따른 동작 차이

```java
public class ParentSettingTest {
    public static void main(String[] args) throws Exception {

        // Case 1: 부모 = SystemClassLoader (권장)
        ClassLoader withParent = new SimpleFileClassLoader(
            Path.of("./build"),
            ClassLoader.getSystemClassLoader()  // 부모 명시
        ) {};

        // Case 2: 부모 = null (Bootstrap만)
        ClassLoader withoutParent = new SimpleFileClassLoader(
            Path.of("./build"),
            null  // Bootstrap만 부모
        ) {};

        // Case 1: classpath의 클래스도 로드 가능
        Class<?> c1 = withParent.loadClass("java.util.ArrayList");    // OK: Bootstrap
        Class<?> c2 = withParent.loadClass("com.example.MyService");  // OK: 부모(App) 또는 내가 로드

        // Case 2: classpath 클래스 로드 불가 (Bootstrap에 없으면)
        Class<?> c3 = withoutParent.loadClass("java.lang.String");    // OK: Bootstrap
        try {
            Class<?> c4 = withoutParent.loadClass("com.example.MyService"); // App ClassLoader 없음
        } catch (ClassNotFoundException e) {
            System.out.println("부모 없으면 classpath 클래스 접근 불가: " + e.getMessage());
        }
    }
}
```

---

## ⚡ 실무 임팩트

### Spring Boot DevTools의 Restart ClassLoader

```
Spring Boot DevTools 내부 구조:

Application 시작:
  Base ClassLoader (parent: SystemClassLoader)
    → 3rd party 라이브러리 클래스 로드 (변경 없음)
  Restart ClassLoader (parent: Base ClassLoader)
    → 개발자 코드 클래스 로드 (변경 가능)

파일 변경 감지 시:
  1. 기존 Restart ClassLoader 폐기 (참조 제거)
  2. 새 Restart ClassLoader 생성
  3. 새 ClassLoader로 변경된 클래스만 재로드
  4. Spring Context 재시작

장점:
  Base ClassLoader는 유지 → 3rd party 라이브러리 재로드 없음
  Restart ClassLoader만 교체 → 수 초 내 완료 (전체 재시작 수십 초 대비)
```

### 커스텀 ClassLoader 설계 체크리스트

```
□ findClass()를 오버라이드하는가? (loadClass() 오버라이드는 신중하게)
□ 부모 ClassLoader를 명시적으로 설정했는가?
□ defineClass() 전에 바이트 배열 유효성을 확인하는가?
□ ClassLoader 자체에 대한 누수 가능성을 검토했는가?
  (static 필드, ThreadLocal, 프레임워크 레지스트리 등)
□ 병렬 로딩이 필요한가? (ClassLoader.registerAsParallelCapable())
□ close() 메서드를 구현해 리소스를 반납하는가?
  (URLClassLoader처럼 Closeable 구현 고려)
□ Hot Reload 시 인터페이스/추상 클래스는 부모 ClassLoader에 두었는가?
```

### 병렬 클래스 로딩 최적화

```java
// 멀티스레드 환경에서 동시 클래스 로딩 성능 향상
public class ParallelClassLoader extends ClassLoader {

    static {
        // JVM에게 이 ClassLoader가 병렬 로딩을 지원함을 알림
        // 클래스별 잠금 사용 → 동시에 다른 클래스 로드 가능
        registerAsParallelCapable();
    }

    // registerAsParallelCapable() 없이는:
    //   ClassLoader 인스턴스 전체에 잠금
    //   스레드 A가 FooClass 로드 중 → 스레드 B의 BarClass 로드 대기

    // registerAsParallelCapable() 이후:
    //   클래스명 단위 잠금
    //   FooClass와 BarClass를 동시에 로드 가능
}
```

---

## 🚫 흔한 오해

### "loadClass()를 오버라이드해야 커스텀 로딩이 된다"

```
❌ 잘못된 이해:
  클래스 로딩을 커스터마이징하려면 loadClass()를 오버라이드해야 한다.

✅ 실제:
  findClass()만 오버라이드하면 충분한 경우가 대부분.
  
  findClass()는 loadClass()의 마지막 단계에서 호출됨.
  Parent Delegation(캐시 확인 + 부모 위임)은 유지한 채
  "내가 직접 찾는" 부분만 커스터마이징 가능.
  
  loadClass() 오버라이드:
  - Parent Delegation 자체를 바꿔야 할 때만
  - Tomcat처럼 부모보다 자신을 먼저 탐색해야 할 때
  - 잘못 구현하면 java.lang.* 로딩까지 방해 → JVM 붕괴
```

### "defineClass()를 호출하면 클래스가 즉시 초기화된다"

```
❌ 잘못된 이해:
  defineClass() 호출 시 static 블록이 실행된다.

✅ 실제:
  defineClass() → Loading + Linking 까지만 (Verification 포함)
  static 블록(Initializing)은 클래스를 처음 사용할 때 실행
  
  Class<?> clazz = defineClass(...);  // static 블록 실행 안 됨
  Object instance = clazz.newInstance();  // 이 시점에 static 블록 실행
  
  02-loading-linking-initializing에서 다룬
  6가지 active use 조건이 여기서도 동일하게 적용됨
```

### "커스텀 ClassLoader는 Parent Delegation 없이 동작해야 한다"

```
❌ 잘못된 이해:
  커스텀 ClassLoader는 직접 모든 클래스를 로드한다.

✅ 실제:
  findClass()만 오버라이드하면 Parent Delegation은 그대로 유지됨.
  java.lang.String, java.util.List 같은 클래스는
  여전히 Bootstrap이 로드.
  
  내가 만든 클래스(com.example.*)만 내 로직으로 로드됨.
  이것이 "java.lang.Object를 두 번 로드하는" 사태를 막는 방법.
```

---

## 📌 핵심 정리

```
오버라이드 선택
  findClass()  → Parent Delegation 유지, 직접 탐색만 커스터마이징 (권장)
  loadClass()  → 전체 로딩 흐름 제어, 잘못 구현 시 JVM 불안정

구현 뼈대
  super(parent) 로 부모 설정
  findClass()에서 bytes 획득
  defineClass(name, bytes, 0, bytes.length) 로 Class 생성

주요 활용 패턴
  암호화 클래스: 파일 읽기 → 복호화 → defineClass
  Hot Reload:   파일 변경 감지 → 새 ClassLoader 생성 → 재로드
  네트워크 로드: HTTP로 bytes 다운로드 → defineClass

Hot Reload 핵심 원칙
  새 ClassLoader 인스턴스 = 새 캐시 = 파일 재로드 가능
  인터페이스는 부모 ClassLoader에, 구현체만 재로드
  구 ClassLoader 참조 완전 제거 → 언로딩

defineClass 이후 단계
  Loading + Linking (Verification 포함) 까지만
  Initializing(static 블록)은 첫 사용 시 발생

병렬 로딩
  registerAsParallelCapable() 로 클래스별 잠금 사용
  성능이 중요한 ClassLoader에서 필수
```

---

## 🤔 생각해볼 문제

**Q1.** Hot Reload 시스템에서 인터페이스는 부모 ClassLoader에, 구현체만 재로드하는 이유를 `ClassCastException` 발생 조건과 연결해 설명하라.

**Q2.** 암호화된 ClassLoader에서 AES 키를 어디에 보관해야 안전한가? 소스코드 하드코딩, 환경변수, 외부 Key Management Service(KMS) 각각의 장단점을 설명하라.

**Q3.** `ClassLoader.registerAsParallelCapable()`을 호출하지 않은 ClassLoader를 여러 스레드가 동시에 사용하면 어떤 문제가 발생하는가? 어떤 상황에서 데드락이 발생할 수 있는가?

> 💡 **해설**
>
> **Q1.** JVM에서 클래스 정체성은 `(ClassLoader 인스턴스 + FQCN)`으로 결정된다. 새 ClassLoader로 재로드한 `PluginImpl`과 구 ClassLoader의 `PluginImpl`은 이름이 같아도 다른 타입이다. 따라서 구 ClassLoader로 로드된 `Plugin` 인터페이스 타입 변수에 새 ClassLoader의 `PluginImpl` 인스턴스를 담으면 `ClassCastException`이 발생한다. 해결책은 `Plugin` 인터페이스를 부모 ClassLoader(Application 또는 별도 공유 ClassLoader)에 두는 것이다. 부모 ClassLoader는 재로드되지 않으므로 인터페이스 타입이 항상 동일하게 유지된다. 구현체(`PluginImpl`)만 새 ClassLoader로 재로드해도 같은 인터페이스를 구현하므로 캐스팅이 안전하다.
>
> **Q2.** 소스코드 하드코딩: 구현이 간단하지만 소스 코드가 유출되면 키도 유출. Git 히스토리에도 남음. 절대 권장하지 않음. 환경변수: 배포 환경에서 분리 관리 가능, 소스 코드와 분리. 단, 프로세스 목록이나 로그에 노출될 위험. `System.getenv("DECRYPT_KEY")`로 접근. 외부 KMS(AWS KMS, HashiCorp Vault 등): 가장 안전. 키 자체를 서버에 저장하지 않고 암호화/복호화 요청만 KMS에 전달. 접근 감사 로그 제공. 단, 네트워크 의존성과 추가 비용 발생. 실무 권장: 민감도에 따라 환경변수(중간) 또는 KMS(높음) 선택.
>
> **Q3.** `registerAsParallelCapable()` 없으면 ClassLoader 인스턴스 전체에 단일 잠금이 걸린다. 데드락 시나리오: Thread A가 ClassLoader1으로 `ClassA`를 로드하는 중(ClassLoader1 잠금 보유), `ClassA`의 static 초기화가 `ClassB`를 필요로 해 ClassLoader2에 요청. Thread B는 ClassLoader2로 `ClassB`를 로드하는 중(ClassLoader2 잠금 보유), `ClassB`의 static 초기화가 `ClassA`를 필요로 해 ClassLoader1에 요청. → 두 스레드가 서로의 잠금을 기다리는 교착 상태. `registerAsParallelCapable()`을 쓰면 잠금 단위가 클래스명으로 세분화되어 `ClassA`와 `ClassB`를 동시에 로드 가능해 이 데드락을 피할 수 있다.

---

## 📚 참고 자료

- [JVMS §5.3.2 — Loading Using a User-defined Class Loader](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html#jvms-5.3.2)
- [OpenJDK ClassLoader.java](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/lang/ClassLoader.java)
- [Spring Boot DevTools — Restart ClassLoader](https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.devtools.restart)

---

<div align="center">

**[⬅️ 이전: Class Unloading](./05-class-unloading.md)** | **[다음: ClassLoader Isolation ➡️](./07-classloader-isolation.md)**

</div>

# Method Area & Metaspace - 메서드 영역과 메타스페이스

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Method Area는 무엇을 저장하며, Heap과 어떻게 다른가?
- PermGen이 사라지고 Metaspace로 바뀐 이유는 무엇인가?
- Metaspace는 왜 네이티브 메모리를 사용하며, 이것의 장단점은?
- `OutOfMemoryError: Metaspace`는 언제 발생하며, 어떻게 방지하는가?
- `-XX:MetaspaceSize`와 `-XX:MaxMetaspaceSize`의 차이는?

---

## 🔍 왜 이게 존재하는가

### 문제: 클래스 메타데이터를 어디에 저장할 것인가

```
객체 생성 시:
  new MyClass() → 힙에 인스턴스 저장
  
하지만 클래스 자체의 정보는?
  - 클래스 구조 (필드, 메서드 목록)
  - 메서드 바이트코드
  - 상수 풀 (Runtime Constant Pool)
  - static 변수
  - 클래스 메타데이터 (이름, 부모 클래스, 인터페이스 등)

이 정보는 인스턴스와 다름:
  인스턴스: 매번 생성/소멸
  클래스: 로드 후 프로그램 종료까지 유지 (대부분)
```

JVM은 클래스 메타데이터를 별도의 **Method Area**에 저장한다.

---

### PermGen의 한계와 Metaspace 탄생

```
Java 7 이전: PermGen (Permanent Generation)

┌─────────────────────────────────────┐
│          Heap                       │
├──────────────┬──────────────────────┤
│ Young Gen    │ Old Gen    │ PermGen │
│              │            │ (고정)   │
└──────────────┴──────────────────────┘
                            ↑
                    클래스 메타데이터

PermGen 문제:
  1. 고정 크기 (-XX:MaxPermSize)
     → 동적 클래스 로딩 시스템에서 OOM 빈번
     
  2. Heap의 일부로 관리
     → Full GC 시 PermGen도 스캔
     → GC 부담 증가
     
  3. 클래스 언로딩이 어려움
     → 메모리 회수 비효율

실제 사례:
  Tomcat 웹앱 재배포 100번
  → ClassLoader 누수
  → PermGen 가득 참
  → java.lang.OutOfMemoryError: PermGen space
  → 서버 재시작 필요
```

```
Java 8+: Metaspace

┌─────────────────────────────────────┐
│          Heap                       │
├──────────────┬──────────────────────┤
│ Young Gen    │ Old Gen              │
│              │                      │
└──────────────┴──────────────────────┘

Native Memory (Heap 밖)
┌─────────────────────────────────────┐
│         Metaspace                   │
│  (동적 확장, 네이티브 메모리)          │
└─────────────────────────────────────┘

Metaspace 장점:
  1. 동적 크기 조정
     → OOM 위험 감소
  
  2. Heap과 분리
     → Full GC 시 Metaspace 스캔 불필요
     → GC 성능 향상
  
  3. 클래스 언로딩 개선
     → 더 빠른 메모리 회수
```

---

## 📐 내부 구조

### 1. Method Area에 저장되는 내용

```
Class Metadata:
  - Class 구조 정보 (필드, 메서드, 생성자 목록)
  - 클래스 이름, 부모 클래스, 인터페이스
  - 접근 제어자 (public, private, static, final 등)

Method Bytecode:
  - 각 메서드의 바이트코드
  - 메서드 시그니처
  - Exception Table

Runtime Constant Pool:
  - 클래스 파일의 Constant Pool이 런타임에 확장된 것
  - 문자열 리터럴, 심볼릭 참조
  - (Java 7+ 문자열 리터럴은 Heap으로 이동)

Static Variables:
  - static 필드의 참조 (실제 객체는 Heap)
  
  class Config {
      static String API_KEY = "secret";  
      // API_KEY 참조는 Metaspace
      // "secret" String 인스턴스는 Heap
  }

Field & Method 정보:
  - 필드 타입, 이름, 오프셋
  - 메서드 파라미터, 반환 타입
  - vtable (가상 메서드 테이블)
```

---

### 2. Metaspace 메모리 구조

```
Metaspace 내부:

┌─────────────────────────────────────────────┐
│              Metaspace                      │
├───────────────────┬─────────────────────────┤
│   Class Metadata  │  Compressed Class Space │
│   (일반 클래스)     │  (압축 포인터 활성화 시)  │
│                   │                         │
│  - Klass 구조체   │  - Klass* 압축 저장     │
│  - 메서드 정보     │  - 32비트 주소 사용     │
│  - ConstantPool   │  (-XX:+UseCompressedOops)│
└───────────────────┴─────────────────────────┘
        ↑                       ↑
  동적 확장 가능         최대 1GB (기본)

메모리 할당 단위: Chunk
  JVM이 OS로부터 메모리를 Chunk 단위로 할당
  Chunk 크기: 1KB ~ 4MB (동적)
```

---

### 3. Metaspace 크기 설정 플래그

```
-XX:MetaspaceSize=<size>
  초기 Metaspace 크기 (기본: 플랫폼 의존, ~21MB)
  이 크기 초과 시 Full GC 트리거
  → 클래스 언로딩 시도
  
-XX:MaxMetaspaceSize=<size>
  최대 Metaspace 크기
  기본: 무제한 (시스템 메모리까지)
  
  설정 예:
  -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=512m
  
  주의:
  무제한이면 네이티브 메모리 고갈 위험
  컨테이너 환경에서는 반드시 설정

-XX:CompressedClassSpaceSize=<size>
  Compressed Class Space 최대 크기
  기본: 1GB
  -XX:+UseCompressedOops 활성화 시만 사용
```

---

### 4. Metaspace GC와 클래스 언로딩

```
Metaspace GC 트리거:

1. Metaspace 사용량 > MetaspaceSize
   → Full GC 발생
   → 클래스 언로딩 시도
   → 공간 확보

2. MaxMetaspaceSize 도달
   → OutOfMemoryError: Metaspace

클래스 언로딩 조건 (05-class-unloading 참고):
  ✓ 클래스의 모든 인스턴스가 GC됨
  ✓ Class 객체에 대한 참조 없음
  ✓ 해당 ClassLoader가 GC됨

언로딩 후:
  Metaspace 메모리 반납
  OS에 반환 (재사용 가능)
```

---

## 💻 실험으로 확인하기

### 실험 1: Metaspace 사용량 모니터링

```bash
# 실행 중인 JVM의 Metaspace 확인
jcmd <pid> VM.metaspace

# 출력 예시:
# Total: reserved=1065984KB, committed=65536KB
# Class space: reserved=1048576KB, committed=6144KB
# Non-class space: reserved=17408KB, committed=59392KB

# 설명:
# reserved: OS로부터 예약한 메모리 (실제 사용 전)
# committed: 실제 할당된 메모리
# Class space: 압축 클래스 포인터 영역
# Non-class: 일반 클래스 메타데이터

# jstat으로 실시간 모니터링
jstat -gc <pid> 1000

# 출력 컬럼:
# MC: Metaspace Capacity (KB)
# MU: Metaspace Used (KB)
# CCSC: Compressed Class Space Capacity
# CCSU: Compressed Class Space Used
```

---

### 실험 2: Metaspace OOM 재현

```java
import java.io.*;
import java.net.*;

public class MetaspaceLeakDemo {
    public static void main(String[] args) throws Exception {
        int count = 0;
        
        while (true) {
            // 새 ClassLoader로 클래스 동적 생성
            URLClassLoader loader = new URLClassLoader(new URL[]{});
            
            // 바이트코드 직접 생성 (간단한 클래스)
            byte[] classBytes = generateClass("DynamicClass" + count);
            
            // defineClass로 로드
            Class<?> clazz = loader.getClass()
                .getSuperclass()
                .getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class)
                .invoke(loader, "DynamicClass" + count, classBytes, 0, classBytes.length);
            
            count++;
            if (count % 1000 == 0) {
                System.out.println("Loaded classes: " + count);
                System.out.println("Metaspace: " + 
                    Runtime.getRuntime().freeMemory() / 1024 / 1024 + " MB");
            }
        }
    }
    
    static byte[] generateClass(String name) {
        // 최소한의 클래스 바이트코드 생성
        // (실제로는 ASM 라이브러리 사용 권장)
        return new byte[]{
            (byte)0xCA, (byte)0xFE, (byte)0xBA, (byte)0xBE, // magic
            // ... (클래스 구조)
        };
    }
}
```

```bash
# Metaspace 제한해서 실행
java -XX:MaxMetaspaceSize=64m MetaspaceLeakDemo

# 출력:
# Loaded classes: 1000
# Loaded classes: 2000
# ...
# Exception in thread "main" java.lang.OutOfMemoryError: Metaspace
#   at java.lang.ClassLoader.defineClass1(Native Method)
#   at java.lang.ClassLoader.defineClass(ClassLoader.java:...)
```

---

### 실험 3: 클래스 언로딩 관찰

```java
import java.lang.ref.WeakReference;
import java.net.*;

public class ClassUnloadingDemo {
    public static void main(String[] args) throws Exception {
        // ClassLoader에 대한 약한 참조
        WeakReference<ClassLoader> loaderRef = loadAndRelease();
        
        System.out.println("ClassLoader 생성 직후: " + (loaderRef.get() != null));
        
        // Full GC 유도
        for (int i = 0; i < 5; i++) {
            System.gc();
            Thread.sleep(100);
        }
        
        System.out.println("GC 후 ClassLoader: " + (loaderRef.get() != null));
        // false면 ClassLoader와 그 클래스들이 언로딩됨
    }
    
    static WeakReference<ClassLoader> loadAndRelease() throws Exception {
        URLClassLoader loader = new URLClassLoader(new URL[]{
            new File("./lib").toURI().toURL()
        });
        
        Class<?> clazz = loader.loadClass("com.example.TempClass");
        // 이 메서드 종료 후 loader, clazz 모두 참조 소멸
        
        return new WeakReference<>(loader);
    }
}
```

```bash
# 클래스 언로딩 로그 활성화
java -Xlog:class+unload=debug ClassUnloadingDemo

# 출력:
# ClassLoader 생성 직후: true
# [class,unload] unloading class com.example.TempClass 0x00007f...
# GC 후 ClassLoader: false
# ← 클래스가 언로딩됨
```

---

## ⚡ 실무 임팩트

### Metaspace OOM 진단과 해결

```
증상:
  java.lang.OutOfMemoryError: Metaspace

원인 분석:

1. 클래스 누수 확인
   jcmd <pid> GC.class_stats | head -100
   # 로드된 클래스 수와 ClassLoader 수 확인
   
   정상: 수천 ~ 수만 개
   누수: 수십만 개 이상

2. ClassLoader 누수 탐지
   Heap Dump 생성:
   jmap -dump:format=b,file=heap.hprof <pid>
   
   Eclipse MAT로 분석:
   - Leak Suspects → ClassLoader 확인
   - Path to GC Roots → 누수 원인 추적

주요 원인:

1. 프레임워크/라이브러리 버그
   - Tomcat 웹앱 재배포
   - JDBC 드라이버 미등록 해제
   - ThreadLocal 미정리

2. 동적 클래스 생성 과다
   - 과도한 람다 사용
   - 동적 프록시 (CGLib, Javassist)
   - Groovy, Kotlin 스크립트 실행

해결책:

1. MaxMetaspaceSize 증가
   -XX:MaxMetaspaceSize=512m
   → 임시 방편, 근본 해결 아님

2. 클래스 언로딩 강제
   -XX:+CMSClassUnloadingEnabled (CMS)
   -XX:+ClassUnloadingWithConcurrentMark (G1)
   → 대부분 기본 활성화

3. 코드 수정
   - ClassLoader 제대로 닫기 (close())
   - ThreadLocal 정리
   - JDBC 드라이버 등록 해제
```

### 컨테이너 환경 Metaspace 설정

```
Docker / Kubernetes:

잘못된 설정:
  컨테이너 메모리: 2GB
  JVM 힙: -Xmx1.5g
  Metaspace: 제한 없음
  
  문제:
  Metaspace가 500MB 사용
  → 총 2GB 초과
  → 컨테이너 OOM Killed

권장 설정:
  컨테이너 메모리: 2GB
  JVM 힙: -Xmx1.2g
  Metaspace: -XX:MaxMetaspaceSize=256m
  나머지 ~500MB: Thread Stack, Code Cache, Native 등
  
  계산:
  2GB = 1.2GB(힙) + 256MB(Meta) + 512MB(기타)
```

### 람다 남용과 Metaspace

```java
// ❌ 루프 내 람다 → 클래스 폭증
for (int i = 0; i < 1_000_000; i++) {
    list.stream()
        .filter(x -> x > i)  // 람다마다 새 클래스?
        .count();
}

// 실제로는:
// JVM이 람다를 캐시하므로 클래스 1개만 생성
// 하지만 복잡한 람다는 여러 클래스 생성 가능

// ✅ 재사용 가능한 경우 메서드 참조
Predicate<Integer> filter = x -> x > threshold;
list.stream().filter(filter).count();
```

---

## 🚫 흔한 오해

### "Metaspace는 무제한이니 설정 불필요하다"

```
❌ 잘못된 이해:
  MaxMetaspaceSize 기본값이 무제한이니 신경 쓸 필요 없다.

✅ 실제:
  무제한 = 시스템 메모리까지 사용 가능
  → 네이티브 메모리 고갈 위험
  → OS 전체 불안정
  
  특히 컨테이너 환경:
  무제한 Metaspace가 컨테이너 메모리 초과
  → OOM Killer 발동
  → JVM 강제 종료
  
  권장:
  항상 MaxMetaspaceSize 설정
  일반적으로 256MB ~ 512MB
```

### "클래스는 한 번 로드되면 절대 언로딩 안 된다"

```
❌ 잘못된 이해:
  클래스가 Metaspace에 로드되면 JVM 종료까지 유지된다.

✅ 실제:
  Custom ClassLoader가 로드한 클래스는 언로딩 가능
  
  언로딩 조건:
  1. ClassLoader가 GC됨
  2. 해당 클래스의 모든 인스턴스 GC됨
  3. Class 객체 참조 제거
  
  Application ClassLoader는 언로딩 안 됨:
  → 일반적인 비즈니스 클래스는 영구 유지
  
  Custom ClassLoader만 언로딩:
  → Tomcat WebApp, OSGi Bundle 등
```

### "PermGen 제거로 OOM 문제가 사라졌다"

```
❌ 잘못된 이해:
  Metaspace로 바뀌면서 OutOfMemoryError가 사라졌다.

✅ 실제:
  OOM 발생 조건만 바뀜
  
  PermGen 시대:
  고정 크기 → 쉽게 OOM
  
  Metaspace 시대:
  동적 확장 → OOM 덜 빈번하지만 여전히 발생
  ClassLoader 누수 → 계속 증가 → MaxMetaspaceSize 도달
  
  근본 해결:
  ClassLoader 누수 방지
  불필요한 동적 클래스 생성 제거
```

---

## 📌 핵심 정리

```
Method Area
  클래스 메타데이터 저장 영역
  클래스 구조, 메서드 바이트코드, Runtime Constant Pool, static 변수

PermGen → Metaspace
  Java 7: PermGen (Heap 일부, 고정 크기)
  Java 8+: Metaspace (네이티브 메모리, 동적 확장)
  
  변경 이유:
  - PermGen OOM 빈번
  - Full GC 부담
  - 클래스 언로딩 비효율

Metaspace 장점
  동적 크기 조정 → OOM 위험 감소
  Heap 분리 → GC 성능 향상
  클래스 언로딩 개선

Metaspace 크기 설정
  -XX:MetaspaceSize: 초기 크기 (Full GC 트리거)
  -XX:MaxMetaspaceSize: 최대 크기 (반드시 설정)
  
  권장: 256MB ~ 512MB

Metaspace OOM 원인
  ClassLoader 누수 (Tomcat 재배포, ThreadLocal)
  동적 클래스 과다 생성 (람다, 프록시, 스크립트)
  
  진단: jcmd VM.metaspace, Heap Dump + MAT

컨테이너 환경
  반드시 MaxMetaspaceSize 설정
  컨테이너 메모리 = 힙 + Metaspace + Thread Stack + 기타
```

---

## 🤔 생각해볼 문제

**Q1.** 애플리케이션이 시작 시 500개 클래스를 로드하고, 각 클래스 메타데이터가 평균 50KB라면 Metaspace는 최소 몇 MB 필요한가? Runtime Constant Pool과 static 변수도 고려해 추정하라.

**Q2.** `-XX:MaxMetaspaceSize=128m`으로 설정했는데 Metaspace 사용량이 200MB를 넘어서는데도 OOM이 발생하지 않는다. 어떻게 가능한가?

**Q3.** Spring Boot 애플리케이션을 도커 컨테이너(메모리 4GB)에 배포한다. 힙, Metaspace, Thread Stack을 각각 어떻게 설정해야 하는가? 200개 스레드 환경을 가정하고 계산하라.

> 💡 **해설**
>
> **Q1.** 순수 클래스 메타데이터만: 500 * 50KB = 25MB. 하지만 실제로는 Runtime Constant Pool(클래스당 ~10KB), static 변수 참조, 메서드 vtable 등이 추가로 필요하다. 일반적으로 클래스당 전체 메타데이터는 80~100KB 정도이므로 500개 클래스는 약 40~50MB의 Metaspace가 필요하다. 여기에 JVM 내부 클래스, 라이브러리 클래스까지 합치면 초기 MetaspaceSize는 최소 128MB 정도가 안전하다.
>
> **Q2.** MaxMetaspaceSize는 "소프트 리미트"가 아니라 "하드 리미트"다. 200MB를 넘어도 OOM이 안 나는 경우는 MaxMetaspaceSize 설정이 실제로 적용되지 않았거나, Compressed Class Space와 Non-class Space를 따로 계산하는 경우다. `jcmd <pid> VM.metaspace`로 확인하면 "reserved" 값이 MaxMetaspaceSize보다 큰 경우가 있는데, 이는 OS로부터 예약만 한 것이고 실제 "committed"가 중요하다. 또 다른 가능성은 JVM 옵션이 실제로 적용되지 않았거나 (`java -XX:+PrintFlagsFinal`로 확인), 다른 JVM 인스턴스를 보고 있는 경우.
>
> **Q3.** 컨테이너 4GB = 힙 + Metaspace + Thread Stack + Code Cache + Native Memory. 계산: 힙 2.5GB (`-Xms2.5g -Xmx2.5g`), Metaspace 256MB (`-XX:MaxMetaspaceSize=256m`), Thread Stack 200개 * 1MB = 200MB (`-Xss1m`), Code Cache ~50MB, Native Memory(NIO, JNI 등) ~500MB. 총 2.5 + 0.256 + 0.2 + 0.05 + 0.5 ≈ 3.5GB. 약 500MB 여유를 두어 안전. 만약 Thread Stack을 줄이고 싶다면 `-Xss512k`로 설정해 100MB 절약 가능 (단, StackOverflowError 주의).

---

## 📚 참고 자료

- [JVMS §2.5.4 — Method Area](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html#jvms-2.5.4)
- [JEP 122 — Remove the Permanent Generation](https://openjdk.org/jeps/122)
- [Metaspace in HotSpot JVM](https://stuefe.de/posts/metaspace/what-is-metaspace/)

---

<div align="center">

**[⬅️ 이전: Stack And Frames](./03-stack-and-frames.md)** | **[다음: Runtime Constant Pool ➡️](./05-runtime-constant-pool.md)**

</div>

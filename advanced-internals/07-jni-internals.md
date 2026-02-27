# JNI Internals - JNI 내부 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JNI (Java Native Interface)는 어떻게 동작하는가?
- JVM ↔ Native 코드 경계 비용은 얼마나 되는가?
- Global Reference와 Local Reference의 차이는?
- JNI를 안전하게 사용하는 방법은?

---

## 🔍 왜 이게 존재하는가

### 문제: Java만으로 부족하다

```
필요한 경우:
  - 하드웨어 직접 접근
  - OS API 호출
  - Legacy 코드 재사용
  - 성능 크리티컬 영역

JNI:
  Java ↔ C/C++ 경계
```

JNI는 **Java와 Native 세계의 다리**다.

---

## 📐 JNI 구조

### 1. 기본 사용법

```java
// Java 코드
public class Hello {
    static {
        System.loadLibrary("hello");  // libhello.so 로드
    }
    
    public native String sayHello(String name);
    
    public static void main(String[] args) {
        Hello h = new Hello();
        System.out.println(h.sayHello("World"));
    }
}
```

```c
// hello.c
#include <jni.h>
#include <stdio.h>

JNIEXPORT jstring JNICALL 
Java_Hello_sayHello(JNIEnv *env, jobject obj, jstring name) {
    // jstring → C string
    const char *nameStr = (*env)->GetStringUTFChars(env, name, NULL);
    
    // 작업 수행
    char buffer[256];
    snprintf(buffer, sizeof(buffer), "Hello, %s!", nameStr);
    
    // 메모리 해제
    (*env)->ReleaseStringUTFChars(env, name, nameStr);
    
    // C string → jstring
    return (*env)->NewStringUTF(env, buffer);
}
```

```bash
# 컴파일
javac Hello.java
javah -jni Hello
gcc -shared -fPIC -I${JAVA_HOME}/include \
    -I${JAVA_HOME}/include/linux \
    -o libhello.so hello.c

# 실행
java -Djava.library.path=. Hello
```

---

### 2. JNI 호출 경로

```
Java → Native 호출:

1. Java: obj.method()
2. JVM: Method Lookup
3. JNI Stub: Stack 준비
4. JNI Boundary: 상태 전환
   - Java Stack → Native Stack
   - Safepoint 체크
5. Native Code: 실행
6. Return: 역순

비용:
  - Java 메서드 호출: ~1ns
  - JNI 호출: ~20~50ns
  - 20~50배 차이
```

---

### 3. JNIEnv 포인터

```c
// JNIEnv: JNI 함수 테이블
typedef const struct JNINativeInterface *JNIEnv;

struct JNINativeInterface {
    void* reserved0;
    void* reserved1;
    void* reserved2;
    jint (JNICALL *GetVersion)(JNIEnv *env);
    jclass (JNICALL *FindClass)(JNIEnv *env, const char *name);
    jmethodID (JNICALL *GetMethodID)(JNIEnv *env, jclass clazz, 
                                      const char *name, const char *sig);
    // ... 수백 개 함수
};

사용:
  (*env)->GetMethodID(env, clazz, "method", "()V");

특징:
  - Thread-local
  - 스레드 간 공유 불가
  - Native 스레드는 AttachCurrentThread 필요
```

---

### 4. Reference 종류

```c
// Local Reference (기본)
jstring str = (*env)->NewStringUTF(env, "hello");
// 함수 종료 시 자동 해제
// Native 메서드 종료 시 무효화

// Global Reference (명시적)
jclass globalClazz = (*env)->NewGlobalRef(env, clazz);
// GC 방지
// 명시적 DeleteGlobalRef 필요

// Weak Global Reference
jweak weakRef = (*env)->NewWeakGlobalRef(env, obj);
// GC 허용
// 객체 회수 시 NULL

예:
Local: 함수 내 임시 사용
Global: 캐싱, 콜백
Weak: 캐시 (GC 허용)
```

---

## 💻 실험으로 확인하기

### 실험 1: JNI 호출 비용

```java
public class JNIBenchmark {
    static {
        System.loadLibrary("bench");
    }
    
    public native int nativeAdd(int a, int b);
    
    public int javaAdd(int a, int b) {
        return a + b;
    }
    
    public static void main(String[] args) {
        JNIBenchmark bench = new JNIBenchmark();
        
        // Warm-up
        for (int i = 0; i < 10000; i++) {
            bench.javaAdd(1, 2);
            bench.nativeAdd(1, 2);
        }
        
        // Java 측정
        long start = System.nanoTime();
        for (int i = 0; i < 1_000_000; i++) {
            bench.javaAdd(1, 2);
        }
        long javaTime = System.nanoTime() - start;
        
        // JNI 측정
        start = System.nanoTime();
        for (int i = 0; i < 1_000_000; i++) {
            bench.nativeAdd(1, 2);
        }
        long jniTime = System.nanoTime() - start;
        
        System.out.println("Java: " + javaTime / 1_000_000 + "ms");
        System.out.println("JNI: " + jniTime / 1_000_000 + "ms");
        System.out.println("Overhead: " + (jniTime / javaTime) + "x");
    }
}
```

```c
// bench.c
JNIEXPORT jint JNICALL 
Java_JNIBenchmark_nativeAdd(JNIEnv *env, jobject obj, jint a, jint b) {
    return a + b;
}
```

```bash
# 출력:
# Java: 2ms
# JNI: 50ms
# Overhead: 25x
```

---

## ⚡ 실무 임팩트

### 언제 JNI를 사용하는가

```
✅ 좋은 사용:
  - 대량 데이터 처리 (이미지, 비디오)
  - 계산 집약적 작업 (암호화, 압축)
  - OS API 필수 (하드웨어 제어)

❌ 나쁜 사용:
  - 간단한 연산 (a + b)
  - 빈번한 호출 (루프 내)
  - 문자열 변환 많음
```

---

### Batch 처리로 비용 최소화

```java
// ❌ 나쁜 예
for (int i = 0; i < 1_000_000; i++) {
    nativeProcess(data[i]);  // JNI 호출 100만 번
}

// ✅ 좋은 예
nativeProcessBatch(data);  // JNI 호출 1번
```

```c
// Batch 처리
JNIEXPORT void JNICALL 
Java_MyClass_nativeProcessBatch(JNIEnv *env, jobject obj, jintArray arr) {
    jint *data = (*env)->GetIntArrayElements(env, arr, NULL);
    jsize len = (*env)->GetArrayLength(env, arr);
    
    // 한 번에 처리
    for (int i = 0; i < len; i++) {
        data[i] = process(data[i]);
    }
    
    (*env)->ReleaseIntArrayElements(env, arr, data, 0);
}
```

---

### Direct ByteBuffer

```java
// Zero-Copy I/O
ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

// Native에서 직접 접근
nativeProcess(buffer);
```

```c
JNIEXPORT void JNICALL 
Java_MyClass_nativeProcess(JNIEnv *env, jobject obj, jobject buffer) {
    // Direct 메모리 포인터
    void *ptr = (*env)->GetDirectBufferAddress(env, buffer);
    jlong capacity = (*env)->GetDirectBufferCapacity(env, buffer);
    
    // 복사 없이 직접 접근
    memset(ptr, 0, capacity);
}
```

---

## 🚫 흔한 실수

### 메모리 누수

```c
// ❌ 메모리 누수
JNIEXPORT void JNICALL func(JNIEnv *env, jobject obj, jstring str) {
    const char *cstr = (*env)->GetStringUTFChars(env, str, NULL);
    // 작업...
    // ReleaseStringUTFChars 누락!
}

// ✅ 올바른 코드
JNIEXPORT void JNICALL func(JNIEnv *env, jobject obj, jstring str) {
    const char *cstr = (*env)->GetStringUTFChars(env, str, NULL);
    if (cstr == NULL) return;  // OutOfMemory 체크
    
    // 작업...
    
    (*env)->ReleaseStringUTFChars(env, str, cstr);  // 필수!
}
```

---

## 📌 핵심 정리

```
JNI
  Java Native Interface
  Java ↔ C/C++ 경계

호출 비용
  Java 메서드: ~1ns
  JNI 호출: ~20~50ns
  20~50배 차이

JNIEnv
  JNI 함수 테이블
  Thread-local
  스레드별 별도 포인터

Reference
  Local: 자동 해제 (함수 종료 시)
  Global: 명시적 해제 필요
  Weak: GC 허용

성능 최적화
  Batch 처리 (호출 최소화)
  Direct ByteBuffer (Zero-Copy)
  대량 데이터만 JNI 사용

실무
  계산 집약적 작업
  OS API 호출
  Legacy 코드 재사용
  간단한 연산은 Java로
```

---

## 🤔 생각해볼 문제

**Q1.** JNI 호출이 일반 Java 메서드 호출보다 20~50배 느린 이유를 설명하라.

**Q2.** Local Reference와 Global Reference를 언제 사용하는가? 메모리 관리 측면에서 설명하라.

**Q3.** 다음 코드의 문제점을 찾고, 수정하라.

```c
JNIEXPORT jstring JNICALL func(JNIEnv *env, jobject obj) {
    const char *str = "Hello";
    return str;  // 문제!
}
```

> 💡 **해설**
>
> **Q1.** JNI 호출이 느린 이유: ① 상태 전환 — Java Stack → Native Stack 전환 비용. ② Safepoint 체크 — GC를 위해 안전 지점 확인. ③ Parameter Marshalling — Java 객체 → C 타입 변환 (jstring → char*). ④ JNI 함수 테이블 간접 호출 — 포인터 역참조. ⑤ 캐시 미스 — Java/Native 코드 간 캐시 무효화. 총 비용: 20~50ns.
>
> **Q2.** Local vs Global 사용: ① Local Reference — 함수 내 임시 사용, 자동 해제 (함수 종료 시). 메모리 관리: JVM이 자동. 사용: 대부분의 경우. ② Global Reference — 캐싱, 콜백 함수, 다른 함수로 전달. 명시적 DeleteGlobalRef 필요. 메모리 관리: 수동 (누수 주의). 사용: 장기 보관 필요 시. ③ Weak Global — 캐시 (GC 허용). 메모리: GC가 회수 가능.
>
> **Q3.** 문제: `return str;` — C 문자열을 그대로 반환 (타입 불일치). JNI는 jstring 타입 필요. 수정: `return (*env)->NewStringUTF(env, str);` — C 문자열 → jstring 변환. 올바른 코드: `JNIEXPORT jstring JNICALL func(JNIEnv *env, jobject obj) { const char *str = "Hello"; return (*env)->NewStringUTF(env, str); }`.

---

## 📚 참고 자료

- [JNI Specification](https://docs.oracle.com/en/java/javase/17/docs/specs/jni/index.html)
- [JNI Best Practices](https://www.baeldung.com/jni)

---

<div align="center">

**[⬅️ 이전: Instrumentation & Java Agent](./06-instrumentation-and-agent.md)** | **[홈으로 🏠](../README.md)**

</div>

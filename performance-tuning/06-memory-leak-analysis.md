# Memory Leak Analysis - 메모리 누수 분석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 메모리 누수는 어떻게 탐지하는가?
- Heap Dump를 어떻게 분석하는가?
- 흔한 메모리 누수 패턴은 무엇인가?
- Eclipse MAT는 어떻게 사용하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 메모리가 계속 증가한다

```
증상:
  - Old Gen 사용량 증가
  - Full GC 후에도 메모리 높음
  - 결국 OutOfMemoryError

원인:
  객체가 계속 참조됨
  → GC 회수 불가
  → 메모리 누수
```

메모리 누수 분석은 **안정성의 핵심**이다.

---

## 📐 누수 탐지 방법

### 1. GC 로그로 탐지

```bash
# GC 로그 분석
-Xlog:gc*:file=gc.log:time,uptime

# 패턴 확인
grep "Pause Full" gc.log

# After GC 추이
0h: [gc] 500M->100M
1h: [gc] 600M->150M
2h: [gc] 700M->200M  ← 지속 증가
3h: [gc] 800M->300M
4h: [gc] 900M->450M

판단:
  After GC가 지속 증가
  → 메모리 누수 의심
```

---

### 2. 힙 메모리 모니터링

```bash
# jstat으로 모니터링
jstat -gc <pid> 1000

# 출력:
# S0C    S1C    S0U    S1U      EC       EU        OC         OU
# 0.0   0.0    0.0    0.0   512.0   256.0    2048.0    1800.0  ← Old 증가

# 주기적 확인
watch -n 5 'jstat -gc <pid>'

# Old 사용량이 계속 증가
# → 누수 의심
```

---

### 3. Heap Dump 생성

```bash
# 수동 생성
jmap -dump:live,format=b,file=heap.hprof <pid>

# OOM 시 자동 생성
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heapdump.hprof

# 파일 크기 확인
ls -lh heap.hprof
# -rw-r--r-- 1 user user 2.5G heap.hprof
```

---

## 🔍 Eclipse MAT 분석

### 1. MAT 설치 및 실행

```bash
# 다운로드
https://www.eclipse.org/mat/downloads.php

# 실행
./mat

# Heap Dump 열기
File → Open Heap Dump → heap.hprof
```

---

### 2. Leak Suspects 리포트

```
MAT 자동 분석:

1. Leak Suspects Report
   - 자동으로 누수 의심 객체 탐지
   - 큰 객체 우선 표시

2. 문제 요약
   Problem Suspect 1:
   
   Thread "worker-1" keeps local variables with 
   total size 1,234,567,890 bytes
   
   Details:
   - HashMap instance (800MB)
   - Key: String, Value: byte[]
   - 500,000 entries

3. Dominator Tree
   객체 → 참조 → 참조 → ...
   전체 트리 크기 표시
```

---

### 3. Histogram (클래스별 통계)

```
Histogram 탭:

Class Name              Objects    Shallow    Retained
byte[]                  500,000    800MB      800MB
java.lang.String        500,000    16MB       816MB
HashMap$Node           500,000    32MB       848MB
...

분석:
  - byte[] 500,000개 (800MB)
  - String 500,000개
  → HashMap에 대량 데이터

우클릭 → List objects → with incoming references
→ HashMap 찾기
```

---

### 4. Dominator Tree (지배 트리)

```
Dominator Tree:

this$0 MyCache (900MB)
  ├─ HashMap table (850MB)
  │   ├─ Entry[0] (1.7MB)
  │   │   └─ byte[] (1.7MB)
  │   ├─ Entry[1] (1.7MB)
  │   └─ ...
  └─ ...

분석:
  MyCache 인스턴스가 900MB 지배
  → HashMap이 850MB
  → 누수 원인
```

---

## 🐛 흔한 누수 패턴

### 패턴 1: Static Collection

```java
// ❌ 누수
public class Cache {
    private static Map<String, byte[]> cache = new HashMap<>();
    
    public void put(String key, byte[] data) {
        cache.put(key, data);  // 계속 증가
    }
}

문제:
  static → 영원히 참조
  → GC 회수 불가

해결:
  - WeakHashMap 사용
  - 크기 제한 (LRU Cache)
  - 명시적 제거
```

---

### 패턴 2: ThreadLocal 미제거

```java
// ❌ 누수
public class RequestContext {
    private static ThreadLocal<BigData> context = new ThreadLocal<>();
    
    public void handle() {
        context.set(new BigData());  // 100MB
        process();
        // context.remove() 없음!
    }
}

문제:
  Thread Pool 환경
  → Thread 재사용
  → ThreadLocal 값 유지
  → 메모리 누수

해결:
  try {
      context.set(data);
      process();
  } finally {
      context.remove();  // 필수!
  }
```

---

### 패턴 3: Listener 미제거

```java
// ❌ 누수
public class EventBus {
    private List<Listener> listeners = new ArrayList<>();
    
    public void register(Listener listener) {
        listeners.add(listener);
    }
    
    // unregister() 메서드 없음!
}

public class MyComponent {
    void init() {
        eventBus.register(this);
        // 컴포넌트 종료 시 unregister 안 함
    }
}

문제:
  Listener가 계속 참조됨
  → 컴포넌트 GC 불가

해결:
  public void unregister(Listener listener) {
      listeners.remove(listener);
  }
  
  또는 WeakReference 사용
```

---

### 패턴 4: ClassLoader Leak

```java
// ❌ 누수
public class HotDeploy {
    private static Object cache;  // Static 참조
    
    static {
        cache = new HeavyObject();
    }
}

문제:
  애플리케이션 재배포 시
  → 새 ClassLoader 생성
  → 이전 ClassLoader 회수 안 됨
  → Static 참조 유지

해결:
  - Static 필드 최소화
  - 재배포 시 명시적 정리
```

---

## 💻 실무 분석 예시

### 예시 1: Cache 누수

```
증상:
  매일 OOM 발생

GC 로그:
  After GC 지속 증가
  0일: 500MB
  1일: 1GB
  2일: 1.5GB
  3일: 2GB → OOM

Heap Dump 분석:
  Histogram:
  HashMap$Node : 1,000,000개 (1.2GB)
  
  Dominator Tree:
  ProductCache (1.5GB)
    └─ HashMap (1.2GB)

원인:
  ProductCache에 무제한 추가
  → 제거 로직 없음

해결:
  GuavaCache (크기 제한)
  
  Cache<String, Product> cache = CacheBuilder.newBuilder()
      .maximumSize(10_000)
      .expireAfterWrite(1, TimeUnit.HOURS)
      .build();

효과:
  메모리 안정 (500MB 유지)
```

---

### 예시 2: ThreadLocal 누수

```
증상:
  Thread Pool 환경에서 메모리 증가

Heap Dump:
  Histogram:
  BigData : 200개 (20GB)
  
  Incoming References:
  Thread "pool-1-thread-1" → ThreadLocalMap
    → Entry → BigData (100MB)

원인:
  RequestContext.remove() 미호출
  → Thread 재사용 시 누적

해결:
  @Around("@annotation(RequestScoped)")
  public Object handle(ProceedingJoinPoint pjp) {
      try {
          RequestContext.set(data);
          return pjp.proceed();
      } finally {
          RequestContext.remove();
      }
  }

효과:
  메모리 누수 제거
```

---

## 🔧 예방 방법

### 1. WeakReference 활용

```java
// 약한 참조로 캐시
Map<String, WeakReference<Data>> cache = new WeakHashMap<>();

cache.put(key, new WeakReference<>(data));

// 메모리 부족 시 GC가 자동 회수
```

---

### 2. 크기 제한

```java
// Guava Cache
Cache<String, Data> cache = CacheBuilder.newBuilder()
    .maximumSize(10_000)
    .build();

// Apache Commons LRUMap
Map<String, Data> cache = new LRUMap<>(10_000);
```

---

### 3. 명시적 정리

```java
// Lifecycle 관리
public class ResourceManager implements AutoCloseable {
    private List<Resource> resources = new ArrayList<>();
    
    @Override
    public void close() {
        resources.clear();
        resources = null;
    }
}

try (ResourceManager rm = new ResourceManager()) {
    // 사용
}  // 자동 정리
```

---

## 📌 핵심 정리

```
누수 탐지
  GC 로그: After GC 증가 추이
  jstat: Old Gen 지속 증가
  Heap Dump: 큰 객체 분석

Heap Dump
  jmap -dump:live,format=b
  -XX:+HeapDumpOnOutOfMemoryError

Eclipse MAT
  Leak Suspects: 자동 탐지
  Histogram: 클래스별 통계
  Dominator Tree: 참조 구조

흔한 패턴
  Static Collection (무제한 증가)
  ThreadLocal (remove 미호출)
  Listener (unregister 미호출)
  ClassLoader (재배포 시)

해결 방법
  WeakReference/WeakHashMap
  크기 제한 (LRU Cache)
  명시적 정리 (remove, close)
  try-finally 패턴

예방
  Static 최소화
  ThreadLocal 항상 remove
  Listener 항상 unregister
  Cache 크기 제한
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 메모리 누수가 발생하는 이유를 설명하고, 해결 방법을 제시하라.

```java
public class SessionManager {
    private static Map<String, Session> sessions = new HashMap<>();
    
    public void login(String userId, Session session) {
        sessions.put(userId, session);
    }
}
```

**Q2.** Heap Dump에서 HashMap$Node가 100만 개 발견되었다. 누수 원인을 어떻게 추적하는가?

**Q3.** ThreadLocal을 사용하는 웹 애플리케이션에서 메모리 누수를 방지하기 위한 Best Practice 3가지를 제시하라.

> 💡 **해설**
>
> **Q1.** 누수 이유: ① static Map → 영원히 참조 유지. ② login()만 있고 logout() 없음 → Session 계속 추가. ③ 사용자 재로그인 시 덮어쓰기만 (제거 없음). 해결: ① logout() 추가: `public void logout(String userId) { sessions.remove(userId); }`. ② 크기 제한: `Map<String, Session> sessions = Collections.synchronizedMap(new LRUMap<>(10_000));`. ③ 만료 시간: Guava Cache + `expireAfterAccess(30, TimeUnit.MINUTES)`. ④ WeakHashMap: 메모리 부족 시 자동 제거.
>
> **Q2.** HashMap$Node 추적: ① MAT Histogram → HashMap$Node 우클릭 → "List objects" → "with incoming references". ② Incoming References 확인 → HashMap 찾기. ③ HashMap 우클릭 → "Path to GC Roots" → "exclude weak references". ④ 참조 경로: Thread → static 필드 → MyCache → HashMap. ⑤ MyCache 클래스 확인 → 코드 분석 → put()만 있고 remove() 없음. ⑥ 해결: remove() 추가 또는 크기 제한.
>
> **Q3.** ThreadLocal Best Practice: ① try-finally 패턴: `try { threadLocal.set(data); process(); } finally { threadLocal.remove(); }`. ② Servlet Filter에서 제거: `@WebFilter public class CleanupFilter { doFilter() { try { chain.doFilter(); } finally { RequestContext.remove(); } } }`. ③ Spring Interceptor: `@Component public class CleanupInterceptor { afterCompletion() { RequestContext.remove(); } }`. ④ 자동 제거: InheritableThreadLocal 대신 일반 ThreadLocal + 명시적 제거. ⑤ 모니터링: Heap Dump 주기적 확인 → ThreadLocalMap 크기 체크.

---

## 📚 참고 자료

- [Eclipse MAT Tutorial](https://www.eclipse.org/mat/documentation/)
- [Java Memory Leaks](https://www.baeldung.com/java-memory-leaks)
- [Heap Dump Analysis](https://dzone.com/articles/heap-dump-analysis)

---

<div align="center">

**[⬅️ 이전: Profiling with async-profiler](./05-profiling-with-async-profiler.md)** | **[다음: Benchmarking with JMH ➡️](./07-benchmarking-with-jmh.md)**

</div>

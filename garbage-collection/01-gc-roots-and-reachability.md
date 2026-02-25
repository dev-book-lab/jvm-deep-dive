# GC Roots & Reachability - GC 루트와 도달 가능성

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- GC는 어떻게 "살아있는 객체"와 "죽은 객체"를 구분하는가?
- GC Root는 무엇이며, 어떤 종류가 있는가?
- 순환 참조(Circular Reference)는 왜 메모리 누수가 아닌가?
- Reachability Analysis는 어떻게 동작하는가?
- Reference Counting 방식을 JVM이 사용하지 않는 이유는?

---

## 🔍 왜 이게 존재하는가

### 문제: 어떤 객체를 제거해야 하는가

```java
Object a = new Object();
Object b = new Object();
a = null;  // a가 가리키던 객체는?
```

```
단순한 접근 (Reference Counting):
  각 객체마다 참조 카운트 유지
  참조 증가: counter++
  참조 감소: counter--
  counter == 0 → 제거

문제점:
  순환 참조 처리 불가
  
  A → B
  B → A
  
  외부에서 A, B 참조 없어도
  → counter(A) = 1, counter(B) = 1
  → 제거 안 됨 (메모리 누수)
```

JVM은 **Reachability Analysis**로 살아있는 객체를 판단한다.

---

## 📐 내부 구조

### 1. GC Root 개념

```
GC Root:
  "확실히 살아있는" 시작점
  
  GC 알고리즘:
  1. GC Root에서 시작
  2. 도달 가능한 모든 객체 탐색 (그래프 순회)
  3. 도달 불가능한 객체 = 죽은 객체 → 제거

예시:
  
  GC Root (Local Variable)
    ↓
  Object A
    ↓
  Object B → Object C
  
  Object D (아무도 참조 안 함)
  
  Reachable: A, B, C
  Unreachable: D → GC 대상
```

---

### 2. GC Root 종류

```
1. Stack Frame의 Local Variable
   void method() {
       Object obj = new Object();  ← GC Root
       // obj가 가리키는 객체는 살아있음
   }

2. Static 변수
   class MyClass {
       static Object obj = new Object();  ← GC Root
       // 클래스가 언로드되기 전까지 살아있음
   }

3. Active Thread (Thread 객체 자체)
   Thread thread = new Thread(() -> { ... });
   thread.start();
   // thread 객체와 내부 필드들은 살아있음

4. JNI References
   JNIEXPORT void JNICALL Java_MyClass_nativeMethod(JNIEnv *env, jobject obj) {
       // obj는 GC Root (네이티브 코드에서 참조 중)
   }

5. Synchronized Lock이 걸린 객체
   synchronized (obj) {
       // obj는 GC Root (Lock 중)
   }

6. JVM 내부 참조
   - System Class Loader
   - Exception 객체 (현재 처리 중)
   - 기타 JVM 내부 데이터 구조
```

---

### 3. Reachability Analysis 과정

```
Mark Phase (Marking):

1. GC Root 수집
   - Stack Frame 스캔
   - Static 변수 스캔
   - Thread 목록 스캔
   
2. BFS/DFS로 그래프 순회
   
   초기: GC Roots = {A, D}
   
   객체 그래프:
   A → B → C
   D → E
   F (고립)
   
   Mark 과정:
   Step 1: Mark(A), Mark(D)
   Step 2: A의 참조 탐색 → Mark(B)
           D의 참조 탐색 → Mark(E)
   Step 3: B의 참조 탐색 → Mark(C)
   
   결과:
   Marked: A, B, C, D, E
   Unmarked: F → GC 대상

3. Mark 비트 설정
   각 객체의 Object Header에 Mark 비트
   
   Before GC:
   A: [mark=0][data]
   B: [mark=0][data]
   ...
   
   After Marking:
   A: [mark=1][data]  ← Reachable
   B: [mark=1][data]
   F: [mark=0][data]  ← Unreachable
```

---

### 4. 순환 참조 처리

```
순환 참조는 문제 없음:

객체 그래프:
  A → B
  B → C
  C → A  (순환)
  
  D (외부) → A

GC Root: D

Reachability:
  D → A (도달 가능)
  A → B (도달 가능)
  B → C (도달 가능)
  C → A (이미 Mark됨, 순회 종료)
  
결과: A, B, C 모두 살아있음

순환이지만 외부 참조 없을 때:

  A → B
  B → C
  C → A

GC Root에서 출발: (A에 도달 불가)
  → A, B, C 모두 Unreachable
  → 모두 GC 대상

핵심:
  "GC Root에서 도달 가능한가"가 기준
  순환 여부는 무관
```

---

### 5. Reference Counting vs Reachability

```
Reference Counting:

Object A {
    int refCount = 0;
}

new A() → refCount++
a = null → refCount--
refCount == 0 → 제거

장점:
  - 즉시 메모리 회수
  - GC Pause 없음

단점:
  - 순환 참조 처리 불가
  - 모든 참조 변경 시 카운트 업데이트 (오버헤드)
  - 멀티스레드에서 동기화 필요

Reachability Analysis (JVM):

GC 시점에 한 번에 판단
  - 순환 참조 처리 가능
  - Reference 변경 시 오버헤드 없음
  
단점:
  - GC Pause 발생 (STW)
  - 죽은 객체 즉시 회수 안 됨

JVM 선택: Reachability
  → 순환 참조 처리 필수
  → Pause는 다른 기법으로 최소화 (Concurrent GC)
```

---

## 💻 실험으로 확인하기

### 실험 1: 순환 참조 테스트

```java
class Node {
    Node next;
    byte[] data = new byte[1024 * 1024];  // 1MB
}

public class CircularRefTest {
    public static void main(String[] args) throws Exception {
        // 순환 참조 생성
        Node a = new Node();
        Node b = new Node();
        Node c = new Node();
        
        a.next = b;
        b.next = c;
        c.next = a;  // 순환
        
        System.out.println("=== 순환 참조 생성 ===");
        printMemory();
        
        // 외부 참조 제거
        a = null;
        b = null;
        c = null;
        
        System.out.println("=== 참조 제거 후 ===");
        printMemory();
        
        // GC 강제 실행
        System.gc();
        Thread.sleep(100);
        
        System.out.println("=== GC 후 ===");
        printMemory();
    }
    
    static void printMemory() {
        Runtime rt = Runtime.getRuntime();
        long used = (rt.totalMemory() - rt.freeMemory()) / (1024 * 1024);
        System.out.println("Used Memory: " + used + " MB");
    }
}
```

```bash
# 출력:
# === 순환 참조 생성 ===
# Used Memory: 5 MB
# === 참조 제거 후 ===
# Used Memory: 5 MB (아직 회수 안 됨)
# === GC 후 ===
# Used Memory: 2 MB (순환 참조 제거됨)

# → 순환 참조도 GC됨 (메모리 누수 아님)
```

---

### 실험 2: GC Root 확인 (jcmd)

```bash
# 힙 덤프 생성
jcmd <pid> GC.heap_dump heap.hprof

# Eclipse MAT로 분석
# File → Open Heap Dump → heap.hprof

# GC Roots 확인:
# - Java Local (Stack)
# - JNI Local
# - Thread
# - System Class Loader
# - Static fields
```

---

### 실험 3: WeakReference로 Reachability 확인

```java
import java.lang.ref.WeakReference;

public class ReachabilityTest {
    public static void main(String[] args) throws Exception {
        Object strong = new Object();
        WeakReference<Object> weak = new WeakReference<>(strong);
        
        System.out.println("Strong ref exists: " + weak.get());  // Object
        
        strong = null;  // 유일한 Strong Reference 제거
        
        System.gc();
        Thread.sleep(100);
        
        System.out.println("After GC: " + weak.get());  // null
        // → Strong Reference 없으면 GC됨
    }
}
```

---

## ⚡ 실무 임팩트

### 메모리 누수 패턴 — Static 컬렉션

```java
// ❌ 메모리 누수
public class Cache {
    private static Map<String, Object> cache = new HashMap<>();
    
    public static void put(String key, Object value) {
        cache.put(key, value);  // GC Root (static) → 영구 보존
    }
}

// 사용:
for (int i = 0; i < 1_000_000; i++) {
    Cache.put("key" + i, new byte[1024]);
}
// → 1GB 메모리 누수 (절대 GC 안 됨)

// ✅ 개선 1: WeakHashMap
private static Map<String, Object> cache = new WeakHashMap<>();
// → Key가 외부에서 참조 안 되면 GC

// ✅ 개선 2: 명시적 제거
private static Map<String, Object> cache = new LinkedHashMap<>(1000, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > 1000;  // 최대 1000개 유지
    }
};
```

### Thread 메모리 누수

```java
// ❌ Thread 누수
public class ThreadLeakTest {
    void process() {
        Thread thread = new Thread(() -> {
            while (true) {
                // 무한 루프 (종료 안 함)
            }
        });
        thread.start();
        // thread는 GC Root → 절대 GC 안 됨
    }
}

// 100번 호출 → 100개 Thread → 메모리 누수

// ✅ 개선
thread.setDaemon(true);  // Daemon Thread
// 또는
thread.interrupt();  // 적절한 종료
```

### JNI 메모리 누수

```c
// ❌ JNI 누수
JNIEXPORT void JNICALL Java_MyClass_leaky(JNIEnv *env, jobject obj) {
    jobject globalRef = (*env)->NewGlobalRef(env, obj);
    // 사용 후 DeleteGlobalRef 안 함 → 메모리 누수
}

// ✅ 개선
JNIEXPORT void JNICALL Java_MyClass_correct(JNIEnv *env, jobject obj) {
    jobject globalRef = (*env)->NewGlobalRef(env, obj);
    // ... 사용 ...
    (*env)->DeleteGlobalRef(env, globalRef);  // 명시적 해제
}
```

---

## 🚫 흔한 오해

### "순환 참조는 메모리 누수다"

```
❌ 잘못된 이해:
  A → B → A 순환 참조는 GC가 못 한다.

✅ 실제:
  GC Root에서 도달 불가능하면 GC됨
  
  순환 참조 예시:
  class Node {
      Node next;
  }
  
  Node a = new Node();
  Node b = new Node();
  a.next = b;
  b.next = a;
  
  // 외부 참조 제거
  a = null;
  b = null;
  
  → GC Root에서 도달 불가
  → a, b 모두 GC됨
  
  메모리 누수가 되는 경우:
  static Node root;  // GC Root
  root.next = a;
  
  → GC Root에서 a, b 도달 가능
  → GC 안 됨
```

### "null 할당하면 즉시 메모리 회수"

```
❌ 잘못된 이해:
  a = null; → 메모리 즉시 회수

✅ 실제:
  GC가 실행될 때까지 대기
  
  a = null;
  // 메모리는 아직 그대로
  
  System.gc();  // GC 요청 (보장 안 됨)
  // 언젠가 GC가 회수
  
  즉시 회수:
  - 불가능 (JVM 설계)
  - GC는 비동기로 실행
  - Stop-The-World 최소화 위해 일괄 처리
```

### "finalize()로 메모리 관리"

```java
❌ 잘못된 이해:
  finalize()로 리소스 정리하면 안전하다.

✅ 실제:
  finalize()는 deprecated (Java 9+)
  
  문제점:
  1. 실행 시점 불확실
     GC 후 언젠가 실행 (보장 안 됨)
  
  2. 성능 저하
     Finalization Queue 추가 오버헤드
  
  3. 부활(Resurrection) 가능
     finalize()에서 this를 다시 참조 → 부활

대안:
  try-with-resources (AutoCloseable)
  
  try (FileInputStream fis = new FileInputStream("file.txt")) {
      // 사용
  }  // 자동 close() 호출
```

---

## 📌 핵심 정리

```
GC Root
  확실히 살아있는 객체의 시작점
  종류: Local Variable, Static, Thread, JNI, Lock 등

Reachability Analysis
  GC Root에서 시작해 도달 가능한 객체 탐색
  도달 불가능한 객체 = 죽은 객체 → GC 대상

Mark Phase
  GC Root에서 BFS/DFS 순회
  도달 가능한 객체에 Mark 비트 설정
  Unmarked 객체 → 제거

순환 참조
  Reachability로 해결
  GC Root에서 도달 불가능하면 GC됨
  메모리 누수 아님

Reference Counting vs Reachability
  Reference Counting: 순환 참조 처리 불가
  Reachability: 순환 참조 처리 가능
  JVM 선택: Reachability

메모리 누수 주의
  Static 컬렉션 (GC Root)
  Thread 미종료 (GC Root)
  JNI GlobalRef 미해제

null 할당
  즉시 회수 아님
  GC 실행 시점에 회수

finalize()
  사용 금지 (deprecated)
  try-with-resources 사용
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 어떤 객체가 GC 대상인가? GC Root와 Reachability를 고려해 설명하라.

```java
class Node {
    Node next;
    static Node root = new Node();  // A
}

void method() {
    Node b = new Node();
    Node c = new Node();
    b.next = c;
    c.next = b;  // 순환
    
    Node.root.next = b;
    Node.root = null;
}
```

**Q2.** Reference Counting 방식을 JVM이 채택하지 않은 이유 3가지를 설명하라.

**Q3.** 다음 코드에서 메모리 누수가 발생하는가? 발생한다면 그 이유와 해결 방법을 제시하라.

```java
public class EventBus {
    private static List<Listener> listeners = new ArrayList<>();
    
    public static void register(Listener listener) {
        listeners.add(listener);
    }
}

// 사용
for (int i = 0; i < 10000; i++) {
    EventBus.register(new MyListener());
}
```

> 💡 **해설**
>
> **Q1.** 초기 GC Root: `Node.root` (static 변수) → A. A → B (root.next), B → C (b.next), C → B (순환). `Node.root = null` 후: GC Root 사라짐 → A, B, C 모두 도달 불가능 → 모두 GC 대상. `root.next = b`로 연결했지만, `root` 자체가 null이 되면서 GC Root 역할 상실. 순환 참조 (B ↔ C)는 문제 아님, GC Root에서 도달 불가능하므로 GC됨.
>
> **Q2.** ① 순환 참조 처리 불가 — A → B → A 순환 시 refCount가 0이 안 됨 → 메모리 누수. Java는 복잡한 객체 그래프에서 순환 참조 흔함 → 치명적. ② 성능 오버헤드 — 모든 참조 할당/제거 시 카운터 증가/감소 필요 → 멀티스레드에서 동기화 비용 큼 → CAS 연산 등 필요. ③ 즉시 회수의 단점 — 객체 제거가 즉각 발생 → 프로그램 실행 흐름 중 예측 불가능한 시점에 destructor 실행 → 지연(latency) 불균형. GC Pause를 한 번에 모아서 처리하는 게 오히려 예측 가능.
>
> **Q3.** 메모리 누수 발생. 이유: `listeners`는 static (GC Root) → 등록된 모든 Listener가 영구 보존. `register()`만 있고 `unregister()`가 없음 → 10,000개 Listener가 계속 메모리 차지. 해결: ① `WeakHashMap` 또는 `WeakReference` 사용 → Listener가 외부에서 참조 안 되면 자동 GC. ② 명시적 `unregister()` 제공 → 사용 후 제거. ③ 크기 제한 (`LinkedHashMap` + `removeEldestEntry`) → 최대 개수 제한. ④ Listener를 Weak Reference로 감싸기 → `listeners.add(new WeakReference<>(listener))`.

---

## 📚 참고 자료

- [JVMS §2.5.3 — Heap](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html#jvms-2.5.3)
- [Java GC Roots](https://www.baeldung.com/java-gc-roots)
- [Understanding Reachability](https://shipilev.net/jvm/anatomy-quarks/12-inline-caches/)

---

<div align="center">

**[⬅️ 이전: JVM Intrinsics](../execution-engine/07-intrinsics.md)** | **[다음: Reference Types ➡️](./02-reference-types.md)**

</div>

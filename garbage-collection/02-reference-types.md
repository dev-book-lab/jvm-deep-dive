# Reference Types - 참조 타입

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Strong, Soft, Weak, Phantom Reference의 차이는 무엇인가?
- 각 Reference 타입은 GC와 어떻게 상호작용하는가?
- `WeakHashMap`은 어떻게 동작하며, 언제 사용하는가?
- `ReferenceQueue`의 역할은 무엇인가?
- Phantom Reference는 왜 존재하며, `finalize()`와 어떻게 다른가?

---

## 🔍 왜 이게 존재하는가

### 문제: 캐시를 어떻게 구현할 것인가

```java
// 이미지 캐시
Map<String, Image> imageCache = new HashMap<>();

imageCache.put("logo.png", loadImage("logo.png"));
```

```
문제 1: 메모리 부족
  캐시가 무한정 증가
  → OOM (OutOfMemoryError)
  
문제 2: 수동 관리의 어려움
  언제 캐시를 비울 것인가?
  → LRU? TTL? Size limit?
  → 복잡한 로직 필요

이상적인 해결책:
  "메모리가 부족하면 자동으로 GC"
  → Soft Reference
```

Java는 **4가지 Reference 타입**을 제공한다.

---

## 📐 내부 구조

### 1. Reference 타입 개요

```
Strong Reference (일반 참조):
  Object obj = new Object();
  → GC Root에서 도달 가능하면 절대 GC 안 됨

Soft Reference:
  SoftReference<Object> soft = new SoftReference<>(obj);
  → 메모리 부족 시 GC 가능

Weak Reference:
  WeakReference<Object> weak = new WeakReference<>(obj);
  → Strong Reference 없으면 즉시 GC

Phantom Reference:
  PhantomReference<Object> phantom = new PhantomReference<>(obj, queue);
  → finalize() 이후 처리용
```

---

### 2. Strong Reference

```java
// 가장 일반적인 참조
Object obj = new Object();
List<String> list = new ArrayList<>();

특징:
  - GC Root에서 도달 가능하면 절대 GC 안 됨
  - 명시적으로 null 할당해야 GC 대상
  
메모리 관리:
  obj = null;  // 참조 제거
  // → GC 대상 (다른 Strong Reference 없으면)

사용:
  대부분의 일반 객체
```

---

### 3. Soft Reference

```java
SoftReference<Image> imageRef = new SoftReference<>(image);

특징:
  - 메모리 부족 시 GC
  - 충분한 메모리 있으면 유지
  
GC 동작:
  1. Strong Reference 확인
     강한 참조 있음 → GC 안 함
  
  2. 메모리 상태 확인
     충분함 → 유지
     부족함 → GC
  
  3. LRU 방식
     최근 사용 안 한 것부터 GC

사용법:
  SoftReference<Image> ref = new SoftReference<>(image);
  
  // 사용
  Image img = ref.get();  // Image 또는 null
  if (img == null) {
      img = loadImage();
      ref = new SoftReference<>(img);
  }

캐시 구현:
  Map<String, SoftReference<Image>> cache = new HashMap<>();
  
  Image getImage(String key) {
      SoftReference<Image> ref = cache.get(key);
      if (ref != null) {
          Image img = ref.get();
          if (img != null) return img;
      }
      
      // 캐시 미스 또는 GC됨
      Image img = loadImage(key);
      cache.put(key, new SoftReference<>(img));
      return img;
  }
```

---

### 4. Weak Reference

```java
WeakReference<Object> weakRef = new WeakReference<>(obj);

특징:
  - Strong Reference 없으면 즉시 GC
  - 메모리 부족과 무관
  
GC 동작:
  다음 GC 사이클에 무조건 제거
  
예시:
  Object obj = new Object();
  WeakReference<Object> weak = new WeakReference<>(obj);
  
  System.out.println(weak.get());  // Object
  
  obj = null;  // Strong Reference 제거
  System.gc();
  
  System.out.println(weak.get());  // null (GC됨)

사용:
  - WeakHashMap
  - 메타데이터, 캐싱
  - Listener 패턴 (메모리 누수 방지)
```

---

### 5. WeakHashMap

```java
WeakHashMap<Key, Value>

동작 원리:
  Key를 Weak Reference로 저장
  → Key에 대한 Strong Reference 없으면 GC
  → Entry 자동 제거

예시:
  WeakHashMap<User, Session> sessions = new WeakHashMap<>();
  
  User user = new User();
  sessions.put(user, new Session());
  
  // 사용 중
  System.out.println(sessions.size());  // 1
  
  user = null;  // Key에 대한 Strong Reference 제거
  System.gc();
  
  // GC 후
  System.out.println(sessions.size());  // 0 (자동 제거)

vs HashMap:
  HashMap<User, Session> sessions = new HashMap<>();
  
  user = null;
  System.gc();
  
  System.out.println(sessions.size());  // 1 (남아있음)
  // → 메모리 누수 (명시적 제거 필요)
```

---

### 6. Phantom Reference

```java
PhantomReference<Object> phantom = new PhantomReference<>(obj, queue);

특징:
  - get() 항상 null 반환
  - finalize() 이후 알림용
  - ReferenceQueue 필수

동작:
  1. 객체 Unreachable
  2. finalize() 실행 (있으면)
  3. Phantom Reference → ReferenceQueue 추가
  4. queue.poll()로 감지
  5. phantom.clear() 호출 후 객체 제거

용도:
  리소스 정리 (finalize()의 대안)
  
예시:
  ReferenceQueue<Resource> queue = new ReferenceQueue<>();
  PhantomReference<Resource> ref = new PhantomReference<>(resource, queue);
  
  // 모니터링 스레드
  while (true) {
      Reference<?> r = queue.remove();  // Blocking
      if (r == ref) {
          // Resource가 GC됨 → 정리 작업
          cleanupNativeResource();
          r.clear();
      }
  }
```

---

### 7. ReferenceQueue

```java
ReferenceQueue<Object> queue = new ReferenceQueue<>();
WeakReference<Object> ref = new WeakReference<>(obj, queue);

역할:
  GC된 Reference 알림
  
동작:
  1. 객체 GC됨
  2. Reference → ReferenceQueue 추가
  3. queue.poll() 또는 remove()로 확인
  
사용:
  Reference<?> r = queue.poll();  // Non-blocking
  Reference<?> r = queue.remove(); // Blocking
  Reference<?> r = queue.remove(timeout); // Timeout
  
  if (r != null) {
      // 객체가 GC됨 → 정리 작업
  }

예시 (리소스 추적):
  class TrackedResource {
      static Set<PhantomReference<Resource>> refs = new HashSet<>();
      static ReferenceQueue<Resource> queue = new ReferenceQueue<>();
      
      static void track(Resource r) {
          refs.add(new PhantomReference<>(r, queue));
      }
      
      static void cleanup() {
          Reference<?> ref;
          while ((ref = queue.poll()) != null) {
              // 리소스 정리
              refs.remove(ref);
              ref.clear();
          }
      }
  }
```

---

## 💻 실험으로 확인하기

### 실험 1: Soft Reference vs Weak Reference

```java
public class ReferenceTest {
    public static void main(String[] args) throws Exception {
        byte[] data = new byte[10 * 1024 * 1024];  // 10MB
        
        SoftReference<byte[]> soft = new SoftReference<>(data);
        WeakReference<byte[]> weak = new WeakReference<>(data);
        
        System.out.println("Before GC:");
        System.out.println("Soft: " + (soft.get() != null));  // true
        System.out.println("Weak: " + (weak.get() != null));  // true
        
        data = null;  // Strong Reference 제거
        System.gc();
        Thread.sleep(100);
        
        System.out.println("After GC (memory sufficient):");
        System.out.println("Soft: " + (soft.get() != null));  // true (메모리 충분)
        System.out.println("Weak: " + (weak.get() != null));  // false (즉시 GC)
        
        // 메모리 압박
        try {
            byte[][] pressure = new byte[1000][];
            for (int i = 0; i < 1000; i++) {
                pressure[i] = new byte[10 * 1024 * 1024];  // 10GB 할당 시도
            }
        } catch (OutOfMemoryError e) {
            System.out.println("OOM");
        }
        
        System.out.println("After memory pressure:");
        System.out.println("Soft: " + (soft.get() != null));  // false (메모리 부족)
    }
}
```

---

### 실험 2: WeakHashMap 동작

```java
import java.util.*;

public class WeakHashMapTest {
    static class Key {
        String name;
        Key(String name) { this.name = name; }
    }
    
    public static void main(String[] args) throws Exception {
        WeakHashMap<Key, String> weakMap = new WeakHashMap<>();
        HashMap<Key, String> strongMap = new HashMap<>();
        
        Key key1 = new Key("A");
        Key key2 = new Key("B");
        
        weakMap.put(key1, "Value A");
        weakMap.put(key2, "Value B");
        strongMap.put(key1, "Value A");
        strongMap.put(key2, "Value B");
        
        System.out.println("Initial:");
        System.out.println("WeakHashMap size: " + weakMap.size());    // 2
        System.out.println("HashMap size: " + strongMap.size());       // 2
        
        key1 = null;  // Key A에 대한 Strong Reference 제거
        System.gc();
        Thread.sleep(100);
        
        System.out.println("After GC:");
        System.out.println("WeakHashMap size: " + weakMap.size());    // 1 (A 제거)
        System.out.println("HashMap size: " + strongMap.size());       // 2 (유지)
    }
}
```

---

### 실험 3: ReferenceQueue 알림

```java
import java.lang.ref.*;

public class ReferenceQueueTest {
    public static void main(String[] args) throws Exception {
        ReferenceQueue<Object> queue = new ReferenceQueue<>();
        
        Object obj1 = new Object();
        Object obj2 = new Object();
        
        WeakReference<Object> ref1 = new WeakReference<>(obj1, queue);
        WeakReference<Object> ref2 = new WeakReference<>(obj2, queue);
        
        obj1 = null;  // ref1 GC 대상
        System.gc();
        Thread.sleep(100);
        
        Reference<?> r;
        while ((r = queue.poll()) != null) {
            System.out.println("GC detected: " + r);  // ref1
        }
        
        System.out.println("ref1.get(): " + ref1.get());  // null
        System.out.println("ref2.get(): " + ref2.get());  // Object (obj2 유지)
    }
}
```

---

## ⚡ 실무 임팩트

### 캐시 구현 — Soft Reference

```java
// 이미지 캐시
public class ImageCache {
    private Map<String, SoftReference<Image>> cache = new HashMap<>();
    
    public Image getImage(String path) {
        SoftReference<Image> ref = cache.get(path);
        
        if (ref != null) {
            Image img = ref.get();
            if (img != null) {
                return img;  // 캐시 히트
            }
        }
        
        // 캐시 미스 또는 GC됨
        Image img = loadImage(path);
        cache.put(path, new SoftReference<>(img));
        return img;
    }
    
    private Image loadImage(String path) {
        // 디스크/네트워크에서 로드 (느림)
        return ...;
    }
}

장점:
  - 메모리 부족 시 자동 제거
  - OOM 방지
  - 수동 관리 불필요
```

### Listener 패턴 — Weak Reference

```java
// ❌ 메모리 누수
public class EventBus {
    private List<Listener> listeners = new ArrayList<>();
    
    public void register(Listener listener) {
        listeners.add(listener);  // Strong Reference
    }
    // unregister() 안 하면 메모리 누수
}

// ✅ Weak Reference 사용
public class EventBus {
    private List<WeakReference<Listener>> listeners = new ArrayList<>();
    
    public void register(Listener listener) {
        listeners.add(new WeakReference<>(listener));
    }
    
    public void fire(Event event) {
        Iterator<WeakReference<Listener>> it = listeners.iterator();
        while (it.hasNext()) {
            Listener listener = it.next().get();
            if (listener == null) {
                it.remove();  // GC된 Listener 제거
            } else {
                listener.onEvent(event);
            }
        }
    }
}
```

### Phantom Reference — 리소스 정리

```java
// finalize() 대안
public class FileResource {
    private static ReferenceQueue<FileResource> queue = new ReferenceQueue<>();
    private static Set<PhantomReference<FileResource>> refs = new HashSet<>();
    
    static {
        // 정리 스레드
        Thread cleaner = new Thread(() -> {
            while (true) {
                try {
                    Reference<?> ref = queue.remove();
                    // 파일 핸들 정리
                    ((FilePhantomRef)ref).cleanup();
                    refs.remove(ref);
                } catch (InterruptedException e) {
                    break;
                }
            }
        });
        cleaner.setDaemon(true);
        cleaner.start();
    }
    
    private int fileHandle;
    
    public FileResource(String path) {
        this.fileHandle = openNative(path);
        refs.add(new FilePhantomRef(this, queue, fileHandle));
    }
    
    static class FilePhantomRef extends PhantomReference<FileResource> {
        private int handle;
        
        FilePhantomRef(FileResource ref, ReferenceQueue<FileResource> q, int handle) {
            super(ref, q);
            this.handle = handle;
        }
        
        void cleanup() {
            closeNative(handle);
        }
    }
    
    private native int openNative(String path);
    private static native void closeNative(int handle);
}
```

---

## 🚫 흔한 오해

### "Soft Reference는 절대 GC 안 된다"

```
❌ 잘못된 이해:
  Soft Reference는 메모리 부족 전까지 유지된다.

✅ 실제:
  LRU 방식으로 GC됨
  
  메모리 부족 시:
  - 최근 사용 안 한 Soft Reference부터 GC
  - Free Memory와 Last Access Time 고려
  
  알고리즘 (개략):
  timestamp - last_access_time > free_memory / constant
  → GC
  
  결과:
  - 자주 사용하는 캐시: 유지
  - 사용 안 하는 캐시: 제거
  
  완전히 안전하지는 않음:
  - OOM 직전까지 유지 가능
  - Weak Reference보다 aggressive한 제거 권장 시 사용
```

### "WeakHashMap은 Value도 Weak다"

```
❌ 잘못된 이해:
  WeakHashMap의 Key와 Value 모두 Weak Reference다.

✅ 실제:
  Key만 Weak Reference
  Value는 Strong Reference
  
  WeakHashMap<Key, Value>
  
  내부 구조:
  Entry {
      WeakReference<Key> key;
      Value value;  // Strong Reference
  }
  
  주의:
  Value가 Key를 Strong Reference하면 메모리 누수
  
  // ❌ 누수
  WeakHashMap<User, UserData> map = new WeakHashMap<>();
  User user = new User();
  UserData data = new UserData(user);  // user를 참조
  map.put(user, data);
  
  user = null;
  // → Key는 Weak이지만, Value(data)가 User 참조
  // → User가 GC 안 됨 (순환)
```

### "Phantom Reference는 객체를 부활시킨다"

```
❌ 잘못된 이해:
  Phantom Reference로 객체를 다시 살릴 수 있다.

✅ 실제:
  get()이 항상 null 반환 → 부활 불가능
  
  Weak Reference:
  Object obj = weakRef.get();  // Object 또는 null
  if (obj != null) {
      // 부활 가능 (다시 Strong Reference)
  }
  
  Phantom Reference:
  Object obj = phantomRef.get();  // 항상 null
  // 부활 불가능
  
  용도:
  객체 접근 아닌 "알림"만
  → 리소스 정리 목적
```

---

## 📌 핵심 정리

```
Reference 타입
  Strong: 일반 참조, GC 안 됨
  Soft: 메모리 부족 시 GC
  Weak: Strong Reference 없으면 GC
  Phantom: finalize() 이후 알림

Soft Reference
  캐시 구현에 적합
  LRU 방식으로 GC
  메모리 압박 시 자동 제거

Weak Reference
  WeakHashMap 구현
  Listener 패턴 (메모리 누수 방지)
  Strong Reference 없으면 즉시 GC

WeakHashMap
  Key를 Weak Reference로 저장
  Key GC 시 Entry 자동 제거
  Value는 Strong Reference (주의)

Phantom Reference
  get() 항상 null
  ReferenceQueue 필수
  finalize() 대안 (리소스 정리)

ReferenceQueue
  GC된 Reference 알림
  poll() / remove()로 확인
  정리 작업 트리거

실무 사용
  캐시: Soft Reference
  Listener: Weak Reference
  리소스 정리: Phantom Reference
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 캐시 구현 중 어느 것이 더 적합한가? 이유를 설명하라.
- Option A: `Map<String, SoftReference<Data>>`
- Option B: `Map<String, WeakReference<Data>>`

**Q2.** WeakHashMap에서 Value가 Key를 참조하면 어떤 문제가 발생하는가? 예시 코드와 함께 설명하라.

**Q3.** Phantom Reference가 finalize()보다 나은 이유 3가지를 설명하라.

> 💡 **해설**
>
> **Q1.** 캐시 용도에는 Soft Reference (Option A)가 더 적합. 이유: Weak Reference는 Strong Reference 없으면 다음 GC에 무조건 제거 → 캐시 효율 매우 낮음 (거의 항상 캐시 미스). Soft Reference는 메모리 충분하면 유지, 부족할 때만 LRU로 제거 → 캐시 히트율 높음. 단, Soft Reference도 OOM 직전까지 유지될 수 있어 메모리 제약 심하면 수동 관리 (크기 제한) 병행 권장.
>
> **Q2.** 순환 참조로 메모리 누수 발생. 예시: `WeakHashMap<User, UserSession> map; User user = new User(); UserSession session = new UserSession(user); map.put(user, session);`. user를 null로 설정해도: Key(user)는 Weak → GC 대상. 하지만 Value(session)가 user 참조 (Strong) → user가 GC 안 됨 → Entry 제거 안 됨. 해결: Value가 Key를 참조하지 않도록 설계. 또는 Value도 Weak Reference로 감싸기.
>
> **Q3.** ① 실행 시점 명확 — finalize()는 GC 후 언제 실행될지 불명확 (수 초~분). Phantom Reference + ReferenceQueue는 GC 직후 queue.remove()로 즉시 감지 가능. ② 부활 방지 — finalize()에서 this를 다시 참조하면 객체 부활 가능 (버그 유발). Phantom Reference는 get()이 항상 null → 부활 불가능. ③ 성능 — finalize() 있는 객체는 Finalization Queue 거쳐야 해서 GC 2회 필요. Phantom Reference는 별도 큐 사용해 GC 성능 영향 적음.

---

## 📚 참고 자료

- [Java Reference Objects](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/ref/package-summary.html)
- [Understanding Weak References](https://weblogs.java.net/blog/2006/05/04/understanding-weak-references)
- [WeakHashMap Internals](https://www.baeldung.com/java-weakhashmap)

---

<div align="center">

**[⬅️ 이전: GC Roots & Reachability](./01-gc-roots-and-reachability.md)** | **[다음: Mark-Sweep-Compact ➡️](./03-mark-sweep-compact.md)**

</div>

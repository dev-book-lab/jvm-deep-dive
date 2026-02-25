# Runtime Constant Pool - 런타임 상수 풀

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `.class` 파일의 Constant Pool과 Runtime Constant Pool은 어떻게 다른가?
- 문자열 리터럴 `"hello"`는 어디에 저장되며, `intern()`은 무엇을 하는가?
- 심볼릭 참조가 직접 참조로 바뀌는 Resolution은 언제 발생하는가?
- Runtime Constant Pool은 왜 클래스마다 별도로 존재하는가?
- Java 7 이전과 이후의 String Pool 위치 차이는 무엇이며, 왜 바뀌었는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 바이트코드는 메모리 주소를 직접 가질 수 없다

```java
public class Example {
    public void greet() {
        String msg = "Hello";
        System.out.println(msg);
    }
}
```

```
컴파일 시:
  javac는 Example.class를 생성
  greet() 메서드의 바이트코드를 작성
  
  문제:
  - "Hello" 문자열을 어디에 저장?
  - System.out의 실제 주소는?
  - println 메서드는 어떻게 찾을까?
  
  컴파일 타임에는 런타임 메모리 주소를 알 수 없음
  → 이름(심볼릭 참조)으로 저장해야 함
```

JVM은 이를 **Constant Pool**로 해결한다.

---

## 📐 내부 구조

### 1. Class File Constant Pool vs Runtime Constant Pool

```
.class 파일의 Constant Pool (컴파일 타임):

┌────────────────────────────────────────┐
│  Example.class                         │
├────────────────────────────────────────┤
│  Constant Pool:                        │
│  #1 = Methodref  #6.#15                │
│  #2 = String     #16                   │
│  #3 = Fieldref   #17.#18               │
│  #4 = Methodref  #19.#20               │
│  #5 = Class      #21                   │
│  #6 = Class      #22   // java/lang/Object
│  ...                                   │
│  #16 = Utf8      "Hello"               │
│  #17 = Class     #23   // java/lang/System
│  #18 = NameAndType #24:#25  // out:PrintStream
│  #19 = Class     #26   // java/io/PrintStream
│  #20 = NameAndType #27:#28  // println:(String)V
│  ...                                   │
└────────────────────────────────────────┘
        ↓ 클래스 로드 시
┌────────────────────────────────────────┐
│  Runtime Constant Pool (Metaspace)     │
├────────────────────────────────────────┤
│  #1 → Object.<init> (직접 참조)          │
│  #2 → String 인스턴스 "Hello" (Heap)     │
│  #3 → System.out 필드 (직접 참조)         │
│  #4 → PrintStream.println (직접 참조)    │
│  ...                                   │
└────────────────────────────────────────┘

변환 과정:
  심볼릭 참조 → Resolution → 직접 참조
```

---

### 2. Runtime Constant Pool의 내용

```
각 클래스마다 독립적인 Runtime Constant Pool:

┌──────────────────────────────────────────────┐
│  MyClass의 Runtime Constant Pool              │
├──────────────────────────────────────────────┤
│  1. 리터럴 상수                                 │
│     - 숫자: 42, 3.14, true                    │
│     - 문자열: "Hello", "World"                │
│                                              │
│  2. 심볼릭 참조 (Resolution 전)                 │
│     - 클래스/인터페이스: "java/lang/String"      │
│     - 필드: "System.out:PrintStream"          │
│     - 메서드: "println:(Ljava/lang/String;)V" │
│                                              │
│  3. 직접 참조 (Resolution 후)                   │
│     - 메서드 포인터: 0x7f3a2b10                  │
│     - 필드 오프셋: offset 24                    │
│     - 클래스 메타데이터: Klass* 0x...             │
└──────────────────────────────────────────────┘

접근:
  ldc #2  // Constant Pool[#2] 값을 Operand Stack에 push
```

---

### 3. 문자열 리터럴과 String Pool

```
Java 6 이하: String Pool in PermGen

┌─────────────────────────────────────┐
│          PermGen                    │
├─────────────────────────────────────┤
│  String Pool (Interned Strings)     │
│  ┌───────────────────────────────┐  │
│  │ "Hello" → String 인스턴스       │  │
│  │ "World" → String 인스턴스       │  │
│  │ ...                           │  │
│  └───────────────────────────────┘  │
│                                     │
│  Runtime Constant Pool              │
│  (각 클래스마다)                       │
└─────────────────────────────────────┘

문제:
  PermGen 크기 제한 → OOM 빈번
  String.intern() 과다 사용 시 PermGen 고갈

Java 7+: String Pool in Heap

┌─────────────────────────────────────┐
│          Heap                       │
├─────────────────────────────────────┤
│  String Pool                        │
│  ┌───────────────────────────────┐  │
│  │ "Hello" → String 인스턴스       │  │
│  │ "World" → String 인스턴스       │  │
│  └───────────────────────────────┘  │
│                                     │
│  일반 객체들                           │
│  ...                                │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│          Metaspace                  │
├─────────────────────────────────────┤
│  Runtime Constant Pool              │
│  (String Pool은 Heap을 참조)          │
└─────────────────────────────────────┘

장점:
  GC로 관리 → 사용하지 않는 String 제거 가능
  크기 제한 완화 (-XX:StringTableSize)
```

---

### 4. String.intern() 동작 원리

```java
String s1 = "hello";           // 리터럴 → String Pool
String s2 = "hello";           // 같은 리터럴 → 같은 인스턴스
s1 == s2;                      // true

String s3 = new String("hello"); // new → Heap의 새 인스턴스
s1 == s3;                        // false

String s4 = s3.intern();         // String Pool에서 찾거나 추가
s1 == s4;                        // true
```

```
intern() 동작:

1. String Pool에 동일한 문자열 존재?
   → 존재하면 그 인스턴스 반환
   
2. 존재하지 않으면?
   Java 6: String Pool에 복사본 생성 후 반환
   Java 7+: Heap의 현재 인스턴스를 Pool에 등록 후 반환
   
3. String Pool 구조:
   HashMap 형태
   키: 문자열 내용 (해시)
   값: String 인스턴스 참조
```

---

### 5. Resolution 시점과 Constant Pool

```
바이트코드 실행 흐름:

invokedynamic #42  // ConstantPool[#42] 참조

1. ConstantPool[#42]가 이미 Resolution됨?
   → 직접 참조 사용 (빠름)

2. 아직 심볼릭 참조?
   → Resolution 수행:
      - 클래스 로드 (필요 시)
      - 메서드/필드 탐색
      - 접근 권한 확인
      - 직접 참조로 교체
   → 이후 호출은 빠름 (이미 Resolution됨)

예:
  첫 호출: 심볼릭 참조 → Resolution (느림)
  이후 호출: 직접 참조 (빠름)
  
  이것이 "Warm-up"의 일부
```

---

## 💻 실험으로 확인하기

### 실험 1: javap로 Constant Pool 확인

```java
public class ConstantPoolDemo {
    public static void main(String[] args) {
        String s = "Hello";
        int x = 42;
        System.out.println(s + x);
    }
}
```

```bash
javac ConstantPoolDemo.java
javap -v ConstantPoolDemo.class

# 출력:
# Constant pool:
#    #1 = Methodref    #6.#20   // java/lang/Object."<init>":()V
#    #2 = String       #21      // Hello
#    #3 = Fieldref     #22.#23  // java/lang/System.out:Ljava/io/PrintStream;
#    #4 = Class        #24      // java/lang/StringBuilder
#    #5 = Methodref    #4.#20   // java/lang/StringBuilder."<init>":()V
#    ...
#   #21 = Utf8         Hello    ← 리터럴 "Hello"
#   #22 = Class        #27      // java/lang/System
#   #23 = NameAndType  #28:#29  // out:Ljava/io/PrintStream;
#   ...

# main 메서드 바이트코드:
#   0: ldc           #2   // String Hello ← ConstantPool[#2] 사용
#   2: astore_1
#   3: bipush        42
#   5: istore_2
#   ...
```

---

### 실험 2: String Pool 동작 확인

```java
public class StringPoolTest {
    public static void main(String[] args) {
        // 리터럴
        String s1 = "hello";
        String s2 = "hello";
        System.out.println("s1 == s2: " + (s1 == s2));  // true
        
        // new String
        String s3 = new String("hello");
        System.out.println("s1 == s3: " + (s1 == s3));  // false
        
        // intern()
        String s4 = s3.intern();
        System.out.println("s1 == s4: " + (s1 == s4));  // true
        
        // 런타임 생성 후 intern
        String s5 = new String("world");
        String s6 = s5.intern();
        String s7 = "world";
        System.out.println("s6 == s7: " + (s6 == s7));  // true
    }
}
```

---

### 실험 3: String Pool 크기와 성능

```java
public class StringPoolSizeBench {
    public static void main(String[] args) {
        long start = System.nanoTime();
        
        for (int i = 0; i < 100_000; i++) {
            String s = ("str" + i).intern();
        }
        
        long elapsed = System.nanoTime() - start;
        System.out.println("소요 시간: " + elapsed / 1_000_000 + " ms");
    }
}
```

```bash
# String Pool 크기 기본값 (60013)
java StringPoolSizeBench
# 출력: 소요 시간: ~500 ms

# String Pool 크기 증가
java -XX:StringTableSize=1000003 StringPoolSizeBench
# 출력: 소요 시간: ~300 ms (더 빠름)

# String Pool은 내부적으로 HashMap
# 크기가 크면 해시 충돌 감소 → 빠른 탐색
```

---

### 실험 4: String Pool GC 관찰

```java
public class StringPoolGCTest {
    public static void main(String[] args) throws Exception {
        // 대량 String intern
        for (int i = 0; i < 100_000; i++) {
            ("temp" + i).intern();
        }
        
        System.out.println("=== intern 완료 ===");
        System.gc();
        Thread.sleep(1000);
        
        // 사용하지 않는 String은 GC됨 (Java 7+)
        System.out.println("=== GC 후 ===");
    }
}
```

```bash
# String Pool GC 로그
java -Xlog:stringtable*=trace StringPoolGCTest

# 출력:
# [stringtable] Table size: 60013, number of entries: 100000
# [gc] GC(1) Pause Young (Normal)
# [stringtable] Dead strings: 99800, deduped: 0
# ← GC가 String Pool에서 사용하지 않는 String 제거
```

---

## ⚡ 실무 임팩트

### String.intern() 사용 시 주의사항

```java
// ❌ 안티패턴: 무분별한 intern()
for (String line : largeFile) {
    String trimmed = line.trim().intern();  // 수백만 String을 Pool에 추가
    // String Pool 크기 폭증 → GC 압박
}

// ✅ 제한된 값에만 intern()
enum Status { ACTIVE, INACTIVE }
String status = getStatus().intern();  // 값이 몇 개 안 되면 OK

// ✅ 명시적 캐시 사용
Map<String, String> cache = new HashMap<>();
String cached = cache.computeIfAbsent(key, k -> k);
// 스코프 제어 가능, GC 쉬움
```

### Constant Pool과 클래스 파일 크기

```java
// 많은 문자열 리터럴 → Constant Pool 크기 증가

public class Config {
    // ❌ 수백 개의 상수
    public static final String URL_1 = "http://...";
    public static final String URL_2 = "http://...";
    // ...
    // → .class 파일 크기 증가
    // → 클래스 로딩 시간 증가
}

// ✅ 외부 파일 (properties, YAML)
Properties props = new Properties();
props.load(new FileInputStream("config.properties"));
// → .class 파일 작아짐
// → 런타임에만 로드
```

### Lambda와 Constant Pool

```java
// 람다도 Constant Pool에 저장됨

public class LambdaPool {
    public void process() {
        list.forEach(x -> System.out.println(x));
        // → BootstrapMethods 속성에 람다 정보 저장
        // → invokedynamic으로 호출
    }
}
```

```bash
javap -v LambdaPool.class

# BootstrapMethods:
#   0: #27 REF_invokeStatic java/lang/invoke/LambdaMetafactory.metafactory:...
#      Method arguments:
#        #28 (Ljava/lang/Object;)V
#        #29 REF_invokeVirtual java/io/PrintStream.println:(Ljava/lang/Object;)V
#        #28 (Ljava/lang/Object;)V

# Constant Pool에 람다 관련 정보 저장
# 람다는 클래스 로딩 시 동적으로 생성
```

---

## 🚫 흔한 오해

### "String 리터럴은 Metaspace에 저장된다"

```
❌ 잘못된 이해:
  "Hello" 같은 문자열 리터럴이 Metaspace에 있다.

✅ 실제:
  Java 6: PermGen (Metaspace 전신)
  Java 7+: Heap
  
  Runtime Constant Pool (Metaspace)은
  String 인스턴스를 참조만 함
  실제 String 객체는 Heap
  
  "Hello" → Heap의 String 인스턴스
  Runtime Constant Pool → Heap 참조 보유
```

### "intern()은 항상 새 String을 생성한다"

```java
❌ 잘못된 이해:
  s.intern()은 새 String을 만든다.

✅ 실제:
  이미 Pool에 있으면 기존 인스턴스 반환
  없을 때만 등록 (Java 7+는 복사 없이 참조만 등록)

String s1 = new String("hello");
String s2 = s1.intern();  // Pool에 "hello" 없으면 s1 등록, 있으면 기존 반환

// Java 7+ 동작:
// Pool에 없을 때: s1의 참조를 Pool에 저장 (복사 X)
// Pool에 있을 때: 기존 인스턴스 반환
```

### "Constant Pool은 모든 클래스가 공유한다"

```
❌ 잘못된 이해:
  하나의 Constant Pool을 모든 클래스가 사용한다.

✅ 실제:
  각 클래스는 자신의 Runtime Constant Pool을 가짐
  
  ClassA의 Runtime Constant Pool (Metaspace)
  ClassB의 Runtime Constant Pool (Metaspace)
  ClassC의 Runtime Constant Pool (Metaspace)
  
  → 독립적으로 존재
  
  단, String Pool (Heap)은 JVM 전체가 공유
  "Hello" 리터럴이 여러 클래스에 있어도
  Heap에는 하나의 String 인스턴스만 존재
```

---

## 📌 핵심 정리

```
Constant Pool 종류
  Class File Constant Pool: 컴파일 타임, .class 파일 내
  Runtime Constant Pool: 런타임, Metaspace (클래스별)

Runtime Constant Pool 내용
  리터럴 상수 (숫자, 문자열)
  심볼릭 참조 (클래스, 필드, 메서드)
  Resolution 후 직접 참조

String Pool 변천
  Java 6: PermGen (고정 크기, OOM 빈번)
  Java 7+: Heap (GC 관리, 크기 유연)

String.intern()
  Pool에 같은 문자열 있으면 기존 반환
  없으면 등록 후 반환
  Java 7+: 복사 없이 참조만 등록

String Pool 크기
  -XX:StringTableSize=<n> (기본 60013)
  값이 많으면 크기 증가 권장 (소수 추천)

Resolution
  심볼릭 참조 → 직접 참조 변환
  첫 사용 시 발생 (지연 실행)
  이후 호출은 직접 참조 사용 (빠름)

실무 주의
  무분별한 intern() → String Pool 비대화
  대량 리터럴 → .class 파일 크기 증가
  외부 설정 파일 활용 권장
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 각 String이 어디에 저장되는지 설명하라. Heap, String Pool, Metaspace 중 정확한 위치를 지정하고 이유를 설명하라.

```java
String s1 = "hello";
String s2 = new String("world");
String s3 = s2.intern();
```

**Q2.** String Pool 크기가 60013(기본값)일 때와 1000003일 때 `intern()` 성능 차이가 발생하는 이유를 HashMap 충돌과 연결해 설명하라.

**Q3.** 다음 두 방법 중 어느 것이 메모리 효율적인가? Runtime Constant Pool과 String Pool 관점에서 비교하라.

```java
// 방법 1: 클래스 내 상수
public class ErrorMessages {
    public static final String ERR_001 = "Invalid input";
    public static final String ERR_002 = "Network timeout";
    // ... 1000개
}

// 방법 2: properties 파일
Properties errors = new Properties();
errors.load(new FileInputStream("errors.properties"));
```

> 💡 **해설**
>
> **Q1.** `s1 = "hello"`: "hello" String 인스턴스는 Heap의 String Pool에 저장. Runtime Constant Pool(Metaspace)은 이 Heap 주소를 참조만 함. `s2 = new String("world")`: "world" 리터럴은 String Pool에, `new`로 생성한 인스턴스는 Heap의 일반 영역(Pool 밖)에 따로 존재. `s3 = s2.intern()`: String Pool에서 "world" 찾음 (리터럴로 이미 존재). Pool의 "world" 인스턴스 반환. s3와 리터럴 "world"는 같은 인스턴스.
>
> **Q2.** String Pool은 내부적으로 HashMap으로 구현된다. 크기 60013일 때 100,000개 String을 intern하면 평균 버킷당 ~1.67개씩 충돌 발생. 탐색 시 링크드 리스트 순회 필요 → O(n). 크기 1000003으로 늘리면 평균 버킷당 ~0.1개 → 충돌 거의 없음 → O(1) 탐색. 따라서 intern() 성능이 2~3배 향상된다. 크기는 소수(prime number)로 설정하는 것이 해시 충돌을 최소화하는 데 유리하다.
>
> **Q3.** 방법 1(클래스 상수): 모든 문자열이 Constant Pool에 저장되어 .class 파일 크기 증가. 클래스 로딩 시 1000개 문자열이 모두 String Pool로 올라가 메모리 상주. 사용하지 않는 상수도 메모리 차지. 방법 2(properties): .class 파일에는 상수 없음 (작음). 런타임에 필요한 것만 로드. properties 값은 일반 HashMap에 저장되고 String Pool을 거치지 않으므로 GC로 관리 가능. 스코프 제어 용이. 결론: 값이 수십 개 이하면 방법 1, 수백 개 이상이거나 동적 변경이 필요하면 방법 2가 메모리 효율적.

---

## 📚 참고 자료

- [JVMS §4.4 — The Constant Pool](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html#jvms-4.4)
- [JVMS §5.1 — The Run-Time Constant Pool](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html#jvms-5.1)
- [JEP 192 — String Deduplication in G1](https://openjdk.org/jeps/192)

---

<div align="center">

**[⬅️ 이전: Method Area & Metaspace](./04-method-area-metaspace.md)** | **[다음: Object Layout In Memory ➡️](./06-object-layout-in-memory.md)**

</div>

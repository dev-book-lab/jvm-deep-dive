# String Pool & Interning - 문자열 풀과 인터닝

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- String Pool은 어디에 위치하며, 왜 이동했는가?
- `intern()`은 어떻게 동작하며, 언제 사용하는가?
- String Pool의 크기는 어떻게 조정하는가?
- `intern()` 비용은 얼마나 되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 문자열 중복

```java
String s1 = "hello";
String s2 = "hello";
String s3 = new String("hello");

메모리:
  s1, s2: 같은 인스턴스 (Pool)
  s3: 새 인스턴스 (Heap)
  
String Pool:
  중복 제거 → 메모리 절약
```

String Pool은 **문자열 재사용 메커니즘**이다.

---

## 📐 String Pool 구조

### 1. 위치 변화 (Java 7+)

```
Java 6:
  PermGen (Permanent Generation)
  - 크기 고정 (기본 64MB)
  - Full GC로만 회수
  - OutOfMemoryError 빈번

Java 7+:
  Heap (Young/Old Generation)
  - 크기 동적
  - Minor/Major GC로 회수
  - 유연성 증가

이유:
  PermGen 제약 제거
  동적 문자열 처리
```

---

### 2. 내부 구조

```
String Pool = HashTable

구조:
  HashMap<String, WeakReference<String>>
  
  Key: String 내용
  Value: String 인스턴스 참조

크기:
  기본: 60013 (Java 8+)
  조정: -XX:StringTableSize=N

충돌:
  Chaining (연결 리스트)
```

---

### 3. intern() 동작

```java
String s1 = new String("hello");
String s2 = s1.intern();

과정:
  1. String Pool에서 "hello" 검색
  2. 있으면: 기존 인스턴스 반환
  3. 없으면: 
     - Java 6: Pool에 복사
     - Java 7+: Heap 인스턴스 참조 추가
  
결과:
  s1 != s2 (Java 6)
  s1 == s2 (Java 7+)
```

---

## 💻 실험으로 확인하기

### 실험 1: String Pool 동작

```java
public class StringPoolTest {
    public static void main(String[] args) {
        String s1 = "hello";
        String s2 = "hello";
        String s3 = new String("hello");
        String s4 = s3.intern();
        
        System.out.println(s1 == s2);  // true (Pool)
        System.out.println(s1 == s3);  // false (Heap)
        System.out.println(s1 == s4);  // true (intern)
        
        System.out.println(System.identityHashCode(s1));
        System.out.println(System.identityHashCode(s2));  // 같음
        System.out.println(System.identityHashCode(s3));  // 다름
        System.out.println(System.identityHashCode(s4));  // s1과 같음
    }
}
```

---

### 실험 2: String Pool 크기

```bash
# 기본 크기 확인
java -XX:+PrintFlagsFinal -version | grep StringTableSize
# StringTableSize = 60013

# 크기 조정
java -XX:StringTableSize=100003 MyApp

# 통계 확인
jcmd <pid> VM.stringtable
# StringTable statistics:
# Number of buckets: 60013
# Number of entries: 12345
# Average bucket size: 0.21
```

---

## ⚡ 실무 임팩트

### intern() 사용 시나리오

```java
// ✅ 좋은 사용: 제한된 문자열 집합
enum Status {
    SUCCESS, FAILURE, PENDING
}

Map<String, Data> cache = new HashMap<>();
for (Status status : Status.values()) {
    cache.put(status.name().intern(), data);
}

// ❌ 나쁜 사용: 무제한 문자열
List<String> logs = readLogsFromFile();
for (String log : logs) {
    log.intern();  // Pool 무한 증가!
}
```

---

### String Pool 크기 튜닝

```bash
# 증상: 높은 String Pool 충돌
jcmd <pid> VM.stringtable
# Average bucket size: 5.2  ← 높음 (> 1)

# 해결: 크기 증가
-XX:StringTableSize=200003

# 재확인
# Average bucket size: 1.3  ← 개선
```

---

### 메모리 영향

```java
// Before intern (중복)
List<String> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    list.add(new String("status_" + (i % 10)));
}
// 메모리: ~100MB (100만 개 인스턴스)

// After intern (중복 제거)
List<String> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    list.add(("status_" + (i % 10)).intern());
}
// 메모리: ~1MB (10개 인스턴스 재사용)
```

---

## 🚫 흔한 오해

### "모든 String은 Pool에 있다"

```
❌ 잘못된 이해:
  모든 String이 자동으로 Pool

✅ 실제:
  리터럴만 자동 Pool
  
  String s1 = "hello";  // Pool
  String s2 = new String("hello");  // Heap
  String s3 = s1 + s2;  // Heap
  
  명시적 intern() 필요
```

---

### "intern()은 항상 좋다"

```
❌ 잘못된 이해:
  모든 String에 intern() 호출

✅ 실제:
  제한된 집합만
  
  좋은 사용:
  - Enum 값
  - 고정 코드
  - 제한된 상태
  
  나쁜 사용:
  - 사용자 입력
  - UUID
  - 타임스탬프
  - 로그 메시지
  
  → Pool 무한 증가 → Memory Leak
```

---

## 📌 핵심 정리

```
String Pool
  중복 제거 메커니즘
  HashTable 구조

위치
  Java 6: PermGen
  Java 7+: Heap

intern()
  Pool에 추가/조회
  Java 7+: 참조만 추가 (복사 없음)

크기
  기본: 60013 (Java 8+)
  조정: -XX:StringTableSize=N

비용
  검색: O(1) 평균
  충돌: O(n) 최악

사용 시나리오
  ✅ 제한된 문자열 집합
  ❌ 무제한 입력

튜닝
  Average bucket size < 2
  크기 증가로 충돌 감소

Memory Leak 주의
  무분별한 intern()
  → Pool 무한 증가
```

---

## 🤔 생각해볼 문제

**Q1.** Java 6과 Java 7+에서 `intern()` 동작의 차이를 설명하고, 메모리 관점에서 어느 것이 유리한가?

**Q2.** 다음 코드에서 메모리 누수가 발생하는 이유를 설명하라.

```java
while (true) {
    String uuid = UUID.randomUUID().toString();
    uuid.intern();
}
```

**Q3.** String Pool의 Average bucket size가 10이다. 이것이 성능에 미치는 영향과 해결 방법을 설명하라.

> 💡 **해설**
>
> **Q1.** intern() 차이: ① Java 6 — Pool은 PermGen, intern()시 문자열 복사 → PermGen에 저장. 단점: PermGen 제한 (64MB), 복사 비용. ② Java 7+ — Pool은 Heap, intern()시 참조만 추가 (복사 없음). 장점: 동적 크기, 복사 비용 없음, GC 자동 회수. ③ 유리한 것: Java 7+ — 메모리 효율적 (복사 없음), PermGen 제약 없음.
>
> **Q2.** 메모리 누수 이유: ① UUID는 매번 다른 값 → 중복 없음. ② intern() 호출 → Pool에 계속 추가. ③ Pool은 Strong Reference → GC 회수 안 됨. ④ 무한 루프 → Pool 무한 증가 → OutOfMemoryError. 해결: UUID 같은 무제한 값은 intern() 사용 금지.
>
> **Q3.** Average bucket size = 10 영향: ① String Pool은 HashTable → 충돌 시 Chaining. ② Bucket size 10 → 평균 10개 연결 → 검색 O(10). ③ 성능 저하: intern() 느림, 많은 문자열 처리 시 병목. 해결: StringTableSize 증가 — 기본 60013 → 600013 (10배) → Bucket size ~1로 감소 → O(1) 검색. 플래그: `-XX:StringTableSize=600013`.

---

## 📚 참고 자료

- [String Interning](https://www.baeldung.com/java-string-pool)
- [Java 7 String Pool](https://www.oracle.com/technical-resources/articles/java/string-performance.html)

---

<div align="center">

**[⬅️ 이전: Compressed Oops](./02-compressed-oops.md)** | **[다음: Unsafe API ➡️](./04-unsafe-api.md)**

</div>

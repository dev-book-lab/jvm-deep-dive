# Object Header & Mark Word - 객체 헤더와 마크 워드

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Object Header는 무엇이며, 어떻게 구성되는가?
- Mark Word의 64비트는 어떻게 사용되는가?
- 해시코드, Lock 상태, GC 나이는 어디에 저장되는가?
- Mark Word는 왜 상태에 따라 레이아웃이 바뀌는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 모든 객체는 메타데이터가 필요하다

```
객체의 정보:
  - 클래스 타입
  - 해시코드
  - Lock 상태
  - GC 나이

어디에 저장?
  → Object Header
  → 최소 공간으로 최대 정보
```

Object Header는 **객체의 메타데이터 저장소**다.

---

## 📐 Object Header 구조

### 1. 전체 레이아웃 (64bit JVM)

```
Java 객체 메모리 레이아웃:

┌─────────────────────────────────────┐
│       Mark Word (8 bytes)           │ ← 상태 정보
├─────────────────────────────────────┤
│    Class Pointer (4 bytes)          │ ← 클래스 메타데이터
├─────────────────────────────────────┤  (Compressed Oops)
│       Array Length (4 bytes)        │ ← 배열만 (optional)
├─────────────────────────────────────┤
│          Field Data                 │ ← 실제 필드
│              ...                    │
└─────────────────────────────────────┘

크기:
  일반 객체: 12 bytes (Mark + Class)
  배열: 16 bytes (Mark + Class + Length)
  + Padding (8 byte 정렬)
```

---

### 2. Mark Word 상태별 레이아웃

```
64bit Mark Word (상태에 따라 다름):

1. Unlocked (Normal):
┌──────────┬──────┬───┬──┬──┐
│ hash:25  │age:4 │0:1│01│  │ 64bit
└──────────┴──────┴───┴──┴──┘
 unused:25  age:4 biased lock:2

2. Biased Lock:
┌──────────┬────────┬─────┬──┬──┐
│thread:54 │epoch:2 │age:4│1 │01│
└──────────┴────────┴─────┴──┴──┘
 thread ptr:54  epoch:2  age:4  biased:1  lock:01

3. Lightweight Lock (Thin):
┌─────────────────────────┬──┐
│    lock record ptr:62   │00│
└─────────────────────────┴──┘

4. Heavyweight Lock (Fat):
┌─────────────────────────┬──┐
│   monitor ptr:62        │10│
└─────────────────────────┴──┘

5. GC Mark:
┌─────────────────────────┬──┐
│    (varies)             │11│
└─────────────────────────┴──┘

마지막 2bit (lock):
  01: Unlocked or Biased
  00: Lightweight Lock
  10: Heavyweight Lock
  11: GC Mark
```

---

### 3. 필드별 의미

```
hash (25~31 bit):
  - identityHashCode() 결과
  - 지연 생성 (호출 시)
  - 한 번 생성되면 고정

age (4 bit):
  - GC Young Generation 생존 횟수
  - 최대 15회
  - Tenuring Threshold

biased (1 bit):
  - Biased Lock 활성화 여부
  - 1: Biased
  - 0: Normal

lock (2 bit):
  - 현재 Lock 상태
  - 01/00/10/11

thread (54 bit):
  - Biased Lock의 소유 Thread ID

epoch (2 bit):
  - Biased Lock epoch (재바이어스)

lock record ptr (62 bit):
  - Thin Lock의 Stack 포인터

monitor ptr (62 bit):
  - Heavyweight Monitor 포인터
```

---

## 💻 실험으로 확인하기

### 실험 1: JOL로 Object Header 확인

```java
import org.openjdk.jol.info.ClassLayout;

public class ObjectHeaderTest {
    static class SimpleObject {
        int x;
        long y;
    }
    
    public static void main(String[] args) {
        SimpleObject obj = new SimpleObject();
        
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
    }
}
```

```bash
# 출력:
# SimpleObject object internals:
#  OFFSET  SIZE   TYPE DESCRIPTION               VALUE
#       0     8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
#       8     4        (object header: class)    0x0000c3e8
#      12     4    int SimpleObject.x            0
#      16     8   long SimpleObject.y            0
# Instance size: 24 bytes
# Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

---

### 실험 2: hashCode() 호출 전후

```java
public class HashCodeTest {
    public static void main(String[] args) {
        Object obj = new Object();
        
        System.out.println("Before hashCode:");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        
        System.out.println("\nhashCode: " + obj.hashCode());
        
        System.out.println("\nAfter hashCode:");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
    }
}
```

```bash
# 출력:
# Before hashCode:
# mark: 0x0000000000000001 (no hash)
#
# hashCode: 460141958
#
# After hashCode:
# mark: 0x000000001b5d8f01 (hash: 0x1b5d8f)  ← hash 생성됨
```

---

### 실험 3: Lock 상태 변화

```java
public class LockStateTest {
    public static void main(String[] args) throws Exception {
        Object obj = new Object();
        
        System.out.println("Unlocked:");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        
        synchronized (obj) {
            System.out.println("\nLocked:");
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        }
        
        System.out.println("\nAfter unlock:");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
    }
}
```

---

## ⚡ 실무 임팩트

### 메모리 오버헤드

```
객체 크기 계산:

class Point {
    int x;  // 4 bytes
    int y;  // 4 bytes
}

실제 크기:
  Mark Word: 8 bytes
  Class Pointer: 4 bytes (Compressed)
  Fields: 8 bytes (x + y)
  Padding: 4 bytes (24 → 24, 8의 배수)
  Total: 24 bytes

오버헤드:
  12 bytes (Header) / 24 bytes (Total) = 50%

작은 객체일수록 오버헤드 비율 높음
```

---

### 배열 오버헤드

```
int[] array = new int[10];

크기:
  Mark Word: 8 bytes
  Class Pointer: 4 bytes
  Array Length: 4 bytes
  Elements: 40 bytes (10 × 4)
  Total: 56 bytes

작은 배열:
  int[1] = 24 bytes (헤더 16 + 데이터 4 + 패딩 4)
  오버헤드: 16/24 = 67%
```

---

### Identity HashCode 충돌

```java
// ❌ 문제
class MyClass {
    int x;
    
    @Override
    public int hashCode() {
        return x;  // 필드 기반
    }
}

Object obj = new MyClass();
int hash1 = obj.hashCode();  // 필드 기반
int hash2 = System.identityHashCode(obj);  // Mark Word 기반

문제:
  hash1 != hash2
  → HashMap에서 혼란

// ✅ 해결
불변 객체에서만 hashCode 오버라이드
또는 identityHashCode 사용 금지
```

---

## 🚫 흔한 오해

### "모든 객체는 hashCode를 저장한다"

```
❌ 잘못된 이해:
  모든 객체 생성 시 hashCode 생성

✅ 실제:
  Lazy 생성
  
  Object obj = new Object();
  → hashCode 없음 (Mark Word에 공간만)
  
  obj.hashCode();
  → 이 시점에 생성
  → Mark Word에 저장
  
  메모리 절약
  (대부분 객체는 hashCode 안 씀)
```

---

### "GC age는 무한정 증가한다"

```
❌ 잘못된 이해:
  GC age가 계속 증가

✅ 실제:
  최대 15 (4 bit)
  
  age = 15 도달
  → Old Generation 이동 (Promotion)
  
  또는 Tenuring Threshold 도달
  (기본: 15, 조정 가능)
```

---

## 📌 핵심 정리

```
Object Header
  Mark Word (8 bytes)
  Class Pointer (4 bytes, Compressed)
  Array Length (4 bytes, 배열만)

Mark Word 상태
  Unlocked: hash, age, biased, lock
  Biased Lock: thread, epoch, age
  Thin Lock: lock record ptr
  Fat Lock: monitor ptr
  GC Mark: varies

필드
  hash: 25~31 bit (identityHashCode)
  age: 4 bit (GC 생존 횟수, 최대 15)
  biased: 1 bit (Biased Lock)
  lock: 2 bit (상태: 01/00/10/11)

Lazy 생성
  hashCode: 호출 시 생성
  Monitor: 경쟁 시 생성

메모리 오버헤드
  작은 객체: 50% 이상
  배열: 헤더 16 bytes
  8 byte 정렬 (Padding)

실무
  작은 객체 다수 → 메모리 낭비
  Flyweight Pattern 고려
  Primitive 배열 선호
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 객체의 실제 메모리 크기를 계산하라. (64bit JVM, Compressed Oops)

```java
class Person {
    byte age;
    boolean alive;
}
```

**Q2.** hashCode()를 호출한 객체에 Biased Lock을 적용할 수 없는 이유를 Mark Word 레이아웃으로 설명하라.

**Q3.** GC age가 4bit인 이유와, 이것이 Generational GC에 미치는 영향을 설명하라.

> 💡 **해설**
>
> **Q1.** Person 객체 크기: ① Mark Word: 8 bytes. ② Class Pointer: 4 bytes (Compressed Oops). ③ Fields: byte (1) + boolean (1) = 2 bytes. ④ Padding: 8의 배수 맞추기 → 8+4+2 = 14 → 16으로 (2 bytes padding). Total: 16 bytes. 오버헤드: 12 bytes / 16 bytes = 75% (필드는 2 bytes뿐).
>
> **Q2.** hashCode 후 Biased Lock 불가 이유: ① Unlocked 상태 Mark Word: hash (25bit) + age (4) + biased (1) + lock (2) = 32bit 사용. ② Biased Lock 상태: thread (54) + epoch (2) + age (4) + biased (1) + lock (2) = 63bit. ③ hash와 thread는 동시 저장 불가 (공간 충돌). ④ hashCode() 호출 → hash 저장됨 → Biased Lock 전환 시 hash 손실 → 불가능. ⑤ 따라서 hash 있으면 Thin/Fat Lock만 사용.
>
> **Q3.** 4bit age 이유와 영향: ① 4bit = 최대 15회 생존. ② 이유: Mark Word 공간 절약 (64bit 제약). Young GC는 빈번 (분당 수십 회) → 15회면 충분히 Old 이동 판단. ③ 영향: MaxTenuringThreshold=15 (기본). 15회 생존 → Old Gen 이동. 너무 크면 (예: 100) → Young Gen 압박, 너무 작으면 (예: 1) → 조기 Promotion. ④ 4bit = 적절한 균형점 (메모리 vs 성능).

---

## 📚 참고 자료

- [JOL (Java Object Layout)](https://openjdk.org/projects/code-tools/jol/)
- [HotSpot Object Model](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/oops/markWord.hpp)

---

<div align="center">

**[다음: Compressed Oops ➡️](./02-compressed-oops.md)**

</div>

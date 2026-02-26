# Happens-Before - 해펀스-비포어

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Happens-Before 관계는 무엇이며, 왜 중요한가?
- Happens-Before 8가지 규칙은 무엇인가?
- "실행 순서"와 "가시성 보장 순서"가 왜 다른가?
- 어떻게 Happens-Before로 동시성 안전성을 보장하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 메모리 가시성을 어떻게 보장할 것인가

```java
int a = 0;
boolean ready = false;

// Thread 1
a = 42;
ready = true;

// Thread 2
if (ready) {
    System.out.println(a);  // 42 보장?
}
```

```
문제:
  ready = true를 봤다고
  a = 42를 볼 수 있는가?
  
  CPU 캐시, 재정렬로 불확실

해결:
  Happens-Before 관계
  → 명확한 가시성 보장
```

Happens-Before는 **메모리 가시성의 규칙**이다.

---

## 📐 Happens-Before 규칙

### 1. Program Order Rule

```
단일 스레드 내에서는 프로그램 순서

int a = 1;  // A
int b = 2;  // B

A happens-before B (단일 스레드)
→ A 실행 후 B 실행 (논리적 순서)

주의:
  실제 실행 순서는 재정렬 가능
  하지만 관찰 시 프로그램 순서 보장
```

---

### 2. Monitor Lock Rule

```
synchronized의 unlock happens-before lock

Thread 1:
synchronized (lock) {
    x = 1;
}  // unlock

Thread 2:
synchronized (lock) {  // lock
    System.out.println(x);  // 1 보장
}

보장:
  Thread 1의 모든 변경
  → Thread 2가 확실히 봄
```

---

### 3. Volatile Variable Rule

```
volatile 쓰기 happens-before 읽기

volatile boolean ready = false;
int x = 0;

Thread 1:
x = 42;
ready = true;  // volatile 쓰기

Thread 2:
if (ready) {  // volatile 읽기
    System.out.println(x);  // 42 보장
}

보장:
  ready 쓰기 전 모든 변경
  → ready 읽기 후 보임
```

---

### 4. Thread Start Rule

```
Thread.start() happens-before 스레드 실행

int x = 0;

x = 42;
Thread t = new Thread(() -> {
    System.out.println(x);  // 42 보장
});
t.start();

보장:
  start() 전 모든 변경
  → 새 스레드가 봄
```

---

### 5. Thread Join Rule

```
스레드 종료 happens-before join() 리턴

int x = 0;

Thread t = new Thread(() -> {
    x = 42;
});
t.start();
t.join();

System.out.println(x);  // 42 보장

보장:
  join() 리턴 후
  → 종료된 스레드의 모든 변경 보임
```

---

### 6. Thread Interruption Rule

```
interrupt() happens-before 인터럽트 감지

Thread t = new Thread(() -> {
    while (!Thread.interrupted()) {
        // 작업
    }
});
t.start();
t.interrupt();  // 보장: interrupted() true 반환
```

---

### 7. Object Finalization Rule

```
생성자 완료 happens-before finalize()

class MyClass {
    final int x;
    
    MyClass() {
        x = 42;  // 생성자 완료
    }
    
    protected void finalize() {
        System.out.println(x);  // 42 보장
    }
}
```

---

### 8. Transitivity (전이성)

```
A happens-before B
B happens-before C
→ A happens-before C

예:
Thread 1:
x = 1;  // A
ready = true;  // B (volatile)

Thread 2:
if (ready) {  // C
    System.out.println(x);  // A happens-before C
}

연쇄:
  A happens-before B (Program Order)
  B happens-before C (Volatile Rule)
  → A happens-before C (Transitivity)
```

---

## 💻 실험으로 확인하기

### 실험 1: volatile 없이 문제 발생

```java
public class WithoutVolatile {
    private static int x = 0;
    private static boolean ready = false;
    
    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(() -> {
            x = 42;
            ready = true;
        });
        
        Thread t2 = new Thread(() -> {
            while (!ready) {
                // Spin
            }
            System.out.println(x);
        });
        
        t1.start();
        t2.start();
        t1.join();
        t2.join(5000);
    }
}
```

```bash
# 출력:
# 0 (가능)
# 또는 무한 루프
```

---

### 실험 2: volatile로 해결

```java
private static volatile boolean ready = false;
```

```bash
# 출력:
# 42 (보장)
```

---

### 실험 3: Thread.join() 보장

```java
public class JoinTest {
    private static int x = 0;
    
    public static void main(String[] args) throws Exception {
        Thread t = new Thread(() -> {
            x = 42;
        });
        t.start();
        t.join();
        
        System.out.println(x);  // 42 보장
    }
}
```

---

## ⚡ 실무 임팩트

### Double-Checked Locking

```java
// ❌ 잘못된 구현
class Singleton {
    private static Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

```
문제:
  instance = new Singleton() 재정렬
  1. 메모리 할당
  2. instance = 주소 (재정렬!)
  3. 생성자 실행
  
  Thread 1: instance = 주소 (생성자 전)
  Thread 2: instance != null → 사용 → NPE
```

```java
// ✅ 올바른 구현
private static volatile Singleton instance;
```

---

### Safe Publication

```java
class Data {
    private final int value;
    
    Data(int value) {
        this.value = value;
    }
}

// ❌ 불안전
Data data = new Data(42);

// ✅ 안전 (volatile)
volatile Data data = new Data(42);

// ✅ 안전 (synchronized)
synchronized (lock) {
    data = new Data(42);
}
```

---

## 🚫 흔한 오해

### "Happens-Before = 실행 순서"

```
❌ 잘못된 이해:
  A happens-before B
  → A가 B보다 먼저 실행

✅ 실제:
  가시성 보장일 뿐
  
  실행 순서: 재정렬 가능
  가시성: A의 결과를 B가 봄
  
  예:
  x = 1;  // A
  y = 2;  // B
  
  실행: B → A (재정렬)
  관찰: A happens-before B (보장)
```

---

### "volatile은 모든 변수를 보호한다"

```
❌ 잘못된 이해:
  volatile ready 하나면 충분

✅ 실제:
  volatile 전후만 보장
  
  int x = 0;
  volatile boolean ready = false;
  
  x = 1;
  ready = true;  // ← 여기 전까지만
  x = 2;  // ← 이건 보장 안 됨
  
  올바른 패턴:
  모든 변경 → volatile 쓰기
  volatile 읽기 → 사용
```

---

## 📌 핵심 정리

```
Happens-Before
  메모리 가시성 보장 규칙
  A happens-before B
  → A의 변경을 B가 확실히 봄

8가지 규칙
  1. Program Order (단일 스레드)
  2. Monitor Lock (synchronized)
  3. Volatile Variable
  4. Thread Start
  5. Thread Join
  6. Thread Interruption
  7. Object Finalization
  8. Transitivity (전이성)

실행 순서 vs 가시성
  실행 순서: 재정렬 가능
  가시성: Happens-Before 보장

실무 활용
  volatile: 플래그, 상태
  synchronized: 복합 연산
  final: 불변 객체
  
Double-Checked Locking
  volatile 필수
  재정렬 방지

Safe Publication
  volatile, synchronized, final
  생성자 탈출 금지
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 Thread 2가 x=0을 출력할 수 있는가? Happens-Before 규칙으로 설명하라.

```java
int x = 0;
volatile boolean ready = false;

// Thread 1
x = 42;
ready = true;

// Thread 2
if (ready) {
    System.out.println(x);
}
```

**Q2.** Happens-Before의 Transitivity를 이용해 다음 코드의 안전성을 증명하라.

```java
int a = 0, b = 0;
volatile int c = 0;

// Thread 1
a = 1;
b = 2;
c = 3;

// Thread 2
if (c == 3) {
    System.out.println(a + b);
}
```

**Q3.** Thread.join()을 사용하지 않고 스레드 간 데이터를 안전하게 전달하는 방법 3가지를 제시하라.

> 💡 **해설**
>
> **Q1.** 0 출력 불가능 (42 보장). 이유: ① Program Order Rule — Thread 1에서 `x=42` happens-before `ready=true`. ② Volatile Variable Rule — `ready=true` (쓰기) happens-before `if (ready)` (읽기). ③ Transitivity — `x=42` happens-before `ready=true` happens-before `if (ready)` → `x=42` happens-before `if (ready)` 블록. 따라서 Thread 2가 ready=true를 봤다면 x=42도 확실히 봄.
>
> **Q2.** 안전성 증명: ① `a=1` happens-before `b=2` (Program Order). ② `b=2` happens-before `c=3` (Program Order). ③ `c=3` (쓰기) happens-before `if (c==3)` (읽기, Volatile Rule). ④ Transitivity 적용: `a=1` → `b=2` → `c=3` → `if (c==3)`. 따라서 Thread 2가 c==3을 확인하면 a=1, b=2도 확실히 봄 → `a+b=3` 보장.
>
> **Q3.** join() 없이 안전한 전달: ① volatile 변수 — 데이터를 volatile 필드에 저장 후 volatile 플래그 설정. ② BlockingQueue — put() happens-before take() 보장. Thread 1: queue.put(data). Thread 2: data = queue.take(). ③ synchronized — Thread 1이 synchronized 블록에서 데이터 쓰기, Thread 2가 같은 lock으로 읽기 → Monitor Lock Rule 적용.

---

## 📚 참고 자료

- [JSR 133 FAQ](https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html)
- [The Java Memory Model](http://www.cs.umd.edu/~pugh/java/memoryModel/)

---

<div align="center">

**[⬅️ 이전: CPU Cache & Visibility Problem](./01-cpu-cache-and-visibility-problem.md)** | **[다음: Volatile Deep Dive ➡️](./03-volatile-deep-dive.md)**

</div>

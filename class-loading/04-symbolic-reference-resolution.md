# Symbolic Reference Resolution - 심볼릭 참조 해결

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `.class` 파일 안에 메서드 이름이 문자열로 저장되어 있다면, JVM은 실제 실행 시 어떻게 그 메서드를 찾는가?
- `NoSuchMethodError`와 `NoSuchMethodException`은 어떻게 다른가?
- 빌드 타임에 문제없이 컴파일됐는데 런타임에 `NoClassDefFoundError`가 나는 이유는?
- Resolution이 Linking 단계에서 즉시 실행되지 않는 이유는 무엇인가?
- 인터페이스 메서드 호출이 클래스 메서드 호출보다 느릴 수 있는 이유는?

---

## 🔍 왜 이게 존재하는가

### 문제: .class 파일은 메모리 주소를 알 수 없다

컴파일 타임과 런타임은 다른 세계다.

```
컴파일 타임:
  javac는 Service.process()가 어디에 있는지 모른다.
  JVM이 Service 클래스를 메모리 어느 위치에 로드할지 알 수 없기 때문.
  
  따라서 .class 파일에는 실제 주소 대신 이름(문자열)을 기록한다.
  "com/example/Service.process(Ljava/lang/String;)V"
  ← 이것이 심볼릭 참조(Symbolic Reference)

런타임:
  JVM이 Service를 Metaspace의 특정 위치에 로드함
  이제 process() 메서드의 실제 주소를 알 수 있음
  
  심볼릭 참조 → 실제 메모리 주소/오프셋으로 변환
  ← 이것이 Resolution
```

심볼릭 참조 시스템이 없으면 두 가지가 불가능해진다.

```
불가능해지는 것 1: 독립적 컴파일
  A.java와 B.java를 각각 컴파일할 수 없다.
  A가 B의 주소를 알려면 B를 먼저 메모리에 올려야 하고
  B가 A의 주소를 알려면 A를 먼저 올려야 하는 닭-달걀 문제.

불가능해지는 것 2: 동적 로딩
  애플리케이션 시작 시 모든 클래스를 미리 로드해야 한다.
  플러그인, 동적 로딩, 지연 초기화가 전부 불가능.
```

---

## 📐 내부 구조

### 1. ConstantPool — 심볼릭 참조의 저장소

모든 `.class` 파일은 ConstantPool(상수 풀)을 가진다.  
코드에서 참조하는 모든 클래스, 메서드, 필드, 문자열이 심볼릭 형태로 저장된다.

```
.class 파일의 ConstantPool 구조:

Index  Tag   Value
  #1   Utf8  "com/example/Service"
  #2   Utf8  "process"
  #3   Utf8  "(Ljava/lang/String;)V"
  #4   Class → #1          // com.example.Service 클래스 참조
  #5   NameAndType → #2,#3 // process(String):void 메서드 명세
  #6   Methodref → #4,#5   // Service.process(String) 참조 ← 심볼릭 참조
  #7   Utf8  "Hello"
  #8   String → #7         // "Hello" 문자열 리터럴
  ...

바이트코드에서 사용:
  invokevirtual #6   // #6을 통해 간접 참조
```

```bash
# 실제 ConstantPool 확인
javap -verbose MyClass.class

# 출력 예시:
# Constant pool:
#    #1 = Methodref    #6.#20    // java/lang/Object."<init>":()V
#    #2 = Fieldref     #5.#21    // MyClass.count:I
#    #3 = String       #22       // "hello"
#    #4 = Methodref    #23.#24   // com/example/Service.process:(Ljava/lang/String;)V
#    #5 = Class        #25       // MyClass
#    ...
```

---

### 2. Resolution 동작 원리

Resolution은 ConstantPool의 심볼릭 참조를 직접 참조로 교체한다.

```
Resolution 전:
  ConstantPool[#6] = Methodref {
      class: "com/example/Service"   ← 문자열
      name: "process"
      descriptor: "(Ljava/lang/String;)V"
  }
  
  invokevirtual #6  ← 매번 문자열 탐색 필요 (느림)

Resolution 후:
  ConstantPool[#6] = 0x7f3a2b10  ← 실제 메서드 테이블 오프셋 또는 포인터
  
  invokevirtual #6  ← 직접 주소 참조 (빠름)
```

#### Resolution이 지연(lazy) 실행되는 이유

```
질문: 왜 클래스 로딩 직후 모든 심볼릭 참조를 한꺼번에 해결하지 않는가?

이유 1: 순환 의존성
  A가 B를 참조하고 B가 A를 참조하는 경우
  A를 로드하는 중에 B의 Resolution을 시도하면
  B도 로드 중이어서 데드락 발생 가능

이유 2: 불필요한 로딩 방지
  한 클래스가 수백 개의 다른 클래스를 참조할 수 있다.
  모두 즉시 로드하면 실제 사용하지 않는 클래스도
  전부 메모리에 올라와 기동 시간 폭증

이유 3: 동적 로딩 지원
  런타임에 결정되는 클래스는 컴파일 타임에 알 수 없음
  Class.forName()으로 동적 로딩하는 경우
  해당 시점까지 Resolution을 미뤄야 함

결론:
  JVM 명세는 Resolution을 "사용 직전"에 수행하도록 허용
  실제로 대부분의 JVM 구현(HotSpot)은 lazy Resolution 사용
```

---

### 3. 참조 타입별 Resolution

#### 클래스/인터페이스 Resolution

```
심볼릭 참조: "com/example/Service"

Resolution 절차:
  1. 현재 클래스의 ClassLoader로 "com.example.Service" 로드 요청
  2. ClassLoader 계층에서 탐색 (Parent Delegation)
  3. 클래스 발견 → Class 객체 반환
  4. 접근 권한 확인 (public 이거나 같은 패키지)
  5. ConstantPool 업데이트

실패 케이스:
  ClassNotFoundException → NoClassDefFoundError (런타임 에러)
  접근 권한 없음 → IllegalAccessError
```

#### 필드 Resolution

```
심볼릭 참조: "com/example/Service.maxCount:I"
             클래스명.필드명:타입디스크립터

Resolution 절차:
  1. 클래스 "com/example/Service" 먼저 Resolution
  2. Service 클래스에서 "maxCount" 이름, 타입 "I(int)" 필드 탐색
  3. 없으면 부모 클래스들로 올라가며 탐색 (상속 체인)
  4. 없으면 구현한 인터페이스들에서 탐색
  5. 발견 → 필드 오프셋으로 교체
  6. 접근 권한 확인

실패 케이스:
  필드 없음 → NoSuchFieldError
  접근 권한 없음 → IllegalAccessError
```

#### 메서드 Resolution — invokevirtual vs invokeinterface

```
invokevirtual (클래스 메서드 호출):

  심볼릭 참조: "com/example/Animal.speak:()V"
  
  Resolution:
    1. Animal 클래스에서 speak() 탐색
    2. 없으면 부모 클래스 체인 탐색
    3. 발견 → vtable(가상 메서드 테이블) 인덱스로 교체
    
  실행 시:
    객체의 실제 타입(예: Dog)의 vtable에서 해당 인덱스 조회
    Dog.speak() 주소를 가져와 호출
    
  vtable 구조:
    Animal vtable:  [speak → Animal.speak]
    Dog vtable:     [speak → Dog.speak]     ← 오버라이드됨
    
    vtable의 speak() 인덱스는 Animal이든 Dog든 동일
    → 인덱스 하나만 알면 O(1) 다형성 호출 가능

invokeinterface (인터페이스 메서드 호출):

  심볼릭 참조: "com/example/Flyable.fly:()V"
  
  문제:
    클래스는 여러 인터페이스를 구현할 수 있음
    각 인터페이스의 메서드 인덱스를 고정할 수 없음
    
    class Bird implements Flyable, Singable {
        void fly() {...}
        void sing() {...}
    }
    
    Flyable의 fly()가 Bird vtable의 몇 번 인덱스인지
    컴파일 타임에 결정할 수 없음 (다중 인터페이스 구현 때문)
  
  해결:
    itable(인터페이스 메서드 테이블) 별도 관리
    실행 시마다 itable에서 인터페이스 탐색 → 메서드 주소 획득
    
  비용:
    vtable: 인덱스 직접 참조 → O(1)
    itable: 인터페이스 탐색 후 메서드 조회 → O(n) 또는 캐시 사용
    
    이것이 "인터페이스 호출이 클래스 호출보다 느릴 수 있다"는 근거
    단, JIT 컴파일러가 Inline Cache로 최적화해 실제 차이는 미미
```

---

### 4. Runtime Constant Pool 업데이트

Resolution 완료 후 ConstantPool 엔트리가 업데이트된다.

```
Before Resolution:
  CONSTANT_Methodref_info {
      class_index: 4       // → "com/example/Service"
      name_and_type_index: 5  // → "process:(Ljava/lang/String;)V"
  }

After Resolution (HotSpot 내부):
  직접 참조로 교체 (구현에 따라 다름)
  - 메서드 포인터
  - vtable/itable 인덱스
  - 필드 오프셋

재 Resolution 없음:
  한 번 해결된 심볼릭 참조는 캐시됨
  이후 같은 참조는 ConstantPool에서 즉시 직접 참조 사용
```

---

### 5. NoClassDefFoundError vs ClassNotFoundException

가장 많이 혼동하는 두 에러가 모두 Resolution과 관련 있다.

```
ClassNotFoundException (Checked Exception):
  발생 시점: Class.forName(), ClassLoader.loadClass() 호출 시
  의미: 해당 이름의 클래스를 ClassLoader가 찾지 못함
  원인: classpath에 jar가 없음, 클래스 이름 오타 등
  
  try {
      Class.forName("com.example.NonExistent");
  } catch (ClassNotFoundException e) {
      // 처리 가능 → 복구 로직 작성 가능
  }

NoClassDefFoundError (Error, Unchecked):
  발생 시점: Resolution 단계 — 이미 컴파일 시점에 존재했던 클래스를
             런타임에 찾을 수 없을 때
  의미: 컴파일 타임에는 있었는데 런타임에 없어짐
  원인: 
    ① 런타임 classpath에서 jar 누락
    ② 클래스 초기화 실패 (ExceptionInInitializerError 이후)
    ③ 서로 다른 ClassLoader 간 클래스 공유 문제
  
  new Service();  // ← Service가 컴파일 시 있었지만 런타임에 없음
  // → NoClassDefFoundError: com/example/Service
  
  // 처리하기 어려움 → 대부분 배포 문제 (jar 누락)
```

```
혼동하기 쉬운 시나리오:

컴파일:
  프로젝트에 mysql-connector.jar 포함 → 성공

런타임 (서버):
  mysql-connector.jar를 배포 패키지에 포함하지 않음
  
  → new Connection() 호출 시
     ClassNotFoundException?    → 아니다
     NoClassDefFoundError?      → 맞다!
  
  이유: 컴파일 시점에 분명히 존재했던 클래스라서
        Resolution이 실패하면 NoClassDefFoundError
```

---

## 💻 실험으로 확인하기

### 실험 1: ConstantPool 직접 분석

```java
// ResolutionDemo.java
public class ResolutionDemo {
    public static void main(String[] args) {
        java.util.ArrayList<String> list = new java.util.ArrayList<>();
        list.add("hello");
        System.out.println(list.size());
    }
}
```

```bash
javac ResolutionDemo.java
javap -verbose ResolutionDemo.class

# Constant pool 섹션에서 심볼릭 참조 확인:
# Constant pool:
#   #1 = Methodref   #7.#21  // java/lang/Object."<init>":()V
#   #2 = Class       #22     // java/util/ArrayList
#   #3 = Methodref   #2.#21  // java/util/ArrayList."<init>":()V
#   #4 = String      #23     // hello
#   #5 = Methodref   #2.#24  // java/util/ArrayList.add:(Ljava/lang/Object;)Z
#   #6 = Methodref   #2.#25  // java/util/ArrayList.size:()I
#   ...

# 각 invokevirtual, invokespecial 명령어가
# 위의 Methodref 인덱스를 통해 간접 참조하는 것 확인
```

---

### 실험 2: Resolution 실패 시 에러 종류 확인

```java
// ClassNotFoundDemo.java
public class ClassNotFoundDemo {
    public static void main(String[] args) {
        // Case 1: ClassNotFoundException (동적 로딩)
        try {
            Class.forName("com.example.NonExistent");
        } catch (ClassNotFoundException e) {
            System.out.println("Case 1 - ClassNotFoundException: " + e.getMessage());
        }

        // Case 2: NoClassDefFoundError (정적 참조 → Resolution 실패)
        // 이 케이스는 컴파일 후 classpath에서 jar를 제거해야 재현 가능
        // 아래는 개념 예시
        try {
            // RemovedClass가 컴파일 시 존재했지만 런타임에 없는 경우
            // new RemovedClass();
        } catch (NoClassDefFoundError e) {
            System.out.println("Case 2 - NoClassDefFoundError: " + e.getMessage());
        }
    }
}
```

NoClassDefFoundError 재현:

```bash
# 1. 두 클래스 컴파일
javac Helper.java Main.java

# 2. Helper.class 삭제 (런타임에 없도록)
rm Helper.class

# 3. Main 실행 → Resolution 실패
java Main
# Exception in thread "main" java.lang.NoClassDefFoundError: Helper
#   at Main.main(Main.java:3)
# Caused by: java.lang.ClassNotFoundException: Helper
```

---

### 실험 3: invokevirtual vs invokeinterface 바이트코드 비교

```java
// DispatchDemo.java
import java.util.List;
import java.util.ArrayList;

public class DispatchDemo {
    interface Greeter {
        void greet();
    }

    static class Hello implements Greeter {
        public void greet() { System.out.println("Hello"); }
        public void hello() { System.out.println("Hello!"); } // 클래스 메서드
    }

    public static void main(String[] args) {
        Hello h = new Hello();
        h.hello();       // invokevirtual  (클래스 메서드)

        Greeter g = new Hello();
        g.greet();       // invokeinterface (인터페이스 메서드)
    }
}
```

```bash
javac DispatchDemo.java
javap -c DispatchDemo.class

# 출력에서 차이 확인:
# h.hello() → invokevirtual #x // Hello.hello:()V
# g.greet() → invokeinterface #x, 1 // Greeter.greet:()V
#                             ^^^
#                             인터페이스 메서드임을 표시하는 count
```

---

### 실험 4: Resolution 타이밍 확인

```java
// LazyResolutionDemo.java
public class LazyResolutionDemo {

    static class Holder {
        // 이 클래스가 실제로 사용될 때만 Resolution 발생
        static void use() {
            System.out.println("Holder used");
        }
    }

    public static void main(String[] args) {
        System.out.println("시작");

        boolean useHolder = args.length > 0;

        if (useHolder) {
            Holder.use();  // 이 경로를 통해서만 Holder Resolution 발생
        }

        System.out.println("종료");
    }
}
```

```bash
# Holder 사용 안 함 → Holder.class 없어도 에러 없음
java LazyResolutionDemo
# 시작
# 종료

# Holder 사용 → Holder.class 없으면 NoClassDefFoundError
rm Holder.class (또는 다른 위치에서 실행)
java LazyResolutionDemo trigger
# 시작
# Exception in thread "main" java.lang.NoClassDefFoundError: Holder
```

---

## ⚡ 실무 임팩트

### 배포 시 classpath 불일치 문제 진단

```
증상: 로컬에서는 정상인데 서버에서 NoClassDefFoundError

진단 절차:
  1. 에러 메시지에서 누락 클래스명 확인
     NoClassDefFoundError: com/example/util/StringHelper
  
  2. 해당 클래스가 어느 jar에 있는지 확인
     # Linux/Mac
     find ~/.m2 -name "*.jar" | xargs -I{} jar tf {} | grep StringHelper
     
     # Maven 의존성 트리
     mvn dependency:tree | grep string-utils
  
  3. 서버 배포 패키지에 해당 jar 포함 여부 확인
     jar tf myapp.war | grep string-utils
  
  4. ClassLoader가 어느 경로를 탐색하는지 확인
     java -verbose:class MyApp 2>&1 | grep StringHelper
     # [Loaded com.example.util.StringHelper from ...]
```

### 인터페이스 vs 추상 클래스 성능 고려

```
invokeinterface vs invokevirtual 실제 성능 차이:

이론:
  invokeinterface: itable 탐색 필요 → 더 느림

실제 (JIT 최적화 적용 후):
  JIT의 Inline Cache: 한 번 호출한 메서드 주소를 캐시
  대부분의 경우 monomorphic call site (한 구현체만 사용)
  → 캐시 히트 시 invokevirtual과 동일한 성능

  벤치마크 결과 (JMH):
  단순 인터페이스 호출: invokevirtual 대비 ~2% 느린 수준
  실무에서 의미 있는 차이는 거의 없음

결론:
  인터페이스 사용 여부를 성능으로 결정하지 말 것
  설계 관점(유연성, 테스트 가능성)을 우선으로
  
  성능이 실제 문제라면 JMH로 측정 후 결정
  (JIT가 대부분 최적화함)
```

### 리플렉션과 Resolution

```java
// 리플렉션 메서드 호출도 내부적으로 Resolution 과정을 거침

Method method = Service.class.getMethod("process", String.class);
// → Resolution: Service 클래스에서 process(String) 탐색
// → 없으면 NoSuchMethodException (Checked Exception)

method.invoke(instance, "hello");
// → 실제 메서드 호출 (Resolution 결과 사용)

// 주의: getMethod vs getDeclaredMethod
Service.class.getMethod("process");     // public + 상속된 메서드 포함
Service.class.getDeclaredMethod("process"); // 현재 클래스에 선언된 메서드만
                                             // (private 포함, 상속 제외)
```

---

## 🚫 흔한 오해

### "NoClassDefFoundError = ClassNotFoundException"

```
❌ 잘못된 이해:
  두 에러 모두 클래스를 찾지 못한 것이니 같은 원인이다.

✅ 실제 차이:

  ClassNotFoundException (Exception):
    동적 로딩 중 발생 (Class.forName, loadClass)
    "처음부터 없는 클래스"
    Checked Exception → 복구 가능
    주로 플러그인 시스템, 동적 로딩에서
  
  NoClassDefFoundError (Error):
    Resolution 단계에서 발생
    "컴파일 시 있었는데 런타임에 없어진 클래스"
    또는 "클래스 초기화 실패 후 재사용 시도"
    Error → 복구 어려움
    주로 배포 문제, 환경 불일치에서
  
  진단 팁:
    NoClassDefFoundError의 Caused by를 보면
    ClassNotFoundException이 나오는 경우가 많음
    → 결국 ClassLoader가 못 찾은 것이지만 맥락이 다름
```

### "Resolution은 Linking 단계에서 즉시 완료된다"

```
❌ 잘못된 이해:
  Linking의 세 단계(Verification, Preparation, Resolution) 중
  Resolution은 Linking 시점에 전부 완료된다.

✅ 실제:
  JVM 명세는 Resolution을 지연(lazy) 실행 허용
  HotSpot JVM은 실제로 해당 참조가 처음 실행될 때까지 미룸
  
  이것이 LazyResolutionDemo 실험에서
  Holder.class 없어도 그 경로가 실행되지 않으면
  에러가 발생하지 않는 이유
```

### "invokevirtual은 항상 런타임에 다형성을 사용한다"

```
❌ 잘못된 이해:
  invokevirtual은 항상 vtable을 통해 동적 디스패치한다.

✅ 실제:
  JIT 컴파일러의 devirtualization 최적화:
  
  클래스가 final이면:
    오버라이드 불가 → 컴파일 타임에 메서드 결정 가능
    → invokevirtual이지만 직접 호출로 최적화
  
  Escape Analysis로 로컬 객체임이 확인되면:
    외부로 탈출하지 않는 객체는 타입이 확정적
    → 인라이닝까지 적용
  
  Profile 기반 최적화 (PGO):
    실제로 항상 같은 타입만 사용되면
    JIT가 해당 타입으로 특화된 코드 생성
    → Speculative Optimization (나중에 Deoptimization 가능)
```

---

## 📌 핵심 정리

```
심볼릭 참조란
  컴파일 타임에 실제 메모리 주소를 알 수 없어
  이름(문자열)으로 참조하는 방식
  ConstantPool에 저장됨

Resolution이란
  심볼릭 참조 → 직접 참조(포인터/오프셋)로 변환
  한 번 해결되면 캐시됨 (재 Resolution 없음)

Resolution 시점
  JVM 명세상 지연(lazy) 허용
  HotSpot은 해당 코드가 처음 실행될 때 Resolution

에러 구분
  ClassNotFoundException: 동적 로딩 실패, Checked, 복구 가능
  NoClassDefFoundError:  Resolution 실패, Error, 배포 문제가 주원인

invokevirtual vs invokeinterface
  invokevirtual: vtable 인덱스 → O(1) 다형성 호출
  invokeinterface: itable 탐색 → 원리상 느리지만
                   JIT Inline Cache로 실제 차이 미미

노클래스 에러 진단 순서
  에러 메시지 클래스명 확인
  → 어느 jar에 속하는지 파악
  → 배포 패키지에 해당 jar 포함 여부 확인
  → -verbose:class 로 ClassLoader 탐색 경로 점검
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 코드에서 `Holder.class`가 classpath에 없어도 정상 실행되는 경우와 그렇지 않은 경우를 설명하라. Resolution의 지연 실행 원리와 연결하라.

```java
public class Main {
    public static void main(String[] args) {
        if (System.getProperty("use.holder") != null) {
            new Holder().doSomething();
        }
        System.out.println("done");
    }
}
```

**Q2.** 다음 두 코드의 바이트코드 명령어(`invokevirtual` vs `invokeinterface`)가 다른 이유와, JIT 최적화 이후 실제 성능 차이가 거의 없는 이유를 함께 설명하라.

```java
ArrayList<String> list = new ArrayList<>();
list.size();          // Case A

List<String> list2 = new ArrayList<>();
list2.size();         // Case B
```

**Q3.** 마이크로서비스 환경에서 jar를 Thin JAR로 배포할 때 `NoClassDefFoundError`가 자주 발생한다. 빌드/배포 파이프라인에서 이를 사전에 방지하는 방법을 제안하라.

> 💡 **해설**
>
> **Q1.** `System.getProperty("use.holder") == null`인 경우 `new Holder()` 코드가 실행되지 않는다. HotSpot JVM은 lazy Resolution을 사용하므로 해당 바이트코드가 실제로 실행될 때 Resolution이 발생한다. 실행되지 않으면 Resolution 시도 자체가 없어 `Holder.class`가 없어도 `NoClassDefFoundError`가 발생하지 않는다. `-Dprop`을 설정해 if 블록이 실행되는 순간 Resolution이 시도되고, `Holder.class`가 없으면 그 시점에 `NoClassDefFoundError`가 발생한다.
>
> **Q2.** `list.size()`는 변수 타입이 `ArrayList`(클래스)이므로 `invokevirtual` 사용. `list2.size()`는 변수 타입이 `List`(인터페이스)이므로 `invokeinterface` 사용. 원리상 `invokeinterface`는 itable을 탐색해야 하지만, JIT는 Inline Cache를 통해 한 번 호출한 구현체의 메서드 주소를 캐시한다. `list2`가 항상 `ArrayList`라면 JIT가 이를 감지해 직접 `ArrayList.size()`를 호출하도록 최적화하며, 실제 성능 차이는 측정하기 어려운 수준(~1~2%)이다.
>
> **Q3.** ① 빌드 단계: Maven/Gradle의 `dependency:analyze` 또는 `dependency:tree`로 누락 의존성 탐지. ② 패키징 단계: `maven-dependency-plugin`의 `check` goal로 런타임 classpath 완전성 검증. ③ 테스트 단계: 실제 배포 환경과 동일한 classpath로 통합 테스트 실행 (Docker 컨테이너 활용). ④ CI 단계: `jdeps` 도구로 클래스 의존성 분석 후 누락 jar 자동 감지. ⑤ 런타임: `-verbose:class` 로그를 모니터링해 `NoClassDefFoundError` 발생 시 즉시 알림.

---

## 📚 참고 자료

- [JVMS §5.4 — Resolution](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html#jvms-5.4)
- [JVMS §6.5 — invokevirtual, invokeinterface](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-6.html#jvms-6.5.invokevirtual)
- [OpenJDK: constantPoolOop.cpp](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/oops/constantPool.cpp)

---

<div align="center">

**[⬅️ 이전: Bytecode Verification](./03-bytecode-verification.md)** | **[다음: Class Unloading ➡️](./05-class-unloading.md)**

</div>

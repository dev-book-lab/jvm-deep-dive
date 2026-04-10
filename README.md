<div align="center">

# ⚙️ JVM Deep Dive

**"JVM을 블랙박스가 아닌, 완전히 해부된 기계로 이해하기"**

<br/>

> *"Java 코드를 작성하는 것과, Java 코드가 어떻게 살아 움직이는지 아는 것은 다르다"*

바이트코드부터 GC 알고리즘, JIT 컴파일러, Java Memory Model까지  
**왜 이렇게 설계됐는가** 라는 질문으로 JVM 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Java](https://img.shields.io/badge/Java-8%2B-orange?style=flat-square&logo=openjdk)](https://www.java.com)
[![Docs](https://img.shields.io/badge/Docs-69개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

JVM에 관한 자료는 많습니다. 하지만 대부분은 **"무엇인가"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "GC는 메모리를 수거합니다" | GC Root 탐색 알고리즘과 Stop-The-World가 발생하는 정확한 시점 |
| "volatile은 가시성을 보장합니다" | 메모리 배리어가 CPU 캐시에 내리는 명령 수준까지 |
| "JIT는 코드를 최적화합니다" | Tiered Compilation 5단계, OSR, Deoptimization 메커니즘 |
| "synchronized는 무겁습니다" | Biased → Thin → Fat Lock 전이와 Mark Word 변화 |
| 이론 나열 | 실행 가능한 코드 + `javap`, `JFR`, `JMH` 실측 포함 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Class Loading](https://img.shields.io/badge/🔹_Class_Loading-ClassLoader_Hierarchy-E76F00?style=for-the-badge&logo=openjdk&logoColor=white)](./class-loading/01-classloader-hierarchy.md)
[![Runtime Data Areas](https://img.shields.io/badge/🔹_Runtime_Data-Heap_Structure-E76F00?style=for-the-badge&logo=openjdk&logoColor=white)](./runtime-data-areas/01-heap-structure.md)
[![Bytecode](https://img.shields.io/badge/🔹_Bytecode-Class_File_Format-E76F00?style=for-the-badge&logo=openjdk&logoColor=white)](./bytecode/01-class-file-format.md)
[![Execution Engine](https://img.shields.io/badge/🔹_Execution_Engine-Interpreter_Mechanism-E76F00?style=for-the-badge&logo=openjdk&logoColor=white)](./execution-engine/01-interpreter-mechanism.md)
[![GC](https://img.shields.io/badge/🔹_Garbage_Collection-GC_Roots_%26_Reachability-E76F00?style=for-the-badge&logo=openjdk&logoColor=white)](./garbage-collection/01-gc-roots-and-reachability.md)
[![JMM](https://img.shields.io/badge/🔹_Java_Memory_Model-CPU_Cache_%26_Visibility-E76F00?style=for-the-badge&logo=openjdk&logoColor=white)](./java-memory-model/01-cpu-cache-and-visibility-problem.md)
[![Concurrency](https://img.shields.io/badge/🔹_Concurrency_Internals-Object_Monitor-E76F00?style=for-the-badge&logo=openjdk&logoColor=white)](./concurrency-internals/01-object-monitor.md)
[![Performance](https://img.shields.io/badge/🔹_Performance_Tuning-JVM_Flags_Guide-E76F00?style=for-the-badge&logo=openjdk&logoColor=white)](./performance-tuning/01-jvm-flags-complete-guide.md)
[![Advanced](https://img.shields.io/badge/🔹_Advanced_Internals-Object_Header_%26_Mark_Word-E76F00?style=for-the-badge&logo=openjdk&logoColor=white)](./advanced-internals/01-object-header-and-mark-word.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 클래스 로딩 시스템 (Class Loading)

> **핵심 질문:** `new MyClass()` 를 호출하기 전, JVM은 무엇을 하는가?

<details>
<summary><b>클래스가 JVM에 올라오는 전체 과정 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. ClassLoader Hierarchy](./class-loading/01-classloader-hierarchy.md) | Bootstrap / Extension / Application 계층과 Parent Delegation Model |
| [02. Loading → Linking → Initializing](./class-loading/02-loading-linking-initializing.md) | 3단계 책임 분리, static 초기화 블록이 실행되는 정확한 시점 |
| [03. Bytecode Verification](./class-loading/03-bytecode-verification.md) | JVM이 .class 파일을 어떻게 신뢰하는가, Verifier 동작 원리 |
| [04. Symbolic Reference Resolution](./class-loading/04-symbolic-reference-resolution.md) | ConstantPool의 심볼릭 참조가 직접 참조로 변환되는 과정 |
| [05. Class Unloading](./class-loading/05-class-unloading.md) | 클래스가 언로딩되는 조건, ClassLoader 누수와 메모리 누수 |
| [06. Custom ClassLoader](./class-loading/06-custom-classloader.md) | `findClass()` vs `loadClass()`, 암호화된 클래스 런타임 복호화 |
| [07. ClassLoader Isolation](./class-loading/07-classloader-isolation.md) | 같은 클래스명이 두 ClassLoader에서 로드되면 `==` 결과는? |

</details>

<br/>

### 🔹 런타임 데이터 영역 (Runtime Data Areas)

> **핵심 질문:** 내 객체는 JVM 메모리 어디에, 어떤 모습으로 존재하는가?

<details>
<summary><b>JVM이 메모리를 나누고 관리하는 방식 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Heap Structure](./runtime-data-areas/01-heap-structure.md) | Eden / Survivor / Old Generation 물리적 구조, 객체 이동 조건 |
| [02. TLAB (Thread-Local Allocation Buffer)](./runtime-data-areas/02-tlab-thread-local-allocation.md) | TLAB가 없으면 생기는 경합, 스레드별 Eden 파티셔닝 원리 |
| [03. Stack And Frames](./runtime-data-areas/03-stack-and-frames.md) | Stack Frame 구조(LVA / Operand Stack / Frame Data), StackOverflowError 시점 |
| [04. Method Area & Metaspace](./runtime-data-areas/04-method-area-metaspace.md) | PermGen이 사라진 이유, Metaspace OOM 시나리오 |
| [05. Runtime Constant Pool](./runtime-data-areas/05-runtime-constant-pool.md) | 클래스 파일 상수풀 vs 런타임 상수풀, 문자열 리터럴의 위치 |
| [06. Object Layout In Memory](./runtime-data-areas/06-object-layout-in-memory.md) | Object Header + Instance Data + Padding, JOL로 실측 |
| [07. Off-Heap & Direct Memory](./runtime-data-areas/07-off-heap-direct-memory.md) | ByteBuffer, `sun.misc.Unsafe`, GC가 닿지 않는 메모리 |

</details>

<br/>

### 🔹 바이트코드 (Bytecode)

> **핵심 질문:** 내가 짠 Java 코드가 JVM의 언어로 어떻게 번역되는가?

<details>
<summary><b>Java 코드와 JVM 사이의 언어, 바이트코드 완전 분석 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Class File Format](./bytecode/01-class-file-format.md) | `.class` 파일 바이너리 구조, magic number부터 attributes까지 |
| [02. Bytecode Instruction Set](./bytecode/02-bytecode-instruction-set.md) | 200+ 명령어 카테고리 분류, 타입별 명령어 분리 이유 |
| [03. Operand Stack Mechanism](./bytecode/03-operand-stack-mechanism.md) | 스택 기반 VM vs 레지스터 기반 VM, 명령어가 스택에 하는 일 |
| [04. Method Invocation Instructions](./bytecode/04-method-invocation-instructions.md) | `invokevirtual` / `invokeinterface` / `invokespecial` / `invokestatic` 차이 |
| [05. Exception Handling Bytecode](./bytecode/05-exception-handling-bytecode.md) | try-catch-finally가 bytecode에서 Exception Table로 변환되는 방식 |
| [06. Lambda & InvokeDynamic](./bytecode/06-lambda-and-invokedynamic.md) | Lambda가 내부 클래스가 아닌 이유, `LambdaMetafactory` 동작 원리 |
| [07. Bytecode Manipulation (ASM)](./bytecode/07-bytecode-manipulation-asm.md) | ASM으로 런타임에 바이트코드 조작, AOP 구현 원리 |

</details>

<br/>

### 🔹 실행 엔진 (Execution Engine)

> **핵심 질문:** JVM은 bytecode를 어떻게 "빠르게" 실행하는가?

<details>
<summary><b>Interpreter에서 JIT까지, 코드가 실행되는 방식의 진화 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Interpreter Mechanism](./execution-engine/01-interpreter-mechanism.md) | Template Interpreter 구조, bytecode → 기계어 디스패치 테이블 |
| [02. JIT Compilation Basics](./execution-engine/02-jit-compilation-basics.md) | Warm-up 임계값, 컴파일 대상 선정 기준 (`-XX:+PrintCompilation`) |
| [03. Tiered Compilation](./execution-engine/03-tiered-compilation.md) | Level 0~4 전환 조건, C1 / C2 컴파일러 역할 분리 |
| [04. JIT Optimizations](./execution-engine/04-jit-optimizations.md) | Inlining, Escape Analysis, Loop Unrolling, Dead Code Elimination |
| [05. On-Stack Replacement (OSR)](./execution-engine/05-on-stack-replacement.md) | 이미 실행 중인 메서드를 JIT 버전으로 교체하는 메커니즘 |
| [06. Deoptimization](./execution-engine/06-deoptimization.md) | Speculative Optimization 실패 시 Interpreter로 복귀하는 과정 |
| [07. JVM Intrinsics](./execution-engine/07-intrinsics.md) | JVM이 특정 메서드를 CPU 명령어로 직접 대체하는 방식 |

</details>

<br/>

### 🔹 가비지 컬렉션 (Garbage Collection)

> **핵심 질문:** JVM은 어떻게 "죽은 객체"를 판단하고, 어떻게 제거하는가?

<details>
<summary><b>Serial GC부터 ZGC까지, GC 알고리즘의 진화와 원리 (11개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. GC Roots & Reachability](./garbage-collection/01-gc-roots-and-reachability.md) | GC Root 종류, 순환 참조가 왜 문제가 안 되는가 |
| [02. Reference Types](./garbage-collection/02-reference-types.md) | Strong / Soft / Weak / Phantom Reference별 GC 동작, `WeakHashMap` |
| [03. Mark-Sweep-Compact](./garbage-collection/03-mark-sweep-compact.md) | 3단계 알고리즘, Fragmentation 문제와 Compaction 비용 |
| [04. Generational Hypothesis](./garbage-collection/04-generational-hypothesis.md) | "대부분의 객체는 젊어서 죽는다"는 가설이 GC 설계에 미친 영향 |
| [05. Serial & Parallel GC](./garbage-collection/05-serial-parallel-gc.md) | 단순 GC 동작 원리, Stop-The-World 비용 |
| [06. CMS GC & Problems](./garbage-collection/06-cms-gc-and-problems.md) | Concurrent Mark의 혁신과 Concurrent Mode Failure 한계, G1 탄생 배경 |
| [07. G1 GC Deep Dive](./garbage-collection/07-g1-gc-deep-dive.md) | Region 기반 구조, Concurrent Marking → Evacuation, Pause Prediction Model |
| [08. ZGC Deep Dive](./garbage-collection/08-zgc-deep-dive.md) | Colored Pointer, Load Barrier, Concurrent Relocation — pause < 1ms 원리 |
| [09. Shenandoah GC](./garbage-collection/09-shenandoah-gc.md) | Brooks Pointer, ZGC와의 설계 철학 차이 |
| [10. GC Tuning Flags](./garbage-collection/10-gc-tuning-flags.md) | 실전에서 쓰는 JVM 플래그 완전 정리 |
| [11. GC Log Analysis](./garbage-collection/11-gc-log-analysis.md) | `-Xlog:gc*` 로그 해석, STW 시간 측정, 메모리 누수 탐지 |

</details>

<br/>

### 🔹 자바 메모리 모델 (Java Memory Model)

> **핵심 질문:** 멀티코어 CPU에서 Java 코드는 왜 예상과 다르게 동작하는가?

<details>
<summary><b>CPU 캐시부터 Happens-Before까지, 동시성의 근본 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. CPU Cache & Visibility Problem](./java-memory-model/01-cpu-cache-and-visibility-problem.md) | 캐시 계층 구조, 명령어 재정렬, JMM이 이 모든 것을 추상화하는 이유 |
| [02. Happens-Before](./java-memory-model/02-happens-before.md) | HB 규칙 8가지, "실행 순서"와 "가시성 보장 순서"가 다른 이유 |
| [03. Volatile Deep Dive](./java-memory-model/03-volatile-deep-dive.md) | volatile이 보장하는 것(가시성 + 재정렬 금지)과 보장 안 하는 것(원자성) |
| [04. Final Field Semantics](./java-memory-model/04-final-field-semantics.md) | 생성자 완료 후 final 필드가 보장되는 범위, 안전한 불변 객체 공개 |
| [05. Publication & Escape](./java-memory-model/05-publication-and-escape.md) | 객체가 "탈출"하는 경우, 안전한 공개(Safe Publication) 패턴 |
| [06. Synchronized Internals](./java-memory-model/06-synchronized-internals.md) | synchronized가 삽입하는 Memory Barrier, 모니터 락의 메모리 의미론 |
| [07. Memory Barriers](./java-memory-model/07-memory-barriers.md) | LoadLoad / StoreStore / LoadStore / StoreLoad 배리어와 CPU 명령어 |

</details>

<br/>

### 🔹 동시성 내부 구조 (Concurrency Internals)

> **핵심 질문:** `synchronized`와 `ReentrantLock`은 내부에서 어떻게 다른가?

<details>
<summary><b>Lock 메커니즘과 스레드 스케줄링의 실제 구현 (9개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Object Monitor](./concurrency-internals/01-object-monitor.md) | Monitor 구조, Entry Set / Wait Set, `wait()` / `notify()` 내부 동작 |
| [02. Lock: Biased → Thin → Fat](./concurrency-internals/02-lock-biased-thin-fat.md) | Mark Word 변화로 보는 Lock 상태 전이, Biased Lock deprecated 이유 |
| [03. CAS & Atomic Operations](./concurrency-internals/03-cas-and-atomic-operations.md) | CPU의 `CMPXCHG` 명령어, ABA 문제, AtomicInteger 내부 구현 |
| [04. False Sharing & Cache Line](./concurrency-internals/04-false-sharing-and-cache-line.md) | 64바이트 캐시라인, `@Contended`, JMH로 False Sharing 실측 |
| [05. AQS Internals](./concurrency-internals/05-aqs-internals.md) | CLH Queue, `ReentrantLock` / `Semaphore` / `CountDownLatch` 공통 기반 |
| [06. Thread States & Scheduler](./concurrency-internals/06-thread-states-and-scheduler.md) | OS 스레드 상태 vs JVM 스레드 상태, Context Switching 비용 |
| [07. ThreadLocal Internals](./concurrency-internals/07-thread-local-internals.md) | `ThreadLocalMap` 내부 구조, 메모리 누수 발생 조건 |
| [08. Virtual Threads (Project Loom)](./concurrency-internals/08-virtual-threads-loom.md) | Carrier Thread, Structured Concurrency, pinning 주의사항 |
| [09. Safepoint Mechanism](./concurrency-internals/09-safepoint-mechanism.md) | Safepoint가 필요한 이유, Time-To-Safepoint 지연 원인과 분석 |

</details>

<br/>

### 🔹 성능 튜닝 (Performance Tuning)

> **핵심 질문:** JVM을 어떻게 측정하고, 어떻게 최적화하는가?

<details>
<summary><b>JFR, async-profiler, JMH로 JVM을 실측하고 개선하는 방법 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. JVM Flags Complete Guide](./performance-tuning/01-jvm-flags-complete-guide.md) | 실전에서 쓰는 플래그 전체 정리 (`-Xms`, `-Xmx`, `-XX:*`) |
| [02. Heap Sizing Strategy](./performance-tuning/02-heap-sizing-strategy.md) | Initial / Max Heap 비율, Young/Old 비율, 컨테이너 환경 주의사항 |
| [03. GC Ergonomics](./performance-tuning/03-gc-ergonomics.md) | JVM이 스스로 GC와 힙 크기를 조정하는 자동 튜닝 원리 |
| [04. Profiling with JFR](./performance-tuning/04-profiling-with-jfr.md) | Java Flight Recorder + JDK Mission Control, Flame Graph 읽기 |
| [05. Profiling with async-profiler](./performance-tuning/05-profiling-with-async-profiler.md) | CPU / 메모리 / 락 프로파일링, `alloc` 모드로 GC 압박 찾기 |
| [06. Memory Leak Analysis](./performance-tuning/06-memory-leak-analysis.md) | Heap Dump 분석, 누수 패턴 (static, ThreadLocal, ClassLoader) |
| [07. Benchmarking with JMH](./performance-tuning/07-benchmarking-with-jmh.md) | 왜 `System.nanoTime()`은 부정확한가, Warm-up / Blackhole / @State |

</details>

<br/>

### 🔹 JVM 내부 심화 (Advanced Internals)

> **핵심 질문:** JVM이 숨기고 있는 더 깊은 층에는 무엇이 있는가?

<details>
<summary><b>Mark Word, Compressed Oops, Java Agent까지 — JVM의 가장 깊은 곳 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Object Header & Mark Word](./advanced-internals/01-object-header-and-mark-word.md) | 64비트 Mark Word 레이아웃, 해시코드 / Lock 상태 / GC 나이 필드 |
| [02. Compressed Oops](./advanced-internals/02-compressed-oops.md) | 64비트 JVM에서 포인터를 32비트로 압축하는 원리, 32GB 힙 제한 이유 |
| [03. String Pool & Interning](./advanced-internals/03-string-pool-interning.md) | 문자열 상수풀 위치 변화 (PermGen → Heap), `intern()` 비용 |
| [04. Unsafe API](./advanced-internals/04-unsafe-api.md) | `sun.misc.Unsafe`로 직접 메모리 조작, JDK 내부 코드가 쓰는 이유 |
| [05. Reflection & Performance](./advanced-internals/05-reflection-and-performance.md) | Reflection 호출 경로, 15회 임계값 후 바이트코드 생성, JIT와의 관계 |
| [06. Instrumentation & Java Agent](./advanced-internals/06-instrumentation-and-agent.md) | `-javaagent` 동작 원리, `ClassFileTransformer`로 클래스 변환 |
| [07. JNI Internals](./advanced-internals/07-jni-internals.md) | JVM ↔ Native 코드 경계, JNI 호출 비용, Global / Local Reference |

</details>

---

## 🗺️ 목적별 학습 경로

<details>
<summary><b>🟢 Java 개발을 시작하는 분 / 기술 면접 준비 (3~4주)</b></summary>

<br/>

**Week 1 — JVM 메모리 구조 확립**
```
heap-structure
stack-and-frames
object-layout-in-memory
method-area-metaspace
```

**Week 2 — GC 원리 이해**
```
gc-roots-and-reachability
generational-hypothesis
g1-gc-deep-dive
gc-log-analysis
```

**Week 3 — 동시성과 JMM**
```
cpu-cache-and-visibility-problem
happens-before
volatile-deep-dive
lock-biased-thin-fat
```

**Week 4 — 실전 튜닝**
```
jvm-flags-complete-guide
heap-sizing-strategy
benchmarking-with-jmh
```

</details>

<details>
<summary><b>🔵 Java 코드를 원리로 이해하고 싶은 개발자 (6~8주)</b></summary>

<br/>

```
class-loading 전체
→ runtime-data-areas 전체
→ bytecode 전체 (javap로 직접 분석)
→ execution-engine 전체 (JIT 동작 실측)
→ garbage-collection 전체
```

</details>

<details>
<summary><b>🔴 JVM 마스터 목표 (3~4개월)</b></summary>

<br/>

```
전체 순서대로 + code/ 예제 직접 실행
→ advanced-internals 챕터로 마무리
→ JFR + async-profiler로 실제 프로젝트 분석
```

</details>

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이게 존재하는가** | 문제 상황과 설계 배경 |
| 📐 **내부 구조** | 원리 + 다이어그램 |
| 💻 **실험으로 확인하기** | 직접 실행 가능한 코드 + 예상 결과 + 측정 도구 |
| ⚡ **실무 임팩트** | 이 지식이 실제 코드 작성 / 장애 대응에 어떤 영향을 주는가 |
| 🚫 **흔한 오해** | 잘못 알려진 내용 교정 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🙏 Reference

- [Java Virtual Machine Specification (JVMS)](https://docs.oracle.com/javase/specs/jvms/se21/html/index.html)
- [Inside the Java Virtual Machine — Bill Venners](https://www.artima.com/insidejvm/ed2/)
- [The Garbage Collection Handbook — Richard Jones](https://gchandbook.org/)
- [JEP Index](https://openjdk.org/jeps/0)
- [OpenJDK Source Code](https://github.com/openjdk/jdk)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

**"Java 코드를 작성하는 것과, Java 코드가 어떻게 살아 움직이는지 아는 것은 다르다"**

</div>

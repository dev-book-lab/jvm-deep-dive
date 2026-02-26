# Shenandoah GC - 셰넌도어 GC

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Shenandoah GC는 무엇이며, ZGC와 어떻게 다른가?
- Brooks Pointer는 무엇이며, 왜 사용하는가?
- Shenandoah의 Concurrent Evacuation은 어떻게 동작하는가?
- ZGC vs Shenandoah 선택 기준은 무엇인가?

---

## 🔍 왜 이게 존재하는가

### 문제: Red Hat의 Low Latency 요구

```
Red Hat 고객 요구사항:
  - Large Heap (수백 GB)
  - Low Pause (< 10ms)
  - Java 8 지원 필요

ZGC 한계:
  - Java 11+ 전용
  - Red Hat 라이센스 이슈

해결:
  독자적인 Low Latency GC 개발
  → Shenandoah
```

Shenandoah는 **ZGC와 다른 접근**의 Low Latency GC다.

---

## 📐 내부 구조

### 1. Shenandoah 개요

```
활성화:
  -XX:+UseShenandoahGC (Java 12+)

특징:
  - Pause < 10ms
  - Brooks Pointer (간접 참조)
  - Concurrent Evacuation
  - Region 기반

차이점 (vs ZGC):
  - Brooks Pointer vs Colored Pointer
  - 모든 플랫폼 지원 (32/64bit)
  - Java 8 백포트 가능

목표:
  ZGC와 동일 (Low Latency)
  구현 방식 다름
```

---

### 2. Brooks Pointer

```
Brooks Pointer: 간접 참조 헤더

일반 객체:
  [Object Header][Fields...]

Shenandoah 객체:
  [Brooks Pointer][Object Header][Fields...]
   ↑ 8 bytes (자기 자신 또는 Forwarding)

초기:
  Brooks Pointer → 자기 자신
  
  0x1000: [ptr=0x1000][header][fields]
          ↑ 자기 참조

Relocation 후:
  Old: 0x1000: [ptr=0x2000][header][fields]
                ↑ New 주소
  New: 0x2000: [ptr=0x2000][header][fields]
                ↑ 자기 참조

접근:
  obj = field;  // field = 0x1000
  actual = *obj;  // Brooks Pointer 따라감
  → 0x2000 (Forwarding)

장점:
  - 단순한 구현
  - 32bit 지원
  
단점:
  - 메모리 오버헤드 (객체당 8 bytes)
  - 간접 참조 오버헤드
```

---

### 3. Shenandoah 단계

```
1. Init Mark (STW)
   GC Root 스캔
   시간: < 1ms

2. Concurrent Mark
   Reachable 객체 Mark
   시간: Concurrent

3. Final Mark (STW)
   누락 객체 보정
   시간: < 1ms

4. Concurrent Cleanup
   빈 Region 회수
   시간: Concurrent

5. Concurrent Evacuation
   객체 복사 (Brooks Pointer 사용)
   시간: Concurrent

6. Init Update Refs (STW)
   Reference 업데이트 시작
   시간: < 1ms

7. Concurrent Update Refs
   모든 포인터 업데이트
   시간: Concurrent

8. Final Update Refs (STW)
   GC Root 포인터 업데이트
   시간: < 1ms

STW: 4개 (각 < 1ms)
→ 총 Pause < 10ms
```

---

### 4. Concurrent Evacuation

```
Brooks Pointer 활용:

1. Evacuation 시작
   Region A 선택 (Garbage 많음)

2. 객체 복사 (Concurrent)
   Old: 0x1000 [ptr=0x1000][obj]
   New: 0x2000 [ptr=0x2000][obj]

3. Brooks Pointer 업데이트
   Old: 0x1000 [ptr=0x2000][obj]
                ↑ New로 Forwarding

4. 애플리케이션 접근
   field = 0x1000
   actual = *(field)  // Brooks Pointer
   → 0x2000 (자동 Forwarding)

5. Concurrent Update Refs
   모든 포인터를 0x2000으로 업데이트
   field = 0x2000

6. Old Region 회수
   0x1000 Region 제거

특징:
  - ZGC의 Load Barrier 대신
  - Brooks Pointer로 간접 참조
  - 단순하지만 메모리 오버헤드
```

---

### 5. ZGC vs Shenandoah 비교

```
Colored Pointer (ZGC):
  포인터: [Metadata 4bit][Address 42bit]
  장점: 메모리 효율, Atomic 업데이트
  단점: 64bit 전용, 복잡한 구현

Brooks Pointer (Shenandoah):
  객체: [Forwarding 8byte][Header][Fields]
  장점: 단순 구현, 32/64bit 모두
  단점: 메모리 오버헤드, 간접 참조

Load Barrier:
  ZGC: 모든 로드에 체크
  Shenandoah: Brooks Pointer 자동

성능:
  ZGC: Throughput 95% (G1 대비)
  Shenandoah: Throughput 90~93%
  
Pause:
  둘 다 < 10ms (비슷)

플랫폼:
  ZGC: 64bit Linux/macOS/Windows
  Shenandoah: 32/64bit, 더 넓음
```

---

## 💻 실험으로 확인하기

### 실험 1: Shenandoah Pause Time

```bash
java -Xmx8g \
     -XX:+UseShenandoahGC \
     -Xlog:gc*:file=shenandoah.log \
     MyApp

# shenandoah.log:
# [gc] GC(0) Pause Init Mark 0.6ms
# [gc] GC(0) Concurrent Mark 1500ms
# [gc] GC(0) Pause Final Mark 0.9ms
# [gc] GC(0) Concurrent Evacuation 800ms
# [gc] GC(0) Pause Init Update Refs 0.4ms
# [gc] GC(0) Concurrent Update Refs 600ms
# [gc] GC(0) Pause Final Update Refs 0.7ms
#
# 총 Pause: 2.6ms
```

---

## ⚡ 실무 임팩트

### Shenandoah 사용 케이스

```
적합:
  - 32bit 환경
  - Java 8 필요 (백포트)
  - OpenJDK 사용
  - Red Hat 에코시스템

ZGC 선택:
  - 64bit 환경
  - Java 11+
  - 약간 더 나은 Throughput

선택 기준:
  플랫폼 제약: Shenandoah
  성능 최우선: ZGC
  큰 차이 없음: 둘 다 OK
```

---

### 튜닝

```
기본:
  -XX:+UseShenandoahGC
  -Xmx16g

Heuristics (휴리스틱):
  -XX:ShenandoahGCHeuristics=adaptive (기본)
  -XX:ShenandoahGCHeuristics=static
  -XX:ShenandoahGCHeuristics=compact
  
  adaptive: 자동 조정
  static: 고정 임계값
  compact: Compaction 우선

Concurrent 스레드:
  -XX:ConcGCThreads=4
```

---

## 📌 핵심 정리

```
Shenandoah GC
  Red Hat 개발
  Low Latency (< 10ms)
  Java 12+, Java 8 백포트

Brooks Pointer
  객체당 8byte Forwarding Pointer
  간접 참조로 Concurrent Evacuation
  메모리 오버헤드

Concurrent Evacuation
  객체 복사 중 애플리케이션 실행
  Brooks Pointer로 자동 Forwarding

vs ZGC
  구현 방식 다름 (Brooks vs Colored)
  성능 비슷 (Pause < 10ms)
  Shenandoah: 32bit 지원, 약간 느림
  ZGC: 64bit 전용, 약간 빠름

선택
  플랫폼 제약: Shenandoah
  성능: ZGC
  실무: 큰 차이 없음
```

---

## 🤔 생각해볼 문제

**Q1.** Brooks Pointer가 객체당 8바이트 오버헤드를 발생시킨다. 1억 개 객체를 가진 힙에서 총 메모리 오버헤드는 얼마이며, 이것이 실무에 미치는 영향을 설명하라.

**Q2.** ZGC의 Colored Pointer와 Shenandoah의 Brooks Pointer 중 어느 것이 메모리 효율적인가? 각각의 장단점을 비교하라.

**Q3.** 다음 시나리오에서 ZGC와 Shenandoah 중 어느 것을 선택하겠는가? 이유를 설명하라.
- 시나리오 A: 32bit 임베디드 시스템, Java 8 필요
- 시나리오 B: 64bit 서버, Java 17, 100GB 힙

> 💡 **해설**
>
> **Q1.** 메모리 오버헤드: 1억 개 × 8 bytes = 800MB. 실무 영향: ① 8GB 힙 → 10% 오버헤드 (허용 가능). ② 80GB 힙 → 1% 오버헤드 (무시 가능). ③ 작은 힙 (< 4GB)에서는 부담 (20% 이상). 하지만 Pause < 10ms 이득이 오버헤드보다 큼. 추가 영향: ① 간접 참조로 CPU 캐시 미스 증가 (성능 2~3% 감소). ② GC 시 Brooks Pointer 업데이트 비용. 결론: 큰 힙에서는 문제 없음, 작은 힙에서는 G1 권장.
>
> **Q2.** 메모리 효율: ZGC가 우수. Colored Pointer: 포인터에 메타데이터 (추가 메모리 없음). Brooks Pointer: 객체당 8바이트 추가. 100만 객체: ZGC 0MB, Shenandoah 8MB 차이. 장점 비교: ZGC — 메모리 효율, 약간 빠른 Throughput (95% vs 90%). Shenandoah — 단순 구현, 32bit 지원, 이식성. 결론: 64bit 환경이면 ZGC, 플랫폼 제약 있으면 Shenandoah.
>
> **Q3.** A: Shenandoah 선택. 이유: ① 32bit 시스템 → ZGC 불가능 (64bit 전용). ② Java 8 필요 → ZGC는 Java 11+, Shenandoah는 Java 8 백포트 가능. ③ 임베디드라 힙 작을 것 (< 4GB) → 둘 다 오버킬이지만 선택지 없음. B: ZGC 선택. 이유: ① 64bit 서버 → ZGC 제약 없음. ② Java 17 → ZGC Production Ready. ③ 100GB 힙 → 둘 다 적합하지만 ZGC가 Throughput 약간 우수 (95% vs 90%). ④ Brooks Pointer 오버헤드 (800MB) 회피.

---

## 📚 참고 자료

- [Shenandoah GC Wiki](https://wiki.openjdk.org/display/shenandoah)
- [JEP 189: Shenandoah GC](https://openjdk.org/jeps/189)

---

<div align="center">

**[⬅️ 이전: ZGC Deep Dive](./08-zgc-deep-dive.md)** | **[다음: GC Tuning Flags ➡️](./10-gc-tuning-flags.md)**

</div>

# ZGC Deep Dive - ZGC 심층 분석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- ZGC (Z Garbage Collector)는 무엇이며, 어떻게 < 10ms Pause를 달성하는가?
- Colored Pointer는 무엇이며, 어떻게 동작하는가?
- Load Barrier는 무엇이며, 성능 영향은 얼마나 되는가?
- Concurrent Relocation은 어떻게 동작하는가?
- 언제 ZGC를 사용하고, 어떻게 튜닝하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: G1의 한계

```
G1 GC (큰 힙):
  100GB 힙
  Pause Time: 200~500ms
  
  요구사항:
  초저지연 서버
  99.99th percentile < 10ms
  
  → G1으로 불가능
```

```
목표:
  - Pause Time < 10ms (일정)
  - 힙 크기 무관 (TB급도 지원)
  - Concurrent 모든 단계
```

ZGC는 **< 10ms Pause를 보장**한다.

---

## 📐 내부 구조

### 1. ZGC 개요

```
활성화:
  -XX:+UseZGC (Java 11+)

특징:
  - Pause < 10ms (힙 크기 무관)
  - Concurrent Compaction
  - Colored Pointer
  - Load Barrier
  - Non-Generational (세대 구분 없음)

지원:
  Java 11: Experimental
  Java 15: Production Ready
  Java 21: Generational ZGC 추가

힙 크기:
  최소: 8GB 권장
  최대: 16TB 지원
```

---

### 2. Colored Pointer

```
64-bit 포인터 활용:

일반 포인터:
  [64 bit 주소]

ZGC Colored Pointer:
  [Metadata 4bit][Unused 18bit][Address 42bit]
   ↑ Marking/Relocation 정보

Metadata 비트:
  Finalizable: 0/1
  Remapped:    0/1
  Marked0:     0/1
  Marked1:     0/1

예:
  0x0000_1234_5678_0000
  
  Marked0=1:   0x0001_1234_5678_0000
  Remapped=1:  0x0002_1234_5678_0000

장점:
  - 포인터만 보고 상태 파악
  - 별도 메타데이터 불필요
  - Atomic 업데이트 가능

제약:
  - 64-bit 전용
  - Address 42bit (4TB 제한)
  → Java 15+는 확장 (16TB)
```

---

### 3. Load Barrier

```
Load Barrier: 객체 로드 시 삽입되는 코드

Java:
  Object obj = field;

바이트코드:
  aload_0
  getfield #2

ZGC 컴파일 후:
  mov rax, [rbp+8]     // field 로드
  test rax, MASK       // Colored Pointer 체크
  jnz slow_path        // Bad Color → Slow Path
  
slow_path:
  call fix_pointer     // Pointer 수정
  mov [rbp+8], rax     // 수정된 Pointer 저장

목적:
  - Relocation 중인 객체 감지
  - Old 주소 → New 주소 변환
  - Concurrent Compaction 가능

성능:
  - 오버헤드: 5~10%
  - Branch Prediction으로 최소화
```

---

### 4. ZGC 단계

```
1. Pause Mark Start (STW)
   GC Root 스캔 시작
   시간: < 1ms

2. Concurrent Mark
   Reachable 객체 Mark
   Load Barrier로 누락 방지
   시간: 수백 ms ~ 수 초 (Concurrent)

3. Pause Mark End (STW)
   Final Mark
   시간: < 1ms

4. Concurrent Process References
   Soft/Weak/Phantom Reference 처리
   시간: Concurrent

5. Concurrent Prepare Relocation
   Relocation할 Region 선택
   시간: Concurrent

6. Pause Relocate Start (STW)
   Relocation 준비
   시간: < 1ms

7. Concurrent Relocate
   객체 이동
   시간: 수백 ms ~ 수 초 (Concurrent)

STW 단계: 3개 (각 < 1ms)
→ 총 Pause < 10ms
```

---

### 5. Concurrent Relocation

```
문제:
  객체 이동 중 애플리케이션이 접근하면?

ZGC 해결:

1. Forwarding Table 생성
   Old Address → New Address

2. 객체 복사 (Concurrent)
   Old Region → New Region
   
   Old:  0x1000 [Object A]
   New:  0x2000 [Object A]
   
   Forwarding: 0x1000 → 0x2000

3. Load Barrier로 자동 수정
   
   애플리케이션:
   obj = field;  // field = 0x1000 (Old)
   
   Load Barrier:
   if (pointer has old color):
       new_addr = forwarding_table[old_addr]
       update field to new_addr
       return new_addr

4. Remap
   모든 포인터를 New로 업데이트
   (다음 GC Cycle에서 Concurrent)

결과:
  객체 이동 중에도 애플리케이션 정상 실행
  → Concurrent Compaction
```

---

### 6. Page (Region)

```
ZGC Page:

Small Page: 2MB
  작은 객체 (< 256KB)

Medium Page: 32MB
  중간 객체 (256KB ~ 4MB)

Large Page: N × 2MB
  큰 객체 (> 4MB)

동적 할당:
  G1처럼 고정 크기 Region 아님
  → 객체 크기에 맞춰 Page 할당

Relocation 선택:
  Garbage 비율 높은 Page 우선
  → G1의 "Garbage First"와 유사
```

---

## 💻 실험으로 확인하기

### 실험 1: ZGC Pause Time 측정

```bash
java -Xmx16g \
     -XX:+UseZGC \
     -Xlog:gc*:file=zgc.log \
     MyApp

# zgc.log:
# [gc] GC(0) Pause Mark Start 0.5ms
# [gc] GC(0) Concurrent Mark 1200ms
# [gc] GC(0) Pause Mark End 0.8ms
# [gc] GC(0) Concurrent Relocate 800ms
# [gc] GC(0) Pause Relocate Start 0.3ms
#
# → 모든 Pause < 1ms
# → 총 Pause: 1.6ms
```

---

### 실험 2: ZGC vs G1 비교

```java
public class GCCompare {
    static List<byte[]> list = new ArrayList<>();
    
    public static void main(String[] args) throws Exception {
        for (int i = 0; i < 100000; i++) {
            list.add(new byte[1024 * 10]);  // 10KB
            
            if (i % 1000 == 0) {
                list.subList(0, 500).clear();
            }
        }
    }
}
```

```bash
# G1 GC
java -Xmx8g -XX:+UseG1GC -Xlog:gc GCCompare
# Pause: 50~200ms

# ZGC
java -Xmx8g -XX:+UseZGC -Xlog:gc GCCompare
# Pause: 1~5ms (40배 차이)
```

---

## ⚡ 실무 임팩트

### ZGC 사용 케이스

```
적합:
  - 초저지연 요구 (< 10ms)
  - 큰 힙 (8GB+, 수백 GB)
  - 실시간 거래 시스템
  - 게임 서버
  - 금융 거래

부적합:
  - 작은 힙 (< 4GB)
  - Throughput 중시 배치
  - Load Barrier 오버헤드 민감

Throughput 비교:
  G1:  100%
  ZGC: 95% (5% 감소)
  → Load Barrier 오버헤드

Pause Time:
  G1:  50~200ms
  ZGC: 1~10ms (10~100배 개선)
```

---

### ZGC 튜닝

```
기본 설정:
  -XX:+UseZGC
  -Xmx16g

Concurrent 스레드:
  -XX:ConcGCThreads=4
  기본: CPU 코어 수 / 8

힙 크기:
  최소 8GB 권장
  -Xms=Xmx (고정 권장)

Pause 목표:
  -XX:MaxGCPauseMillis 무시
  → 항상 < 10ms 목표

Large Page:
  -XX:+UseLargePages
  → Throughput 향상

NUMA:
  -XX:+UseNUMA
  → 멀티소켓 서버 최적화
```

---

## 🚫 흔한 오해

### "ZGC는 Pause가 0ms다"

```
❌ 잘못된 이해:
  ZGC는 STW가 전혀 없다.

✅ 실제:
  < 10ms Pause (일정)
  
  STW 단계:
  - Pause Mark Start: < 1ms
  - Pause Mark End: < 1ms
  - Pause Relocate Start: < 1ms
  
  총 Pause: 1~5ms (일반적)
  최대: < 10ms (보장)
  
  "No Pause" 아님
  "Ultra Low Pause"
```

---

### "ZGC는 작은 힙에도 좋다"

```
❌ 잘못된 이해:
  ZGC는 모든 힙 크기에 최적이다.

✅ 실제:
  큰 힙 (8GB+)에 최적화
  
  작은 힙 (< 4GB):
  - Load Barrier 오버헤드
  - Colored Pointer 메모리
  - G1이 더 나음
  
  권장:
  < 4GB: G1 GC
  8~100GB: ZGC
  > 100GB: ZGC 필수
```

---

### "ZGC는 항상 G1보다 빠르다"

```
❌ 잘못된 이해:
  ZGC가 무조건 성능이 좋다.

✅ 실제:
  Pause Time: ZGC 압도적
  Throughput: G1이 약간 우수
  
  벤치마크:
  처리량: G1 100, ZGC 95
  Pause: G1 200ms, ZGC 5ms
  
  선택:
  Latency 중시: ZGC
  Throughput 중시: G1/Parallel
```

---

## 📌 핵심 정리

```
ZGC
  Z Garbage Collector
  Ultra Low Latency (< 10ms)
  Java 11+, Production Ready (Java 15+)

핵심 기술
  Colored Pointer: 포인터에 메타데이터
  Load Barrier: 객체 로드 시 체크
  Concurrent Relocation: 동시 압축

Pause Time
  < 10ms 보장 (힙 크기 무관)
  STW: Mark Start/End, Relocate Start
  나머지: Concurrent

Colored Pointer
  64bit 포인터의 4bit 활용
  Marking/Relocation 상태 저장
  Atomic 업데이트

Load Barrier
  객체 로드 시 상태 체크
  Old → New 주소 자동 변환
  오버헤드: 5~10%

Concurrent Relocation
  Forwarding Table
  객체 이동 중 애플리케이션 실행
  Load Barrier로 자동 처리

성능
  Pause: 1~10ms (G1 대비 20~100배 개선)
  Throughput: 5% 감소 (Load Barrier)

사용 케이스
  큰 힙 (8GB+)
  초저지연 (실시간 거래, 게임)
  Pause 민감 애플리케이션

튜닝
  -XX:+UseZGC
  -Xmx=Xms (고정)
  ConcGCThreads 조정
  UseLargePages
```

---

## 🤔 생각해볼 문제

**Q1.** ZGC의 Load Barrier가 성능에 미치는 영향을 설명하라. 왜 5~10% Throughput 감소가 발생하는가?

**Q2.** Colored Pointer의 4bit Metadata가 Concurrent Relocation을 어떻게 가능하게 하는가? Forwarding Table과 연결해 설명하라.

**Q3.** 4GB 힙 애플리케이션에 ZGC와 G1 GC 중 어느 것이 적합한가? Pause Time, Throughput, 오버헤드를 고려하라.

> 💡 **해설**
>
> **Q1.** Load Barrier 오버헤드: 모든 객체 참조 로드 시 추가 코드 실행 — ① Colored Pointer 체크 (test/jnz 명령어), ② Bad Color면 Slow Path 호출, ③ Forwarding Table 조회, ④ Pointer 업데이트. Hot Path에서 빈번한 객체 접근 → 누적 오버헤드. 하지만 Branch Prediction으로 완화 — 대부분 Good Color (Fast Path) → 예측 성공 → 오버헤드 최소. 5~10% 감소는 허용 가능 (Pause 100배 개선 대비).
>
> **Q2.** Colored Pointer와 Relocation: ① Old Region의 객체는 "Old Color" (Remapped=0). ② Relocation 시작 → 객체 복사 → Forwarding Table 생성 (Old Addr → New Addr). ③ New Region의 객체는 "New Color" (Remapped=1). ④ 애플리케이션이 Old 포인터 로드 → Load Barrier 감지 (Remapped=0) → Forwarding Table 조회 → New 주소 반환 + Pointer 업데이트. ⑤ 다음 로드는 New Color → Fast Path (체크만). 결과: 객체 이동 중에도 투명하게 처리 → Concurrent Compaction.
>
> **Q3.** 4GB 힙: G1 GC가 더 적합. 이유: ① ZGC는 8GB 이상에 최적화 — 작은 힙에서 Load Barrier 오버헤드가 상대적으로 큼. ② G1은 4GB에서 충분히 좋은 Pause (50~100ms) → 대부분 애플리케이션 허용 범위. ③ ZGC의 < 10ms Pause는 Overkill (과도한 최적화) → 5~10% Throughput 손실이 아까움. ④ G1이 메모리 효율 좋음 (Colored Pointer 메타데이터 없음). 예외: Pause < 50ms 필수면 ZGC 고려.

---

## 📚 참고 자료

- [ZGC Official Guide](https://wiki.openjdk.org/display/zgc)
- [JEP 333: ZGC (Experimental)](https://openjdk.org/jeps/333)
- [Understanding ZGC](https://malloc.se/blog/zgc-jdk15)

---

<div align="center">

**[⬅️ 이전: G1 GC Deep Dive](./07-g1-gc-deep-dive.md)** | **[다음: Shenandoah GC ➡️](./09-shenandoah-gc.md)**

</div>

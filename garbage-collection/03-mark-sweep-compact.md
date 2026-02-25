# Mark-Sweep-Compact - 표시-청소-압축

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Mark-Sweep-Compact 알고리즘은 어떻게 동작하는가?
- Mark, Sweep, Compact 각 단계의 역할은 무엇인가?
- Fragmentation은 왜 발생하며, 어떤 문제를 일으키는가?
- Compaction은 왜 필요하며, 비용은 얼마나 드는가?
- Mark-Sweep만 하고 Compact를 생략할 수 있는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 죽은 객체를 어떻게 제거할 것인가

```
힙 메모리 상태:
  [A][B][C][D][E][F]
  
  Reachability 분석:
  A, C, E: Reachable (살아있음)
  B, D, F: Unreachable (죽음)
  
  목표:
  B, D, F 제거 → 메모리 회수
```

```
단순 접근 (Free List):
  [A][ ][C][ ][E][ ]
     ↑     ↑     ↑ 빈 공간
  
  문제: Fragmentation (단편화)
  큰 객체 할당 불가
  
  새 객체 크기 = 3 slots
  → 할당 실패 (연속 공간 없음)
  → OutOfMemoryError
  
  하지만 총 빈 공간 = 3 slots (충분함)
```

Mark-Sweep-Compact는 **단편화를 해결**한다.

---

## 📐 내부 구조

### 1. Mark Phase (표시 단계)

```
목적: 살아있는 객체 표시

알고리즘:
  1. GC Root 수집
  2. BFS/DFS로 도달 가능한 객체 탐색
  3. Object Header에 Mark 비트 설정

예시:
  GC Root → A
  A → B, C
  C → D
  
  Mark 순서:
  Mark(A) → Mark(B), Mark(C) → Mark(D)

Object Header 변화:
  Before:
  A: [mark=0][class*][fields...]
  
  After:
  A: [mark=1][class*][fields...]
     ↑ Mark 비트 설정

비용:
  - 시간: O(live objects)
  - 공간: O(1) (Object Header 재사용)
  - STW: 필요 (객체 그래프 변경 방지)
```

---

### 2. Sweep Phase (청소 단계)

```
목적: Unmarked 객체 제거

알고리즘:
  힙 전체 순회
  for each object:
      if (mark == 0):
          free(object)  // 메모리 회수
      else:
          mark = 0      // 다음 GC를 위해 리셋

Free List 생성:
  Unmarked 객체 위치를 Free List에 추가
  
  예:
  힙: [A(marked)][B(unmarked)][C(marked)][D(unmarked)]
  
  Sweep 후:
  Free List: [B의 주소, 크기], [D의 주소, 크기]
  
  메모리:
  [A][Free B][C][Free D]

비용:
  - 시간: O(heap size)
  - 공간: O(free blocks) (Free List)
  - STW: 필요

문제: Fragmentation
  [A][Free 10KB][B][Free 5KB][C][Free 8KB]
  
  새 객체 15KB 할당 요청
  → 실패 (연속 공간 없음)
  → 총 Free = 23KB (충분하지만)
```

---

### 3. Compact Phase (압축 단계)

```
목적: Fragmentation 제거

알고리즘:
  1. 살아있는 객체를 힙 한쪽으로 이동
  2. 빈 공간을 다른 쪽으로 모음

예시:
  Before Compact:
  [A][Free][B][Free][C][Free]
  
  After Compact:
  [A][B][C][Free Free Free]
  
  결과: 연속된 큰 빈 공간

이동 과정:
  1. Forwarding Address 계산
     A → 0x1000 (그대로)
     B → 0x1010 (A 다음)
     C → 0x1020 (B 다음)
  
  2. 포인터 업데이트
     모든 참조를 새 주소로 변경
     
     X → A (0x1050 → 0x1000)
     Y → B (0x1070 → 0x1010)
  
  3. 객체 이동
     memcpy(new_addr, old_addr, size)

비용:
  - 시간: O(live objects) (이동 + 포인터 업데이트)
  - 공간: O(live objects) (Forwarding Table)
  - STW: 필요 (포인터 무결성)
  
  매우 비쌈!
```

---

### 4. Mark-Sweep vs Mark-Sweep-Compact

```
Mark-Sweep (Compact 없음):

장점:
  - 빠름 (Compact 비용 없음)
  
단점:
  - Fragmentation 발생
  - 큰 객체 할당 실패 가능
  
사용:
  CMS GC (Old Generation)

Mark-Sweep-Compact:

장점:
  - Fragmentation 없음
  - 메모리 효율 최대
  
단점:
  - 느림 (Compact 비용)
  
사용:
  Serial GC, Parallel GC (Old Generation)

선택 기준:
  Pause Time 중시 → Mark-Sweep (CMS)
  Throughput 중시 → Mark-Sweep-Compact (Parallel)
```

---

### 5. Compaction 알고리즘 변형

```
Two-Pointer Compaction (Lisp2 알고리즘):

Pass 1: 새 주소 계산
  for each live object:
      new_addr = compute_forward_addr(obj)
      obj->forward = new_addr

Pass 2: 포인터 업데이트
  for each pointer in heap:
      *ptr = ptr->forward

Pass 3: 객체 이동
  for each live object:
      memcpy(obj->forward, obj, obj->size)

비용: 3 Pass (힙 3회 순회)

Sliding Compaction:
  객체를 힙 시작으로 밀어냄 (슬라이딩)
  
  [A][Dead][B][Dead][C]
  → [A][B][C][Free Free]
  
  장점: 순서 보존 (할당 순서 유지)
  단점: 여전히 3 Pass 필요

Mark-Compact (Concurrent):
  Concurrent Marking + Compaction
  
  G1 GC, ZGC에서 사용
  STW 최소화
```

---

## 💻 실험으로 확인하기

### 실험 1: Fragmentation 시뮬레이션

```java
import java.util.*;

public class FragmentationTest {
    static List<byte[]> objects = new ArrayList<>();
    
    public static void main(String[] args) throws Exception {
        System.out.println("=== Allocating objects ===");
        
        // 교대로 할당 (유지 vs 제거)
        for (int i = 0; i < 100; i++) {
            byte[] data = new byte[1024 * 1024];  // 1MB
            if (i % 2 == 0) {
                objects.add(data);  // 유지
            }
            // i % 2 == 1: 제거 대상 (GC될 것)
        }
        
        printMemory();
        
        System.out.println("=== After GC (Fragmentation) ===");
        System.gc();
        Thread.sleep(100);
        
        printMemory();
        
        // 큰 객체 할당 시도
        System.out.println("=== Allocating large object ===");
        try {
            byte[] large = new byte[60 * 1024 * 1024];  // 60MB
            System.out.println("Success: Large object allocated");
        } catch (OutOfMemoryError e) {
            System.out.println("Failed: OutOfMemoryError (Fragmentation?)");
        }
    }
    
    static void printMemory() {
        Runtime rt = Runtime.getRuntime();
        long total = rt.totalMemory() / (1024 * 1024);
        long free = rt.freeMemory() / (1024 * 1024);
        long used = total - free;
        System.out.printf("Total: %d MB, Used: %d MB, Free: %d MB%n", total, used, free);
    }
}
```

```bash
# -XX:+UseSerialGC (Compact 있음)
java -Xmx100m -XX:+UseSerialGC FragmentationTest
# Success: Large object allocated

# -XX:+UseConcMarkSweepGC (Compact 없음)
java -Xmx100m -XX:+UseConcMarkSweepGC FragmentationTest
# Failed: OutOfMemoryError (Fragmentation)
```

---

### 실험 2: GC 로그로 Compact 확인

```bash
java -Xlog:gc*:file=gc.log \
     -XX:+UseSerialGC \
     MyApp

# gc.log 분석:
# [gc] GC(0) Pause Young
# [gc] GC(0) Pause Full (Allocation Failure)
# [gc] GC(0) Phase 1: Mark live objects
# [gc] GC(0) Phase 2: Compute new addresses
# [gc] GC(0) Phase 3: Adjust pointers
# [gc] GC(0) Phase 4: Move objects
# → Mark-Sweep-Compact 확인
```

---

## ⚡ 실무 임팩트

### Compaction 비용

```
Full GC 시간 구성 (Parallel GC):

Mark:       30% (0.3s)
Sweep:      20% (0.2s)
Compact:    50% (0.5s)  ← 가장 비쌈
Total:      100% (1.0s)

큰 힙에서:
  10GB 힙, 5GB live objects
  Compact: 객체 이동 + 포인터 업데이트
  → 수 초 소요 (STW)

결과:
  사용자 경험 저하
  → GC Pause 최소화 필요
  → CMS, G1, ZGC 탄생 배경
```

### Fragmentation 영향

```
시나리오: 장시간 실행 서버

초기 (1시간):
  힙: [Object들이 고르게 분포]
  Fragmentation: 낮음
  
1주일 후:
  힙: [Live][Free][Live][Free]...
  Fragmentation: 높음
  
  증상:
  - 작은 Free 공간이 많음
  - 큰 객체 할당 실패
  - Full GC 빈번 발생
  
  해결:
  Full GC → Compact
  또는
  G1 GC로 전환 (Region 기반)
```

### GC 선택 기준

```
Throughput 중시 (배치):
  Parallel GC
  → Mark-Sweep-Compact
  → Pause 길어도 OK
  → 전체 처리량 최대

Latency 중시 (웹 서버):
  G1 GC, ZGC
  → Concurrent Mark + Incremental Compact
  → Pause 최소화
  → 일부 Fragmentation 허용

메모리 제약:
  Serial GC
  → 단일 스레드 (메모리 절약)
  → Mark-Sweep-Compact
  → 느리지만 안정적
```

---

## 🚫 흔한 오해

### "Sweep 후 Compact는 선택사항이다"

```
❌ 잘못된 이해:
  Compact 없이 Sweep만 해도 된다.

✅ 실제:
  GC 알고리즘에 따라 다름
  
  Compact 없음:
  CMS GC (Old Generation)
  → Free List 방식
  → Fragmentation 발생 가능
  → 주기적으로 Full GC + Compact 필요
  
  Compact 있음:
  Serial GC, Parallel GC
  → 매번 Compact
  → Fragmentation 없음
  → Pause 길어짐
  
  Trade-off:
  Pause Time ↔ Fragmentation
```

### "Compaction은 항상 힙 전체를 대상으로 한다"

```
❌ 잘못된 이해:
  Compact는 힙 전체를 정리한다.

✅ 실제:
  선택적 Compaction 가능
  
  G1 GC:
  Region 단위 Compaction
  → Garbage 많은 Region만
  → Mixed GC
  
  예:
  10개 Region 중 3개만 Compact
  → Pause Time 단축
  
  ZGC:
  Concurrent Compaction
  → STW 없이 Compact
  → Page 단위 재배치
```

### "Mark 비트는 별도 메모리를 사용한다"

```
❌ 잘못된 이해:
  Mark를 위한 별도 비트맵이 필요하다.

✅ 실제:
  Object Header 재사용
  
  Object Header (64-bit):
  [Mark Word (8 bytes)][Class Pointer (4 bytes)]
  
  Mark Word 구조:
  [hash:25][age:4][biased:1][lock:2][...]
  
  GC 중:
  [mark:1][forwarding:31][...]
     ↑ Mark 비트
        ↑ Forwarding Address (Compact용)
  
  재사용:
  - Mark Phase: mark 비트 사용
  - Compact Phase: forwarding 필드 사용
  - 추가 메모리 불필요
```

---

## 📌 핵심 정리

```
Mark-Sweep-Compact

Mark Phase
  GC Root에서 Reachable 객체 탐색
  Object Header에 Mark 비트 설정
  시간: O(live objects)

Sweep Phase
  힙 전체 순회
  Unmarked 객체 제거
  Free List 생성
  시간: O(heap size)

Compact Phase
  살아있는 객체를 한쪽으로 이동
  포인터 업데이트
  Fragmentation 제거
  시간: O(live objects)

Fragmentation
  Mark-Sweep만: 발생
  Mark-Sweep-Compact: 제거
  큰 객체 할당 실패 원인

Compaction 비용
  객체 이동 + 포인터 업데이트
  Full GC 시간의 50%
  STW 필수

알고리즘 선택
  Serial/Parallel GC: 항상 Compact
  CMS GC: Compact 없음 (일부 Full GC만)
  G1/ZGC: 선택적/Concurrent Compact

Trade-off
  Pause Time ↔ Fragmentation
  Throughput ↔ Latency
```

---

## 🤔 생각해볼 문제

**Q1.** Mark-Sweep과 Mark-Sweep-Compact의 시간 복잡도를 비교하라. 왜 Compact가 가장 비싼 단계인가?

**Q2.** Fragmentation이 심한 힙에서 작은 객체는 할당되지만 큰 객체는 OOM이 발생하는 이유를 설명하라.

**Q3.** G1 GC가 "선택적 Compaction"을 하는 이유는 무엇인가? 전체 힙을 Compact하지 않는 이유를 설명하라.

> 💡 **해설**
>
> **Q1.** Mark: O(live objects) — GC Root에서 도달 가능한 객체만 탐색. Sweep: O(heap size) — 힙 전체를 순회해 Unmarked 제거. Compact: O(live objects) × 3 — ① 새 주소 계산 (1 pass), ② 포인터 업데이트 (1 pass, 모든 참조), ③ 객체 이동 (1 pass, memcpy). 포인터 업데이트가 특히 비쌈 — 힙 전체에서 포인터 찾아 변경, 캐시 미스 빈번. 따라서 Compact가 전체 GC 시간의 50% 차지.
>
> **Q2.** Free List 상태: `[A][Free 10KB][B][Free 5KB][C][Free 8KB]`. 작은 객체 6KB 요청 → 10KB 블록에 할당 가능. 큰 객체 15KB 요청 → 연속된 15KB 공간 없음 → Allocation Failure → OOM. 총 Free = 23KB (충분)하지만, Fragmentation으로 연속 공간 부족. 해결: Full GC + Compact → `[A][B][C][Free 23KB]` → 큰 객체 할당 가능.
>
> **Q3.** Pause Time 목표 달성: G1 GC는 Pause Time 목표 (예: 200ms) 설정 가능. 전체 힙 Compact → 수 초 소요 → 목표 초과. 선택적 Compaction → Garbage 많은 Region만 (Mixed GC) → 목표 시간 내 완료. 예: 100개 Region 중 10개만 Compact → Pause 10배 단축. Fragmentation 일부 허용하되, 점진적으로 해결. 최악의 경우 Full GC로 전체 Compact (드물게).

---

## 📚 참고 자료

- [GC Algorithms: Mark and Sweep](https://en.wikipedia.org/wiki/Tracing_garbage_collection#Na%C3%AFve_mark-and-sweep)
- [Memory Compaction in GC](https://mechanical-sympathy.blogspot.com/2013/07/java-garbage-collection-distilled.html)
- [HotSpot GC Internals](https://shipilev.net/jvm/anatomy-quarks/4-tlab-allocation/)

---

<div align="center">

**[⬅️ 이전: Reference Types](./02-reference-types.md)** | **[다음: Generational Hypothesis ➡️](./04-generational-hypothesis.md)**

</div>

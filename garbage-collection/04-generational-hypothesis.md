# Generational Hypothesis - 세대별 가설

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Generational Hypothesis는 무엇이며, 왜 중요한가?
- Young Generation과 Old Generation은 어떻게 나뉘는가?
- Minor GC와 Major/Full GC의 차이는 무엇인가?
- Promotion은 언제 발생하며, 어떤 기준으로 결정되는가?
- 왜 대부분의 최신 GC가 Generational을 채택하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 힙 전체를 매번 GC하면 느리다

```
10GB 힙, Full GC:
  Mark: 모든 객체 탐색 (수백만 개)
  Sweep: 힙 전체 순회 (10GB)
  Compact: 객체 이동 (수 GB)
  
  시간: 수 초 ~ 수십 초
  → 사용자 경험 저하
```

```
관찰:
  대부분의 객체는 금방 죽는다
  
  예:
  void process(Request req) {
      String temp = req.getBody();  // ← 메서드 종료 시 죽음
      Data data = parse(temp);       // ← 메서드 종료 시 죽음
      save(data);
  }
  
  req, temp, data → 메서드 내에서만 사용
  → 즉시 Unreachable
```

Generational GC는 **자주 죽는 객체만 자주 수집**한다.

---

## 📐 내부 구조

### 1. Generational Hypothesis

```
가설 1: "대부분의 객체는 젊어서 죽는다"
  Weak Generational Hypothesis
  
  통계:
  - 객체의 98%는 첫 GC에서 죽음
  - 수명이 긴 객체는 2% 미만

가설 2: "오래된 객체가 젊은 객체를 참조하는 경우는 드물다"
  Strong Generational Hypothesis
  
  예:
  오래된 객체: Application, Config
  → 거의 변하지 않음
  → 젊은 객체 참조할 이유 없음

설계 영향:
  1. 힙을 세대로 분리
  2. 젊은 세대만 자주 GC
  3. 늙은 세대는 가끔 GC
  
  결과:
  GC 시간 = 힙 크기의 일부만
  → 빠른 GC
```

---

### 2. Young Generation vs Old Generation

```
힙 구조:

┌──────────────────────────────────────┐
│          Young Generation            │
│  ┌──────┬──────────┬──────────┐      │
│  │ Eden │Survivor 0│Survivor 1│      │
│  └──────┴──────────┴──────────┘      │
├──────────────────────────────────────┤
│          Old Generation              │
│                                      │
└──────────────────────────────────────┘

Young Generation (Young Gen):
  - 크기: 힙의 1/3 (기본)
  - 특징: 자주 GC, 빠름
  - 대상: 새로 생성된 객체
  
Old Generation (Old Gen, Tenured):
  - 크기: 힙의 2/3
  - 특징: 가끔 GC, 느림
  - 대상: 오래 살아남은 객체

Permanent Generation (Java 7):
  - 클래스 메타데이터
  - Java 8+ → Metaspace (네이티브 메모리)
```

---

### 3. Minor GC vs Major/Full GC

```
Minor GC (Young GC):
  대상: Young Generation만
  빈도: 매우 자주 (초당 수십 회)
  시간: 수십 ms (빠름)
  
  트리거:
  Eden 영역이 가득 참
  
  동작:
  1. Eden + Survivor 0 스캔
  2. 살아있는 객체 → Survivor 1로 복사
  3. Eden + Survivor 0 비움
  
  결과:
  98% 객체 제거
  → 빠른 회수

Major GC (Old GC):
  대상: Old Generation만
  빈도: 가끔 (분당 수 회)
  시간: 수백 ms ~ 수 초
  
  트리거:
  Old Generation이 가득 참
  또는 Promotion 실패

Full GC:
  대상: Young + Old Generation
  빈도: 드물게 (시간당 수 회)
  시간: 수 초 ~ 수십 초 (매우 느림)
  
  트리거:
  System.gc() 호출
  Metaspace 부족
  Old Gen 부족 + Minor GC 실패
```

---

### 4. Promotion (승격)

```
Promotion: Young → Old 이동

기준:
  Age (나이) Threshold
  → 기본값: 15 (Minor GC 15회 생존)

과정:
  1. 객체 생성 → Eden (age=0)
  2. Minor GC 생존 → Survivor (age=1)
  3. Minor GC 생존 → Survivor (age=2)
  ...
  15. Minor GC 생존 → Old (age=15)

Object Header Age:
  [mark:25][age:4][biased:1][lock:2][...]
            ↑ 4 bit (0~15)

예:
  Object obj = new Object();
  
  0ms:    Eden (age=0)
  100ms:  Minor GC → Survivor (age=1)
  200ms:  Minor GC → Survivor (age=2)
  ...
  1500ms: Minor GC → Old (age=15)
  
  이후: Old에서 계속 유지

동적 Age 조정:
  Survivor 공간 부족 시
  → MaxTenuringThreshold보다 낮은 age에서도 Promotion
  
  -XX:MaxTenuringThreshold=15 (기본)
```

---

### 5. Card Table (세대 간 참조)

```
문제:
  Old → Young 참조가 있다면?
  
  Old 객체 A → Young 객체 B
  
  Minor GC 시:
  Young Gen만 스캔
  → A를 못 봄
  → B가 Reachable인지 모름
  → 잘못된 GC?

해결: Card Table

Old Generation을 Card로 분할:
  Card 크기: 512 bytes
  
  ┌──────┬──────┬──────┬──────┐
  │Card 0│Card 1│Card 2│Card 3│
  └──────┴──────┴──────┴──────┘
  
  Card Table (Bit Array):
  [0][1][0][0]  ← Card 1이 Young 참조

Write Barrier:
  Old 객체가 Young 참조 시
  → Card Table Mark
  
  예:
  oldObj.field = youngObj;
  → Card Table[card_index] = 1

Minor GC 시:
  1. GC Root 스캔
  2. Card Table 스캔
     → Mark된 Card만 스캔
  3. Young Gen 스캔
  
  결과:
  Old Gen 전체 스캔 불필요
  → 빠른 Minor GC
```

---

## 💻 실험으로 확인하기

### 실험 1: Minor GC 관찰

```java
public class MinorGCTest {
    public static void main(String[] args) throws Exception {
        List<byte[]> survivors = new ArrayList<>();
        
        for (int i = 0; i < 100; i++) {
            // 짧은 수명 객체 (대부분 Minor GC에서 제거)
            for (int j = 0; j < 1000; j++) {
                byte[] temp = new byte[1024];  // 1KB
            }
            
            // 긴 수명 객체 (Promotion될 것)
            if (i % 10 == 0) {
                survivors.add(new byte[1024]);
            }
            
            Thread.sleep(10);
        }
        
        System.out.println("Survivors: " + survivors.size());
    }
}
```

```bash
java -Xmx100m -Xmn30m \
     -Xlog:gc*:file=gc.log \
     MinorGCTest

# gc.log 분석:
# [gc] GC(0) Pause Young (Allocation Failure)
# [gc] GC(0) Eden: 20M -> 0M
# [gc] GC(0) Survivor: 0M -> 2M
# [gc] GC(0) Old: 0M -> 0M
# → Minor GC만 발생 (Old 변화 없음)
```

---

### 실험 2: Promotion 관찰

```java
public class PromotionTest {
    public static void main(String[] args) throws Exception {
        List<byte[]> longLived = new ArrayList<>();
        
        for (int i = 0; i < 100; i++) {
            byte[] data = new byte[1024 * 1024];  // 1MB
            longLived.add(data);  // 계속 참조 (죽지 않음)
            Thread.sleep(100);
        }
    }
}
```

```bash
java -Xmx200m -Xmn50m \
     -Xlog:gc*:file=gc.log \
     PromotionTest

# gc.log:
# [gc] GC(5) Pause Young
# [gc] GC(5) Survivor: 10M -> 0M
# [gc] GC(5) Old: 20M -> 30M
# → Survivor에서 Old로 Promotion
```

---

### 실험 3: Age Threshold 조정

```bash
# 기본 Threshold (15)
java -XX:+PrintTenuringDistribution MyApp

# 출력:
# Desired survivor size 10485760 bytes, new threshold 15 (max 15)
# - age   1:    1048576 bytes,    1048576 total
# - age   2:     524288 bytes,    1572864 total
# ...
# - age  15:      65536 bytes,   10485760 total

# Threshold 낮추기 (5)
java -XX:MaxTenuringThreshold=5 MyApp
# → 5회 GC 생존 시 Old로 Promotion
```

---

## ⚡ 실무 임팩트

### Young/Old 비율 튜닝

```
기본 비율: Young 1/3, Old 2/3

Young 크게:
  -Xmn2g (Young 2GB)
  
  장점:
  - Minor GC 빈도 감소
  - Promotion 감소 (더 많이 죽음)
  
  단점:
  - Minor GC 시간 증가
  - Survivor 공간 부족 가능

Young 작게:
  -Xmn500m (Young 500MB)
  
  장점:
  - Minor GC 시간 단축
  
  단점:
  - Minor GC 빈도 증가
  - Promotion 증가 (Old 압박)

권장:
  처리량 중시: Young 크게 (Xmn=1~2GB)
  지연시간 중시: Young 작게 (Xmn=500MB~1GB)
```

### Premature Promotion 문제

```
조기 승격 (Premature Promotion):
  아직 죽을 객체가 Old로 이동
  
원인:
  Survivor 공간 부족
  → age < MaxTenuringThreshold인데도 Promotion
  
증상:
  Old Gen 빠르게 증가
  → Full GC 빈번 발생
  → 성능 저하

해결:
  1. Survivor 크기 증가
     -XX:SurvivorRatio=6 (기본 8)
     → Eden:Survivor:Survivor = 6:1:1
     
  2. Young Gen 크기 증가
     -Xmn2g
     
  3. MaxTenuringThreshold 증가
     -XX:MaxTenuringThreshold=31
     → 더 오래 Young에 유지

확인:
  -XX:+PrintTenuringDistribution
  → Age 분포 확인
  → Survivor 사용량 확인
```

### Full GC 회피

```
Full GC 원인:
  1. Old Generation 가득 찬
  2. Promotion 실패
  3. Metaspace 부족
  4. System.gc() 호출

회피 전략:
  1. Young Gen 크기 적절히 조정
     → Promotion 최소화
  
  2. Old Gen 크기 충분히 확보
     -Xmx10g (힙 10GB)
     → Old 압박 감소
  
  3. CMS/G1 GC 사용
     → Concurrent GC (STW 최소화)
  
  4. System.gc() 비활성화
     -XX:+DisableExplicitGC

모니터링:
  Full GC 빈도
  → 시간당 1회 미만 목표
  → 초과 시 튜닝 필요
```

---

## 🚫 흔한 오해

### "Young Generation을 크게 하면 항상 좋다"

```
❌ 잘못된 이해:
  Young Gen이 클수록 성능이 좋다.

✅ 실제:
  Trade-off 존재
  
  Young 크게 (2GB):
  장점: Minor GC 빈도 ↓, Promotion ↓
  단점: Minor GC 시간 ↑
  
  예:
  Young 500MB: Minor GC 50ms, 초당 10회
  Young 2GB:   Minor GC 200ms, 초당 2회
  
  총 GC 시간:
  500MB: 50ms × 10 = 500ms/s
  2GB:   200ms × 2 = 400ms/s
  
  → 2GB가 유리
  
  하지만:
  Latency 요구사항 (99th percentile < 100ms)
  → 200ms는 목표 초과
  → 500MB가 나음

권장:
  Throughput: Young 크게
  Latency: Young 작게
```

### "Old Generation 객체는 절대 죽지 않는다"

```
❌ 잘못된 이해:
  Old Gen에 한 번 들어가면 영원히 산다.

✅ 실제:
  Major/Full GC로 제거됨
  
  예:
  Cache 객체 → Old Gen
  → 시간 지나 참조 제거
  → Major GC에서 회수

Old Gen GC:
  - 빈도: 가끔 (분당 수 회)
  - 대상: Unreachable Old 객체
  - 시간: 느림 (수백 ms ~ 수 초)

통계:
  Young: 98% 제거 (Minor GC)
  Old:   20~30% 제거 (Major GC)
  
  → Old도 죽지만, 비율 낮음
```

### "Generational GC는 항상 최선이다"

```
❌ 잘못된 이해:
  모든 애플리케이션에 Generational이 좋다.

✅ 실제:
  워크로드에 따라 다름
  
  Generational 유리:
  - 짧은 수명 객체 많음
  - 웹 서버, API 서버
  
  Generational 불리:
  - 대부분 객체가 오래 살아남음
  - 인메모리 데이터베이스
  - 캐시 서버
  
  예: Redis (in-memory DB)
  → 거의 모든 객체가 장수
  → Generational 이점 없음
  → ZGC (Non-Generational) 고려

최신 트렌드:
  ZGC, Shenandoah → Non-Generational
  → Ultra-low latency 목표
  → < 10ms Pause
```

---

## 📌 핵심 정리

```
Generational Hypothesis
  가설 1: 대부분 객체는 젊어서 죽음 (98%)
  가설 2: Old → Young 참조 드뭄

힙 구조
  Young Generation (1/3)
    Eden + Survivor 0 + Survivor 1
  Old Generation (2/3)
    장수 객체

GC 종류
  Minor GC: Young만, 빠름 (수십 ms)
  Major GC: Old만, 느림 (수백 ms)
  Full GC: Young + Old, 매우 느림 (수 초)

Promotion
  Young → Old 이동
  Age Threshold (기본 15)
  Survivor 공간 부족 시 조기 승격

Card Table
  Old → Young 참조 추적
  Write Barrier로 Mark
  Minor GC 시 Old 전체 스캔 불필요

튜닝
  Young 크게: Throughput ↑, Pause ↑
  Young 작게: Latency ↓, GC 빈도 ↑
  Survivor 크게: Premature Promotion ↓

Full GC 회피
  Young/Old 비율 조정
  CMS/G1 GC 사용
  Promotion 최소화
```

---

## 🤔 생각해볼 문제

**Q1.** Young Generation을 1GB에서 2GB로 늘리면 어떤 변화가 발생하는가? Minor GC 빈도, 시간, Promotion을 고려해 설명하라.

**Q2.** Card Table이 없다면 Minor GC는 어떻게 동작해야 하는가? 성능에 미치는 영향을 설명하라.

**Q3.** 다음 두 워크로드 중 어느 것이 Generational GC에 더 적합한가? 이유를 설명하라.
- 워크로드 A: 웹 서버 (Request/Response, 짧은 수명)
- 워크로드 B: 인메모리 캐시 (데이터 장시간 유지)

> 💡 **해설**
>
> **Q1.** Young 1GB → 2GB 변화: ① Minor GC 빈도 감소 — Eden이 2배 → 가득 차는 시간 2배 → GC 빈도 절반. ② Minor GC 시간 증가 — 스캔 대상 2배 → 시간 약 2배 (50ms → 100ms). ③ Promotion 감소 — Young에서 더 오래 머물 기회 → 더 많이 죽음 → Old 압박 감소. 총 GC 시간: 100ms × 5회/s = 500ms vs 50ms × 10회/s = 500ms (비슷). 하지만 Latency: 100ms vs 50ms (2배 차이) → Latency 중시면 불리.
>
> **Q2.** Card Table 없으면: Minor GC 시 Old Generation 전체 스캔 필요 (Old → Young 참조 찾기). Old Gen 크기 5GB → 스캔 시간 수백 ms 추가. Minor GC가 Full GC 수준으로 느려짐 → Generational 이점 상실. Card Table로: Mark된 Card만 스캔 (Old의 1% 미만) → 수 ms만 추가. 결과: Card Table이 Generational GC의 핵심 기술.
>
> **Q3.** 워크로드 A (웹 서버)가 훨씬 적합. 이유: Request/Response 패턴 → 대부분 객체가 요청 처리 후 즉시 죽음 (Generational Hypothesis 부합). Minor GC에서 98% 제거 → 빠른 회수. Old Gen 거의 증가 안 함 → Full GC 드물게 발생. 워크로드 B (캐시): 대부분 객체가 장시간 유지 → Young에서 금방 Old로 Promotion. Minor GC 효과 낮음 (대부분 Survivor 차지). Old Gen 빠르게 증가 → Full GC 빈번. ZGC 같은 Non-Generational GC가 더 적합 (전체 힙을 동일하게 취급).

---

## 📚 참고 자료

- [Understanding Java Garbage Collection](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)
- [Generational Hypothesis](https://www.memorymanagement.org/glossary/g.html#generational.hypothesis)
- [Tuning Java Garbage Collection](https://docs.oracle.com/en/java/javase/17/gctuning/garbage-collector-implementation.html)

---

<div align="center">

**[⬅️ 이전: Mark-Sweep-Compact](./03-mark-sweep-compact.md)** | **[다음: Serial & Parallel GC ➡️](./05-serial-parallel-gc.md)**

</div>

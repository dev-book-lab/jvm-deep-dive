# Compressed Oops - 압축 포인터

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Compressed Oops는 무엇이며, 왜 필요한가?
- 64비트 포인터를 32비트로 어떻게 압축하는가?
- 32GB 힙 제한은 왜 발생하는가?
- 언제 Compressed Oops가 비활성화되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 64bit JVM의 메모리 낭비

```
32bit JVM:
  포인터 크기: 4 bytes
  최대 힙: 4GB (2^32)

64bit JVM (압축 없음):
  포인터 크기: 8 bytes
  최대 힙: 무제한
  
  하지만:
  포인터 2배 → 메모리 30~50% 증가
  캐시 효율 저하
```

Compressed Oops는 **메모리 효율성 개선**이다.

---

## 📐 Compressed Oops 원리

### 1. 압축 알고리즘

```
핵심 아이디어:
  Java 객체는 8 byte 정렬
  → 하위 3bit 항상 0
  → 하위 3bit 생략 가능

압축:
  64bit 주소: 0x0000_7F00_1234_5678
  → Shift right 3: 0x0000_0FE0_0246_8ACF
  → 32bit 저장: 0x0FE0_0246_8ACF (상위 잘림)

압축 해제:
  32bit: 0x0246_8ACF
  → Shift left 3: 0x0001_2345_678 (×8)
  → 64bit 주소 복원

허용 범위:
  32bit × 8 = 2^35 = 32GB
```

---

### 2. Zero-Based vs Heap-Based

```
Zero-Based (힙 < 4GB):
  Heap Base: 0
  압축: addr / 8
  해제: compressed × 8
  
  예:
  0x1000 → 0x200 → 0x1000

Heap-Based (힙 4~32GB):
  Heap Base: 0x7F000000000
  압축: (addr - base) / 8
  해제: compressed × 8 + base
  
  예:
  0x7F000001000 - 0x7F000000000 = 0x1000
  → 0x200
  → 0x200 × 8 + 0x7F000000000 = 0x7F000001000
```

---

### 3. Object Header 크기

```
Without Compressed Oops (64bit):
┌────────────────┐
│ Mark Word (8)  │
├────────────────┤
│ Class Ptr (8)  │ ← 8 bytes
└────────────────┘
Total: 16 bytes

With Compressed Oops (64bit):
┌────────────────┐
│ Mark Word (8)  │
├────────────────┤
│ Class Ptr (4)  │ ← 4 bytes (압축)
└────────────────┘
Total: 12 bytes

절약: 25% (16 → 12)
```

---

## 💻 실험으로 확인하기

### 실험 1: Compressed Oops 확인

```bash
# 활성화 확인
java -XX:+PrintFlagsFinal -version | grep UseCompressedOops
# UseCompressedOops = true {product} {default}

# 비활성화
java -XX:-UseCompressedOops -version

# 객체 크기 비교
java -XX:+UseCompressedOops -jar test.jar  # 12 bytes header
java -XX:-UseCompressedOops -jar test.jar  # 16 bytes header
```

---

### 실험 2: 힙 크기별 동작

```bash
# < 4GB: Zero-Based
java -Xmx3g -XX:+PrintFlagsFinal | grep -E "UseCompressed|HeapBase"
# UseCompressedOops = true
# HeapBaseMinAddress = 0

# 4~32GB: Heap-Based
java -Xmx20g -XX:+PrintFlagsFinal | grep HeapBase
# HeapBaseMinAddress = 17179869184 (16GB)

# > 32GB: Disabled
java -Xmx40g -XX:+PrintFlagsFinal | grep UseCompressed
# UseCompressedOops = false
```

---

## ⚡ 실무 임팩트

### 32GB 힙 제한

```
메모리 선택:

Option A: 30GB (Compressed Oops)
  포인터: 4 bytes
  실제 사용: ~30GB

Option B: 40GB (No Compression)
  포인터: 8 bytes
  실제 사용: ~52GB (30% 증가)

결론:
  32GB 이하 권장
  → Compressed Oops 활용
```

---

### 컨테이너 환경

```yaml
# Kubernetes
resources:
  limits:
    memory: "30Gi"  # ✅ Compressed Oops

# JVM 설정
-Xmx28g  # 안전 마진

# vs

resources:
  limits:
    memory: "40Gi"  # ❌ 압축 비활성화

-Xmx40g  # 포인터 8 bytes → 메모리 낭비
```

---

## 🚫 흔한 오해

### "32GB 힙이 최적이다"

```
❌ 잘못된 이해:
  항상 32GB 설정

✅ 실제:
  워크로드에 따라 다름
  
  Live Data < 8GB:
  → 16~24GB 힙 충분
  
  Live Data > 20GB:
  → 32GB 제한 문제
  → ZGC 고려 (압축 효율)
```

---

## 📌 핵심 정리

```
Compressed Oops
  64bit 포인터 → 32bit 압축
  8 byte 정렬 활용 (하위 3bit 생략)

압축 범위
  32bit × 8 = 32GB

메모리 절약
  Class Pointer: 8B → 4B
  Object Header: 16B → 12B
  전체: 20~30% 절약

활성화 조건
  힙 ≤ 32GB (기본)
  64bit JVM

비활성화
  힙 > 32GB
  -XX:-UseCompressedOops

실무 권장
  힙 ≤ 32GB 유지
  32~40GB 필요 시 ZGC
```

---

## 🤔 생각해볼 문제

**Q1.** 30GB 힙과 40GB 힙 중 어느 것이 실제 메모리를 더 적게 사용하는가? 이유를 설명하라.

**Q2.** Compressed Oops가 8 byte 정렬에 의존하는 이유를 설명하라. 만약 4 byte 정렬이라면?

**Q3.** ZGC에서 Colored Pointer는 Compressed Oops와 어떻게 다른가?

> 💡 **해설**
>
> **Q1.** 30GB 힙이 더 적게 사용. 이유: ① 30GB: Compressed Oops 활성 → 포인터 4 bytes. Live Data 20GB라면 실제 사용 ~25GB. ② 40GB: Compressed Oops 비활성 → 포인터 8 bytes. Live Data 20GB + 포인터 오버헤드 30% → 실제 사용 ~35GB. ③ 결론: 30GB 힙 < 40GB 힙 (실제 메모리).
>
> **Q2.** 8 byte 정렬 이유: ① Java 객체는 8 byte 정렬 강제 → 주소 하위 3bit 항상 0. ② 이 3bit 생략 → 32bit로 2^35 (32GB) 표현. ③ 4 byte 정렬이라면: 하위 2bit만 0 → 2bit 생략 → 32bit로 2^34 (16GB)만 표현. ④ 8 byte 정렬이 더 효율적.
>
> **Q3.** ZGC Colored Pointer: ① Compressed Oops: 32bit 압축, 힙 ≤ 32GB. ② Colored Pointer: 64bit 전체 사용, 상위 bit에 메타데이터 (Marked, Remapped), 하위 42bit 주소 (4TB). ③ ZGC는 압축 안 하지만 효율적 (메타데이터 포함).

---

## 📚 참고 자료

- [Compressed Oops](https://wiki.openjdk.org/display/HotSpot/CompressedOops)

---

<div align="center">

**[⬅️ 이전: Object Header & Mark Word](./01-object-header-and-mark-word.md)** | **[다음: String Pool & Interning ➡️](./03-string-pool-interning.md)**

</div>

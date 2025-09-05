---
date: 2025-09-05
user: Daemon
topic: "Dalvik/ART 가비지 컬렉션"
---

## 메모리 누수는 왜 발생할까?

가비지 컬렉션이 있는데도 안드로이드 앱에서 메모리 누수가 발생하는 이유는 무엇일까? GC는 **도달 불가능한(unreachable)** 객체만 수거하기 때문에, 실수로 참조가 유지되면 메모리에서 해제되지 않는다.

## GC의 메모리 관리 전략

JVM/ART가 메모리 압박 상황에서 대응하는 순서:

1. **Soft Reference 정리** → 메모리 부족 시 가장 먼저 회수되는 약한 참조
    - 이미지 캐시, 계산 결과 캐시 등 재생성 가능한 데이터
    - System.gc() 호출이나 메모리 압박 시 자동 해제
2. **GC 빈도 증가**
    - Minor GC를 더 자주 실행해서 Young Generation 청소
    - Allocation 실패 시 즉시 GC 트리거
    - GC pause time이 늘어나면서 앱이 버벅거리기 시작
3. **OutOfMemoryError 발생**
    - Heap 최대 크기에 도달하고 GC로도 공간 확보 실패
    - 앱 크래시로 이어짐

### "Soft Reference로 캐시 관리, GC 빈도 증가로 버티다가, 결국 OOM으로 크래시"

## Generational GC의 실제 동작

<img width="996" height="750" alt="image" src="https://github.com/user-attachments/assets/d877823e-746f-46ed-ad3b-1b16523acf19" />

안드로이드의 Heap은 세대별로 나뉘어 관리된다. 왜 이렇게 복잡하게 만들었을까?

### 통계적 관찰: Weak Generational Hypothesis

`실험 결과:
- 생성된 객체의 90%는 1초 내에 죽음
- 5초 이상 살아남은 객체의 80%는 앱 종료까지 생존`

이 패턴을 활용한 것이 Generational GC다.

### Young Generation 구조

<img width="1004" height="388" alt="image" src="https://github.com/user-attachments/assets/dee6b4c6-a6ef-4a8c-829a-f8be2e282a12" />

### RecyclerView처럼 빈번하게 호출되는 곳에서 객체 생성을 최소화하면 Minor GC 빈도가 크게 줄어든다.

## Write Barrier의 실제 구현

Write Barrier는 어떻게 Old→Young 참조를 추적할까?

### Card Table 메커니즘

```kotlin
*// JVM 내부 의사코드*
fun writeBarrier(object: Any, field: Field, newValue: Any) {
    *// 1. 실제 값 할당*
    object.field = newValue
    
    *// 2. Old → Young 참조 체크*
    if (isInOldGen(object) && isInYoungGen(newValue)) {
        *// 3. Card Table에 마킹*
        val cardIndex = getCardIndex(object.address)
        cardTable[cardIndex] = DIRTY
    }
}
```

```kotlin
*// Minor GC 시*
fun minorGC() {
    *// Young Generation 스캔*
    markFromRoots()
    
    *// Dirty Card만 확인 - 전체 Old를 스캔하지 않음!*
    for (card in cardTable) {
        if (card == DIRTY) {
            scanCard(card)
        }
    }
}
```

성능상 이점:

- Card 하나 = 512 bytes
- 4GB Old Gen = 8M개 카드
- Dirty Card는 보통 1% 미만 → 80,000개만 확인

## Concurrent GC의 고질적인 문제

동시 실행 GC는 완벽하지 않다. 앱과 GC가 동시에 실행되면서 발생하는 문제들이 있다.

### Lost Update Problem

시나리오:
1. GC: Object A를 Gray로 마킹
2. App: A.field = B (새 객체 B 할당)
3. GC: A를 Black으로 마킹 (B를 모르고 지나감)
4. App: A.field = null (B 참조 제거)
5. GC: B는 White로 남음 → 잘못 수거!

### SATB (Snapshot-At-The-Beginning) 해결책

```kotlin
*// Write Barrier with SATB*
fun writeBarrier(object: Any, field: Field, newValue: Any) {
    val oldValue = object.field  *// 기존 값 저장*
    
    if (gcPhase == MARKING && oldValue != null) {
        *// SATB 버퍼에 기존 값 기록*
        satbBuffer.add(oldValue)
    }
    
    object.field = newValue
}
```

## 그래서 ART는 Dalvik과 뭐가 다른가

### Dalvik의 한계

1. **Stop-The-World GC**: 전체 앱 정지
2. **Mark & Sweep만 사용**: 단편화 심각
3. **단일 세대**: Young/Old 구분 없음

### ART의 혁신

1. **Concurrent Copying GC**: 실행 중 복사 & 압축
2. **Generational**: Young/Old 분리 관리
3. **Read Barrier**: 항상 최신 객체 위치 추적

### 실제 성능 차이

Dalvik (Android 4.4):
- GC Pause: 50-100ms
- 초당 GC 횟수: 5-10회
- 사용자 체감: 눈에 띄는 버벅임

ART (Android 8.0+):
- GC Pause: 2-5ms
- 초당 GC 횟수: 1-2회
- 사용자 체감: 부드러운 스크롤

## GC 튜닝 실전 팁

### Allocation Tracking

- Allocation Rate: 초당 할당되는 바이트
- Allocation Count: 초당 생성되는 객체 수
- 목표: < 50KB/sec, < 100 objects/sec

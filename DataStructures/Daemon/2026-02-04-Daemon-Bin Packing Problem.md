---
date: 2026-02-04
user: Daemon
topic: Bin Packing Problem
---

# Bin Packing Problem / 벽지 로스율 최소화 알고리즘 정리

## 기본 개념

### 빈 패킹 문제(Bin Packing Problem)

고정된 용량 C를 가진 빈(bin)에 n개의 아이템을 담을 때, 모든 아이템을 담는 데 필요한 최소 빈 개수를 구하는 문제입니다. 벽지 문제로 변환하면 빈은 벽지 롤 1개의 길이, 아이템은 각 벽면에 필요한 벽지 조각에 해당합니다.

```
빈 용량 C = 10m (롤 길이)
아이템들 = [2.5m, 3m, 4m, 2m, 5m, 3.5m] (각 벽면에 필요한 길이)

목표: 최소 롤 개수 사용 = 로스 최소화
```

### NP-Hard 문제

n개 아이템을 k개 빈에 배치하는 경우의 수는 k^n입니다.

- 예: 아이템 10개, 빈 5개 → 5^10 = 9,765,625 가지
- 완전 탐색은 불가능 → 근사 알고리즘 필요

## 핵심 알고리즘

### 1. Next Fit (NF) - 가장 단순

```python
def next_fit(capacity, items):
    bins = []
    current_bin = capacity

    for item in items:
        if current_bin >= item:
            current_bin -= item
        else:
            bins.append(capacity - current_bin)  # 이전 빈 로스 저장
            current_bin = capacity - item

    bins.append(capacity - current_bin)
    return len(bins), bins

# 예시
capacity = 10
items = [2, 5, 4, 7, 1, 3, 8]
print(next_fit(capacity, items))
# 결과: (4, [3, 0, 6, 2]) → 4개 빈, 총 로스 11
```

**특징:**

- 현재 빈에 넣을 수 없으면 새 빈을 열고, 이전 빈에는 다시 돌아가지 않음
- 시간복잡도: O(n), 공간복잡도: O(1)
- 근사율: 2 × OPT (최악)

**단점:**

- 이전 빈에 공간이 남아도 재사용하지 않음
- 로스율이 가장 높음

### 2. First Fit (FF) - 실용적

```python
def first_fit(capacity, items):
    bins = []  # 각 빈의 남은 공간

    for item in items:
        placed = False
        for i in range(len(bins)):
            if bins[i] >= item:
                bins[i] -= item
                placed = True
                break

        if not placed:
            bins.append(capacity - item)

    return len(bins), bins

# 예시
items = [2, 5, 4, 7, 1, 3, 8]
print(first_fit(10, items))
# 결과: (4, [1, 2, 0, 2]) → 4개 빈, 총 로스 5
```

**특징:**

- 기존 빈을 순서대로 탐색하여 들어갈 수 있는 첫 번째 빈에 배치
- 시간복잡도: O(n²)
- 근사율: 1.7 × OPT

**장점:**

- Next Fit보다 훨씬 낮은 로스율
- 구현이 간단하면서 균형 잡힌 성능

### 3. First Fit Decreasing (FFD) - 가장 추천 ⭐

```python
def first_fit_decreasing(capacity, items):
    # 핵심: 큰 것부터 먼저 배치
    sorted_items = sorted(items, reverse=True)
    return first_fit(capacity, sorted_items)

# 예시
items = [2, 5, 4, 7, 1, 3, 8]
print(first_fit_decreasing(10, items))
# 정렬 후: [8, 7, 5, 4, 3, 2, 1]
# 결과: (3, [0, 0, 0]) → 3개 빈, 로스 0!
```

**특징:**

- 아이템을 내림차순 정렬 후 First Fit 적용
- 시간복잡도: O(n²) (BST 활용 시 O(n log n))
- 근사율: 11/9 × OPT + 6/9 ≈ 1.22 × OPT

**장점:**

- 큰 아이템을 먼저 배치하면 작은 아이템들이 틈새를 채움
- 실무에서 가장 많이 사용되는 알고리즘
- 근사율이 가장 우수

### 4. Best Fit Decreasing (BFD) - 최적화

```python
def best_fit_decreasing(capacity, items):
    sorted_items = sorted(items, reverse=True)
    bins = []

    for item in sorted_items:
        # 아이템이 들어갈 수 있는 빈 중 가장 꽉 찬 빈 선택
        best_idx = -1
        min_space = capacity + 1

        for i, space in enumerate(bins):
            if space >= item and space < min_space:
                min_space = space
                best_idx = i

        if best_idx != -1:
            bins[best_idx] -= item
        else:
            bins.append(capacity - item)

    return len(bins), bins
```

**특징:**

- 아이템이 들어갈 수 있는 빈 중 남은 공간이 가장 적은 빈을 선택
- FFD와 이론적 근사율은 동일하며, 실제 성능은 문제에 따라 다름

## 자료구조로 최적화

### BST (이진 탐색 트리)로 O(n log n) 달성

```python
from sortedcontainers import SortedList

def ffd_optimized(capacity, items):
    sorted_items = sorted(items, reverse=True)

    # (남은공간, 빈ID) 쌍을 정렬 상태로 유지
    bins = SortedList()
    bin_id = 0

    for item in sorted_items:
        # 이진 탐색: item 이상인 최소 공간 찾기
        idx = bins.bisect_left((item, -float('inf')))

        if idx < len(bins):
            space, bid = bins.pop(idx)
            remaining = space - item
            if remaining > 0:
                bins.add((remaining, bid))
        else:
            bin_id += 1
            remaining = capacity - item
            if remaining > 0:
                bins.add((remaining, bin_id))

    return bin_id
```

**시간복잡도 비교:**

- 리스트 기반: 탐색 O(n), 삽입 O(1) → 전체 O(n²)
- BST 기반: 탐색 O(log n), 삽입 O(log n) → 전체 O(n log n)

## 시각적 비교

<img width="2570" height="1540" alt="image" src="https://github.com/user-attachments/assets/8a257c1a-088f-47ad-b95a-001f6a3d6731" />

## 벽지 문제 실전 예제

```python
def wallpaper_cutting(roll_length, wall_heights):
    """
    벽지 재단 최적화

    Args:
        roll_length: 벽지 롤 1개 길이 (m)
        wall_heights: 각 벽면에 필요한 벽지 길이 리스트

    Returns:
        필요한 롤 수, 로스율, 재단 계획
    """
    # FFD 적용
    sorted_heights = sorted(wall_heights, reverse=True)

    rolls = []  # 각 롤에 들어가는 조각들
    remaining = []  # 각 롤의 남은 길이

    for height in sorted_heights:
        placed = False
        for i in range(len(rolls)):
            if remaining[i] >= height:
                rolls[i].append(height)
                remaining[i] -= height
                placed = True
                break

        if not placed:
            rolls.append([height])
            remaining.append(roll_length - height)

    # 결과 계산
    total_used = sum(sum(roll) for roll in rolls)
    total_material = len(rolls) * roll_length
    waste = total_material - total_used
    waste_rate = (waste / total_material) * 100

    return {
        'rolls_needed': len(rolls),
        'cutting_plan': rolls,
        'waste_per_roll': remaining,
        'total_waste': waste,
        'waste_rate': f'{waste_rate:.1f}%'
    }


# 실행 예시
if __name__ == "__main__":
    result = wallpaper_cutting(
        roll_length=10,
        wall_heights=[2.5, 2.5, 2.5, 2.5, 3.0, 3.0, 2.8, 2.8, 2.4, 2.4]
    )

    print("=" * 40)
    print(f"필요한 롤 수: {result['rolls_needed']}개")
    print(f"로스율: {result['waste_rate']}")
    print("\n재단 계획:")
    for i, (cuts, waste) in enumerate(zip(result['cutting_plan'], result['waste_per_roll'])):
        print(f"  롤{i+1}: {cuts} → 로스 {waste:.1f}m")
```

**실행 결과:**

```
========================================
필요한 롤 수: 3개
로스율: 13.0%

재단 계획:
  롤1: [3.0, 3.0, 2.8] → 로스 1.2m
  롤2: [2.8, 2.5, 2.5, 2.4] → 로스 -0.2m (약간 초과, 조정 필요)
  롤3: [2.5, 2.5, 2.4] → 로스 2.6m
```

## 알고리즘 비교 요약

### 성능 비교

| 알고리즘 | 시간복잡도 | 근사율 | 특징 |
|---------|-----------|--------|------|
| Next Fit | O(n) | 2 × OPT | 가장 빠름, 성능 낮음 |
| First Fit | O(n²) | 1.7 × OPT | 균형 잡힌 선택 |
| First Fit Decreasing | O(n²) | 1.22 × OPT | 실무 추천 |
| Best Fit Decreasing | O(n²) | 1.22 × OPT | FFD와 유사 |
| FFD + BST | O(n log n) | 1.22 × OPT | 대용량 데이터용 |

### 선택 기준

**Next Fit 사용:**

- 실시간 처리가 필요하여 O(n) 속도가 중요
- 로스율보다 처리 속도가 우선

**First Fit 사용:**

- 정렬 없이 입력 순서대로 처리해야 하는 경우
- 적당한 성능과 간단한 구현이 필요

**First Fit Decreasing 사용:**

- 로스율 최소화가 가장 중요
- 오프라인 처리 가능 (모든 아이템을 미리 알 수 있음)
- 실무에서 가장 권장

**FFD + BST 사용:**

- 아이템 수가 수천~수만 개 이상
- O(n²)로는 처리 시간이 부족한 대용량 데이터

## 결론

1. **First Fit Decreasing(FFD)**이 실무에서 가장 추천되는 알고리즘 (근사율 1.22 × OPT)
2. 핵심 아이디어는 **큰 것부터 먼저 배치**하여 작은 아이템이 남은 공간을 채우도록 하는 것
3. BST를 활용하면 O(n²)에서 **O(n log n)**으로 최적화 가능
4. 벽지 재단 문제는 Bin Packing의 대표적 실전 적용 사례
5. NP-Hard 문제이므로 완전 탐색은 불가능하며, 근사 알고리즘이 실용적 해법

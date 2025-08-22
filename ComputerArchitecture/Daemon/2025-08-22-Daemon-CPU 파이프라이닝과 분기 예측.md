---
date: 2025-08-22
user: Daemon
topic: "CPU 파이프라이닝과 분기 예측"
---

## Part 1: CPU 파이프라이닝 기초

### 파이프라이닝이란?

CPU가 명령어를 처리하는 5단계를 동시에 수행하는 기법입니다.

- **Fetch**: 명령어를 메모리에서 가져옴
- **Decode**: 명령어 해석
- **Execute**: 연산 수행
- **Memory**: 메모리 접근
- **Write Back**: 결과 저장

`순차 처리: 명령어1 완료 → 명령어2 시작 (5사이클 × n개)
파이프라인: 여러 명령어가 단계별로 겹쳐서 실행 (5 + n-1 사이클)`

### 파이프라인 해저드

**제어 해저드(Control Hazard)**: 분기 명령어로 인해 다음 명령어를 모를 때 발생

- if문을 만나면 조건 결과를 알기 전까지 다음 명령어를 fetch할 수 없음
- 잘못된 예측 시 파이프라인을 비우고 다시 시작 (10-20 사이클 손실)

---

## Part 2: 분기 예측 메커니즘

### 분기 예측이 필요한 이유

현대 CPU는 10-20단계 파이프라인을 사용합니다. 분기 예측 실패 시 파이프라인을 전부 비워야 하므로 성능에 큰 영향을 미칩니다.

### 동적 분기 예측

**2-bit Saturating Counter**

- 상태: Strongly Taken → Weakly Taken → Weakly Not Taken → Strongly Not Taken
- 한 번의 예외로 예측이 바뀌지 않아 안정적

### 예측하기 쉬운 패턴 vs 어려운 패턴

kotlin

`*// 🔴 예측 어려움 (50% 정확도)*
if (random.nextBoolean()) { }  *// 랜덤 패턴// 🟢 예측 쉬움 (거의 100% 정확도)*
if (i < 5000) { }  *// 규칙적 패턴*
if (sortedValue > threshold) { }  *// 정렬된 데이터*`

---

## Part 3: 안드로이드 개발 시 적용

### 1. when vs if-else 성능 차이

- **if-else 체인**: 순차적 비교 O(n)
- **when 표현식**: tableswitch/lookupswitch로 컴파일 O(1)
- 3개 이상 조건 비교 시 when 사용 권장

### 2. 인라인 함수의 이점

- 함수 호출 오버헤드 제거
- 분기 명령어 감소
- 작고 자주 호출되는 함수에 inline 키워드 사용

### 3. 정렬을 통한 최적화

kotlin

`*// 정렬되지 않은 배열: 분기 예측 실패 많음// 정렬된 배열: 특정 지점부터 계속 true → 예측 성공률 높음*
val sorted = array.sorted()
for (value in sorted) {
    if (value > threshold) sum += value
}`

### 4. 분기 제거 기법

- **룩업 테이블**: 다중 분기를 배열 인덱싱으로 대체
- **조건부 이동**: 삼항 연산자로 분기 최소화
- **비트 연산**: 수학적 연산으로 조건문 대체

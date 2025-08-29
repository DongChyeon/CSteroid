---
date: 2025-08-29
user: Daemon
topic: "Thread-Safe 자료구조와 동시성 제어"
---

# Thread-Safe 자료구조와 동시성 제어

---

## Thread-Safe란?

여러 스레드가 동시에 접근해도 **데이터 일관성과 정확성이 보장**되는 것

### Thread-Unsafe 예시


```kotlin
*// Thread-Unsafe 예시*
var counter = 0
repeat(1000) {
    thread {
        counter++  *// Race Condition 발생*
    }
}
*// 결과: 1000이 아닌 예측 불가능한 값*`
```

---

## CPU가 실제로 수행하는 단계

`counter++`는 실제로 3단계로 실행됨:

1. **READ**: 메모리에서 현재 값 읽기
2. **ADD**: 읽은 값에 1 더하기
3. **WRITE**: 계산된 값을 메모리에 저장

### 문제 상황: Race Condition

두 스레드가 동시에 실행될 때:

- **정상적인 경우 (순차 실행)**: 각 단계가 순서대로 실행
- **Race Condition (동시 실행)**: CPU 수행 단계가 겹쳐서 실제 값에 오차 발생

```kotlin
Thread A: [READ: 0] → [ADD: 1] → [WRITE: 1]
Thread B: [READ: 0] → [ADD: 1] → [WRITE: 1]
          ↑ 둘 다 0을 읽음!
결과: 1 (2가 되어야 하는데 오류 발생!)
```

---

## 안드로이드 특화 Thread-Safe 패턴

### 1. LiveData

LiveData는 **Thread-Safe한 데이터 홀더**:

- 데이터 변경 시 자동으로 UI 업데이트
- 안드로이드 생명주기 인식으로 메모리 누수 방지

### 기본 사용법

```kotlin
class SimpleExample : ViewModel() {
    private val _text = MutableLiveData<String>()
    val text: LiveData<String> = _text
    
    fun updateText(newText: String) {
        _text.value = newText
    }
}
```

### 내부 동작 원리 (단순화)

```kotlin
class SimplifiedLiveData<T> {
    @Volatile  *// 모든 스레드가 최신 값을 봄*
    private var data: T? = null
    
    private val lock = Any()  *// 동기화용 락*
    
    *// Main 스레드에서만 호출 가능*
    fun setValue(value: T) {
        assertMainThread()  *// Main 스레드 체크*
        data = value
        notifyObservers()  *// UI 업데이트*
    }
    
    *// 어느 스레드에서든 호출 가능*
    fun postValue(value: T) {
        synchronized(lock) {  *// 동시 접근 방지*
            pendingData = value
        }
        *// Main 스레드로 작업 예약*
        Handler(Looper.getMainLooper()).post {
            setValue(pendingData)
        }
    }
}
```

---

## 핵심 Thread-Safety 메커니즘

### 1. 동기화를 위한 Lock 사용

```java
final Object mDataLock = new Object();  *// 동기화용 락*
volatile Object mPendingData = NOT_SET;  *// volatile로 가시성 보장*
private volatile Object mData;           *// 실제 데이터도 volatile*
```

**volatile 키워드**: 모든 스레드가 항상 최신 값을 보도록 보장

### 2. postValue의 Thread-Safe 구현

```java
protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {  *// 동기화 블록*
        postTask = mPendingData == NOT_SET;
        mPendingData = value;  *// 임시 저장*
    }
    if (!postTask) {
        return;  *// 이미 대기 중인 작업이 있으면 덮어쓰기만*
    }
    *// Main 스레드로 작업 전달*
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}
```

**핵심 포인트**:

- `synchronized` 블록으로 여러 스레드가 동시에 postValue 호출해도 안전
- 연속 호출 시 마지막 값만 전달 (`mPendingData` 덮어쓰기)
- 실제 값 설정은 Main 스레드에서만 실행

### 3. Main 스레드에서만 setValue 허용

```java
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");  *// Main 스레드 체크*
    mVersion++;
    mData = value;
    dispatchingValue(null);  *// 옵저버들에게 알림*
}

static void assertMainThread(String methodName) {
    if (!ArchTaskExecutor.getInstance().isMainThread()) {
        throw new IllegalStateException(
            "Cannot invoke " + methodName + " on a background thread"
        );
    }
}
```

**Main 스레드 강제**: setValue는 Main 스레드가 아니면 예외 발생

### 4. postValue → setValue 변환 과정

```java
private final Runnable mPostValueRunnable = new Runnable() {
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {  *// 값 가져올 때도 동기화*
            newValue = mPendingData;
            mPendingData = NOT_SET;  *// 초기화*
        }
        setValue((T) newValue);  *// Main 스레드에서 setValue 호출*
    }
};
```

**작동 순서**:

1. 백그라운드 스레드: `postValue()` → `mPendingData`에 저장
2. Main 스레드 큐에 Runnable 등록
3. Main 스레드: Runnable 실행 → `setValue()` 호출

---

## Thread-Safety 보장 요약

### 세 가지 핵심 메커니즘

### 1. **스레드 분리**

- `setValue`: Main 스레드 전용
- `postValue`: 모든 스레드 가능

### 2. **동기화**

- `synchronized(mDataLock)`: 임계 영역 보호
- `volatile`: 메모리 가시성 보장

### 3. **단방향 데이터 흐름**

`백그라운드 스레드 → postValue → mPendingData
                                     ↓
                             Main 스레드 큐
                                     ↓
                   Main 스레드 → setValue → mData → 옵저버`

---

## 주의사항

### 연속 postValue: 마지막 값만 전달됨

```java
if (!postTask) {
    return;  *// 이미 대기 중이면 값만 덮어쓰고 리턴*
}
```

### getValue()는 Thread-Safe하지 않음

```java
public T getValue() {
    Object data = mData;  *// 동기화 없이 읽기// 최신 값이 아닐 수 있음*
}
```

---

## 결론

LiveData는 **동기화 블록 + volatile + Main 스레드 강제**를 통해 Thread-Safety를 보장합니다.

백그라운드에서는 `postValue`만 사용하고, UI 업데이트는 자동으로 Main 스레드에서 처리됩니다.

---
date: 2025-07-05
user: DongChyeon
topic: 뮤텍스와 세마포어의 차이점은 무엇인가?
---

# 뮤텍스와 세마포어의 차이점은 무엇인가?

뮤텍스에 대해 알기 전에 `상호 배제(Mutual Exclusion)`에 대해서 알아야할 필요가 있다.

## 1. 상호 배제(Mutual Exclusion)란?

- 정의: 여러 프로세스나 스레드가 `공유 자원(shared resource)`에 접근할 때, 동시에 접근하지 못하도록 제어하는 것
- 필요 이유: 공유 자원(예: 변수, 파일, DB Connection 등)에 동시에 접근하면 데이터 무결성이 깨질 수 있다.
  - 예) 동시에 같은 변수 증가 -> 둘 다 이전 값을 읽고 +1 -> 결과가 1만 증가 (원래는 2 증가해야 함)

### 예시 코드 (문제 상황)

```kotlin
var counter = 0

fun main() = runBlocking {
    val jobs = List(1000) {
        launch(Dispatchers.Default) {
            repeat(1000) {
                counter++
            }
        }
    }
    jobs.forEach { it.join() } 
    println("Final counter value: $counter")
}
```

- 의도된 값: 1,000 x 1,000 = 1,000,000
- 실제 출력: 의도된 값보다 작은 값 (race condition 발생)

## 2. 뮤텍스(Mutex) 란 무엇인가?

- 정의: 공유 자원의 상호 배제를 위한 락
- 동작: lock()으로 잠그고 unlock()으로 해제. 락을 획득한 스레드만 해제 가능
- 종류
  - Normal Mutex
    - 기본 뮤텍스. 락을 획득하지 않은 스레드가 unlock()하면 에러 발생
  - Recursive Mutex
    - 락을 획득한 스레드가 같은 뮤텍스를 중복해서 lock() 가능
    - lock() 호출 횟수만큼 unlock() 호출해야 해제됨
  - Error Checking Mutex
    - 락을 획득하지 않은 스레드가 unlock()하면 에러 발생
    - 동일 스레드가 중복 lock() 시 deadlock 대신 에러 발생
  - Robust Mutex
    - 락을 잡은 스레드가 비정상 종료되더라도 다른 스레드가 복구 가능

### 예시 코드 (Kotlin Coroutine Mutex)

```kotlin
var counter = 0
val mutex = Mutex()

fun main() = runBlocking {
    val jobs = List(1000) {
        launch(Dispatchers.Default) {
            repeat(1000) {
                mutex.withLock {
                    counter++
                }
            }
        }
    }
    jobs.forEach { it.join() } 
    println("Final counter value: $counter")
}
```

mutex.withLock { ... } 블록을 통해 한 번에 하나의 코루틴만 counter를 수정 가능 -> 데이터 무결성 보장

## 3. 세마포어(Sempaphore) 란 무엇인가?

- 정의: 공유 자원의 접근 가능 개수를 관리하는 동기화 기법
- 동작: 정수값(counter)을 가지며, `wait(P)` 호출 시 감소, `signal(V)` 호출 시 증가
- 종류
  - Counting Semaphore
    - 0 이상의 정수값을 가짐
    - 용도: 재한된 개수의 자원 접근 제어 (예: 동시에 DB Connection 10개 제한)
  - Binary Semaphore: 0 또는 1만 가지는 이진 세마포어 (뮤텍스와 유사한 소유권 없음)
    - 값이 0 또는 1만 가짐
    - 용도: 단일 접근 제어. 뮤텍스와 유사하나 소유권 없음.

### 예시 코드 (Kotlin Coroutine Semaphore)

```kotlin
var counter = 0
val semaphore = Semaphore(4)

fun main() = runBlocking {
    val jobs = List(1000) {
        launch(Dispatchers.Default) {
            repeat(1000) {
                semaphore.withPermit {
                    counter++
                }
            }
        }
    }
    jobs.forEach { it.join() }
    println("Final counter value: $counter")
}
```

최대 4개의 코루틴만 동시에 실행되도록 제어<br>
mutex와 달리 세마포어는 소유권이 없고 여러 코루틴이 동시에 접근 가능하므로, counter++는 원자 연산이 아니면 의도된 값보다 작게 출력될 수 있다.

```kotlin
var counter = 0
val semaphore = Semaphore(4)

fun main() = runBlocking {
    val jobs = List(1000) {
        launch(Dispatchers.Default) {
            repeat(1000) {
                semaphore.withPermit {
                    counter++
                }
            }
        }
    }
    jobs.forEach { it.join() }
    println("Final counter value: $counter")
}
```

이진 세마포어와 같이 Semaphore(1) 로 초기화하면 뮤텍스처럼 동작한다.<br>
단, 소유권이 없으므로 mutex와 완전히 동일하지는 않다.<br>
출력값은 의도된 값 (1000,000)과 동일하게 나온다.

## 4. 그래서 둘의 차이점은?

| 구분 | 세마포어 | 뮤텍스 |
|------|----------|--------|
| **의미** | 공유 자원 접근 개수 제어 | 상호 배제 락 |
| **값** | 0 이상의 정수 | locked/unlocked |
| **소유권** | 없음 | 있음 |
| **사용 목적** | 자원 개수 제한 | 단일 자원 보호 |
| **signal/unlock** | 누구나 호출 가능 | 락 소유자만 해제 가능 |
| **종류** | Counting, Binary | Normal, Recursive 등 |

- 세마포어는 동시에 접근 가능한 개수를 제어하는 기법으로, 뮤텍스의 일반화된 형태라고 볼 수 있다.
- 반면 뮤텍스는 단 하나의 스레드만 접근하도록 보장하는 상호 배제 전용 락이다.
- 따라서 뮤텍스는 Binary Sempaphore의 특수한 형태이지만, 락의 소유권 관리가 엄격하여 unlock() 안전성이 보장된다는 점이 가장 큰 차이다.
- 실무에서는 자원 개수 제한이 필요하면 세마포어, Critical Section 보호가 필요하면 뮤텍스를 사용하는 것이 일반적이다.
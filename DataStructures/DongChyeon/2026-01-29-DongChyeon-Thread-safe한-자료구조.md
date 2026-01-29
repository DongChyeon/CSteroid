---
date: 2026-01-29
user: DongChyeon
topic: Thread-safe한 자료구조
---

### 1. Thread-safe 하다의 의미

Thread-safe 하다의 의미는 멀티스레드 환경에서 여러 스레드가 동시에 접근하더라도 데이터의 일관성과 무결성이 유지되는 것을 의미한다.<br>
만약 공유 변수 A에 대해 두 스레드가 동시에 +1을 해주는 상황을 가정해보자.

![race_condition.png](assets/race_condition.png)

공유 변수 A의 초기값이 1이라고 할 때, 두 스레드가 동시에 +1 연산을 수행하면 다음과 같은 상황이 발생할 수 있다.

1. 스레드 1이 A의 값을 읽어옴 (A = 1)
2. 스레드 2가 A의 값을 읽어옴 (A = 1)
3. 스레드 1이 A에 1을 더한 값을 저장 (A = 2)
4. 스레드 2가 A에 1을 더한 값을 저장 (A = 2)
5. 최종적으로 A의 값은 2가 됨

즉, 다른 스레드가 동시에 접근하여 값을 변경하는 바람에 의도한 대로 값이 증가하지 않는 문제가 발생한다. 이를 레이스 컨디션(Race Condition)이라고 한다.<br>
이러한 문제를 방지하기 위해 Thread-safe한 자료구조는 내부적으로 동기화 메커니즘을 사용하여 여러 스레드가 동시에 접근하더라도 데이터의 일관성을 유지한다.

### 2. Vector와 SynchronizedList

Vector는 Java에서 제공하는 Thread-safe한 동적 배열 자료구조이다.<br>
내부 코드를 살펴보면, 요소를 추가하거나 제거하는 메서드들이 synchronized 키워드로 선언되어 있는 것을 확인할 수 있다.

```java
public synchronized void addElement(E obj) { /* ... */ }

public synchronized boolean removeElement(Object obj) { /* ... */ }

public synchronized void removeAllElements() { /* ... */ }
```

synchronized 키워드는 해당 메서드가 호출될 때 객체의 모니터 락을 획득하도록 하여, 동시에 여러 스레드가 접근하는 것을 방지하는 상호 배제의 역할을 한다.<br>
만약, 여러 스레드가 addElement 메서드를 호출하면, 한 스레드가 메서드를 실행하는 동안 다른 스레드는 대기하게 된다.

```kotlin
Collections.synchronizedList(ArrayList<Int>())
```

SynchronizedList는 원본 리스트를 감싸서 Thread-safe하게 만들어주는 래퍼 클래스이다. 내부적으로 모든 메서드가 synchronized 블록으로 감싸져 있어, 여러 스레드가 동시에 접근하더라도 안전하다.

```java
// Collections.java 내부
static class SynchronizedCollection<E> implements Collection<E>, Serializable {
    final Collection<E> c;  // 원본 컬렉션 (Backing Collection)
    final Object mutex;     // 동기화에 사용할 잠금 객체

    SynchronizedCollection(Collection<E> c) {
        this.c = Objects.requireNonNull(c);
        this.mutex = this; // 기본값은 자기 자신(this)을 열쇠로 사용
    }

    SynchronizedCollection(Collection<E> c, Object mutex) {
        this.c = c;
        this.mutex = mutex; // 외부에서 주입된 객체를 열쇠로 사용 가능
    }
    // ...
}
```

```java
// Collections.java 내부
static class SynchronizedList<E> extends SynchronizedCollection<E> implements List<E> {
    final List<E> list;

    // 핵심: 원본 리스트의 메서드를 호출하기 전, 반드시 mutex를 잠급니다.
    public E get(int index) {
        synchronized (mutex) { return list.get(index); }
    }

    public E set(int index, E element) {
        synchronized (mutex) { return list.set(index, element); }
    }

    public void add(int index, E element) {
        synchronized (mutex) { list.add(index, element); }
    }

    public E remove(int index) {
        synchronized (mutex) { return list.remove(index); }
    }
    // ... 모든 List 메서드가 이런 식으로 구현되어 있습니다.
}
```

내부적으로는 모든 메서드 호출을 하나의 공유 객체(mutex)를 이용한 synchronized 블록으로 감싸 스레드 안전성을 보장해주고 있다.

### 3. Concurrent Collections

하지만 위와 같이 모든 메서드에 synchronized를 걸면, 성능 저하가 발생할 수 있다.<br>
그 이유는 동기화가 걸린 메서드가 실행되는 동안 다른 스레드들은 대기해야 하기 때문이다.<br>
따라서, Java에서는 더 나은 성능을 제공하는 Concurrent Collections를 제공한다.

ConcurrentHashMap, CopyOnWriteArrayList, ConcurrentLinkedQueue 등이 대표적인 예이다.

#### ConcurrentHashMap

ConcurrentHashMap은 `분할 잠금(Lock Stripping)`과 `CAS(Compare-And-Swap)` 기법을 사용한다.<br>
처음 데이터를 넣을 때는 락을 걸지 않는다. 대신 CAS 연산을 통해 비어있는 버킷에 데이터를 삽입한다.<br>
이미 데이터가 있는 버킷이라면 버킷의 첫 번째 노드(Head)에만 synchronized를 건다. 즉, 서로 다른 버킷에 접근한다면 동시에 일을 할 수 있다.<br>
volatile 키워드를 통해 락 없이도 항상 최신 데이터를 읽을 수 있도록 설계되어 있다.

#### CopyOnWriteArrayList

CopyOnWriteArrayList는 `불변성(Immutability)`의 원리를 이용한다.<br>
읽기에는 락이 필요 없고, 쓰기 작업이 발생할 때마다 내부 배열을 복사하여 새로운 배열을 만든다.<br>
따라서, 읽기 작업이 매우 빈번하고 쓰기 작업이 드문 경우에 적합하다.

#### ConcurrentLinkedQueue

ConcurrentLinkedQueue는 락을 사용하지 않는 `Non-blocking (Lock-free)` 알고리즘을 사용한다.<br>
새 노드를 연결할 때 synchronized 키워드 대신 CAS 연산을 사용하여 원자적으로 연결 작업을 수행한다.<br>
CAS 연산을 통해 만약에 다른 스레드가 먼저 변경을 했다면 다시 시도하는 방식으로 동작한다.

#### volatile 키워드

앞선 ConcurrentHashMap에서는 volatile 키워드를 사용하여 락 없이도 항상 최신 데이터를 읽을 수 있도록 설계되었다고 언급했다.<br>
volatile 키워드는 CPU 캐시가 아닌 항상 메모리에서 값을 읽고 쓰도록 보장한다.

메모리(RAM)보다 CPU가 훨씬 빠르기 때문에 CPU는 자주 사용하는 데이터를 캐시에 저장한다.<br>
하지만 멀티코어 환경에서는 각 코어가 각각의 캐시를 가지고 있기 때문에, 한 코어에서 변경한 값이 다른 코어의 캐시에 반영되지 않을 수 있다.<br>
따라서, volatile 키워드를 사용하면 해당 변수에 대한 모든 읽기/쓰기가 메모리에서 직접 이루어지도록 하여, 여러 스레드가 항상 최신 값을 볼 수 있도록 보장한다.

### 동기화 알고리즘

여기서 나온 Lock-Free, CAS 등과 같은 용어는 해당 글을 참고하면 좋다.<br>
[2025-08-14-DongChyeon-Lock-Free & Wait-Free 알고리즘.md](../../OS/DongChyeon/2025-08-14-DongChyeon-Lock-Free%20%26%20Wait-Free%20%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98.md)
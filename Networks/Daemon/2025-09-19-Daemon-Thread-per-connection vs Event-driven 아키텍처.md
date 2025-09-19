---
date: 2025-09-19
user: Daemon
topic: "Thread-per-connection vs Event-driven 아키텍처"
---

## 기본 개념

### Thread-per-connection

각 네트워크 연결마다 독립적인 스레드를 할당하는 방식입니다. 예를 들어 100명의 클라이언트가 접속하면 100개의 스레드가 생성되어 각각의 연결을 처리합니다.

### Event-driven

단일 스레드나 소수의 스레드가 이벤트 루프를 통해 다수의 연결을 비동기적으로 처리하는 방식입니다. I/O 멀티플렉싱을 사용하여 하나의 스레드가 수천 개의 연결을 관리할 수 있습니다.

**장점:**

- 적은 리소스로 대량의 연결 처리 가능
- Context switching 오버헤드 최소화
- 메모리 효율적 (스레드 스택 메모리 절약)

**단점:**

- 복잡한 프로그래밍 모델 (콜백 지옥)
- CPU 집약적 작업에서 블로킹 발생
- 디버깅 어려움

## Thread-per-connection 상세 분석

### 동작 방식

**구조:**

- Accept Thread: 새 연결 수락
- Worker Threads: 각 연결당 하나씩 할당
- Thread Pool: 스레드 재사용 (선택적)

**특징:**

- Blocking I/O 사용
- 동기식 프로그래밍 모델
- 스레드 간 독립성 보장

### 리소스 사용량

**메모리 사용:**

- 스레드당 스택 크기: 1MB (기본값)
- 1000개 연결 시: 1GB 메모리 필요
- 10000개 연결 시: 10GB 메모리 필요

**CPU 오버헤드:**

- Context Switch 시간: ~10μs
- 스레드 수 증가 시 스케줄링 오버헤드 급증

## Event-driven 상세 분석

### I/O 멀티플렉싱 메커니즘

**select (POSIX.1-2001):**

- 최대 1024개 파일 디스크립터 제한
- O(n) 시간 복잡도
- 모든 FD를 순회하며 확인

**poll (POSIX.1-2001):**

- FD 개수 제한 없음
- 여전히 O(n) 시간 복잡도
- select보다 확장성 개선

**epoll (Linux 2.5.44+):**

- O(1) 시간 복잡도
- Edge-triggered/Level-triggered 모드
- 대규모 연결에 최적화

**kqueue (BSD/macOS):**

- epoll과 유사한 성능
- 파일 시스템 이벤트도 지원

### Event Loop 구현

**핵심 구성요소:**

- Event Queue: I/O 이벤트 대기열
- Callback Queue: 실행 대기 콜백
- Event Loop: 이벤트 처리 순환

**처리 과정:**

1. I/O 이벤트 대기 (select/poll/epoll)
2. 준비된 이벤트 수집
3. 각 이벤트에 대한 콜백 실행
4. 다음 이벤트 대기

### 응답 시간 특성

**Thread-per-connection:**

- 일정한 응답 시간
- CPU 작업에서 우수
- I/O 대기 시 비효율

**Event-driven:**

- I/O 작업에서 빠른 응답
- CPU 작업 시 전체 블로킹
- 이벤트 큐 길이에 따라 가변적

## C10K Problem

### 문제점

10,000개 동시 연결 처리 시 Thread-per-connection의 한계:

**시스템 리소스 고갈:**

- 메모리 부족 (10GB 필요)
- 파일 디스크립터 한계
- 프로세스/스레드 수 제한

**성능 저하:**

- Context switching 오버헤드
- 캐시 미스 증가
- 스케줄러 과부하

### 해결 방안

**Event-driven 아키텍처:**

- 단일 스레드로 다수 연결 처리
- Non-blocking I/O 사용
- 효율적인 이벤트 통지 메커니즘

## 하이브리드 접근법

### Reactor + Thread Pool

**구조:**

- Main Thread: Event Loop (I/O 처리)
- Thread Pool: CPU 집약적 작업
- Task Queue: 작업 분배

**장점:**

- I/O와 CPU 작업 모두 효율적
- 블로킹 방지
- 확장성 확보

### Multi-Reactor Pattern

**구조:**

- CPU 코어당 하나의 Event Loop
- SO_REUSEPORT로 포트 공유
- 로드 밸런싱 자동화

## 선택 가이드

### Thread-per-connection 선택 기준

**적합한 경우:**

- 동시 연결 수 < 100
- CPU 집약적 작업
- 단순한 구현 필요
- 레거시 시스템 통합

**사용 예시:**

- 내부 관리 시스템
- 소규모 데이터베이스 서버
- 배치 처리 시스템

### Event-driven 선택 기준

**적합한 경우:**

- 동시 연결 수 > 1000
- I/O 집약적 작업
- 실시간 통신 필요
- 메모리 제약 존재

**사용 예시:**

- 웹 서버
- 채팅 서버
- IoT 게이트웨이
- 스트리밍 서비스

## 최신 발전 동향

### io_uring (Linux 5.1+)

**특징:**

- 진정한 비동기 I/O
- 시스템 콜 오버헤드 최소화
- 제출/완료 큐 분리

### eBPF 기반 최적화

**특징:**

- 커널 우회 네트워킹
- XDP (eXpress Data Path)
- 사용자 공간 프로토콜 스택

## 결론

1. **Thread-per-connection**은 구현이 단순하지만 확장성에 한계
2. **Event-driven**은 대규모 동시성 처리에 최적화
3. **하이브리드 접근**이 실무에서 가장 실용적
4. **시스템 요구사항**에 따라 적절한 모델 선택 필수
5. **최신 기술** (io_uring, eBPF)로 성능 한계 극복 가능

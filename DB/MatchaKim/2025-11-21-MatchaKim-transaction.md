---
date: 2025-07-03
user: MatchaKim
topic: Transaction
---

## 트랜잭션이 뭐야?

더 쪼갤 수 없는 최소 작업단위로 commit 되거나 rollback 되어야하는 작업단위

## commit은 뭔데?

Commit은 트랜잭션 내부의 모든 작업을 완료하고, 트랜잭션 내부의 작업 변경을 모두 승인 후 반영하는거야

## rollback은?

트랜잭션 내부의 모든 작업을 결과에 반영시키지 않고, 트랜잭션 시작 이전의 상태로 되돌리는거야

## 트랜잭션은 어떤 상황에서 쓰여?

금융 서버에서 내가 A한테 3000원을 보냈는데 B가 3000원을 받았어 근데 A의 금액이 줄어야하는 쿼리 도중에 에러가 생기면 어떻게 되겠어?

## 그럼 데이터가 불일치하는 문제가 생기겠지?

그래서 트랜잭션이 있는거야 모든 쿼리문에 대해서 승인하고 반영 허가를 해야하니까

많이 사용되는 자바 프레임워크에서 트랜잭션을 보면 그 쓰임을 알 수 있어 내부에서도 트랜잭션을 지원하거든

## 어떤식으로 지원해?

다음과 같이 지원해

> isolation :
> 트랜잭션의 격리 수준 설정, 다른 트랜잭션에 어느정도 개입할 수 있는지에 대한 수준 설정이 가능.

> propagation :
> 트랜잭션 동작 도중, 다른 트랜잭션을 호출 할 때, 어떤 방식으로 동작하는지, 전파되는지 설정이 가능.

> noRollbackFor :
> 특정 예외 발생 시 rollback이 동작하지 않도록 설정

> rollbackFor :
> 특정 예외 발생 시 rollback이 동작하도록 설정

> timeout :
> 지정한 시간 내에 메소드 수행이 완료되지 않으면 rollback이 동작하도록 설정

> readOnly :
> 트랜잭션을 읽기 전용으로 설정

그러니까 트랜잭션에서 중요한건 이 항목들인데

격리수준, 전파, 롤백, 타임아웃, 읽기전용 이런 항목들이 중요한거지

## 트랜잭션의 특징은 무엇이있어?

원자성(Atomicity)

- 트랜잭션이 DB에 모두 반영되거나, 혹은 전혀 반영되지 않아야 된다.

일관성(Consistency)

- 트랜잭션의 작업 처리 결과는 항상 일관성 있어야 한다.

독립성(Isolation)

- 둘 이상의 트랜잭션이 동시에 병행 실행되고 있을 때, 어떤 트랜잭션도 다른 트랜잭션 연산에 끼어들 수 없다.

지속성(Durability)

- 트랜잭션이 성공적으로 완료되었으면, 결과는 영구적으로 반영되어야 한다.

이 개념들은 DB의 락과도 연관지어져

#DB LOCK?

락의 목적은

> 트랜잭션들이 동시에 작업할 때 데이터 정합성을 깨지 않도록 방지하는 것!

트랜잭션 격리 수준(Isolation Level)은 “트랜잭션끼리 서로 얼마나 간섭하지 못하게 할 것인가”를 결정하는 옵션이야.

DB는 이 격리 수준을 유지하기 위해 락(lock)과 MVCC를 서로 다른 방식으로 사용해

## MVCC는 뭔데?

MVCC는 Multi Version Concurrency Control, 즉
"다중 버전 동시성 제어"라고 부르는 DB의 동시성 처리 기법이야.

한 줄로 말하면

> 같은 데이터를 여러 버전으로 관리해서, 읽기 작업이 쓰기 작업을 방해하지 않도록 하는 기술인거지

트랜잭션 A가 어떤 행을 UPDATE하면서 X-Lock(쓰기 락)을 걸고 있는 상황에서
트랜잭션 B가 그 행을 SELECT하려고 하면 원래는 “락 때문에 기다려야” 맞아

근데 실무에서 읽기 때문에 쓰기를 막거나,
쓰기 때문에 읽기를 기다리게 하는 건 엄청난 병목을 만들어

그래서 DB는 생각하는거지:

“읽기(tran B)는 그냥 과거 버전을 읽어가면 되잖아?”

결국 현대 데이터베이스는 다음과 같은 구조로 이루어져있는거야

읽기 = MVCC 기반(버전 스냅샷을 읽음)
쓰기 = 락(X-Lock, Next-Key Lock 등)을 사용

이런 기반에서 락의 범위에 대한 이해가 필요해

## DB 락의 범위?

락은 단순히 “행을 잠근다” 수준이 아니라
DB가 어떤 범위를 잠그는지가 매우 중요해.

DB는 기본적으로 다음 세 가지 단위를 잠글 수 있어:

1. Row Lock (Record Lock)
2. Gap Lock
3. Next-Key Lock (Record + Gap)

그리고 이 범위는 조건절, 인덱스 유무, 격리 수준에 따라 달라져.

### 1) Record Lock (행 단위 락)

“정확히 어떤 행을 잠글지 DB가 알고 있을 때” 걸리는 락

예: PK 또는 Unique 인덱스로 특정 행을 UPDATE 할 때

UPDATE user SET age = 30 WHERE id = 1;

id가 PK이므로
→ DB는 바로 해당 row를 찾고
→ 딱 그 한 행만 잠금(Record X-Lock)

특징

- 가장 작은 범위의 락
- 충돌 적음
- 가장 좋은 성능

결론: PK/Unique 조건이면 Record Lock 하나만 걸린다.

### 2) Gap Lock (행과 행 사이의 범위 잠금)

존재하는 행 사이의 “공백”을 잠그는 락
주로 Phantom Read 방지용

예:
SELECT \* FROM user WHERE age BETWEEN 20 AND 30 FOR UPDATE;

이 경우:
나이 20~30 사이에 없는 값(예: 25)을 새로 INSERT하는 것을 막기 위해

20~30 구간 전체를 Gap Lock으로 잠근다

특징

1. 실제 데이터가 없는 구간에도 락이 걸림
2. “이 구간에 새로운 행 들어오는 것”을 막음
3. Phantom Read 방지의 핵심

## 3) Next-Key Lock (Record Lock + Gap Lock)

특정 행 + 앞뒤 구간까지 한 번에 잠그는 락
MySQL REPEATABLE READ 격리 수준의 기본 락

예:
SELECT \* FROM user WHERE age = 25 FOR UPDATE;

DB는 이렇게 잠근다:
age = 25 행 (Record Lock)
age 25 이전/이후 인덱스 범위(Gap Lock)
→ 이 전체 묶음을 Next-Key Lock이라고 함.

✔ 왜 이렇게까지 잠글까?

INSERT로 새로운 행이 생기면
"조건의 조회 결과가 바뀌기" 때문에
즉, Phantom Read 방지 때문.

## 락은 그럼 개발자가 컨트롤하는거야?

락은 “개발자가 명령하지 않아도 DB가 자동으로 잠궈”

락(lock)은 DB가 트랜잭션을 안전하게 처리하기 위해 스스로 알아서 걸어버리는 것이고,
개발자가 직접 “락 걸어라”라고 명령하는 경우는 거의 없어.

개발자가 할 수 있는 건:

- 어떤 쿼리를 실행하느냐 (UPDATE/DELETE/SELECT FOR UPDATE 등)
- 어떤 인덱스를 만들었느냐
- 어떤 격리 수준을 쓰느냐

  이걸로 락이 어떤 범위/종류로 걸릴지 간접적으로 결정하는 것뿐이야.

그러니까 락은 DB가 ACID를 지키려고 “자동으로 판단해서 잠구는 것”이지
개발자가 수동으로 제어하는 기능이 아님.

## DB가 어떤 순간에 락을 자동으로 걸까?

DB는 쿼리 종류 + 격리 수준 + 인덱스 구조를 보고 자동으로 락을 건다.

다음 경우에 DB가 알아서 락을 건다:

1. UPDATE / DELETE → 무조건 X-Lock(쓰기 락)
   UPDATE user SET age = 30 WHERE id = 1;

→ DB가 id=1 행에 Exclusive Lock 자동으로 걸어버림

개발자는 “락 걸어”라고 말한 적 없음
DB가 ACID 지키려고 자동 수행.

2. SELECT … FOR UPDATE → X-Lock
   SELECT \* FROM user WHERE age BETWEEN 20 AND 30 FOR UPDATE;

→ 지정된 인덱스 범위 전체 Next-Key Lock 자동 생성

3. SELECT (일반 조회) → 락 거의 안 걸림

일반 SELECT는 MVCC 때문에 락을 안 걸거나 거의 안 걸어.

4. 격리 수준이 REPEATABLE READ면 범위 잠금(Gap/Next-Key) 자동
   SELECT \* FROM user WHERE age = 25 FOR UPDATE;

→ Next-Key Lock 자동 생성
DB가 “너 이 조회의 일관성 유지하려고 하는구나?” 하고 판단해서.

5. SERIALIZABLE 모드 → SELECT에도 Shared Lock 자동
   SELECT \* FROM user WHERE id = 1;

→ 그냥 조회해도 Shared Lock 걸림

## 진짜 중요한 핵심

DB는 락을 언제/어떻게 걸지 전적으로 "스스로 판단해서" 운영한다.

개발자의 역할은 “조건을 주고”, “인덱스를 설계하고”, “격리 수준을 선택하는 것”.
락은 그 선택의 “자동 결과물”이야.

## 그럼 왜 개발자가 락을 직접 걸지 않을까?

이유는 단순해:

1.  DB 내부 구조(InnoDB MVCC + B+Tree 인덱스)는 매우 복잡하고
2.  어떤 범위를 잠궈야 ACID가 깨지지 않을지 사람이 계산하기 힘들기 때문

예를 들어,

인덱스가 여러 개일 때

범위 조건이 있을 때

JOIN이 있을 때
어떤 레코드/갭을 잠궈야 안전한지 사람은 알기 어려워.
그래서 DB가 직접 계산해서 안전한 범위를 잠궈

## 락은 개발자가 컨트롤하는건줄 알았어

보통 “개발자가 컨트롤하는 거다”라고 오해하는 경우가 훨씬 많아.
하지만 사실은 반대야

### 왜 개발자가 락을 직접 제어하지 못할까?

이유는 간단해.

→ 락은 ACID를 지키기 위해 DB가 ‘자동’으로 계산해서 걸어야 하는 영역이라서 그래.

인덱스 구조(B+Tree)
Undo Log 버전
현재 트랜잭션 상태
다른 트랜잭션이 잡고 있는 락 범위
격리 수준(REPEATABLE READ, READ COMMITTED 등)

이런 복잡한 내부 정보를
개발자가 직접 계산해서 어느 범위를 잠금해야 하는지 정한다?
현실적으로 불가능해.

그래서 DB가 직접 “지금 상황에서 이 범위를 잠가야 ACID가 깨지지 않음” 하고 알아서 잠궈

그럼 개발자는 락에 대해 뭘 할 수 있을까?

직접 락을 걸 수는 없고,
다음 세 가지만으로 간접적으로 영향을 줄 수 있어

### 1. 어떤 쿼리를 쓰느냐

UPDATE

DELETE

SELECT FOR UPDATE

SELECT LOCK IN SHARE MODE

이런 명령에 따라 DB가
“아, 이건 쓰기 충돌 방지해야겠구나”
하고 알아서 락 종류와 범위를 계산함.

### 2. 어떤 인덱스를 만들었느냐

예)

UPDATE user SET age = 30 WHERE name = 'Kim';

name에 인덱스가 없으면
→ DB는 full scan
→ 스캔하는 모든 행에 Record Lock 걸림
→ (= 사실상 테이블 전체 락)

반면

UPDATE user SET age = 30 WHERE id = 1;

id PK이면
→ 단 한 행(row)에만 Record Lock
→ 빠르고 충돌 없음

즉,

“인덱스를 어떻게 설계했는지”가 락 범위에 결정적인 영향을 미치는거지

### 3. 격리 수준을 어떻게 설정했느냐

READ COMMITTED → Gap Lock 없음 → 락 적다
REPEATABLE READ → Gap/Next-Key Lock 있음 → 락 범위 크다
SERIALIZABLE → SELECT에도 락 → 범위 최대로 큼

DB는 이 설정을 보고
“이 트랜잭션은 이정도의 락이 필요하겠군”
하고 자동으로 잠금 전략을 설정해

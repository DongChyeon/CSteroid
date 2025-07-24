---
date: 2025-07-21
user: DongChyeon
topic: 트랜잭션과 ACID 원칙
---

## 1. 트랜잭션(Transaction)이란?

트랜잭션이란 데이터베이스의 상태를 변환시키는 작업의 논리적인 단위를 의미한다. 즉, 여러 쿼리들을 하나의 작업 단위로 묶어, 모두 성공하거나 모두 실패하게 만들어 데이터의 무결성을 보장한다.

데이터베이스 트랜잭션은 ACID 원칙을 따르며, 이는 트랜잭션의 신뢰성과 일관성을 보장하는 데 중요한 역할을 한다.

## 2. ACID 원칙이란?

| 원칙 | 의미 | 설명 | 예시 |
|------|------|------|------|
| A - Atomicity (원자성) | 트랜잭션은 모두 수행되거나, 전혀 수행되지 않아야 함 | 중간에 실패하면 전체 작업을 롤백하여 이전 상태로 되돌림 | A → B 송금 시, A 계좌에서 출금되었지만 B에 입금 실패하면 전체 취소되어야 함 |
| C - Consistency (일관성) | 트랜잭션 전후로 DB의 제약조건과 규칙이 항상 만족되어야 함 | 무결성 제약조건(기본키, 외래키 등)이 항상 유지되어야 함 | A 계좌 잔고가 음수가 되지 않도록 제약조건 설정, 이를 위반하면 트랜잭션 거부 |
| I - Isolation (격리성) | 동시에 실행되는 트랜잭션이 서로 영향을 주지 않아야 함 | 트랜잭션 간 간섭 방지, 격리 수준 설정으로 동시성 조절 | 두 명이 동시에 같은 재고 상품 구매 시, 초과 판매되지 않도록 처리 필요 |
| D - Durability (지속성) | 트랜잭션이 커밋되면 결과는 영구 저장되어야 함 | 장애 발생 시에도 변경 내용은 손실되지 않음 | 전원 장애 후에도 B 계좌에 입금된 데이터는 유지되어야 함 (로그/저널 활용) |

## 3. Isolation Level (격리 수준)

| 수준 | 설명 | 발생 가능한 문제 | 성능 |
|------|------|------------------|------|
| READ UNCOMMITTED | 다른 트랜잭션의 미완료 데이터를 읽을 수 있음 | Dirty Read | 가장 빠름 |
| READ COMMITTED | 커밋된 데이터만 읽음 | Non-Repeatable Read | 빠름 |
| REPEATABLE READ | 트랜잭션 내 동일 쿼리는 항상 동일한 결과 반환 | Phantom Read | 중간 |
| SERIALIZABLE | 트랜잭션을 순차적으로 실행한 것처럼 처리 | 없음 (가장 엄격) | 가장 느림 |

> 💡 참고: 대부분의 RDBMS는 기본적으로 `READ COMMITTED` 또는 `REPEATABLE READ`를 사용

1. Dirty Read
    - 발생 조건: 다른 트랜잭션이 아직 커밋되지 않은 데이터를 읽는 경우
    - 설명: 트랜잭션 간 임시 변경 데이터를 읽음
    - 시나리오: 트랜잭션 A가 금액을 200으로 바꾸고 아직 커밋을 하지 않았지만, 트랜잭션 B가 해당 값을 읽음
2. Non-Repeatable Read
    - 발생 조건: 같은 쿼리를 두 번 실행했는데, 다른 결과가 나옴
    - 설명: 중간에 다른 트랜잭션이 수정/커밋해서 값이 바뀜
    - 시나리오: 트랜잭션 A가 100원 조회 후, 트랜잭션 B가 200원으로 업데이트하고 커밋함. 다시 트랜잭션 A가 조회하면 200원이 나옴.
3. Phantom Read
    - 발생 조건: 같은 조건의 SELECT인데 레코드 개수가 달라짐
    - 설명: 중간에 다른 트랜잭션이 insert/delete해서 행이 추가됨
    - 시나리오: 트랜잭션 A가 특정 조건으로 10개의 행을 조회했는데, 트랜잭션 B가 새로운 행을 추가해서 다음 조회 시 11개가 나옴.

## 4. 트랜잭션 실패 시 시나리오

1. 시도 작업:
    - 상품 재고 차감
    - 결제 테이블에 결제 내역 저장
    - 포인트 차감
    - 주문 완료 처리

2. 중간 실패: 포인트 차감 중 DB 오류 발생

3. Rollback 수행:
    - 재고 차감 → 복구
    - 결제 내역 → 삭제
    - 포인트 → 원래대로 복구
    - 주문 상태 → 취소

4. 사용자 입장:
    - "결제 실패" 메시지 노출
    - 모든 상태는 트랜잭션 이전으로 되돌아감 (원자성 보장)

## 5. 트랜잭션 사용 예시 (코드)

```kotlin
internal object DatabaseMigrations {

    val MIGRATION_1_2 = object : Migration(1, 2) {
        override fun migrate(database: SupportSQLiteDatabase) {
            database.beginTransaction()
            try {
                // 1. 새 스키마로 임시 테이블 생성 (isAm 컬럼 제외, missionType, missionCount 추가 및 기본값 변경)
                database.execSQL(
                    """
                CREATE TABLE ${DATABASE_NAME}_new (
                    id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
                    hour INTEGER NOT NULL,
                    minute INTEGER NOT NULL,
                    second INTEGER NOT NULL,
                    repeatDays INTEGER NOT NULL,
                    isHolidayAlarmOff INTEGER NOT NULL,
                    isSnoozeEnabled INTEGER NOT NULL,
                    snoozeInterval INTEGER NOT NULL,
                    snoozeCount INTEGER NOT NULL,
                    isVibrationEnabled INTEGER NOT NULL,
                    isSoundEnabled INTEGER NOT NULL,
                    soundUri TEXT NOT NULL,
                    soundVolume INTEGER NOT NULL,
                    isAlarmActive INTEGER NOT NULL,
                    missionType INTEGER NOT NULL DEFAULT 1,  -- 타입 INTEGER, 기본값 1
                    missionCount INTEGER NOT NULL DEFAULT 10 -- 타입 INTEGER, 기본값 10
                )
                    """.trimIndent(),
                )

                // 2. 기존 테이블에서 새 임시 테이블로 데이터 복사 (isAm 컬럼은 복사하지 않음)
                database.execSQL(
                    """
                INSERT INTO ${DATABASE_NAME}_new (
                    id, hour, minute, second, repeatDays, isHolidayAlarmOff,
                    isSnoozeEnabled, snoozeInterval, snoozeCount, isVibrationEnabled,
                    isSoundEnabled, soundUri, soundVolume, isAlarmActive
                    -- missionType, missionCount는 CREATE TABLE에서 정의된 기본값으로 자동 채워짐
                )
                SELECT
                    id,
                    -- hour를 24시간 형식으로 변환합니다.
                    -- 예시: isAm 컬럼이 0 (PM)이고 hour가 12가 아니면 hour + 12
                    -- 예시: isAm 컬럼이 1 (AM)이고 hour가 12 (자정)이면 0으로 변환
                    -- 실제 isAm 컬럼의 의미와 값에 따라 아래 로직을 조정해야 합니다.
                    CASE
                        WHEN isAm = 0 AND hour != 12 THEN hour + 12 -- 오후 1시 ~ 11시 -> 13 ~ 23시
                        WHEN isAm = 1 AND hour = 12 THEN 0          -- 오전 12시 (자정) -> 0시
                        ELSE hour                                   -- 그 외 (오전 1시 ~ 11시, 오후 12시(정오))
                    END AS hour_24,
                    minute,
                    second,
                    repeatDays,
                    isHolidayAlarmOff,
                    isSnoozeEnabled,
                    snoozeInterval,
                    snoozeCount,
                    isVibrationEnabled,
                    isSoundEnabled,
                    soundUri,
                    soundVolume,
                    isAlarmActive
                FROM $DATABASE_NAME
                    """.trimIndent(),
                )

                // 3. 기존 테이블 삭제
                database.execSQL("DROP TABLE $DATABASE_NAME")

                // 4. 임시 테이블의 이름을 기존 테이블 이름으로 변경
                database.execSQL("ALTER TABLE ${DATABASE_NAME}_new RENAME TO $DATABASE_NAME")

                // 5. 커밋
                database.setTransactionSuccessful()
            } finally {
                database.endTransaction()
            }
        }
    }
}
```

안드로이드 앱 개발 당시 버전 업데이트에 따라 기존 데이터베이스 스키마를 변경해야 했던 상황이다.
이때, 트랜잭션을 사용하여 데이터베이스 스키마 변경 작업을 안전하게 수행했다.
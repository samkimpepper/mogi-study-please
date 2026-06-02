---
tags:
  - backend
  - mssql
  - transaction
  - database
---

# MSSQL 트랜잭션 로그 FULL과 ACTIVE_TRANSACTION

## 한 줄 정리

`ACTIVE_TRANSACTION` 때문에 트랜잭션 로그가 찼다는 것은, 대량 변경 로그가 아직 끝나지 않은 트랜잭션에 묶여 있어서 SQL Server가 로그를 재사용하지 못하는 상태라는 뜻이다.

## 발생한 에러

재고조정 장부재고 프로시저 호출 중 다음 에러가 발생했다.

```sql
The transaction log for database 'STM' is full due to 'ACTIVE_TRANSACTION'
```

문제가 된 구간은 과거 장부재고 데이터를 삭제하는 쿼리였다.

```sql
DELETE FROM STM.dbo.ST_INVSTCK_MGR
WHERE CO_CD = @VC_CO_CD
  AND STCK_YMD < @VC_STCK_DT
  AND ORG_CD = @VC_ORG_CD;
```

장부재고 가져오기 기능을 오랫동안 사용하지 않다가 다시 사용하면, 이전 일자 데이터가 많이 쌓여 있다. 이때 한 번에 삭제해야 하는 데이터가 한 점포당 `9000개 x N` 수준까지 커질 수 있다.

> [!summary]
> 문제의 직접 계기는 대량 `DELETE`였고, 더 중요한 원인은 이 삭제가 긴 트랜잭션 안에서 실행되어 로그가 중간에 재사용되지 못했다는 점이다.

## 트랜잭션 로그가 하는 일

MSSQL은 데이터 변경 작업을 할 때 변경 내역을 트랜잭션 로그에 기록한다.

```text
DELETE 실행
-> 어떤 row가 삭제되는지 로그에 기록
-> 장애가 나면 롤백하거나 복구할 수 있어야 함
-> 커밋 전까지 필요한 로그를 함부로 버릴 수 없음
```

`DELETE`는 단순히 데이터를 지우는 작업처럼 보이지만, DB 입장에서는 "나중에 롤백할 수도 있는 변경"이다. 그래서 삭제 대상이 많을수록 트랜잭션 로그도 많이 사용한다.

> [!note]
> 대량 `DELETE`는 테이블에서 row를 없애는 동시에, 그 삭제 변경 내역을 로그 파일에 계속 쌓는 작업이다.

## 로그가 필요한 이유

트랜잭션 로그는 DB 변경 작업의 복구용 일지에 가깝다.

```text
INSERT 했다
UPDATE 했다
DELETE 했다
트랜잭션을 시작했다
커밋했다
롤백해야 할 수도 있다
```

SQL Server는 이런 내용을 순서대로 기록한다. 이유는 장애가 나도 DB의 일관성을 지켜야 하기 때문이다.

```text
트랜잭션 실행 중 서버 장애 발생
-> 어디까지 반영됐는지 확인해야 함
-> 커밋된 변경은 살림
-> 커밋되지 않은 변경은 되돌림
```

그래서 SQL Server는 실제 데이터 페이지를 바꾸기 전에 로그를 먼저 남긴다. 이 흐름을 write-ahead logging으로 이해할 수 있다.

```text
1. 로그에 "이 row를 삭제할 예정"이라고 기록
2. 실제 데이터 페이지 변경
3. 커밋 로그 기록
```

> [!summary]
> 트랜잭션 로그는 성능을 위해 대충 남기는 기록이 아니라, 장애 복구와 롤백을 위해 반드시 필요한 변경 이력이다.

## 로그가 찬다는 말의 의미

트랜잭션 로그 파일은 실제 디스크에 있는 물리 파일이다.

```text
STM_log.ldf
크기: 1MB
```

DB 변경 작업이 많아지면 이 파일 안에 로그 기록이 쌓인다.

```text
[사용 중][사용 중][사용 중][빈 공간]
```

계속 변경 작업이 발생하면 다음처럼 새 로그를 쓸 공간이 부족해질 수 있다.

```text
[사용 중][사용 중][사용 중][사용 중]
```

이 상태에서 더 이상 새 로그를 쓸 수 없으면 다음 에러가 발생한다.

```sql
The transaction log for database 'STM' is full
```

여기서 중요한 것은 로그 파일 크기와 로그 사용량을 구분하는 것이다.

```text
로그 파일 크기
-> 물리적인 .ldf 파일 크기

로그 사용량
-> 그 파일 안에서 현재 사용 중인 비율
```

예를 들면 다음과 같은 상태가 가능하다.

```text
로그 파일 크기: 1024MB
사용량: 900MB
사용률: 약 88%
```

파일 자동 증가가 가능하면 로그 파일은 더 커질 수 있다.

```text
1024MB -> 1280MB -> 1536MB
```

하지만 자동 증가가 막혀 있거나, 디스크가 부족하거나, 증가 단위가 너무 작거나, 로그 재사용이 막혀 있으면 로그 full 에러가 발생할 수 있다.

> [!warning]
> "로그가 찼다"는 말은 물리 파일 크기만의 문제가 아니다. 파일 안에 재사용 가능한 공간이 있는지, 새 로그를 쓸 수 있는지도 함께 봐야 한다.

## Truncate와 Shrink의 차이

트랜잭션 로그에서 `truncate`는 물리 파일 크기를 줄인다는 뜻이 아니다.

`truncate`는 더 이상 필요 없는 로그 영역을 재사용 가능하게 표시하는 것이다.

```text
truncate 전

[필요한 로그][필요한 로그][이제 필요 없는 로그][이제 필요 없는 로그]
```

```text
truncate 후

[필요한 로그][필요한 로그][재사용 가능][재사용 가능]
```

이때 물리 파일 크기는 그대로일 수 있다.

```text
STM_log.ldf = 1024MB
```

반면 `shrink`는 물리적인 로그 파일 크기를 줄이는 작업이다.

```text
STM_log.ldf 1024MB
-> shrink
STM_log.ldf 256MB
```

실무에서는 shrink를 함부로 사용하면 안 된다. 로그 파일을 줄여놓고 다시 업무 중에 로그가 필요해지면 SQL Server가 파일을 다시 키워야 하기 때문이다.

```text
shrink로 256MB까지 줄임
-> 대량 작업 발생
-> 다시 1024MB까지 autogrowth
-> 증가 작업 반복
-> 성능 저하
```

> [!summary]
> `truncate`는 로그 내부 공간을 재사용 가능하게 만드는 정상적인 로그 관리에 가깝고, `shrink`는 물리 파일 크기를 줄이는 특수 작업에 가깝다.

## ACTIVE_TRANSACTION의 의미

`ACTIVE_TRANSACTION`은 아직 완료되지 않은 트랜잭션 때문에 로그를 재사용할 수 없다는 뜻이다.

```text
BEGIN TRANSACTION
DELETE 100건
DELETE 100건
DELETE 100건
...
아직 COMMIT 안 됨
```

이 상태에서는 앞에서 발생한 로그도 아직 필요하다. 트랜잭션이 실패하면 전체를 되돌려야 하기 때문이다.

```text
커밋 전
-> 롤백 가능성이 있음
-> 삭제 로그가 필요함
-> SQL Server가 로그를 재사용하지 못함

커밋 후
-> 트랜잭션이 끝남
-> 조건이 맞으면 로그 재사용 가능
```

> [!warning]
> 로그 파일이 찼다는 말은 "로그가 아예 정리되지 않는다"는 뜻이 아니라, 현재 필요한 로그가 너무 많아서 재사용 가능한 공간이 부족해졌다는 뜻에 가깝다.

## ACTIVE_TRANSACTION이 Truncate를 막는 이유

아직 끝나지 않은 트랜잭션이 있으면 SQL Server는 그 트랜잭션의 로그를 버릴 수 없다.

```text
BEGIN TRANSACTION

DELETE 100건
DELETE 100건
DELETE 100건
...

아직 COMMIT 안 됨
```

이 상태에서는 앞에서 삭제한 로그가 여전히 필요하다.

```text
아직 커밋 안 됨
-> 롤백할 수도 있음
-> 롤백하려면 삭제 로그가 필요함
-> 로그를 재사용 가능 처리하면 안 됨
```

SQL Server 입장에서는 다음과 같다.

```text
이 트랜잭션이 아직 살아 있음
-> 이 트랜잭션 시작 이후의 로그는 복구나 롤백에 필요할 수 있음
-> active log로 유지
-> truncate 불가
```

> [!note]
> `ACTIVE_TRANSACTION`은 "로그 백업을 안 해서"가 아니라, 현재 열려 있는 트랜잭션 때문에 로그 재사용이 막힌 상태를 가리킨다.

## Recovery Model과 로그 재사용

SQL Server의 로그 관리는 데이터베이스의 recovery model과도 연결된다.

대표적으로 다음 두 가지를 먼저 보면 된다.

```text
SIMPLE
FULL
```

`SIMPLE` 복구 모델은 체크포인트 이후 더 이상 필요 없는 로그를 비교적 자동으로 재사용 가능하게 만든다. 대신 특정 시점 복구에는 제한이 있다.

```text
SIMPLE
-> 로그 백업 없음
-> 로그 관리가 비교적 단순
-> point-in-time restore 불가
```

`FULL` 복구 모델은 로그 백업이 필요하다. 로그 백업을 해야 로그가 정상적으로 재사용될 수 있고, 특정 시점 복구가 가능하다.

```text
FULL
-> 로그 백업 필요
-> 특정 시점 복구 가능
-> 로그 백업이 없으면 로그가 계속 커질 수 있음
```

하지만 `SIMPLE`이라고 해서 로그 full 문제가 절대 안 생기는 것은 아니다.

```text
SIMPLE 복구 모델
-> 체크포인트 후 로그 재사용 가능성이 높음
-> 하지만 긴 트랜잭션이 살아 있으면 ACTIVE_TRANSACTION으로 재사용 불가
```

즉 recovery model과 별개로, 오래 열린 트랜잭션은 로그 재사용을 막을 수 있다.

> [!summary]
> `FULL`에서는 로그 백업이 중요하고, `SIMPLE`에서는 체크포인트 이후 로그 재사용이 비교적 자동이다. 하지만 두 경우 모두 긴 active transaction이 있으면 로그를 재사용할 수 없다.

## DELETE TOP으로 쪼갠 테스트

삭제 위치를 확인하기 위해 로그를 출력하면서 테스트했고, 삭제문을 주석 처리하면 에러가 발생하지 않았다.

```sql
RAISERROR('STEP 1 START', 0, 1) WITH NOWAIT;

WHILE 1 = 1
BEGIN
    DELETE TOP (100)
    FROM STM.dbo.ST_INVSTCK_MGR
    WHERE CO_CD = @VC_CO_CD
      AND STCK_YMD < @VC_STCK_DT
      AND ORG_CD = @VC_ORG_CD;

    IF @@ROWCOUNT = 0
        BREAK;
END
```

SSMS에서 직접 호출했을 때는 `DELETE TOP (100)` 반복으로 에러가 나지 않았다. 하지만 Java/iBatis를 통해 호출하면 같은 에러가 발생했다.

## SSMS와 Java 실행 결과가 달랐던 이유

SSMS에서 명시적인 외부 트랜잭션 없이 프로시저를 실행하면, 각 SQL 문장이 비교적 짧은 트랜잭션으로 끝날 수 있다.

```text
SSMS 실행

DELETE TOP (100)
-> 커밋 가능

DELETE TOP (100)
-> 커밋 가능

DELETE TOP (100)
-> 커밋 가능
```

반면 Java/iBatis 호출에서는 DB 커넥션이 외부 트랜잭션 상태로 유지되었다. 이 경우 프로시저 내부에서 `DELETE TOP (100)`으로 쪼개도 전체 호출이 하나의 큰 트랜잭션에 묶인다.

```text
Java 외부 트랜잭션 시작

프로시저 호출
  DELETE TOP (100)
  DELETE TOP (100)
  DELETE TOP (100)
  ...

Java 외부 트랜잭션 커밋
```

그래서 SQL 문장은 100건씩 나뉘었지만, 트랜잭션 로그 관점에서는 아직 큰 트랜잭션 하나가 끝나지 않은 상태였다.

> [!summary]
> `DELETE TOP (100)`은 실행 단위를 작게 만든 것이지, 항상 트랜잭션 단위를 작게 만드는 것은 아니다. 바깥 트랜잭션이 전체를 감싸고 있으면 로그는 계속 붙잡힌다.

백엔드 코드로 보면 다음과 비슷하다.

```java
@Transactional
public void run() {
    delete100();
    delete100();
    delete100();
}
```

각 메서드가 100건씩 지워도 바깥 `@Transactional`이 전체를 감싸면 커밋은 마지막에 한 번만 일어난다.

## 로그 파일 크기 문제

프로시저 실행 중 다음 명령어로 로그 사용량을 확인했다.

```sql
DBCC SQLPERF(LOGSPACE);
```

평소에는 약 40% 수준이던 로그 사용률이, 프로시저 실행 중에는 100% 이상으로 증가했다.

또한 테스트 환경의 `STM` 로그 파일 크기가 `1MB`로 매우 작았다. 반면 운영 DB의 `STM` 로그 파일은 약 `186GB`였다.

```text
테스트 STM 로그 파일
-> 1MB
-> 대량 DELETE를 감당하기 어려움

운영 STM 로그 파일
-> 약 186GB
-> 대량 변경을 버틸 여지가 큼
```

> [!note]
> 테스트 DB의 로그 파일이 1MB라면 운영과 조건이 완전히 다르다. 대량 삭제 테스트에서는 로그 파일의 초기 크기와 증가 단위도 중요한 환경 변수다.

## 1차 대책: 로그 파일 크기 조정

현재 상황에서 로그 파일 크기를 현실적인 수준으로 키우는 것은 필요한 조치다.

```sql
ALTER DATABASE [STM]
MODIFY FILE (
    NAME = N'STM_log',
    SIZE = 1024MB,
    FILEGROWTH = 256MB
);
```

이 설정은 최소 로그 파일 크기를 `1024MB`로 늘리고, 자동 증가 단위를 `256MB`로 조정한다.

```text
SIZE
-> 현재 로그 파일의 기본 크기

FILEGROWTH
-> 공간이 부족할 때 한 번에 증가하는 크기
```

> [!tip]
> 로그 파일을 키우는 것은 증상을 버티게 해주는 1차 대응이다. 근본적으로는 대량 삭제가 하나의 긴 트랜잭션에 묶이지 않도록 트랜잭션 경계를 같이 봐야 한다.

## 근본적으로 같이 봐야 할 부분

### Java 트랜잭션 경계

프로시저 호출이 반드시 외부 업무 트랜잭션과 원자적으로 묶여야 하는지 확인해야 한다.

```text
반드시 같이 성공/실패해야 한다
-> 하나의 트랜잭션으로 묶는 이유가 있음
-> 로그 파일 크기와 삭제량을 더 보수적으로 관리해야 함

삭제 작업은 별도 정리 작업이어도 된다
-> 외부 트랜잭션 밖에서 실행하는 방향 검토 가능
-> 배치 단위 커밋 설계 가능
```

### 진짜 배치 단위 커밋

프로시저 안에서 `DELETE TOP (100)`을 반복하는 것만으로는 부족할 수 있다. 바깥 트랜잭션이 전체를 감싸면 커밋이 마지막에 한 번만 일어나기 때문이다.

진짜로 로그 부담을 줄이려면 호출 단위 자체가 여러 번 나뉘어야 한다.

```text
delete_old_inventory_batch(1000) 호출
-> 커밋

delete_old_inventory_batch(1000) 호출
-> 커밋

delete_old_inventory_batch(1000) 호출
-> 커밋
```

### 삭제 조건 인덱스

삭제 조건은 다음 컬럼을 사용한다.

```sql
WHERE CO_CD = @VC_CO_CD
  AND STCK_YMD < @VC_STCK_DT
  AND ORG_CD = @VC_ORG_CD
```

이 조건을 잘 타는 인덱스가 없으면 삭제 대상 탐색이 느려지고, 락을 오래 잡으며, 전체 트랜잭션 시간이 길어진다.

검토 후보는 다음과 같은 형태다.

```sql
CREATE INDEX IX_ST_INVSTCK_MGR_DELETE_TARGET
ON STM.dbo.ST_INVSTCK_MGR (CO_CD, ORG_CD, STCK_YMD);
```

> [!warning]
> 인덱스는 무조건 추가하면 되는 것이 아니다. 기존 PK, 기존 인덱스, 조회 패턴, 테이블 크기, 삭제 빈도를 보고 판단해야 한다.

## 확인하면 좋은 쿼리

로그 재사용을 막는 원인은 다음 쿼리로 확인할 수 있다.

```sql
SELECT
    name,
    recovery_model_desc,
    log_reuse_wait_desc
FROM sys.databases
WHERE name = 'STM';
```

여기서 `log_reuse_wait_desc`가 `ACTIVE_TRANSACTION`이면 아직 끝나지 않은 트랜잭션 때문에 로그 재사용이 막힌 상태다.

가능한 원인은 `ACTIVE_TRANSACTION`만 있는 것은 아니다.

```text
ACTIVE_TRANSACTION
-> 긴 트랜잭션 때문에 로그 재사용 불가

LOG_BACKUP
-> FULL 복구 모델인데 로그 백업이 필요함

REPLICATION
-> 복제에서 아직 로그가 필요함

AVAILABILITY_REPLICA
-> Always On 보조 replica 쪽에서 아직 로그가 필요함
```

이번 케이스에서는 에러 메시지가 직접 `ACTIVE_TRANSACTION`을 가리켰기 때문에, 핵심은 로그 백업보다 긴 트랜잭션이었다.

로그 사용량은 다음 DMV로도 확인할 수 있다.

```sql
SELECT
    database_id,
    total_log_size_in_bytes / 1024 / 1024 AS total_log_mb,
    used_log_space_in_bytes / 1024 / 1024 AS used_log_mb,
    used_log_space_in_percent
FROM sys.dm_db_log_space_usage;
```

## 실무에서 가져갈 점

이번 문제는 "DELETE가 많아서 에러가 났다"보다 더 구체적으로 봐야 한다.

```text
대량 DELETE
-> 트랜잭션 로그 대량 발생
-> Java/iBatis 외부 트랜잭션이 전체 프로시저를 감쌈
-> DELETE TOP으로 쪼개도 커밋은 마지막에 한 번
-> ACTIVE_TRANSACTION 때문에 로그 재사용 불가
-> 테스트 DB 로그 파일 1MB라 더 빨리 한계 도달
```

따라서 대책도 두 층으로 나눠야 한다.

```text
환경 보정
-> 로그 파일 크기와 증가 단위를 현실적으로 조정

설계 보정
-> 트랜잭션 경계 확인
-> 배치 단위 커밋 가능 여부 검토
-> 삭제 조건 인덱스 확인
```

> [!summary]
> 로그 파일 크기를 키우는 것은 맞는 1차 대응이다. 하지만 같은 구조로 삭제량이 더 커지면 다시 문제가 날 수 있으므로, Java 트랜잭션 경계와 삭제 배치 설계를 같이 확인해야 한다.

## 복습 질문

- [ ] MSSQL 트랜잭션 로그는 데이터 변경 작업에서 어떤 역할을 하는가?
- [ ] `ACTIVE_TRANSACTION` 때문에 로그가 찼다는 말은 어떤 상태를 의미하는가?
- [ ] `DELETE TOP (100)`으로 쪼갰는데도 Java 호출에서는 로그가 계속 찰 수 있는 이유는 무엇인가?
- [ ] SSMS 직접 실행과 Java/iBatis 호출의 트랜잭션 경계는 어떻게 달라질 수 있는가?
- [ ] 테스트 DB의 로그 파일 크기 1MB가 왜 문제를 더 빨리 드러나게 만들었는가?
- [ ] 로그 파일 크기 조정과 트랜잭션 경계 조정은 각각 어떤 문제를 해결하는가?
- [ ] 트랜잭션 로그의 `truncate`와 `shrink`는 무엇이 다른가?
- [ ] `SIMPLE` 복구 모델에서도 긴 트랜잭션 때문에 로그 full이 발생할 수 있는 이유는 무엇인가?
- [ ] `FULL` 복구 모델에서 로그 백업이 중요한 이유는 무엇인가?

## 한 줄 회고

- 헷갈렸던 점: `DELETE TOP`으로 SQL 문장을 작게 나누면 트랜잭션도 자동으로 작아진다고 생각할 수 있지만, Java 외부 트랜잭션이 전체 프로시저를 감싸고 있으면 로그 관점에서는 여전히 하나의 큰 트랜잭션이라는 점. 그리고 로그 `truncate`는 파일 크기를 줄이는 것이 아니라 재사용 가능 영역을 만드는 것이고, `shrink`가 물리 파일 크기를 줄이는 작업이라는 점.

---
title : 2주차 - SQL for Data Engineers
date : 2023-07-26 13:00 +09:00
categories : [Data, Data Engineering]
tags : [프로그래머스, 실리콘밸리에서 날라온 데이터 엔지니어링 스타터 키트 with python, DE]

---


[1. 1주차 과제](#1-1주차-과제)
<br>
[2. SQL의 장단점](#2-sql의-장단점)
<br>
[3. 기억 해야 할 점](#3-기억-해야-할-점)
<br>
[4. SQL - DDL과 DML](#4-sql---ddl과-dml)
<br>
[5. 기본 SQL](#5-기본-sql)
<br>
[6. SQL JOIN](#6-sql-join)
<br>
[7. SQL 고급 문법](#7-sql-고급-문법)
<br>

# 1. 1주차 과제 

## 1. 내가 작성한 코드

```python
%%sql

SELECT TO_CHAR(st.ts,'yyyy-mm') date, COUNT(usc.userID)
FROM raw_data.session_timestamp st
JOIN raw_data.user_session_channel usc ON st.sessionID = usc.sessionID
GROUP BY 1
ORDER BY 1
;
```

## 2. 강사님이 작성한 코드

```python
%%sql

SELECT TO_CHAR(st.ts,'yyyy-mm') as date, COUNT(DISTINCT usc.userID) as cnt
FROM raw_data.session_timestamp st
JOIN raw_data.user_session_channel usc ON st.sessionID = usc.sessionID
GROUP BY 1
ORDER BY 1 DESC
;
```

## 3. 메모

1. **TO_CHAR()**
- TO_CHAR (A.ts, ‘YYYY-MM’)
- LEFT(A.ts, 7)
- DATE_TRUNC(‘month’, A.ts)
- SUBSTRING(A.ts, 1, 7)

2. **DISTINCT**
- Unique한 사용자 수를 Count 해야함
- DISTINCT 사용 X → 월별 모든 사용자 COUNT
    - ex) 2019-11에 user_id 1,1,2,3,3이 있을 경우 COUNT 값이 5
- DISTINCT 사용o → 월별 사용자 COUNT
    - ex) 2019-11에 user_id 1,1,2,3,3이 있을 경우 COUNT 값이 3

3. COUNT

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/16f1225a-f6e4-4636-bdab-f07ee00b19a0)

# 2. SQL의 장단점

## 1. 장점

- SQL - Structured Query Language의 약자 / 1970년대 IBM에서 개발
- **구조화된 데이터** 처리하는데 **Best**

## 2. 단점

- 비구조화된 데이터를 처리하게이는 오버헤드가 크고, 적합X

# 3. 기억 해야 할 점

### 1. 현업에서 깨끗한 데이터는 존재X

- 믿을 수 있는 데이터 인지 확인 → 실제 레코드 몇개 확인

### 2. 데이터 일을 한다면 항상 데이터의 품질을 의심, 체크

- 중복된 레코드들 체크
- 최근 데이터의 존재 여부 체크(freshness)
- **Primary key uniqueness**가 지켜지는지 체크
- 값이 비어있는 컬럼이 있는지 체크
- 위의 체크는 코딩의 unit(단위) test 형태로 만들어 매번 쉽게 체크 할 수 있음

# 4. SQL - DDL과 DML

## 1. SQL 기본

### 1. 세미콜론으로 분리

- SQL문1; SQL2; SQL3;

### 2. SQL주석

- -- : 한줄 짜리 주석 ex) //
- /* --*/ : 여러 줄에 걸쳐 사용하는 주석

### 3. SQL 사용 규칙

- SQL 키워드를 대문자 / 소문자 사용할 규칙 정하기
- 테이블/필드이름 명명 규칙 정하기

## 2. SQL DDL

### 1. DDL

- DB 스키마를 정의하는 SQL 명령어

### 2. CREATE TABLE

- Primary key 속성을 지정할 수 있으나 무시됨
    - Big Data 데이터웨어하우스에서는 지켜지지 않음 (Redshift, Snowflake,
    BigQuery)


- CTAS
	- CREATE TABLE schema_name.table_name AS SELECT
	- vs. **CREATE TABLE and then INSERT**

```python
CREATE TABLE raw_data.user_session_channel (
userid int,
sessionid varchar(32) primary key,
channel varchar(32)
);
```

### 3. DROP TABLE

- DROP TABLE schema_name.table_name;
- 없는 테이블을 지우려고 하는 경우 에러를 냄
- DROP TABLE IF EXISTS table_name;
- vs. DELETE FROM
- DELETE FROM : 조건에 맞는 레코드들을 지움 (테이블 자체는 존재)

### 4. ALTER TABLE

- 새로운 컬럼 추가:
    - **ALTER TABLE** 테이블이름 **ADD COLUMN** 필드이름 필드타입;
- 기존 컬럼 이름 변경:
    - **ALTER TABLE** 테이블이름 **RENAME** 현재필드이름 to 새필드이름
- 기존 컬럼 제거:
    - **ALTER TABLE** 테이블이름 **DROP COLUMN** 필드이름;
- 테이블 이름 변경:
    - **ALTER TABLE** 현재테이블이름 **RENAME to** 새테이블이름;

## 3. SQL DML

### 1. DML

- 정의된 데이터베이스에 입력된 레코드를 조회하거나 수정하거나 삭제하는 등의 역할

### 2. 레코드 질의 언어 : SELECT

- SELECT FROM: 테이블에서 레코드와 필드를 읽어오는데 사용
- WHERE : 사용해서 레코드 선택 조건을 지정
- GROUP BY :  통해 정보를 그룹 레벨에서 뽑는데 사용하기도 함
    - DAU, WAU, MAU 계산은 GROUP BY를 필요로 함
- ORDER BY : 사용해서 레코드 순서를 결정하기도 함
- 보통 다수의 테이블의 조인해서 사용하기도 함

### 3. 레코드 수정 언어:

- INSERT INTO: 테이블에 레코드를 추가하는데 사용 (COPY)
- UPDATE FROM: 테이블 레코드의 필드 값 수정
- DELETE FROM: 테이블에서 레코드를 삭제 vs. TRUNCATE

# 5. 기본 SQL

## 1. SELECT

```python
SELECT 필드이름1, 필드이름2, …

FROM 테이블이름 [JOIN …]

WHERE 선택조건 GROUP BY 필드이름1, 필드이름2, ...

ORDER BY 필드이름 [ASC|DESC] -- 필드 이름 대신에 숫자 사용 가능

LIMIT N; 출력 개수 제한
```

## 2. WHERE

### 1. IN

```python
WHERE channel in (‘Google’, ‘Youtube’)

WHERE channel = ‘Google’ OR channel = ‘Youtube
```

### 2. **LIKE and ILIKE**

- LIKE: 정해진 문자 탐색

- ILIKE: 대문자 소문자 상관없이 탐색

- WHERE channel LIKE ‘G%’ -> ‘G*’

-  WHERE channel LIKE ‘%o%’ -> ‘*o*’

-  NOT LIKE or NOT ILIKE

### 3. BETWEEN

- 날짜 사이

## 3. **STRING 함수**

- LEFT(str, N) 왼쪽부터

- REPLACE(str, exp1, exp2) 문자 교체

- UPPER(str)

- LOWER(str)

- LEN(str)

- LPAD, RPAD 길이가 기준보다 작을때 특정문자로 채우기

- SUBSTRING 위치 상관없이 대체

## 4. **INSERT INTO vs. COPY(Bulk update)**

- 속도 COPY가 INSERT보다 빠르다

## 5. **ORDER BY**

- ORDER BY 1 ASC - ASC가 기
- null값의 기준은 DB보다 달라서 기억하기보다 코드로 해결하는 것이 편하다
- ORDER BY 1 DESC; -- NULL값이 가장 앞에 옴
- ORDER BY 1 DESC NULLS LAST; -- NULL값이 맨뒤로 이동

## 6. **Type Cast**

- category :: int or cast(category as int)
- 일시적으로 타입 지정

## 7. **NULL**

- 값이 존재하지 않음을 의미
- 0이나 비어있는 string과는 다르다
- =이 아니라 IS를 사용해서 연산해야한다

# 6. SQL JOIN
![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/14bd6c75-1040-442c-858b-6addeac12707)

## 1. JOIN시 고려해야할 점

- **스타 스키마**(데이터베이스에서 데이터를 정리하는 데 사용하는 다차원적 데이터 모델)에서는 항상 필요
- 먼저 중복 레코드가 없고 **Primary Key의 uniqueness**가 보장됨을 체크
- 조인하는 테이블들간의 관계를 명확하게 정의
    - ex) one to one, one to many, many to many
- 어느 테이블을 베이스로 잡을지 (From에 사용할지) 결정해야함

### 2. 기본 문법

```python
SELECT A.*, B.*
FROM raw_data.table1 A
____ JOIN raw_data.table2 B ON A.key1 = B.key1 and A.key2 = B.key2;
```

### 3. INNER JOIN

- join default
- 양쪽 테이블에서 매치 되는 레코드들만 return
- 양쪽 테이블의 필드가 모두 채워진 상태로 return

### 4. LEFT JOIN

- 왼쪽 테이블(Base)의 모든 레코드들을 return
- 오른쪽 테이블의 필드는 왼쪽 레코드와 매칭되는 경우에만 채워진 상태로
return

### 5. FULL JOIN

- 왼쪽 테이블과 오른쪽 테이블의 모든 레코드들을 return
- 매칭되는 경우에만 양쪽 테이블들의 모든 필드들이 채워진 상태로 return

### 6. CROSS JOIN

- 왼쪽 테이블과 오른쪽 테이블의 모든 레코드들의 조합을 리턴함

### 7. SELF JOIN

- 동일한 테이블을 alias를 달리해서 자기 자신과 조인함

# 7. SQL 고급 문법

## 1. UNION, EXCEPT

### 1. UNION(합집합)

- 여러개의 테이블들이나 SELECT 결과를 하나의 결과로 합쳐줌
- UNION vs. UNION ALL
    - UNION은 중복을 제거

### 2. EXCEPT (MINUS)

- 하나의 SELECT 결과에서 다른 SELECT 결과를 빼주는 것이 가능

### 3. INTERSECT (교집합)

- 여러 개의 SELECT문에서 같은 레코드들만 찾아줌

## 2. COALESCE, NULLIF

### 1. COALESCE(Expression1, Expression2, …):

- 첫번째 Expression부터 값이 NULL이 아닌 것이 나오면 그 값을 리턴하고 모두 NULL이면
NULL을 리턴
- NULL값을 다른 값으로 바꾸고 싶을 때 사용

### 2. NULLIF(Expression1, Expression2):

- Expression1과 Expression2의 값이 같으면 NULL을 리턴

### 3. DELETE FROM vs. TRUNCATE

- DELETE FROM table_name (not DELETE * FROM)
    - 테이블에서 모든 레코드를 삭제
    - vs. DROP TABLE table_name
    - WHERE 사용해 특정 레코드만 삭제 가능:
        - DELETE FROM raw_data.user_session_channel WHERE channel = ‘Google’

- TRUNCATE table_name도 테이블에서 모든 레코드를 삭제
    - DELETE FROM은 속도가 느림
    - TRUNCATE이 전체 테이블의 내용 삭제시에는 여러모로 유리
    - 하지만 두가지 단점이 존재
        - TRUNCATE는 WHERE을 지원X
        - TRUNCATE는 Transaction을 지원X

## 3. WINDOW

### 1. Syntax:

- function(expression) OVER ( [ PARTITION BY expression] [ ORDER BY expression ] )

### 2. Useful functions:

- ROW_NUMBER, FIRST_VALUE, LAST_VALUE
- Math functions: AVG, SUM, COUNT, MAX, MIN, MEDIAN, NTH_VALUE

## 4. SUB Query (CTE)

- SELECT를 하기 전에 임시 테이블을 만들어서 사용하는 것이 가능
    - 임시 테이블을 별도의 CREATE TABLE로 생성하는 것이 아니라 SELECT 문의 앞단에서
    하나의 SQL 문으로 생성

- 문법은 아래와 같음 (channel이라는 임시 테이블을 생성)

```python
WITH channel AS (
select DISTINCT channel from raw_data.user_session_channel
),
temp AS (select ...),
...
SELECT *
FROM channel c
JOIN temp t ON c.userId = t.userId
```



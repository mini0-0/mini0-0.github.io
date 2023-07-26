---
title : 3주차 - ETL_Airflow 소개
date : 2023-07-26 13:00 +09:00
categories : [Data, Data Engineering]
tags : [프로그래머스, 실리콘밸리에서 날라온 데이터 엔지니어링 스타터 키트 with python, DE,Airflow]

---

[1. 2주차 숙제](#1-2주차-숙제)
<br>
[2. 데이터 파이프](#2-데이터-파이프)
<br>
[3. Airflow](#3-airflow)
<br>
[4. 데이터 파이프라인을 만들 때 고려할 점](#4-데이터-파이프라인을-만들-때-고려할-점)
<br>


# 1. 2주차 숙제

## 1. 사용자별로 처음/마지막 채널 알아내기(1) - ROW_NUMBER()사용

### 1. 내 코드

```python
%%sql

SELECT ts, channel
FROM raw_data.user_session_channel usc
JOIN raw_data.session_timestamp st ON usc.sessionid = st.sessionid
WHERE userid = 251
ORDER BY 1
LIMIT 15;
```

```python
%%sql

SELECT ROW_NUMBER() OVER(PARTITION BY usc.userid ORDER BY ts) as num,
	usc.userid, ts, channel
FROM raw_data.user_session_channel usc
JOIN raw_data.session_timestamp st ON usc.sessionid = st.sessionid
WHERE userid = 251;
```

### 2. 강사님 코드

```python
%%sql

WITH cte AS (
SELECT userid, channel, (ROW_NUMBER() OVER (PARTITION BY usc.userid ORDER BY st.ts asc))
AS arn, (ROW_NUMBER() OVER (PARTITION BY usc.userid ORDER BY st.ts desc)) AS drn
FROM raw_data.user_session_channel usc
JOIN raw_data.session_timestamp st ON usc.sessionid = st.sessionid
)
SELECT cte1.userid, cte1.channel AS first_touch, cte2.channel AS last_touch
FROM cte cte1
JOIN cte cte2 ON cte1.userid = cte2.userid and cte2.drn = 1
WHERE cte1.arn = 1
ORDER BY 1;
```

### 3. 메모

1. **WITH 이름 AS** 

```python
WITH [ 별명1 ] [ (컬럼명1 [,컬럼명2]) ] AS (
    SUB QUERY
)[, 별명2 AS ... ]
MAIN QUERY
```

- ****WITH Subquery를 빌딩블록으로 사용하여 SELF JOIN을 하고 있음
- Query의 전체적인 가독성을 높이고, 재사용할 수 있음
- 모든 DML에서 사용 가능

1. 문제는 사용자별 → 즉, uerid별로 해야하는 데 문제를 잘못 읽어서 userid = 251 만 추출함

## 2. 사용자별로 처음/마지막 채널 알아내기(2) - FIRST_VALUE(), LAST_VALUE() 사용

### 1. 내 코드

```python
%%sql

SELECT
  DISTINCT usc.userid,
  FIRST_VALUE(usc.channel)
    OVER (
      PARTITION BY usc.userid
      ORDER BY st.ts
      ROWS BETWEEN
        UNBOUNDED PRECEDING
        AND UNBOUNDED FOLLOWING
    ) as first_channel
FROM raw_data.user_session_channel usc
JOIN raw_data.session_timestamp st
  ON usc.sessionid = st.sessionid
WHERE userid =251
ORDER BY 1;
```

```python
%%sql

SELECT
  DISTINCT usc.userid,
  LAST_VALUE(usc.channel)
    OVER (
      PARTITION BY usc.userid
      ORDER BY st.ts
      ROWS BETWEEN
        UNBOUNDED PRECEDING
        AND UNBOUNDED FOLLOWING
    ) as last_channel
FROM raw_data.user_session_channel usc
JOIN raw_data.session_timestamp st
  ON usc.sessionid = st.sessionid
WHERE userid = 251
ORDER BY 1;
```

### 2. 강사님 코드

```python
%%sql

SELECT DISTINCT A.userid,
FIRST_VALUE(A.channel) over(partition by A.userid order by B.ts
rows between unbounded preceding and unbounded following) AS First_Channel,
LAST_VALUE(A.channel) over(partition by A.userid order by B.ts
rows between unbounded preceding and unbounded following) AS Last_Channel
FROM raw_data.user_session_channel A
LEFT JOIN raw_data.session_timestamp B
ON A.sessionid = B.sessionid;
```

### 3. 메모

1. Window 함수 문법
- ROW - 부분집합인 window 크기를 물리적인 단위로 행 집합을 지정
- RANGE - 논리적인 주소에 의해 행 집합을 지정
- UNBOUNDED PRECEDING - 윈도우의 시작위치가 첫번째 ROW
- UNBOUNDED FOLLOWING - 윈도우의 마지막 위치가 마지막 ROW

1. Windows 함수 문법 - 예시

```python
SELECT
**SUM(value)**
OVER (**order by value rows between 2 preceding and 2 following**) AS rolling_sum
FROM raw_data.rows_test ;
```

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/47ba88ce-a24d-4e44-9858-904f6f7f34b3)
![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/66dd3a6f-0234-476d-8f81-4ad188a1f322)


- 위의 코드는 현재 위치에서 앞에2개 뒤에2개 sum한 값
- 즉, 현재 index가 3인 위치에서
    - 앞에 2개→ index 1
    - 뒤에 2개 → index 5
    - 까지의 합 → 1 + 2 +  3 + 4 + 5 =15
- index 3의 값 → 15

## 3. Gross Revenue가 가장 큰 UserID 10개 찾기

### 1. 내 코드

```python
%%sql

SELECT DISTINCT userID,SUM(amount)
FROM raw_data.session_transaction st
LEFT JOIN raw_data.user_session_channel usc ON usc.sessionid = st.sessionid
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10;
```

### 2. 강사님 코드

```python
%%sql

SELECT userID,
SUM(amount) gross_revenue,
SUM(CASE WHEN st.refunded is not TRUE THEN amount END) net_revenue
FROM raw_data.session_transaction st
JOIN raw_data.user_session_channel usc ON st.sessionid = usc.sessionid
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10;
```

### 3. 메모

- 내코드에는 Cross revenue : Refund 포함한 매출 빼먹음
- 문제 잘 읽어보자..^^

## 4. ****채널별 월 매출액 테이블 만들기 (본인 스키마 밑에 CTAS로 테이블 만들기)****

### 1. 내 코드

```python
%%sql

SELECT
  LEFT(ts,7) AS "Month",usc.channel,
  COUNT(DISTINCT userid) AS uniqueUsers,
  COUNT(DISTINCT CASE WHEN amount > 0 THEN usc.userid END ) AS paidUsers,
  ROUND(paidUsers*100.0 / NULLIF(uniqueUsers,0),3) AS conversionRate,
  SUM(DISTINCT amount) AS grossRevenue,
  SUM(DISTINCT CASE WHEN refunded	is FALSE THEN amount END) AS netRevenue
FROM raw_data.user_session_channel usc
JOIN raw_data.session_timestamp ti ON ti.sessionid = usc.sessionid
LEFT JOIN raw_data.session_transaction  st ON st.sessionid = usc.sessionid
GROUP BY 1,2
ORDER BY 1,2;
```

### 2. 강사님 코드

```python
%%sql

SELECT LEFT(ts, 7) "month",
usc.channel,
COUNT(DISTINCT userid) uniqueUsers,
COUNT(DISTINCT (CASE WHEN amount is not NULL THEN userid END)) paidUsers,
ROUND(paidUsers::float*100/NULLIF(uniqueUsers, 0),2) conversionRate, -- 51.49%
SUM(amount) grossRevenue,
SUM(CASE WHEN refunded is not True THEN amount END) netRevenue
FROM raw_data.user_session_channel usc
LEFT JOIN raw_data.session_timestamp t ON t.sessionid = usc.sessionid
LEFT JOIN raw_data.session_transaction st ON st.sessionid = usc.sessionid
GROUP BY 1, 2
ORDER BY 1, 2;
```

### 3. 메모

- 문제에 “netRevenue (refund 제외) -> CASE WHEN 사용” 이라고 작성됨
- BUT netRevenue에 refunded가 False인 경우를 뺐음…(문제 제대로 읽자)

---

# 2. 데이터 파이프

## 1. ETL vs ELT

### 1. ETL(Extract, Transform, Load)

- **데이터를 데이터 웨어하우스 외부에서 내부로 가져오는 프로세스**

1. Extract(추출) : 데이터를 데이터 소스에서 읽어내는 과
2. Transform(변환) : 필요하다면 그 원본 데이터의 포맷을 원하는 형태로 변경시키는 과정 → 굳이 변환할 필요는X
3. Load(로드) : DW(Data Warehouse)에 테이블로 Load하는 작업

### 2. ELT(Extract, Load, Transform)

- **데이터 웨어하우스 내부 데이터를 조작해서(보통은 좀더 추상화되고 요약된) 새로운 데이터를 만드는 프로세스,DBT(Analytics Engineering)**

### 3. 비교

- ETL에서 데이터는 데이터 소스에서 스테이징을 거쳐 데이터 대상으로 이동합니다.
- ELT에서는 데이터 대상에서 변환을 수행하므로 데이터 스테이징이 필요하지 않습니다.
- ETL은 민감한 데이터가 데이터 대상에 로드되기 전에 정리되므로 데이터 개인 정보 보호와 규정 준수에 도움이 되는 반면, ELT는 더 간단하며 데이터 요구 사항이 많지 않은 기업에 적합합니다.

## 2. Data Lake vs Data Warehouse

### 1. 데이터 레이크(Data Lake)

- 구조화 데이터 + 비구조화 데이터
- 보존 기한이 없는 모든 데이터를 원래 형태대로 보존하는 스토리지에 가까움
- 보통은 데이터 웨어하우스보다 몇배는 더 큰 스토리지

### 2. 데이터 웨어하우스 (Data Warehouse)

- 보존 기한이 있는 구조화된 데이터를 저장하고 처리하는 스토리지
- 보통 BI 툴들(룩커, 태블로, 수퍼셋, …)은 데이터 웨어하우스를 백엔드로 사용함

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/4aae84b2-cf7c-4aa2-ad6f-f94cda99b42e)
![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/b8050190-3030-49f2-816d-c1b4d2835c7a)


## 3. Data Pipleline의 정의

- Data Pipeline, ETL(Extract Transform Load), Data Workflow, Dag(Directed acyclic graph)
- 데이터를 **소스**로부터 **목적지**로 복사하는 작업
    - 데이터 소스 예시
        - Click stream, call data, ads performance data, transactions, sensor data, metadata, …
        - More concrete examples: production databases, log files, API, stream data (Kafka topic)
    
    - 데이터 목적지 예
        - 데이터 웨어하우스, 캐시 시스템 (Redis, Memcache), 프로덕션 데이터베이스, NoSQL, S3
- 보통 python, scala 혹은 SQL으로 작업

## 4. 데이터 파이프라인의 종류

### 1. Raw Data ETL Jobs - 데이터 엔지니어

- 외부와 내부 데이터 소스에서 데이터를 읽어(Extract)다가 (많은 경우 API를 통하게 됨)
- 적당한 데이터 포맷 변환(Transform) 후 (데이터의 크기가 커지면 Spark등이 필요해짐)
- 데이터 웨어하우스 로드(Load)

### 2. Summary / Report Jobs - Analytics Engineer (DBT)

- DW(혹은 DL)로부터 데이터를 읽어 다시 DW에 쓰는 ETL
- Raw Data를 읽어서 일종의 리포트 형태나 써머리 형태의 테이블을 다시
만드는 용도
- 특수한 형태로는 AB 테스트 결과를 분석하는 데이터 파이프라인도 존재

### 3. Production Data Jobs

1. DW로부터 데이터를 읽어 다른 Storage(많은 경우 프로덕션 환경)로 쓰는 ETL
    - summary 정보가 프로덕션 환경에서 성능 이유로 필요한 경우
    - 혹은 머신러닝 모델에서 필요한 피쳐들을 미리 계산해두는 경우
2. 흔한 타켓 스토리지:
    - Cassandra/HBase/DynamoDB와 같은 NoSQL
    - MySQL과 같은 관계형 데이터베이스 (OLTP)
    - Redis/Memcache와 같은 캐시
    - ElasticSearch와 같은 검색엔진
    

### 5. 간단한 ETL

[Google Colaboratory](https://colab.research.google.com/drive/1t7q_EuE-0BfKx8PghTFY4BElOwa0R4zy)

# 3. Airflow

## 1. Airflow 소개

- python으로 작성된 데이터 파이프라인 (ETL) 플랫폼
- 데이터 파이프라인 스케줄링 + UI지원
- 데이터 파이프라인 관리/작성 프레임
- Airflow에서는 데이터 파이프라인을 DAG(Directed Acyclic Graph)라고 부름

## 2. Airflow 구성

### 1. Web Server

- Airflow의 웹 UI 서버

### 2. Scheduler

- 모든 DAG와 Task에 대하여 모니터링 및 관리하고, 실행해야할 Task를 스케줄링

### 3. Worker

- 실제 Task를 실행하는 주체입니다. Executor 종류에 따라 동작 방식이 다양

### 4. Database(Sqlite가 기본으로 설치됨)

- Airflow에 존재하는 DAG와 Task들의 메타데이터를 저장하는 데이터베이스

### 5. Queue(기본적으로는 멀티노드 구성인 경우에만 사용)

 

## 3. Airflow 구조

### 1. 단일 서버

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/30d3e2cc-1f95-4ae1-a10c-6102a13b3a42/Untitled.png)

**1. Airflow 스케일링 방법**

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/9a36b2e0-ee60-4bb5-bd9a-b3b25ab774a8)

- 스케일 업(더 좋은 사양 서버 사용)
- 스케일 아웃(서버 추가)
- Docker, K8s 사용


### 2. 다수 서버

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/13ca9cf3-f5c1-4efa-8d2c-686e4f5eaab0)


## 4. Airflow 개발의 장단점

### 1. 장점

- 데이터 파이프라인을 세밀하게 제어 가능
- 다양한 데이터 소스와 데이터 웨어하우스를 지원
- 백필(Backfill)이 쉬움

### 2. 단점

- 배우기가 쉽지 않음
- 상대적으로 개발환경을 구성하기가 쉽지 않음
- 직접 운영이 쉽지 않음. 클라우드 버전 사용이 선호됨
    - GCP provides “Cloud Composer”
    - AWS provides “Managed Workflows for Apache Airflow”
    - Azure provides “Azure Data Factory Managed Airflow”

## 5. DAG

### 1. DAG(Directed Acyclic Graph)란?

- Airflow에서 ETL을 부르는 명칭
- DAG는 Task로 구성됨
    - Task란? - Airflow의 Operator로 만들어짐
        - Airflow에서 이미 다양한 종류의 Operator를 제공함
        - 경우에 맞게 사용 Operator를 결정하거나 필요하다면 직접 개발
        - e.g., Redshift writing, Postgres query, S3 Read/Write, Hive query, Spark job, shell script
        

### 2. DAG 구성 예제(1)

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/98ebdd45-21b8-4c51-83fe-6bd730d426e1)
![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/aff5c479-e4be-45a5-ab88-ae22f40bebdb)


### 4. 모든 Task에 필요한 기본정보

```python
default_args = {
'owner': 'keeyong',
'start_date': datetime(2020, 8, 7, hour=0, minute=00),
'end_date': datetime(2020, 8, 31, hour=23, minute=00),
'email': ['keeyonghan@hotmail.com'],
'retries': 1,
'retry_delay': timedelta(minutes=3),
}
```

### 5. 모든 DAG에 필요한 기본정보

```python
from airflow import DAG

test_dag = DAG(
"HelloWorld", # DAG name
schedule="0 9 * * *",
tags=['test']
default_args=default_args
)
```

- 0 * * * *: 매시 0 분에 실행되는 DAG, 1시간에 한번
- 0 12 * * *: 매일 한번 12:00에 실행되는 DAG, 하루에 한번

# 4. 데이터 파이프라인을 만들 때 고려할 점

## 1. 이상과 현실간의 괴리

### 1.  이상 혹은 환상

- 내가 만든 데이터 파이프라인은 문제 없이 동작할 것이다
- 내가 만든 데이터 파이프라인을 관리하는 것은 어렵지 않을 것이다

→ 즉, 본인의 착각

### 2. 현실 혹은 실상

- 데이터 파이프라인은 많은 이유로 실패함
    - 버그 :)
    - 데이터 소스상의 이슈: What if data sources are not available or change its data format
    - 데이터 파이프라인들간의 의존도에 이해도 부족
    
- 데이터 파이프라인의 수가 늘어나면 유지보수 비용이 기하급수적으로 늘어남
    - 데이터 소스간의 의존도가 생기면서 이는 더 복잡해짐. 만일 마케팅 채널 정보가
    업데이트가 안된다면 마케팅 관련 다른 모든 정보들이 갱신되지 않음
    - More tables needs to be managed (source of truth, search cost, …)

## 2. Best Pratices

### 1. Full Refresh

- 가능하면 데이터가 작을 경우 매번 통채로 복사해서 테이블 생성
- Incremental update만이 가능하다면, 대상 데이터소스가 갖춰야할 몇 가지
조건(created, modified, deleted)

### 2. Idempotency(멱등성)

- 동일한 입력 데이터로 데이터 파이프라인을 다수 실행해도 최종 테이블의 내용이 달라지지
말아야함
- 중복 데이터 X

### 3. 실패한 데이터 파이프라인을 재실행이 쉬어야 함

- 과거 데이터를 다시 채우는 과정(Backfill)이 쉬어야 함
- airflow가 backfill에 강점을 갖고 있음
    - Backfill : 실패한 데이터 파이프랄인을 재실행 or 읽어온 데이터들의 문제로 다시 읽어와야하는 경
    - DAG의 **catchup** 파라미터가 **True**가 되어야하고 **start_date**과 **end_date**이 적절하게
    설정되어야함
    - 대상 테이블이 **incremental update**가 되는 경우에만 의미가 있음

### 4. 명확성

- 데이터 파이프라인의 입력과 출력을 명확히 하고 문서화
- 주기적으로 쓸모없는 데이터들을 삭제

### 5. 문서화

- 데이터 파이프라인 사고시 마다 사고 리포트(post-mortem) 쓰기
- 중요 데이터 파이프라인의 입력과 출력을 체크하기

## 3. Full Refresh vs Incremental Update

### 1. Full Refresh: 매번 소스의 내용을 다 읽어오는 방식

- 효율성이 떨어질 수 있지만 간단하고 소스 데이터에 문제가 생겨도 다시 다 읽어오기에
**유지보수가 쉬움**
- 데이터가 커지면 사용불가

### 2. Incremental Update

- **효율성이 좋지만** 복잡해지고 유지보수가 힘들어짐
- 보통 daily나 hourly로 동작해서 그 전 시간 혹은 그 전 날 데이터를 읽어오는 형태로 동작





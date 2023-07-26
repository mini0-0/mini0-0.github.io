---
title : 6주차- Airflow OLTP 복사와 ETL구현
date : 2023-07-26 13:00 +09:00
categories : [Data, Data Engineering]
tags : [프로그래머스, 실리콘밸리에서 날라온 데이터 엔지니어링 스타터 키트 with python, DE, Airflow]

---

[1. 5주차 숙제](#1-5주차-숙제)
<br>
[2. MySQL 테이블 복사하기](#2-mysql-테이블-복사하기)
<br>
[3. Backfill 실행](#3-backfill-실행)
<br>
[4. Summary 테이블 구현](#4-summary-테이블-구현)
<br>

# 1. 5주차 숙제

## 1. 퀴즈 리뷰

<details>
<summary>퀴즈</summary>
<div markdown="1">


### Q1. Airflow에서 하나의 DAG는 다수의 ()로 구성된다. ()에 들어갈 말은?

- Task

### Q2. 매일 동작하는 DAG의 Start date이 2021-02-05라면 이 DAG의 첫 실행 날짜는?

- 2021-02-06

### Q3. 위 DAG의 경우 이때 execution_date으로 들어오는 날짜는?

- 2021-02-05

### Q4. Schedule interval이 "30 * * * *"으로 설정된 DAG에 대한 올바른 설명은?

- 매시 30분마다 한번씩 실행된다

### Q5. Schedule interval이 "0 * * * *"으로 설정된 DAG의 start date이 "2021-02-04 00:00:00"으로 잡혀있다면 이 DAG의 첫 번째 실행 날짜와 시간은 언제인가?

- 2021-02-04 01:00:00

### Q6. Airflow의 DAG가 처음 ON이 되었을 때 start_date과 현재 날짜 사이에 실행이 안된 run들이 있을 경우 이를 실행한다. 이는 (??) 파라미터에 의해 결정된다. 이 파라미터를 False로 세팅하면 과거 실행이 안된 run을 무시한다

- catchup

### Q7. Redshift에서 큰 데이터를 테이블로 복사하는 방식을 제대로 설명한 것은?

- 복사할 레코드들을 파일로 저장해서 S3로 올린 후에 거기서 Redshift로 벌크 복사한다

</div>
</details>

<br>

## 2. UpdateSymbol_v2의 Incremental Update 방식 수정해보기

### 1. 내 코드

<details>
<summary>코드</summary>
<div markdown="1">

``` python
def _create_table(cur, schema, table, drop_first):
    if drop_first:
        cur.execute(f"DROP TABLE IF EXISTS {schema}.{table};")
    cur.execute(f"""
CREATE TABLE IF NOT EXISTS {schema}.{table} (
    date date,
    "open" float,
    high float,
    low float,
    close float,
    volume bigint,
    created_date timestamp default GETDATE()
);""")

@task
def load(schema, table, records):
    logging.info("load started")
    cur = get_Redshift_connection()
    try:
        cur.execute("BEGIN;")
        # 원본 테이블이 없으면 생성 - 테이블이 처음 한번 만들어질 때 필요한 코드
        _create_table(cur, schema, table, False)
        # 임시 테이블로 원본 테이블을 복사
        cur.execute(f"CREATE TEMP TABLE t AS SELECT * FROM {schema}.{table};")
        for r in records:
            sql = f"INSERT INTO t VALUES ('{r[0]}', {r[1]}, {r[2]}, {r[3]}, {r[4]}, {r[5]});"
            print(sql)
            cur.execute(sql)

        # 임시 테이블 내용을 원본 테이블로 복사 
        cur.execute(f"DELETE FROM {schema}.{table};")
        cur.execute(f"""INSERT INTO {schema}.{table}
            SELECT date, "open", high,low, close, volume FROM(
                SELECT *, ROW_NUMBER() OVER (PARTITION  BY date ORDER BY created_date DESC) seq
                FROM t
              )
            WHERE seq = 1; """)  
        
        cur.execute("COMMIT;")   # cur.execute("END;")
    except 	Exception as error:
        print(error)
        cur.execute("ROLLBACK;") 
        raise
    logging.info("load done")

with DAG(
    dag_id = 'UpdateSymbol_v3',
    start_date = datetime(2023,5,30),
    catchup=False,
    tags=['API'],
    schedule = '0 10 * * *'
) as dag:

    results = get_historical_prices("AAPL")
    load("nalala8200", "stock_info_v3", results)
```

</div>
</details>

<br>

### 2. 강사님 코드
<details>
<summary>코드</summary>
<div markdown="1">

```python
def _create_table(cur, schema, table, drop_first):
    if drop_first:
        cur.execute(f"DROP TABLE IF EXISTS {schema}.{table};")
    cur.execute(f"""
CREATE TABLE IF NOT EXISTS {schema}.{table} (
    date date,
    "open" float,
    high float,
    low float,
    close float,
    volume bigint,
    created_date timestamp default GETDATE()
);""")

@task
def load(schema, table, records):
    logging.info("load started")
    cur = get_Redshift_connection()
    try:
        cur.execute("BEGIN;")
        # 원본 테이블이 없으면 생성 - 테이블이 처음 한번 만들어질 때 필요한 코드
        _create_table(cur, schema, table, False)
        # 임시 테이블로 원본 테이블을 복사
        cur.execute(f"CREATE TEMP TABLE t AS SELECT * FROM {schema}.{table};")
        for r in records:
            sql = f"INSERT INTO t VALUES ('{r[0]}', {r[1]}, {r[2]}, {r[3]}, {r[4]}, {r[5]});"
            print(sql)
            cur.execute(sql)

        # 임시 테이블 내용을 원본 테이블로 복사 
        cur.execute(f"DELETE FROM {schema}.{table};")
        cur.execute(f"""INSERT INTO {schema}.{table}
            SELECT date, "open", high,low, close, volume FROM(
                SELECT *, ROW_NUMBER() OVER (PARTITION  BY date ORDER BY created_date DESC) seq
                FROM t
              )
            WHERE seq = 1; """)  
        
        cur.execute("COMMIT;")   # cur.execute("END;")
    except Exception as error:
        print(error)
        cur.execute("ROLLBACK;") 
        raise
    logging.info("load done")

with DAG(
    dag_id = 'UpdateSymbol_v3',
    start_date = datetime(2023,5,30),
    catchup=False,
    tags=['API'],
    schedule = '0 10 * * *'
) as dag:

    results = get_historical_prices("AAPL")
    load("nalala8200", "stock_info_v3", results)

```

</div>
</details>

### 3. 메모

- **Incremental Update** 방식 - 새로 Update된 데이터만 적재
- **UpdateSymbol_v2.py(Full Refresh) → UpdateSymbol_v3.py(IncrementalUpdate) 방법**
    1. 원본 테이블이 없으면 **테이블 생성**
    2. **임시 테이블** 생성, **원본 테이블** 내용 입력
    3. 중복을 걸러주는 SQL이 작성된 코드를 **원본 테이블**에 복사
- 강사님이 작성했던 코드기반으로 작성하여서 강시님 코드와 크게 다르지 않음

# 2. MySQL 테이블 복사하기

## 1. ETL 소개

### 1. **MySQL → AWS Redshift로 복사**

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/e00a5eef-3128-428e-af59-902ed7cee850)
![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/1402e3b8-5cc1-4178-8cf1-04fcfe3961a0)

- 레코드가 몇개 없으면 → INSERT INTO
- 레코드가 많다면 → COPY Command
    - 데이터 소스(nps)를 읽음 → 파일 저장 → Amazon S3에 업로드 → AWS Redshift로 COPY

## 2. 코드 구현

### 1. MySQL(OLTP, Production Database)

```python
CREATE TABLE prod.nps (
id INT NOT NULL AUTO_INCREMENT primary key,
created_at timestamp,
score smallint
);
```

- 이미 테이블은 MySQL(nps)에서 소스 생성

### 2. Redshift(OLAP, Data Warehouse)

```python
CREATE TABLE (본인의스키마).nps (
id INT NOT NULL primary key,
created_at timestamp,
score smallint
);
```

- Redshift쪽에 본인 스키마 밑에 별도로 만들고 뒤에서 실습할 DAG를 통해
MySQL쪽 테이블로부터 Redshift 테이블로 복사하는 것을 연습하는게 목표

### 3. MySQL_to_Redshift DAG의 Task 구성

- `SqlToS3Operator`
    - MySQL SQL 결과 -> S3
    - (s3://grepp-data-engineering/{본인ID}-nps)
    - s3://s3_bucket/s3_key
    
- `S3ToRedshiftOperator`
    - S3 -> Redshift 테이블
    - (s3://grepp-data-engineering/{본인ID}-nps) -> Redshift (본인스키마.nps)
    - COPY command is used
    
- COPY 과정

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/ddc65311-d576-41d4-8b49-4450fb807a6c)

### 4. MySQL_to_Redshift.py
<details>
<summary>코드</summary>
<div markdown="1">

    from airflow import DAG
    from airflow.operators.python import PythonOperator
    from airflow.providers.amazon.aws.transfers.sql_to_s3 import SqlToS3Operator
    from airflow.providers.amazon.aws.transfers.s3_to_redshift import S3ToRedshiftOperator
    from airflow.models import Variable
    
    from datetime import datetime
    from datetime import timedelta
    
    import requests
    import logging
    import psycopg2
    import json
    
    from plugins import slack
    
    dag = DAG(
        dag_id = 'MySQL_to_Redshift',
        start_date = datetime(2022,8,24), # 날짜가 미래인 경우 실행이 안됨
        schedule = '0 9 * * *',  # 적당히 조절
        max_active_runs = 1,
        catchup = False,
        default_args = {
            'retries': 1,
            'retry_delay': timedelta(minutes=3),
            'on_failure_callback': slack.on_failure_callback,
        }
    )
    
    schema = "사용자"
    table = "nps"
    s3_bucket = "grepp-data-engineering"
    s3_key = schema + "-" + table
    
    # Mysql -> Amazon S3
    mysql_to_s3_nps = SqlToS3Operator(
        task_id = 'mysql_to_s3_nps',
        query = "SELECT * FROM prod.nps",
        s3_bucket = s3_bucket,
        s3_key = s3_key,
        sql_conn_id = "mysql_conn_id",
        aws_conn_id = "aws_conn_id",
        verify = False,
        replace = True,
        pd_kwargs={"index": False, "header": False},    
        dag = dag
    )
    
    # Amazon S3 -> AWS Redshift
    s3_to_redshift_nps = S3ToRedshiftOperator(
        task_id = 's3_to_redshift_nps',
        s3_bucket = s3_bucket,
        s3_key = s3_key,
        schema = schema,
        table = table,
        copy_options=['csv'],
        method = 'REPLACE',
        redshift_conn_id = "redshift_dev_db",
        aws_conn_id = "aws_conn_id",
        dag = dag
    )
    
    mysql_to_s3_nps >> s3_to_redshift_nps
    
</div>
</details>
    

→ MySQL 있는 테이블 nps를 Redshift내의 각자 스키마 밑의 nps 테이블로 복사

- S3를 경유해서 COPY 명령으로 복사

### 5. MySQL_to_Redshift_v2.py (Incremental Update)

<details>
<summary>코드</summary>
<div markdown="1">
``` python

    from airflow import DAG
    from airflow.operators.python import PythonOperator
    from airflow.providers.amazon.aws.transfers.sql_to_s3 import SqlToS3Operator
    from airflow.providers.amazon.aws.transfers.s3_to_redshift import S3ToRedshiftOperator
    from airflow.models import Variable
    
    from datetime import datetime
    from datetime import timedelta
    
    import requests
    import logging
    import psycopg2
    import json
    
    dag = DAG(
        dag_id = 'MySQL_to_Redshift_v2',
        start_date = datetime(2023,1,1), # 날짜가 미래인 경우 실행이 안됨
        schedule = '0 9 * * *',  # 적당히 조절
        max_active_runs = 1,
        catchup = False,
        default_args = {
            'retries': 1,
            'retry_delay': timedelta(minutes=3),
        }
    )
    
    schema = "사용자"
    table = "nps"
    s3_bucket = "grepp-data-engineering"
    s3_key = schema + "-" + table       # s3_key = schema + "/" + table
    
    sql = "SELECT * FROM prod.nps WHERE DATE(created_at) = DATE('{{ execution_date }}')"
    print(sql)
    mysql_to_s3_nps = SqlToS3Operator(
        task_id = 'mysql_to_s3_nps',
        query = sql,
        s3_bucket = s3_bucket,
        s3_key = s3_key,
        sql_conn_id = "mysql_conn_id",
        aws_conn_id = "aws_conn_id",
        verify = False,
        replace = True,
        pd_kwargs={"index": False, "header": False},    
        dag = dag
    )
    
    s3_to_redshift_nps = S3ToRedshiftOperator(
        task_id = 's3_to_redshift_nps',
        s3_bucket = s3_bucket,
        s3_key = s3_key,
        schema = schema,
        table = table,
        copy_options=['csv'],
        redshift_conn_id = "redshift_dev_db",
        aws_conn_id = "aws_conn_id",    
        method = "UPSERT",
        upsert_keys = ["id"],
        dag = dag
    )
    
    mysql_to_s3_nps >> s3_to_redshift_nps
    
```
</div>
</details>


    
- MySQL/PostgreSQL 테이블이면 만족해야 할 조건
    - **`created (timestamp)`**: Optional → 레코드가 생성된 시간
    - `**modified (timestamp)**`: → 레코드가 수정된 시간
    - `**deleted (boolean)**`: 레코드를 삭제하지 않고 deleted를 True로 설정 → 레코드가 삭제된 시간

- **Daily Update**이고 테이블의 이름이 A이고 MySQL에서 읽어온다면
1. **ROW_NUMBER로 직접 구현하는 경우**
    - 먼저 Redshift의 A 테이블의 내용을 temp_A로 복사
    - MySQL의 A 테이블의 레코드 중 **modified**의 날짜가 지난 일(execution_date)에 해당하는 모든 레코드를 읽어다가 temp_A로 복사
        - 아래는 MySQL에 보내는 쿼리. 결과를 파일로 저장한 후 S3로 업로드하고 COPY 수행
            - SELECT * FROM A WHERE DATE(modified) = DATE(execution_date)
        - temp_A의 레코드들을 primary key를 기준으로 파티션한 다음에 modified 값을 기준으로
        DESC 정렬해서, 일련번호가 1인 것들만 다시 A로 복사

1. **S3ToRedshiftOperator로 구현하는 경우**
    - query 파라미터로 아래를 지정
    - SELECT * FROM A WHERE DATE(modified) = DATE(execution_date)
        - method 파라미터로 “UPSERT”를 지정
        - upsert_keys 파라미터로 Primary key를 지정
            - 앞서 nps 테이블이라면 “id” 필드를 사용
            

# 3. Backfill 실행

## 1. Backfill을 Command에서 실행

```python
airflow dags backfill dag_id -s 2018- 07- 01 -e 2018- 08- 01
```

- 2018년 7월 데이터 다시 읽어오려고 함
- catchup = True, execution_date을 사용해서 Incremental Update가 구현되어 있어야함
- start_date부터 시작하지만 end_date은 포함하지 않음
- 실행순서는 날짜/시간순은 아니고 랜덤. 만일 날짜순으로 하고 싶다면
    - DAG default_args의 depends_on_past를 True로 설정
    
    ```python
    default_args = {
    'depends_on_past': True,
    …
    ```
    

## 2. Backfill ready

- 먼저 모든 DAG가 backfill을 필요로 하지는 않음
    - Full Refresh를 한다면 backfill은 의미X
- 여기서 backfill은 일별 혹은 시간별로 업데이트하는 경우를 의미함
    - 마지막 업데이트 시간 기준 backfill을 하는 경우라면 (Data Warehouse 테이블에 기록된 시간
    기준) 이런 경우에도 execution_date을 이용한 backfill은 필요X
- 데이터의 크기가 굉장히 커지면 backfill 기능을 구현해 두는 것이 필수
    - airflow가 큰 도움이 됨
    - 하지만 데이터 소스의 도움 없이는 불가능

- 어떻게 backfill로 구현할 것인가
    - 제일 중요한 것은 데이터 소스가 backfill 방식을 지원해야함
    - “execution_date”을 사용해서 업데이트할 데이터 결정
    - “catchup” = True
    - start_date/end_date을 backfill하려는 날짜로 설정
    - 다음으로 중요한 것은 DAG 구현이 execution_date을 고려해야 하는 것이고 idempotent
    해야함
    

# 4. Summary 테이블 구현

## 1. 간단한 Summary Table

- **Build_Summary.py** 코드

<details>
<summary>코드</summary>
<div markdown="1">
    
```python

    from airflow import DAG
    from airflow.operators.python import PythonOperator
    from airflow.models import Variable
    from airflow.hooks.postgres_hook import PostgresHook
    from datetime import datetime
    from datetime import timedelta
    
    from airflow import AirflowException
    
    import requests
    import logging
    import psycopg2
    
    from airflow.exceptions import AirflowException
    
    def get_Redshift_connection():
        hook = PostgresHook(postgres_conn_id = 'redshift_dev_db')
        return hook.get_conn().cursor()
    
    def execSQL(**context):
    
        schema = context['params']['schema'] 
        table = context['params']['table']
        select_sql = context['params']['sql']
    
        logging.info(schema)
        logging.info(table)
        logging.info(select_sql)
    
        cur = get_Redshift_connection()
    
        sql = f"""DROP TABLE IF EXISTS {schema}.temp_{table};CREATE TABLE {schema}.temp_{table} AS """
        sql += select_sql
        cur.execute(sql)
    
        cur.execute(f"""SELECT COUNT(1) FROM {schema}.temp_{table}""")
        count = cur.fetchone()[0]
        if count == 0:
            raise ValueError(f"{schema}.{table} didn't have any record")
    
        try:
            sql = f"""DROP TABLE IF EXISTS {schema}.{table};ALTER TABLE {schema}.temp_{table} RENAME to {table};"""
            sql += "COMMIT;"
            logging.info(sql)
            cur.execute(sql)
        except Exception as e:
            cur.execute("ROLLBACK")
            logging.error('Failed to sql. Completed ROLLBACK!')
            raise AirflowException("")
    
    dag = DAG(
        dag_id = "Build_Summary",
        start_date = datetime(2021,12,10),
        schedule = '@once',
        catchup = True
    )
    
    execsql = PythonOperator(
        task_id = 'mau_summary',
        python_callable = execSQL,
        params = {
            'schema' : 'nalala8200',
            'table': 'mau_summary',
            'sql' : """SELECT 
      TO_CHAR(A.ts, 'YYYY-MM') AS month,
      COUNT(DISTINCT B.userid) AS mau
    FROM raw_data.session_timestamp A
    JOIN raw_data.user_session_channel B ON A.sessionid = B.sessionid
    GROUP BY 1 
    ;"""
        },
        dag = dag
    )

```
</div>
</details>

    
- MAU 요약 테이블

## 2. NPS Summary Table

### 1. NPS(Net Promoter Score)

- 10점 만점으로 '주변에 추천하겠는가?'라는 질문을 기반으로 고객 만족도를 계산
- 10, 9점 추천하겠다는 고객(promoter)의 비율에서 0-6점의 불평고객(detractor)의 비율을 뺀
것이 NPS
    - 7, 8점은 계산에 안들어

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/6b912360-de3c-42d9-9342-6b1e70809dd5)

### 2. 일별NPS 계산 SQL

```python
SELECT LEFT(created_at, 10) AS date,
ROUND(
SUM(
CASE
WHEN score >= 9 THEN 1
WHEN score <= 6 THEN -1
END
)::float*100/COUNT(1), 2
) nps
FROM 사용자.nps
GROUP BY 1
ORDER BY 1;
```

### 3. NPS 파일

```python
{
          'table': 'nps_summary',
          'schema': '사용자',
          'main_sql': """
            SELECT LEFT(created_at, 10) AS date,
            ROUND(SUM(CASE
            WHEN score >= 9 THEN 1 
            WHEN score <= 6 THEN -1 END)::float*100/COUNT(1), 2)
            FROM nalala8200.nps
            GROUP BY 1
            ORDER BY 1;""",

          'input_check':
          [
            {
              'sql': 'SELECT COUNT(1) FROM nalala8200.nps',
              'count': 150000
            },
          ],
          'output_check':
          [
            {
              'sql': 'SELECT COUNT(1) FROM {schema}.temp_{table}',
              'count': 12
            }
          ],
}
```





---
title : 5주차 - Airflow Deepdive_v2
date : 2023-07-26 13:00 +09:00
categories : [Data, Data Engineering]
tags : [프로그래머스, 실리콘밸리에서 날라온 데이터 엔지니어링 스타터 키트 with python, DE, Airflow]

---

[1. 4주차 숙제](#1-4주차-숙제)
<br>
[2. Open Wearhter DAG 구현하기](#2-open-wearhter-dag-구현하기)
<br>
[3. Primary Key Uniqueness 보장하기](#3-primary-key-uniqueness-보장하기)
<br>
[4. Backfill과 Airflow](#4-backfill과-airflow)
<br>

# 1. 4주차 숙제 
## 1. airflow.cfg 퀴즈

<details>
<summary>퀴즈</summary>
<div markdown="1">


### 1. DAGs 폴더는 어디에 지정되는가?

- core 섹션의 dags_folder 키
- /opt/airflow/dags

### 2. DAGs 폴더에 새로운 Dag 생성시 언제 Airflow 시스템에서 이를 알게 되며 이
스캔 주기를 결정해주는 키의 이름이 무엇인가?

- core 섹션의 dag_dir_list_interval 키

### 3. Airflow를 API 형태로 외부에서 조작하고 싶다면 어느 섹션을 변경해야하는가?

- api 섹션의 auth_backend를 airflow.api.auth.backend.basic_auth로 변경

### 4. Variable에서 변수의 값이 encrypted가 되려면 변수의 이름에 어떤 단어들이
들어가야 하는데 이 단어들은 무엇일까? :)

- password, secret, passwd, authorization, api_key, apikey, access_token

### 5. 이 환경 설정 파일이 수정되었다면 이를 실제로 반영하기 위해서 해야 하는
일은?

- 스케줄러와 웹서버를 재시작해야함. Docker라면 docker-compose.yaml 파일이 변경되고
docker compose down과 docker compose up을 차례로 실행

### 6. Metadata DB의 내용을 암호화하는데 사용되는 키는 무엇인가?

- fernet_key
-> fornet_key가 사라지면 DB복구 못함
</div>
</details>


## 2. 세계 나라 정보 API 사용 DAG 작성

### 1. 내가 작성한 코드

<details>
<summary>코드</summary>
<div markdown="1">


```python
from airflow import DAG
from airflow.decorators import task
from airflow.providers.postgres.hooks.postgres import PostgresHook
from datetime import datetime

import logging
import requests

def get_Redshift_connection(autocommit=True): 
    hook = PostgresHook(postgres_conn_id='redshift_dev_db')
    conn = hook.get_conn()
    conn.autocommit = autocommit
    return conn.cursor()

@task
def extract_transform():
    logging.info("Extract Start")
    # 데이터 url
    response = requests.get("https://restcountries.com/v3/all")
    countries = response.json()
    records = []

    for country in countries:
        name = country['name']['common']
        population = country['population']
        area = country['area']

        records.append([name, population, area])

    logging.info("Extract End")
    return records

@task
def load(schema, table, records):
        logging.info("load started")
        cur = get_Redshift_connection()
        try:
            cur.execute("BEGIN;")
            # 기존 테이블 존재하면 삭제
            cur.execute(f"DROP TABLE IF EXISTS {schema}.{table};")
            cur.execute(f"""CREATE TABLE IF NOT EXISTS {schema}.{table} (
                country VARCHAR(256) primary key ,
                area FLOAT,
                population INT
                );""")

                    
            for r in records:
                sql = f"INSERT INTO {schema}.{table} VALUES ('{r[0]}', {r[1]}, {r[2]});"
                print(sql)
                cur.execute(sql)
               

            cur.execute("COMMIT;")   # cur.execute("END;")
        
        except Exception as error:
                print(error)
                cur.execute("ROLLBACK;")
                raise
        logging.info("load done")

with DAG(
    dag_id = 'WorldInfo',
    start_date = datetime(2023,5,30),
    catchup=False,
    tags=['API'],
    schedule = '30 6 * * 6'
) as dag:
    records = extract_transform()
    load("nalala8200", "worldinfo", records)
```
</div>
</details>


### 2. 강사님이 작성한 코드

<details>
<summary>코드</summary>
<div markdown="1">


```python
import requests

@task
def extract_transform():
response = requests.get('https://restcountries.com/v3/all')
countries = response.json()
records = []
for country in countries:
name = country['name']['common']
population = country['population']
area = country['area']
records.append([name, population, area])
return records

@task
def load(schema, table, records):
cur = get_Redshift_connection()
try:
	cur.execute("BEGIN;")
	cur.execute(f"DROP TABLE IF EXISTS {schema}.{table};")
	cur.execute(f"""CREATE TABLE {schema}.{table} (
	name varchar(256) PRIMARY KEY, population int, area float);""")
for r in records:
	cur.execute(f"INSERT INTO {schema}.{table} VALUES ('{r[0]}', {r[1]},
	{r[2]});")
	cur.execute("COMMIT;")
except (Exception, psycopg2.DatabaseError) as error:
	cur.execute("ROLLBACK;")
raise

with DAG(
dag_id = 'CountryInfo',
start_date = datetime(2023,5,30),
catchup=False,
tags=['API'],
schedule = '30 6 * * 6' # 0 - Sunday, …, 6 - Saturday
) as dag:
results = extract_transform()
load("keeyong", "country_info", results)


```
</div>
</details>


### 3. 메모

1. airflow는 dags 폴더를 주기적으로 스캔함
    - dag_dir_list_interval = 300 → 300초마다 새 파일을 검색
2. 이때 DAG 모듈이 들어있는 모든 파일들의 메인 함수가 실행이 됨
    - 본의 아니게 개발 중인 테스트 코드도 실행될 수 있음

→ .airflowignore 파일을 dags 폴더에 두기

## 3. Airflow와 TimeZone

### 1. airflow.cfg에는 두 종류의 타임존 관련 키가 존재

- default_timezone
- default_ui_timezone

### 2. start_date, end_date, schedule

- default_timezone에 지정된 타임존을 따름

### 3. execution_date와 로그 시간

- 항상 UTC를 따름
- 즉 execution_date를 사용할 때는 타임존을 고려해서 변환 후 사용 필요

→ 현재로서 UTC를 일관되게 사용하는게 Best

# 2. Open Wearhter DAG 구현하기

## 1. 서울 8일 낮/최소/최대 온도 읽기

### 1. Open Weather API 호출 응답 보기

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/7615660a-6e52-4092-842d-e6da1d6d6c2f)

- dt : 필드 날짜
- temp : 필드 온도 정

### 2. 실행 코드

<details>
<summary>코드</summary>
<div markdown="1">


```python
from airflow import DAG
from airflow.models import Variable
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.decorators import task

from datetime import datetime
from datetime import timedelta

import requests
import logging
import json

def get_Redshift_connection():
    # autocommit is False by default
    hook = PostgresHook(postgres_conn_id='redshift_dev_db')
    return hook.get_conn().cursor()

@task
def etl(schema, table):
    api_key = Variable.get("open_weather_api_key")
    # 서울의 위도/경도
    lat = 37.5665
    lon = 126.9780

    # https://openweathermap.org/api/one-call-api
    url = f"https://api.openweathermap.org/data/2.5/onecall?lat={lat}&lon={lon}&appid={api_key}&units=metric&exclude=current,minutely,hourly,alerts"
    response = requests.get(url)
    data = json.loads(response.text) # date = respense.json(url)도 가능
    """
    {'dt': 1622948400, 'sunrise': 1622923873, 'sunset': 1622976631, 'moonrise': 1622915520, 'moonset': 1622962620, 'moon_phase': 0.87, 'temp': {'day': 26.59, 'min': 15.67, 'max': 28.11, 'night': 22.68, 'eve': 26.29, 'morn': 15.67}, 'feels_like': {'day': 26.59, 'night': 22.2, 'eve': 26.29, 'morn': 15.36}, 'pressure': 1003, 'humidity': 30, 'dew_point': 7.56, 'wind_speed': 4.05, 'wind_deg': 250, 'wind_gust': 9.2, 'weather': [{'id': 802, 'main': 'Clouds', 'description': 'scattered clouds', 'icon': '03d'}], 'clouds': 44, 'pop': 0, 'uvi': 3}
    """
    ret = []
    for d in data["daily"]:
        day = datetime.fromtimestamp(d["dt"]).strftime('%Y-%m-%d')
        ret.append("('{}',{},{},{})".format(day, d["temp"]["day"], d["temp"]["min"], d["temp"]["max"]))

    cur = get_Redshift_connection()
    drop_recreate_sql = f"""DROP TABLE IF EXISTS {schema}.{table};
CREATE TABLE {schema}.{table} (
    date date,
    temp float,
    min_temp float,
    max_temp float,
    created_date timestamp default GETDATE()
);
"""
    insert_sql = f"""INSERT INTO {schema}.{table} VALUES """ + ",".join(ret)
    logging.info(drop_recreate_sql)
    logging.info(insert_sql)
    try:
        cur.execute(drop_recreate_sql)
        cur.execute(insert_sql)
        cur.execute("Commit;")
    except Exception as e:
        cur.execute("Rollback;")
        raise

with DAG(
    dag_id = 'Weather_to_Redshift',
    start_date = datetime(2023,5,30), # 날짜가 미래인 경우 실행이 안됨
    schedule = '0 2 * * *',  # 적당히 조절
    max_active_runs = 1,
    catchup = False,
    default_args = {
        'retries': 1,
        'retry_delay': timedelta(minutes=3),
    }
) as dag:

    etl("keeyong", "weather_forecast")
```
</div>
</details>


- **GETDATE()** : 오늘 날짜를 반환
- **strftime()** : 날짜, 시간을 문자열로 출
- **strptime() :** 날짜, 시간 문자열을 datetime으로 변환

# 3. Primary Key Uniqueness 보장하기

## 1. Primary Key Uniqueness란?

### 1. 테이블에서 하나의 레코드를 유일하게 지칭할 수 있는 필드(들)

- 하나의 필드가 일반적이지만 다수의 필드를 사용할 수도 있음
- 이를 CREATE TABLE 사용시 지정

### 2. 관계형 데이터베이스 시스템이 Primary key의 값이 중복 존재하는 것을 막아줌

- 예 1) Users 테이블에서 email 필드
- 예 2) Products 테이블에서 product_id 필드

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/ce0cbf4a-41fb-4661-9a14-9718909ca4f8)

## 2. 빅데이터 기반 데이터 웨어하우스들은 Primary Key를 지켜주지 X

### 1. Primary key를 기준으로 유일성 보장을 해주지 않음

- 이를 보장하는 것은 데이터 인력의 책임

### 2. Primary key 유일성을 보장해주지 않는 이유는?

- 보장하는데 메모리와 시간이 더 들기 때문에 대용량 데이터의 적재가 걸림돌이 됨

### 3. 예시

```python
CREATE TABLE keeyong.test (
date date primary key,
value bigint
);
Primary Key Uniqueness 보장하기
INSERT INTO keeyong.test VALUES ('2023-05-10', 100); (1)
INSERT INTO keeyong.test VALUES ('2023-05-10', 150); (2)
```

- date가 primary key이기 때문에 원래 date는 Unique해야함
- 근데 (2)에서 작업 성공

## 3. Primary Key 유지하는 방법

### 1. 코드

```python
CREATE TABLE keeyong.weather_forecast (
date date primary key,
temp float,
min_temp float,
max_temp float,
created_date timestamp default GETDATE()
);
```

- 날씨 정보이기 때문에 최근 정보가 더 신뢰할 수 있음
- 그래서 어느 정보가 더 최근 정보 인지를 `created_date` 필드에 기록하고 이를 활용
- 즉 date이 같은 레코드들이 있다면 `created_date`을 기준으로 더 최근 정보를 선택

### 2. date, created_date를 통한 순서

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/a51fcb57-4f60-4f2f-819a-30e7a3c928a0)

- date : 날짜 정보
- created_date : 기록한 날

1. date 별로 created_date의 역순으로 일련번호 작성
    - seq 컬럼 추가 date별로 레코드 모으기
    - created_date의 역순으로 sort 후 순서대로 seq부여
2. ROW_NUMBER() 사용
    - ROW_NUMBER() OVER (partition by date order by created_date DESC) seq

### 3. 임시 테이블과 원본 테이블

1. 임시 테이블(스테이징 테이블)을 만들고 거기로 현재 모든 레코드를 복사
2. 임시 테이블에 새로 데이터소스에서 읽어들인 레코드들을 복사 → 이때 중복 존재 가능함
3. 중복을 걸러주는 SQL 작성
    - 최신 레코드를 우선 순위로 선택
    - ROW_NUMBER를 이용해서 primary key로 partition을 잡고 적당한 다른 필드(보통 타임스탬프 필드)로 ordering(역순 DESC)을 수행해 primary key별로 하나의 레코드를 잡아냄
4. 위의 SQL을 바탕으로 최종 원본 테이블로 복사
    - 이때 원본 테이블에서 레코드들을 삭제
    - 임시 temp 테이블을 원본 테이블로 복사 (일련번호가 1번인 것들만 선택)

```python
CREATE TEMP TABLE t AS SELECT * FROM keeyong.weather_forecast; 
# 1. 원래 테이블의 내용을 임시 테이블 t로 복사 

DAG는 임시 테이블(스테이징 테이블)에 레코드를 추가
#2. 이때 중복 데이터가 들어갈 수 있음

DELETE FROM keeyong.weather_forecast;

INSERT INTO keeyong.weather_forecast
SELECT date, temp, min_temp, max_temp, created_date
FROM (
SELECT *, ROW_NUMBER() OVER (PARTITION BY date ORDER BY created_date DESC) seq
FROM t
)
WHERE seq = 1; 
# 3. 중복을 없앤 형태로 새로운 테이블 생성
```

## 4. Weather_Forecase DAG를 Incremental Update로 구현한 코드

<details>
<summary>코드</summary>
<div markdown="1">



```python
from airflow import DAG
from airflow.decorators import task
from airflow.models import Variable
from airflow.providers.postgres.hooks.postgres import PostgresHook

from datetime import datetime
from datetime import timedelta

import requests
import logging
import json

def get_Redshift_connection(): # redshirft 연
    # autocommit is False by default
    hook = PostgresHook(postgres_conn_id='redshift_dev_db')
    return hook.get_conn().cursor()

@task
def etl(schema, table, lat, lon, api_key):

    # https://openweathermap.org/api/one-call-api
    url = f"https://api.openweathermap.org/data/2.5/onecall?lat={lat}&lon={lon}&appid={api_key}&units=metric&exclude=current,minutely,hourly,alerts"
    response = requests.get(url)
    data = json.loads(response.text)

    """
    {'dt': 1622948400, 'sunrise': 1622923873, 'sunset': 1622976631, 'moonrise': 1622915520, 'moonset': 1622962620, 'moon_phase': 0.87, 'temp': {'day': 26.59, 'min': 15.67, 'max': 28.11, 'night': 22.68, 'eve': 26.29, 'morn': 15.67}, 'feels_like': {'day': 26.59, 'night': 22.2, 'eve': 26.29, 'morn': 15.36}, 'pressure': 1003, 'humidity': 30, 'dew_point': 7.56, 'wind_speed': 4.05, 'wind_deg': 250, 'wind_gust': 9.2, 'weather': [{'id': 802, 'main': 'Clouds', 'description': 'scattered clouds', 'icon': '03d'}], 'clouds': 44, 'pop': 0, 'uvi': 3}
    """
    ret = []
    for d in data["daily"]:
        day = datetime.fromtimestamp(d["dt"]).strftime('%Y-%m-%d')
        ret.append("('{}',{},{},{})".format(day, d["temp"]["day"], d["temp"]["min"], d["temp"]["max"]))

    cur = get_Redshift_connection()
    
    # 원본 테이블이 없다면 생성
    create_table_sql = f"""CREATE TABLE IF NOT EXISTS {schema}.{table} (
    date date,
    temp float,
    min_temp float,
    max_temp float,
    created_date timestamp default GETDATE()
);"""
    logging.info(create_table_sql)

    # 임시 테이블 생성
    create_t_sql = f"""CREATE TEMP TABLE t AS SELECT * FROM {schema}.{table};"""
    logging.info(create_t_sql)
    try:
        cur.execute(create_table_sql)
        cur.execute(create_t_sql)
        cur.execute("COMMIT;")
    except Exception as e:
        cur.execute("ROLLBACK;")
        raise

    # 임시 테이블 데이터 입력
    insert_sql = f"INSERT INTO t VALUES " + ",".join(ret)
    logging.info(insert_sql)
    try:
        cur.execute(insert_sql)
        cur.execute("COMMIT;")
    except Exception as e:
        cur.execute("ROLLBACK;")
        raise

    # 기존 테이블 대체
    alter_sql = f"""DELETE FROM {schema}.{table};
      INSERT INTO {schema}.{table}
      SELECT date, temp, min_temp, max_temp FROM (
        SELECT *, ROW_NUMBER() OVER (PARTITION BY date ORDER BY created_date DESC) seq
        FROM t
      )
      WHERE seq = 1;"""
    logging.info(alter_sql)
    try:
        cur.execute(alter_sql)
        cur.execute("COMMIT;")
    except Exception as e:
        cur.execute("ROLLBACK;")
        raise

with DAG(
    dag_id = 'Weather_to_Redshift_v2',
    start_date = datetime(2022,8,24), # 날짜가 미래인 경우 실행이 안됨
    schedule = '0 4 * * *',  # 적당히 조절
    max_active_runs = 1,
    catchup = False,
    default_args = {
        'retries': 1,
        'retry_delay': timedelta(minutes=3),
    }
) as dag:

    etl("keeyong", "weather_forecast_v2", 37.5665, 126.9780, Variable.get("open_weather_api_key"))
```
</div>
</details>


## 4. Upsert란?

1. **Upsert** = **Insert** + **Update**
2. Primary Key를 기준으로 레코드 존재O →  새 정보로 수정
3. 레코드 존재X → 새 레코드로 적재
4. 보통 데이터 웨어하우스마다 UPSERT를 효율적으로 해주는 문법을 지원해줌

# 4. Backfill과 Airflow

## 1. Incremental Update가 실패하면?

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/9137ca10-3eb1-48a1-aed9-a051a2bd93bf)

- 만약 5월24, 25일에 실패했다면 5월 24일은 23일, 25일은 24일 정보를 읽어오도록 되어 있기 때문에 빠지는 데이터 발생
- 이럴때 `bakfill`이 필요함 → airflow는 `backfill`이 쉽게 된다
- `backfill` 은 `Incremetal Update`를 할 때만 사용가능
- `Incremetal Update` 은 효율성이 좋지만/ 유지보수의 난이도는 올라감
    
    → 실수등으로 데이터가 빠지는 일이 생길 수 있음
    
    과거 데이터를 다시 다 읽어와야하는 경우 다시 모두 재실행을 해주어야함
    

→ 가능하면 FULL Refresh를 사용하는 게 좋음

## 2. Backfill

### 1. Backfill이란?

- 실패한 데이터 파이프라인을 재실행 혹은 읽어온 데이터들의 문제로 다시 다 읽어와야하는 경우

### 2. Backfill 해결

- Incremental Update에서 복잡해짐
- Full Refresh에서는 간단. 그냥 다시 실행하면 됨

→ 즉 실패한 데이터 파이프라인의 재실행이 얼마나 용이한 구조인가?

- 이게 잘 디자인된 것이 바로 Airflow

### 3. 어떻게 ETL을 구현해놓으면 이런 일이 편해질까?

- **날짜별로 backfill 결과를 기록하고 성공 여부를 기록**
- **날짜를 시스템에서 ETL 인자로 제공**하여 데이터 엔지니어는 읽어와야 되는 데이터의 날짜를 계산하지 않고 시스템이 지정해 준 날짜를 사용
- airflow는 **모든 날짜에 대해서 실행이 실패했는지 성공했는지 그리고 그에 따른 날짜를 기록**해 둔다. 모든 DAG 실행 시에는 `execution_date`이 지정되어 있는데 **airflow에서 `execution_date`를 읽어온 후 이 날짜를 바탕으로 데이터 갱신이 되도록 코드를 작성**해 준다.
- 이를 통해 `backfill`이 쉬워진다.

## 3. start_date와 execution_date

### 1. 문제

- 아래 사진은 2020-08-10 02:00:00로 start date로 설정된 daily job 이다(catchup=True)
- 현재 시간이 2020-08-13 20:00:00이고 처음으로 이 job이 활성화되었다
- 이 경우 이 job은 몇번 실행될까? (execution_date)


![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/8d72585d-c8d7-404f-9ef8-ebc9abfb1c06)

### 2. 해설

- **2020-08-11 02:00:00**
- **2020-08-12 02:00:00**
- **2020-08-13 02:00:00**총 **세 번 실행**된다.

## 4. Backfill과 관련된 Airflow 변수들

### 1. start_date

- DAG가 처음 실행되는 날짜가 아니라 DAG가 처음 읽어와야하는 데이터의
날짜/시간. 실제 **첫 실행날짜는 start_date + DAG의 실행주기**

### 2. execution_date

- DAG가 읽어와야하는 데이터의 날짜와 시간

### 3. catchup

- DAG가 처음 활성화된 시점이 `start_date`보다 미래라면 그 사이에 실행이 안된 것들을 어떻게 할 것인지 결정해주는 파라미터. 
- `True`가 **디폴트값**이고 이 경우실행안 된 것들을 모두 따라잡으려고 함. 
- `False`가 되면 실행 안된 것들을 무시함

### 4. end_date

- 이 값은 보통 필요하지 않으며 Backfill을 날짜 범위에 대해 하는 경우에만 필요
    - airflow dags backfill -s …. -e ….






---
title : 4주차 - Airflow Deepdive
date : 2023-07-26 13:00 +09:00
categories : [Data, Data Engineering]
tags : [프로그래머스, 실리콘밸리에서 날라온 데이터 엔지니어링 스타터 키트 with python, DE, Airflow]

---

[1. 3주차 숙제](#1-3주차-숙제)
<br>
[2. Hello World 예제 프로그램 살펴보기](#2-hello-world-예제-프로그램-살펴보기)
<br>
[3. Name Gender DAG 개선하기](#3-name-gender-dag-개선하기)
<br>

# 1. 3주차 숙제
## 1. 트랜잭션(Transaction)

### 1. 트랜잭션이란?

- Atomic하게 실행되어야 하는 SQL들을 묶어서 하나의 작업처럼 처리하는
방법
- BEGIN과 END 혹은 BEGIN과 COMMIT 사이에 해당 SQL들을 사용
- ROLLBACK

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/6c1919c7-c006-4189-bee3-e0ee60906826)

### 2. 트랜잭션 구현방법

1. autocommit : 레코드 변경을 바로 반영하는지
2. autocommit=True
    - 기본적으로 모든 SQL statement가 바로 커밋(COMMIT)됨
    - 이를 바꾸고 싶다면 BEGIN;END; 혹은 BEGIN;COMMIT을 사용 (혹은 ROLLBACK)
3. autocommit=False
    - 기본적으로 모든 SQL statement가 커밋(COMMIT)되지 않음
    - 커넥션 객체의 .commit()과 .rollback()함수로 커밋할지 말지 결정
    

→ 무엇을 사용할지는 개인의 취향 

- python은 try/catch와 같이 사용하는 것이 일반적
- try/catch로 에러가 나면 rollback을 명시적으로 실행. 에러가 안 나면 commit을 실행

### 3. try/except 사용시 유의사항

```python
try:
	cur.execute(create_sql)
	cur.execute("COMMIT;")
except Exception as e:
	cur.execute("ROLLBACK;")
	raise
```

1. raise - 예외를 발생함
2. 위 코드에서 raise를 호출하면 예외 발생 → exceptiond으로 가서 예외 처리(예외 처리를 안했다면 에러 출력)
    - ETL을 관리하는 입장에서 어떤 에러가 감춰지는 것보다는 명확하게 드러나는 것이 더 좋음

# 2. Hello World 예제 프로그램 살펴보기

# 1. python 코드 살펴보기

### 1. Python Operator

```python
from airflow.operators.python import PythonOperator

load_nps = PythonOperator(
        dag = dag,
        task_id = "task_id",
        python_callable = python_func,
        params = {
                "table": "delighted_nps",
                "schema": "raw_data"
        }
)

# 태스크 실행 시 호출할 Python 함수
def python_func(**cxt):
        table = cxt["params"]["table"]
        schema = cxt["params"]["schema"]

        ex_date = cxt["execution_date"]
```

### 2. 실습 코드

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

dag = DAG(
    dag_id = 'HelloWorld',
    start_date = datetime(2022,6,14),
    catchup=False,
    tags=['example'],
    schedule = '0 2 * * *')

def print_hello():
    print("hello!")
    return "hello!"

def print_goodbye():
    print("goodbye!")
    return "goodbye!"

print_hello = PythonOperator(
    task_id = 'print_hello',
    #python_callable param points to the function you want to run 
    python_callable = print_hello,
    #dag param points to the DAG that this task is a part of
    dag = dag)

print_goodbye = PythonOperator(
    task_id = 'print_goodbye',
    python_callable = print_goodbye,
    dag = dag)

#Assign the order of the tasks in our DAG
print_hello >> print_goodbye
```

- print_hello: PythonOperator로 구성되어 있으며 먼저 실행
- print_goodbye: PythonOperator로 구성되어 있으며 두번째로 실행

### 3. Airflow Decorators - 프로그래밍이 단순해짐

```python
from airflow.decorators import task

@task
def print_hello():
        print("hello!")
        return "hello!"

@task
def print_goodbye():
        print("goodbye!")
        return "goodbye!"

with DAG(
        dag_id = "HelloWorld",
        startdate = datetime(2022, 5, 5),
        catchup = False,
        tags = ["example"],
        schedule = "0 2 * * *"
) as dag:

# Assign the tasks to the DAG in order
print_hello() >> print_goodbye() #함수 이름이 Task ID가 
```

### 4. default_args 예시

```python
from datetime import datetime, timedelta

default_args = {
                'owner': 'ownerid',
                'retries': 0,
                'retry_delay': timedelta(seconds=20),
                'depends_on_past': False
                }
```

### 5. DAG 예시

```python
dag = DAG(  
			"dag_v1", # DAG name
            'start_date': datetime(2020,8,7,hour=0,minute=00),  
            # schedule (same as cronjob) 
            schedule_interval="0 * * * *",   
            # common settings 
            default_args=default_args 
          )
```

### 6. Task 예시

```python
from datetime import datetime, timedelta

default_args = {
'owner': 'keeyong',
'email': ['keeyonghan@hotmail.com'],
'retries': 1,
'retry_delay': timedelta(minutes=3),
}
```

### 7. 중요한 DAG 파라미터(not task 파라미터)

```python
with DAG(
dag_id = 'HelloWorld_v2',
start_date = datetime(2022,5,5),
catchup=False,
tags=['example'],
schedule = '0 2 * * *'
) as dag:
```

-

- `max_active_runs` : 동시 실행 가능한 DAG 수(default=16)
- `max_active_tasks` : 동시 실행 가능한 Task 수(default=16)
- `catchup` : 밀린 거 자동으로 채울건지위와 같은 DAG parameters는 DAG레벨로 적용되는 것이고, `default_args`로 지정되는 건 Task parameters로 Task레벨로 적용되는 것!

## 2. DAG 실행

### 1. Web UI로 실행

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/c3fb1d34-f991-47ec-abc6-a63baa1e9f89)

### 2. 터미널에서 실행(1)

```python
docker exec -it 컨테이너ID sh
```

```python
airflow dags list
airflow tasks list DAG이름
airflow tasks test DAG이름 Task이름 날짜 # test vs. run
```

- 날짜는 YYYY-MM-DD
- start_date보다 과거인 경우는 실행이 되지만 오늘 날짜보다 미래인 경우 실행
안됨
- 이게 바로 execution_date의 값이 됨

### 3. 터미널에서 실행(2)

```python
docker ps
docker exec -it 컨테이너ID sh
airflow dags list
airflow tasks list my_first_dag
airflow tasks test my_first_dag print_hello 2020-08-09
airflow dags test MySQL_to_Redshift_v3 2019-12-08
airflow dags backfill MySQL_to_Redshift_v3 -s 2019-01-01 -e 2019-12-31
```

# 3. Name Gender DAG 개선하기

## 1. NameGenderCSVtoREdshift.py

- https://colab.research.google.com/drive/1nITNr8_z6DDHVXtZ08C8RYDxJDdtHb0O?usp=sharing
- 3주차에서 Colab으로 ETL한 것을 python을 통해 개선

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime
import requests
import logging
import psycopg2

def get_Redshift_connection():
    host = "learnde.cduaw970ssvt.ap-northeast-2.redshift.amazonaws.com"
    user = "keeyong"  # 본인 ID 사용
    password = "..."  # 본인 Password 사용
    port = 5439
    dbname = "dev"
    conn = psycopg2.connect(f"dbname={dbname} user={user} host={host} password={password} port={port}")
    conn.set_session(autocommit=True)
    return conn.cursor()

def extract(url):
    logging.info("Extract started")
    f = requests.get(url)
    logging.info("Extract done")
    return (f.text)

def transform(text):
    logging.info("Transform started")	
    lines = text.strip().split("\n")[1:] # 첫 번째 라인을 제외하고 처리
    records = []
    for l in lines:
      (name, gender) = l.split(",") # l = "Keeyong,M" -> [ 'keeyong', 'M' ]
      records.append([name, gender])
    logging.info("Transform ended")
    return records

def load(records):
    logging.info("load started")
    """
    records = [
      [ "Keeyong", "M" ],
      [ "Claire", "F" ],
      ...
    ]
    """
    schema = "keeyong"
    # BEGIN과 END를 사용해서 SQL 결과를 트랜잭션으로 만들어주는 것이 좋음
    cur = get_Redshift_connection()
    try:
        cur.execute("BEGIN;")
        cur.execute(f"DELETE FROM {schema}.name_gender;") 
        # DELETE FROM을 먼저 수행 -> FULL REFRESH을 하는 형태
        for r in records:
            name = r[0]
            gender = r[1]
            print(name, "-", gender)
            sql = f"INSERT INTO {schema}.name_gender VALUES ('{name}', '{gender}')"
            cur.execute(sql)
        cur.execute("COMMIT;")   # cur.execute("END;") 
    except (Exception, psycopg2.DatabaseError) as error:
        print(error)
        cur.execute("ROLLBACK;")   
    logging.info("load done")

def etl():
    link = "https://s3-geospatial.s3-us-west-2.amazonaws.com/name_gender.csv"
    data = extract(link)
    lines = transform(data)
    load(lines)

dag_second_assignment = DAG(
	dag_id = 'name_gender',
	catchup = False,
	start_date = datetime(2023,4,6), # 날짜가 미래인 경우 실행이 안됨
	schedule = '0 2 * * *')  # 적당히 조절

task = PythonOperator(
	task_id = 'perform_etl',
	python_callable = etl,
	dag = dag_second_assignment)
```

## 2. NameGenderCSVtoREdshift.py 개선하기 (1)

- params를 통해 변수 넘기기
- execution_date 얻어내기
- “delete from” vs. “truncate”
    - DELETE FROM raw_data.name_gender; -- WHERE 사용 가능
    - TRUNCATE raw_data.name_gender;

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.models import Variable

from datetime import datetime
from datetime import timedelta
import requests
import logging
import psycopg2

def get_Redshift_connection():
    host = "learnde.cduaw970ssvt.ap-northeast-2.redshift.amazonaws.com"
    redshift_user = "keeyong"  # 본인 ID 사용
    redshift_pass = "..."  # 본인 Password 사용
    port = 5439
    dbname = "dev"
    conn = psycopg2.connect(f"dbname={dbname} user={redshift_user} host={host} password={redshift_pass} port={port}")
    conn.set_session(autocommit=True)
    return conn.cursor()

def extract(url):
    logging.info("Extract started")
    f = requests.get(url)
    logging.info("Extract done")
    return (f.text)

def transform(text):
    logging.info("Transform started")	
    lines = text.strip().split("\n")[1:] # 첫 번째 라인을 제외하고 처리
    records = []
    for l in lines:
      (name, gender) = l.split(",") # l = "Keeyong,M" -> [ 'keeyong', 'M' ]
      records.append([name, gender])
    logging.info("Transform ended")
    return records

def load(records):
    logging.info("load started")
    """
    records = [
      [ "Keeyong", "M" ],
      [ "Claire", "F" ],
      ...
    ]
    """
    schema = "keeyong"
    # BEGIN과 END를 사용해서 SQL 결과를 트랜잭션으로 만들어주는 것이 좋음
    cur = get_Redshift_connection()
    try:
        cur.execute("BEGIN;")
        cur.execute(f"DELETE FROM {schema}.name_gender;") 
        # DELETE FROM을 먼저 수행 -> FULL REFRESH을 하는 형태
        for r in records:
            name = r[0]
            gender = r[1]
            print(name, "-", gender)
            sql = f"INSERT INTO {schema}.name_gender VALUES ('{name}', '{gender}')"
            cur.execute(sql)
        cur.execute("COMMIT;")   # cur.execute("END;") 
    except (Exception, psycopg2.DatabaseError) as error:
        print(error)
        cur.execute("ROLLBACK;")   
    logging.info("load done")

def etl(**context):
    link = context["params"]["url"]
    # task 자체에 대한 정보 (일부는 DAG의 정보가 되기도 함)를 읽고 싶다면 context['task_instance'] 혹은 context['ti']를 통해 가능
    # https://airflow.readthedocs.io/en/latest/_api/airflow/models/taskinstance/index.html#airflow.models.TaskInstance
    task_instance = context['task_instance']
    execution_date = context['execution_date']

    logging.info(execution_date)

    data = extract(link)
    lines = transform(data)
    load(lines)

dag = DAG(
    dag_id = 'name_gender_v2',
    start_date = datetime(2023,4,6), # 날짜가 미래인 경우 실행이 안됨
    schedule = '0 2 * * *',  # 적당히 조절
    catchup = False,
    max_active_runs = 1,	
    default_args = {
        'retries': 1,
        'retry_delay': timedelta(minutes=3),
    }
)

task = PythonOperator(
    task_id = 'perform_etl',
    python_callable = etl,
    params = {
        'url': "https://s3-geospatial.s3-us-west-2.amazonaws.com/name_gender.csv"
    },
    dag = dag)
```

## 3. NameGenderCSVtoREdshift.py 개선하기 (2)

- Xcom 객체를 사용해서 세 개의 task로 나누기
- Redshift의 스키마와 테이블 이름을 params로 넘기기

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.models import Variable

from datetime import datetime
from datetime import timedelta
import requests
import logging
import psycopg2

def get_Redshift_connection():
    host = "learnde.cduaw970ssvt.ap-northeast-2.redshift.amazonaws.com"
    redshift_user = "keeyong"  # 본인 ID 사용
    redshift_pass = "..."  # 본인 Password 사용
    port = 5439
    dbname = "dev"
    conn = psycopg2.connect(f"dbname={dbname} user={redshift_user} host={host} password={redshift_pass} port={port}")
    conn.set_session(autocommit=True)
    return conn.cursor()

def extract(**context):
    link = context["params"]["url"]
    task_instance = context['task_instance']
    execution_date = context['execution_date']

    logging.info(execution_date)
    f = requests.get(link)
    return (f.text)

def transform(**context):
    logging.info("Transform started")    
    text = context["task_instance"].xcom_pull(key="return_value", task_ids="extract")
    lines = text.strip().split("\n")[1:] # 첫 번째 라인을 제외하고 처리
    records = []
    for l in lines:
      (name, gender) = l.split(",") # l = "Keeyong,M" -> [ 'keeyong', 'M' ]
      records.append([name, gender])
    logging.info("Transform ended")
    return records

def load(**context):
    logging.info("load started")    
    schema = context["params"]["schema"]
    table = context["params"]["table"]

    lines = context["task_instance"].xcom_pull(key="return_value", task_ids="transform")
    """
    records = [
      [ "Keeyong", "M" ],
      [ "Claire", "F" ],
      ...
    ]
    """
    # BEGIN과 END를 사용해서 SQL 결과를 트랜잭션으로 만들어주는 것이 좋음
    cur = get_Redshift_connection()
    try:
        cur.execute("BEGIN;")
        cur.execute(f"DELETE FROM {schema}.name_gender;") 
        # DELETE FROM을 먼저 수행 -> FULL REFRESH을 하는 형태
        for r in records:
            name = r[0]
            gender = r[1]
            print(name, "-", gender)
            sql = f"INSERT INTO {schema}.name_gender VALUES ('{name}', '{gender}')"
            cur.execute(sql)
        cur.execute("COMMIT;")   # cur.execute("END;") 
    except (Exception, psycopg2.DatabaseError) as error:
        print(error)
        cur.execute("ROLLBACK;")   
    logging.info("load done")

dag = DAG(
    dag_id = 'name_gender_v3',
    start_date = datetime(2023,4,6), # 날짜가 미래인 경우 실행이 안됨
    schedule = '0 2 * * *',  # 적당히 조절
    catchup = False,
    max_active_runs = 1,
    default_args = {
        'retries': 1,
        'retry_delay': timedelta(minutes=3),
    }
)

extract = PythonOperator(
    task_id = 'extract',
    python_callable = extract,
    params = {
        'url':  Variable.get("csv_url")
    },
    dag = dag)

transform = PythonOperator(
    task_id = 'transform',
    python_callable = transform,
    params = { 
    },  
    dag = dag)

load = PythonOperator(
    task_id = 'load',
    python_callable = load,
    params = {
        'schema': 'keeyong',
        'table': 'name_gender'
    },
    dag = dag)

extract >> transform >> load
```

## 4. NameGenderCSVtoREdshift.py 개선하기 (3)

- Variable를 이용해 CSV parameter 넘기기
- csv_url -  https://s3-geospatial.s3-us-west-2.amazonaws.com/name_gender.csv
- 값을 암호화 하려면(*****)

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/5ca5d209-729e-4812-9192-01fb28b8e45a)


```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.models import Variable

from datetime import datetime
from datetime import timedelta
import requests
import logging
import psycopg2

def get_Redshift_connection():
    host = "learnde.cduaw970ssvt.ap-northeast-2.redshift.amazonaws.com"
    redshift_user = "keeyong"  # 본인 ID 사용
    redshift_pass = "..."  # 본인 Password 사용
    port = 5439
    dbname = "dev"
    conn = psycopg2.connect(f"dbname={dbname} user={redshift_user} host={host} password={redshift_pass} port={port}")
    conn.set_session(autocommit=True)
    return conn.cursor()

def extract(**context):
    link = context["params"]["url"]
    task_instance = context['task_instance']
    execution_date = context['execution_date']

    logging.info(execution_date)
    f = requests.get(link)
    return (f.text)

def transform(**context):
    logging.info("Transform started")    
    text = context["task_instance"].xcom_pull(key="return_value", task_ids="extract")
    lines = text.strip().split("\n")[1:] # 첫 번째 라인을 제외하고 처리
    records = []
    for l in lines:
      (name, gender) = l.split(",") # l = "Keeyong,M" -> [ 'keeyong', 'M' ]
      records.append([name, gender])
    logging.info("Transform ended")
    return records

def load(**context):
    logging.info("load started")    
    schema = context["params"]["schema"]
    table = context["params"]["table"]

    lines = context["task_instance"].xcom_pull(key="return_value", task_ids="transform")
    """
    records = [
      [ "Keeyong", "M" ],
      [ "Claire", "F" ],
      ...
    ]
    """
    # BEGIN과 END를 사용해서 SQL 결과를 트랜잭션으로 만들어주는 것이 좋음
    cur = get_Redshift_connection()
    try:
        cur.execute("BEGIN;")
        cur.execute(f"DELETE FROM {schema}.name_gender;") 
        # DELETE FROM을 먼저 수행 -> FULL REFRESH을 하는 형태
        for r in records:
            name = r[0]
            gender = r[1]
            print(name, "-", gender)
            sql = f"INSERT INTO {schema}.name_gender VALUES ('{name}', '{gender}')"
            cur.execute(sql)
        cur.execute("COMMIT;")   # cur.execute("END;") 
    except (Exception, psycopg2.DatabaseError) as error:
        print(error)
        cur.execute("ROLLBACK;")   
    logging.info("load done")

dag = DAG(
    dag_id = 'name_gender_v3',
    start_date = datetime(2023,4,6), # 날짜가 미래인 경우 실행이 안됨
    schedule = '0 2 * * *',  # 적당히 조절
    catchup = False,
    max_active_runs = 1,
    default_args = {
        'retries': 1,
        'retry_delay': timedelta(minutes=3),
    }
)

extract = PythonOperator(
    task_id = 'extract',
    python_callable = extract,
    params = {
        'url':  Variable.get("csv_url")
    },
    dag = dag)

transform = PythonOperator(
    task_id = 'transform',
    python_callable = transform,
    params = { 
    },  
    dag = dag)

load = PythonOperator(
    task_id = 'load',
    python_callable = load,
    params = {
        'schema': 'keeyong',
        'table': 'name_gender'
    },
    dag = dag)

extract >> transform >> load
```

## 5. NameGenderCSVtoREdshift.py 개선하기 (4)

- Redshift Connection 사용

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.models import Variable
from airflow.providers.postgres.hooks.postgres import PostgresHook

from datetime import datetime
from datetime import timedelta
from plugins import slack

import requests
import logging
import psycopg2

def get_Redshift_connection(autocommit=True):
    hook = PostgresHook(postgres_conn_id='redshift_dev_db')
    conn = hook.get_conn()
    conn.autocommit = autocommit
    return conn.cursor()

def extract(**context):
    link = context["params"]["url"]
    task_instance = context['task_instance']
    execution_date = context['execution_date']

    logging.info(execution_date)
    f = requests.get(link)
    return (f.text)

def transform(**context):
    logging.info("Transform started")    
    text = context["task_instance"].xcom_pull(key="return_value", task_ids="extract")
    if text is None:
        print("++++++++++++++++++++++++++++++")
    lines = text.strip().split("\n")[1:] # 첫 번째 라인을 제외하고 처리
    records = []
    for l in lines:
      (name, gender) = l.split(",") # l = "Keeyong,M" -> [ 'keeyong', 'M' ]
      records.append([name, gender])
    logging.info("Transform ended")
    return records

def load(**context):
    logging.info("load started")    
    schema = context["params"]["schema"]
    table = context["params"]["table"]
    
    records = context["task_instance"].xcom_pull(key="return_value", task_ids="transform")    
    """
    records = [
      [ "Keeyong", "M" ],
      [ "Claire", "F" ],
      ...
    ]
    """
    # BEGIN과 END를 사용해서 SQL 결과를 트랜잭션으로 만들어주는 것이 좋음
    cur = get_Redshift_connection()
    try:
        cur.execute("BEGIN;")
        cur.execute(f"DELETE FROM {schema}.name_gender;") 
        # DELETE FROM을 먼저 수행 -> FULL REFRESH을 하는 형태
        for r in records:
            name = r[0]
            gender = r[1]
            print(name, "-", gender)
            sql = f"IINSERT INTO {schema}.name_gender VALUES ('{name}', '{gender}')"
            cur.execute(sql)
        cur.execute("COMMIT;")   # cur.execute("END;") 
    except (Exception, psycopg2.DatabaseError) as error:
        print("Error Msg", error)
        cur.execute("ROLLBACK;")
        raise  
    logging.info("load done")

dag = DAG(
    dag_id = 'name_gender_v4',
    start_date = datetime(2023,4,6), # 날짜가 미래인 경우 실행이 안됨
    schedule = '0 2 * * *',  # 적당히 조절
    max_active_runs = 1,
    catchup = False,
    default_args = {
        'retries': 1,
        'retry_delay': timedelta(minutes=3),
        'on_failure_callback': slack.on_failure_callback,
    }
)

extract = PythonOperator(
    task_id = 'extract',
    python_callable = extract,
    params = {
        'url':  Variable.get("csv_url")
    },
    dag = dag)

transform = PythonOperator(
    task_id = 'transform',
    python_callable = transform,
    params = { 
    },  
    dag = dag)

load = PythonOperator(
    task_id = 'load',
    python_callable = load,
    params = {
        'schema': 'keeyong',   ## 자신의 스키마로 변경
        'table': 'name_gender'
    },
    dag = dag)

extract >> transform >> load
```

## 6. Connections과 Variables

- `Connections`
    - **hostname**, **port numbe**r, **access** 정보를 저장하는 데 사용
    - **Postgres** 연결 또는 **Redshift** 연결 정보를 저장 할 수 있음

- `Variables` : 자주 사용되는 configuration info들을 미리 저장해 두는 것.
    - **API key** 또는 일부 구성 정보를 저장하는 데 사용
    - 값을 암호화하려면 이름에 “**access**” 또는 “**secret**”를 사용





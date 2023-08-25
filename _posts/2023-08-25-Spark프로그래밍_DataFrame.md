---
title : Spark 프로그래밍 - DataFrame 
date : 2023-08-14 23:00 +09:00
categories : [Data, spark]
tags : [프로그래머스, 파이썬으로 해보는 spark 프로그래밍 with 프로그래머스, DE, Spark]

---

[1. Spark 데이터 처리](#1-spark-데이터-처리)
<br>
[2. Spark 데이터 구조: RDD, DataFrame, Dataset](#2-spark-데이터-구조-rdd-dataframe-dataset)
<br>
[3. 프로그램 구조](#3-프로그램-구조)
<br>
[4. 개발/실습 환경 소개](#4-개발실습-환경-소개)
<br>


# 1. Spark 데이터 처리

## 1. Spark 데이터 시스템 구조
![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/16a3f026-0ce1-4847-acd2-9629ccf1ad46)

- Spark은 파일 시스템을 별도로 X → 기존에 존재하는 파일 분산시스템 사용
    - HDFS, AWS S3, Azure Blob, GCP, Cloud Storage
- Spark를 사용하는 큰 이유 중 하나는 단일 시스템 기반으로 다양한 기능을 사용할 수 있기 때문

## 2. 데이터 병렬처리가 가능하려면?

### 1. 데이터가 먼저 분산되어야함

- 하둡 맵의 데이터 처리 단위는 디스크에 있는 데이터 블록 (128MB)
    - hdfs-site.xml에 있는 dfs.block.size 프로퍼티가 결정
- Spark에서는 이를 파티션 (Partition)이라 부름. 파티션의 기본크기도 128MB
    - `spark.sql.files.maxPartitionBytes`: HDFS등에 있는 파일을 읽어올 때만 적용됨

### 2. 다음으로 나눠진 데이터를 각각 따로 동시 처리

- 맵리듀스에서 N개의 데이터 블록으로 구성된 파일 처리시 **N**개의 **Map** 태스크가 실행
- **Spark**에서는 **파티션** 단위로 메모리로 로드되어 **Executo**r가 배정됨

### 3. 처리 데이터를 나누기 -> 파티션 -> 병렬처리
![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/2ce50b43-0396-4e58-b481-a69d5c230a60)

1. **처리하려는 데이터 파일 → 파티션**
- ETL process를 따로 만들어서 별도의 프로세스가 입력으로부터 데이터를 읽어서 HDFS에 저장
- 그리고 Spark job을 HDFS에서 읽도록 함

2. 파티션 → 병렬처리
- IF Spark Cluster 내부에 Executor 2개 존재하고 Executor마다 CPU 코어가 1개라면
- 동시에 2개의 task를 처리 할 수 있음

## 3. Spark 데이터 처리 흐름
![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/dd0137fa-a262-4b94-9568-6b5b4e19851e)

- 데이터프레임은 작은 파티션들로 구성됨
    - 데이터프레임은 한번 만들어지면 **수정 불가 (Immutable)**
- 입력 데이터프레임을 원하는 결과 도출까지 다른 데이터 프레임으로
계속 변환
    - sort, group by, filter, map, join, …
- 파티션간에 데이터 이동없이 계속변환이 가능할까?
    - `map`, `filter`는 한 파티션에 있던 데이터가 다음 파티션으로 만들어질 때, 그 파티션의 자체가  physical하게 다른 서버로 이동 할일 X
    - `group by`, `sorting`은 새로운 파티션을 만들어야 함 → 데이터 이동이 필요 → 네트워크 단으로 통해 서버 간 전송 필요(파티션 생성) → 이때 새로운 파티션이 만들어짐, But 파티션 간 **데이터 크기가 균등하지 않을 수 있음**

## 4. 셔플링

### 1. 셔플링이란?

- 파티션간에 데이터 이동이 필요한 경우 발생

### 2. 셔플링이 발생하는 경우는?

- 명시적 파티션을 새롭게 하는 경우 (예: 파티션 수를 줄이기)
- 시스템에 의해 이뤄지는 셔플링

### 3. 셔플링이 발생할 때 네트워을 타고 데이터가 이동하게 됨

- IF 셔플링이 발생 할 때, 몇 개의 파티션이 결과로 만들어질까?
    - `spark.sql.shuffle.partitions`이 결정
        - 기본값은 200이며 이는 최대 파티션 수
    - 오퍼레이션에 따라 파티션 수가 결정됨
        - random, hashing partition, range partition 등등
        - sorting의 경우 range partition을 사용함

- 또한 이때 Data Skew(비대칭) 발생 가능!
    - range에 따라 파티션이 구축되고 키 값에 따라 레코드 이동
    - 이때, 샘플링이 잘못되었다면 파티션간 Skew(비대칭)가 발생

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/80a026f6-22a3-43d2-8f68-5728e6ab7fef)

### 4. 셔플링: hashing partition

- Aggregation 오퍼레이션

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/4b455da2-0188-41bd-88ad-2fc50db9579c)

- 키 값으로 주어진 값들을 hashing fauction으로 넘김 → 만들어진 파티션의 수로 나눠서 어느 파티션으로 갈지 결정
- hashing function에서 같은 것끼리 같은 파티션으로 이동

### 5. Data Skewness

**Data partitioning**은 데이터 처리에 **병렬성**을 주지만 단점도 존재

- 이는 데이터가 균등하게 분포하지 않는 경우
    - 주로 데이터 셔플링 후에 발생
- **셔플링**을 **최소화**하는 것이 중요하고 **파티션** **최적화**를 하는 것이 중요 → Spark job 최적화

# 2. Spark 데이터 구조: RDD, DataFrame, Dataset

## 1. Spark 데이터 구조

### 1. RDD, DataFrame, Dataset (Immutable Distributed Data)

- 2016년에 DataFrame과 Dataset은 하나의 API로 통합됨
- 모두 파티션으로 나뉘어 Spark에서 처리됨

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/7151f249-d0e3-4a47-8e2d-c1771127c8b4)

- 보통 python → DataFrame, Scala → Dataset
- DataFrame과 Dataset은 Catalyst Optimizer를 통해 실제 쿼리 RDD o오퍼레이션으로 바꿀 때, 오퍼레이션 별로 비용을 계산 → 계산된 비용을 바탕로 execution plan을 선택하는 구조
    - Catalyst Optimizer란?
        - Query 최적화 프로그램을 구축하는 Spark 엔진의 핵심, Logical Query들을 Physical Query Execute Plan으로 변환

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/b5c8bcb7-1bf2-4570-beb2-e086e301f6c9)
    
참고  <https://www.databricks.com/kr/glossary/catalyst-optimizer>

### 2. RDD (Resilient Distributed Dataset)

- **Low Level** 데이터로 클러스터내의 서버에 분산된 데이터를 지칭
- 레코드별로 존재하지만 **스키마가 존재하지 않음**
    - 구조화된 데이터나 비구조화된 데이터 모두 지원

### 3. DataFrame과 Dataset

- RDD위에 만들어지는 RDD와는 달리 **필드 정보**를 갖고 있음 (테이블)
- Dataset은 타입 정보가 존재하며 **컴파일 언어**에서 사용가능
    - 컴파일 언어: Scala/Java에서 사용가능
- PySpark에서는 DataFrame을 사용


### 4. Spark 구조

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/aab992e0-bafc-4e07-b1e2-5ccbc1298a58)

- RDD위에 Spark SQL 엔진이 올라가서 RDD operation을 최종적으로 최적화하는 역할
- 최적화 과정

<aside>
💡

1. Code Analysis
2. Logical Optimization (Catalyst Optimizer)
3. Physical Planning
4. Code Generation (Project Tungsten)
</aside>

- 1번 과정에서 작성한 DataFrame이나 Spark SQL를 2번 과정을 통해 최적화
- 3번 과정을 통해 최종적으로 RDD Operation으로 만들어 줌
- 4번 과정을 통해 Java Byte 코드로 만들어

## 2. Spark 데이터 구조 - RDD

- 변경이 불가능한 분산 저장된 데이터
    - RDD는 다수의 파티션으로 구성
    - **Low Level**의 함수형 변환 지원 (map, filter, flatMap 등등)

- 일반 파이썬 데이터는 parallelize 함수로 RDD로 변환
    - 반대는 `collect`로 파이썬 데이터로 변환가능

```python
py_list = [
(1, 2, 3, 'a b c'),
(4, 5, 6, 'd e f'),
(7, 8, 9, 'g h i')
]
rdd = sc.parallelize(py_list)
…
print(rdd.collect())
```

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/331f90a0-7822-4d9b-ab05-cfd6444f3e27)

## 3. Spark 데이터 구조 - DataFrame

- **변경이 불가한** 분산 저장된 데이터
- RDD와는 다르게 관계형 데이터베이스 테이블처럼 컬럼으로 나눠
저장
    - 판다스의 데이터 프레임 혹은 관계형 데이터베이스의 테이블과 거의 흡사
    - 다양한 데이터소스 지원: **HDFS, Hive, 외부 데이터베이스, RDD** 등등
- 스칼라, 자바, 파이썬과 같은 언어에서 지원

# 3. 프로그램 구조

## 1. Spark Session 생성

- Spark 프로그램의 시작은 SparkSession을 만드는 것
    - 프로그램마다 하나를 만들어 Spark Cluster와 통신: Singleton 객체
    - Spark 2.0에서 처음 소개됨
- Spark Session을 통해 Spark이 제공해주는 다양한 기능을 사용
    - DataFrame, SQL, Streaming, ML API 모두 이 객체로 통신
    - config 메소드를 이용해 다양한 환경설정 가능
    - 단 RDD와 관련된 작업을 할때는 SparkSession 밑의 sparkContext 객체를 사용
- Spark Session API 문

<https://spark.apache.org/docs/3.1.1/api/python/reference/api/pyspark.sql.SparkSession.html>

## 2. Spark 세션 생성 - PySpark 예제
![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/152959c6-7977-4b07-869a-d4e977ca760d)

```python
from pyspark.sql import SparkSession # Spark SQL Engine이 중심으로 동작함

# SparkSession은 싱글턴 -> Driver 내부 Spark Session
spark = SparkSession.builder\
.master("local[*]")\
.appName('PySpark Tutorial')\
.getOrCreate()
…

spark.stop()
```

## 3. pyspark.sql 제공 주요 기능

- pyspark.sql.SparkSession
- pyspark.sql.DataFrame
- pyspark.sql.Column
- pyspark.sql.Row
- pyspark.sql.functions
- pyspark.sql.types
- pyspark.sql.Window

## 4. Spark Session 환경 변수

- Spark Session을 만들 때 다양한 환경 설정이 가능

https://spark.apache.org/docs/latest/configuration.html#spark-configuration

- 몇 가지 예
    - **executor별 메모리**: spark.executor.memory (기본값: 1g)
    - **executor별 CPU수**: spark.executor.cores (YARN에서는 기본값 1)
    - **driver 메모리**: spark.driver.memory (기본값: 1g)
    - **Shuffle후 Partition의 수**: spark.sql.shuffle.partitions (기본값: 최대 200)
- 가능한 모든 환경변수 옵션은 여기에서 찾을 수 있음

https://spark.apache.org/docs/latest/configuration.html#application-properties

- 사용하는 Resource Manager에 따라 환경변수가 많이 달라짐

## 5. Spark Session 환경 설정 방법 4가지

1. 환경변수
2. $SPARK_HOME/conf/spark_defaults.conf
    - 보통 Spark Cluster 어드민이 관리
3. spark-submit 명령의 커맨드라인 파라미터
    - 나중에 따로 설명
4. SparkSession 만들때 지정
    - SparkConf

→ 충돌시 우선순위는 높은 번호일수록 높음

### 1. Spark 세션 환경 설정 (1)

- SparkSession 생성시 일일히 지정

```python
from pyspark.sql import SparkSession

# SparkSession은 싱글턴
spark = SparkSession.builder\
.master("local[*]")\
.appName('PySpark Tutorial')\
.config("spark.some.config.option1", "some-value") \
.config("spark.some.config.option2", "some-value") \
.getOrCreate()
```

이 시점의 Spark Configuration은 앞서 언급한 환경변수와 spark_defaults.conf와 spark-submit로 들어온 환경설정이 우선순위를 고려한 상태로 정리된 상태

### 2. Spark 세션 환경 설정 (2)

- SparkConf 객체에 환경 설정하고 SparkSession에 지정

```python
from pyspark.sql import SparkSession
from pyspark import SparkConf

conf = SparkConf()
conf.set("spark.app.name", "PySpark Tutorial")
conf.set("spark.master", "local[*]")

# SparkSession은 싱글턴
spark = SparkSession.builder\
.config(conf=conf) \
.getOrCreate()
```

## 6. 전체적인 플로우

- Spark 세션(SparkSession)을 만들기
- 입력 데이터 로딩
- 데이터 조작 작업 (판다스와 아주 흡사)
    - DataFrame API나 Spark SQL을 사용
    - 원하는 결과가 나올때까지 새로운 DataFrame을 생성
- 최종 결과 저장


![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/7c267deb-f8dd-47af-a358-9ef41ce3cd7b)

## 7. Spark Session이 지원하는 데이터 소스

https://spark.apache.org/docs/latest/sql-data-sources.html

- spark.read(DataFrameReader)를 사용하여 데이터프레임으로 로드
- DataFrame.write(DataFrameWriter)을 사용하여 데이터프레임을 저장
- 많이 사용되는 데이터 소스들
    - HDFS 파일
        - CSV, JSON, Parquet, ORC, Text, Avro
            - Parquet/ORC/Avro에 대해서는 나중에 더 자세히 설명
        - Hive 테이블
- JDBC 관계형 데이터베이스
- 클라우드 기반 데이터 시스템
- 스트리밍 시스템

# 4. 개발/실습 환경 소개

## 1. Spark 개발 환경 옵션

- Local Standalone Spark + Spark Shell
- Python IDE – PyCharm, Visual Studio
- Databricks Cloud – [커뮤니티 에디션을 무료로 사용]
<https://www.databricks.com/try-databricks#account>
- 다른 노트북 – 주피터 노트북, 구글 Colab, 아나콘다 등등

### 1. Local Standalone Spark

- Spark Cluster Manager로 local[n] 지정
    - master를 local[n]으로 지정
    - master는 클러스터 매니저를 지정하는데
    사용
- 주로 개발이나 간단한 테스트 용도
- 하나의 JVM에서 모든 프로세스를 실행
    - 하나의 Driver와 하나의 Executor가 실행됨
    - 1+ 쓰레드가 Executor안에서 실행됨
- Executor안에 생성되는 쓰레드 수
    - local:하나의 쓰레드만 생성
    - local[*]: 컴퓨터 CPU 수만큼 쓰레드를
- Spark 잡을 실행할 때 master를 local[3]으로 지정한 경우

![image](https://github.com/mini0-0/mini0-0.github.io/assets/63296983/a557e91d-c179-4e28-8a29-e6bb5d8a9ecd)

### 2. 구글 Colab에서 Spark 사용

- PySpark + Py4J를 설치
    - 구글 Colab 가상서버 위에 로컬 모드 Spark을 실행
    - 개발 목적으로는 충분하지만 큰 데이터의 처리는 불가
    - Spark Web UI는 기본적으로는 접근 불가
        - ngrok을 통해 억지로 열 수는 있음
    - Py4J
        - 파이썬에서 JVM내에 있는 자바 객체를 사용가능하게 해줌

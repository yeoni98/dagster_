
# Dagster 

## 목차
- [Data Orchestrator](#data-orchestrator)
- [핵심 구성 요소](#핵심-구성-요소)
  - [Op](#op)
  - [Graph](#graph)
  - [Job](#job)
  - [Asset](#asset)
- [asset간 의존성 설정](#다른-asset에-dependency-주입-방법)
  - [Basic dependency](#basic-dependency)
  - [Different code locations](#different-code-locations)
- [asset간 데이터 전달 방법](#자산-간-데이터-전달-방법)
  - [external storage](#external-storage)
  - [I/O Managers](#I/O-Managers)
- [메타데이터 관리](#메타데이터-관리)
  - [메타데이터 및 태그](#메타데이터-및-태그)
  - [Output vs add_output_metadata vs MaterializeResult](#output-vs-add_output_metadata-vs-materializeresult)
- [자동화](#자동화)
  - [스케줄 (Schedule)](#스케줄-schedule)
  - [센서 (Sensor)](#센서-sensor)
- [code locations](#code-locations)
- [artifact](#아티팩트-관리)
- [예제](#예제)




## Data Orchestrator란


## 핵심 구성 요소

<details>
<summary><h3>Op</h3></summary>

- 단일 작업 단위(함수)
- 재사용 및 조합 가능, graph 내에서 연결하여 사용

**예제:**
```python
@dg.op
def extract_data():
    return {"data": [1, 2, 3, 4, 5]}

@dg.op
def transform_data(data):
    return {"transformed_data": [x * 2 for x in data["data"]]}
```
</details>

<details>
<summary><h3>Graph</h3></summary>

- 여러 Op을 연결하여 데이터 흐름을 정의하는 파이프라인 구조

**예제:**
```python
@dg.graph
def etl_process():
    data = extract_data()
    transformed = transform_data(data)
    return transformed
```
</details>

<details>
<summary><h3>Job</h3></summary>

- 실행 가능한 단위로, Graph, Op 혹은 Asset 등으로 구성

**예제:**
```python
# Op 기반 Job
@dg.job
def my_etl_job():
    etl_process()

# 자산 기반 Job
asset_job = dg.define_asset_job(
    name="process_daily_data",
    selection=["daily_extract", "daily_transform", "daily_load"]
)
```
</details>

<details>
<summary><h3>Asset</h3></summary>

- 파이프라인 실행의 결과로 생성되는 데이터 객체(table, file, ml model,..)
- 데이터 중심, 명시적 의존성, 데이터 lineage 추적


**예제:**
```python
@dg.asset
def raw_data():
    return pd.read_csv("data.csv")

@dg.asset(deps=[raw_data])
def cleaned_data(raw_data):
    return raw_data.dropna()
```

케이크 제작 파이프라인에서의 자산 예시:
```python
@dg.asset(
    metadata={
        "description": "레시피 파일에서 재료 정보를 추출합니다.",
        "owner": "ingredients_team"
    },
    tags={"category": "input", "data_type": "json"}
)
def ingredients():
    """레시피 파일에서 재료 정보를 추출합니다."""
    # 구현 코드...
    return dg.MaterializeResult(metadata={...})
```
</details>

## Asset 유형

<details>
<summary><h3>@asset</h3></summary>

가장 기본적인 자산 정의 방법으로, 하나의 함수가 하나의 자산을 생성


**예제:**
```python
@dg.asset
def ingredients():
    # 레시피 정보 추출
    return ingredients_list
```
</details>

<details>
<summary><h3>@multi_asset</h3></summary>

하나의 함수에서 여러 자산을 생성


**예제:**
```python

```
</details>

<details>
<summary><h3>@graph_asset</h3></summary>

여러 Op을 조합하여 하나의 자산을 생성


**예제:**
```python

```
</details>

<details>
<summary><h3>@graph_multi_asset</h3></summary>

여러 Op을 조합하여 여러 자산을 생성 (`@graph_asset`과 `@multi_asset`의 기능 결합)


**예제:**
```python

```
</details>


## asset 간 의존성 설정

두 자산(upstream-downstream)간 의존성을 'deps' 로 부여
(different code locations 에서도 의존성 설정할 수 있음)

<details>
<summary><h3>basic dependency</h3></summary>

**예제**
```python
@dg.asset
def sugary_cereals() -> None:
    execute_query(
        "CREATE TABLE sugary_cereals AS SELECT * FROM cereals WHERE sugar_grams > 10"
    )


@dg.asset(deps=[sugary_cereals])
def shopping_list() -> None:
    execute_query("CREATE TABLE shopping_list AS SELECT * FROM sugary_cereals")
```
</details>

<details>
<summary><h3>dependency in different 'code locations'</h3></summary>

**예제**
```python
@dg.asset
def code_location_1_asset():
    with open("/tmp/data/code_location_1_asset.json", "w+") as f:
        json.dump(5, f)


defs = dg.Definitions(assets=[code_location_1_asset])
```
```python
@dg.asset(deps=["code_location_1_asset"])
def code_location_2_asset():
    with open("/tmp/data/code_location_1_asset.json") as f:
        x = json.load(f)

    with open("/tmp/data/code_location_2_asset.json", "w+") as f:
        json.dump(x + 6, f)
```
</details>

## asset간 데이터 전달 방법

Dagster에서는 자산 간에 데이터를 전달하는 세 가지 주요 방법이 있습니다:

<details>
<summary><h3>1. external storage</h3></summary>


**예제:**
```python

```
</details>

<details>
<summary><h3>2. IO Managers</h3></summary>


**예제:**
```python

```
</details>

<details>
<summary><h3>3. </h3></summary>


**예제:**
```python

```
</details>

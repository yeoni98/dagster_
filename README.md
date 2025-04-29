
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





## Data Orchestrator



## 핵심 구성 요소

### Op

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

### Graph

- 여러 Op을 연결하여 데이터 흐름을 정의하는 파이프라인 구조

**예제:**
```python
@dg.graph
def etl_process():
    data = extract_data()
    transformed = transform_data(data)
    return transformed
```

### Job

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

### Asset

- 파이프라인 실행의 결과로 생성되는 데이터 객체(table, file, ml model 등)
- 데이터 중심, 명시적 의존성, 데이터 계보(lineage) 추적

**기본 예제:**
```python
@dg.asset
def raw_data():
    return pd.read_csv("data.csv")

@dg.asset(deps=[raw_data])
def cleaned_data(raw_data):
    return raw_data.dropna()
```

#### @asset 데코레이터

가장 기본적인 자산 정의 방법으로, 하나의 함수가 하나의 자산을 생성합니다.

**특징:**
- 단일 데이터 산출물 생성
- 기본적으로 함수명이 자산명이 됨
- 간단하고 명확한 구조

**예제:**
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

#### @multi_asset

하나의 함수에서 여러 자산을 생성합니다.

**특징:**
- 한 번의 실행으로 여러 자산 생성
- `yield` 구문으로 여러 자산 반환
- `outs` 매개변수로 출력 자산 정의

**예제:**
```python
@dg.multi_asset(
    outs={
        "production_analysis": dg.AssetOut(description="케이크 생산량 기본 분석"),
        "type_distribution": dg.AssetOut(description="케이크 타입별 분포"),
        "top_cakes": dg.AssetOut(description="가장 많이 생산된 케이크 목록")
    },
    deps={"baked_cakes": baked_cakes}
)
def cake_analysis(baked_cakes):
    # 분석 코드...
    yield dg.MaterializeResult(
        metadata={...},
        asset_key="production_analysis"
    )
    yield dg.MaterializeResult(
        metadata={...},
        asset_key="type_distribution"
    )
    yield dg.MaterializeResult(
        metadata={...},
        asset_key="top_cakes"
    )
```

#### @graph_asset

여러 Op을 조합하여 하나의 자산을 생성합니다.

**특징:**
- Op 기반의 복잡한 로직을 자산으로 표현
- 세분화된 단계를 가진 자산 생성 과정
- 각 단계가 재사용 가능한 Op으로 정의됨

**예제:**
```python
@dg.op
def fetch_recipes():
    # 레시피 데이터 가져오기
    return recipe_data

@dg.op
def process_recipes(recipes):
    # 레시피 처리
    return processed_recipes

@dg.graph_asset
def processed_recipe_data():
    recipes = fetch_recipes()
    return process_recipes(recipes)
```

#### @graph_multi_asset

여러 Op을 조합하여 여러 자산을 생성합니다 (`@graph_asset`과 `@multi_asset`의 기능 결합).

**특징:**
- 여러 단계의 Op에서 여러 자산 생성
- 복잡한 데이터 프로세싱 과정을 모델링

**예제:**
```python
@dg.op
def fetch_cake_data():
    # 케이크 데이터 가져오기
    return cake_data

@dg.op
def analyze_data(data):
    # 데이터 분석
    return analysis, distribution

@dg.graph_multi_asset(
    outs={
        "cake_analysis": dg.AssetOut(description="케이크 분석"),
        "cake_distribution": dg.AssetOut(description="케이크 분포")
    }
)
def cake_analytics():
    data = fetch_cake_data()
    analysis, distribution = analyze_data(data)
    return analysis, distribution
```

## 다른 asset에 dependency 주입 방법

두 자산(upstream-downstream) 간 의존성을 'deps'로 부여합니다.
서로 다른 코드 로케이션에서도 의존성을 설정할 수 있습니다.

### 기본 의존성 (Basic dependency)

**예제:**
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

### 서로 다른 코드 로케이션 (Different code locations)

**예제:**
```python
# 첫 번째 코드 로케이션
@dg.asset
def code_location_1_asset():
    with open("/tmp/data/code_location_1_asset.json", "w+") as f:
        json.dump(5, f)


defs = dg.Definitions(assets=[code_location_1_asset])
```

```python
# 두 번째 코드 로케이션
@dg.asset(deps=["code_location_1_asset"])
def code_location_2_asset():
    with open("/tmp/data/code_location_1_asset.json") as f:
        x = json.load(f)

    with open("/tmp/data/code_location_2_asset.json", "w+") as f:
        json.dump(x + 6, f)
```

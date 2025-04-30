# Dagster 


## 목차
* [Data Orchestrator:Dagster](#data-orchestratordagster)
   * [Architecture](#architecture)
   * [핵심 컴포넌트](#핵심-컴포넌트)
     * [Dagster Daemon](#dagster-daemon)
     * [Code Location Server](#code-location-server)
     * [Executor](#executor)
   * [데이터 흐름](#데이터-흐름)
* [Code Locations](#code-locations)
   * [구현 방법 비교](#구현-방법-비교)
   * [여러 팀 환경 공존 방법](#여러-팀-환경-공존-방법)
* [IO Managers](#io-managers)
* [Key Concepts](#key-concepts)
   * [Op](#op)
   * [Graph](#graph)
   * [Job](#job)
   * [Asset](#asset)
* [Asset 간 dependency 설정](#asset-간-dependency-설정)
   * [Basic dependency](#basic-dependency)
   * [Different code locations](#different-code-locations)
* [Asset간 데이터 전달](#asset간-데이터-전달)
   * [External Storage](#external-storage)
   * [IO Managers](#asset간-io-managers)
* [메타데이터 관리](#메타데이터-관리)
   * [메타데이터 및 태그](#메타데이터-및-태그)
   * [Materialize](#materialize)
* [자동화](#자동화)
   * [스케줄 (Schedule)](#스케줄-schedule)
   * [센서 (Sensor)](#센서-sensor)
* [Artifact](#artifact)
   * [아티팩트 저장 및 관리](#아티팩트-저장-및-관리)
* [케이크 제작 파이프라인 예제](#케이크-제작-파이프라인-예제)
* [분산 실행 환경 구성 (Celery + Docker)](#분산-실행-환경-구성-celery--docker)

## Data Orchestrator:Dagster

데이터 파이프라인의 복잡성을 관리하고, 데이터 처리 작업의 모니터링, 자동화 등을 관리하는 도구. 


- **언제 쓰이나요?**  
  - 데이터 파이프라인이 복잡해지고, 다양한 시스템/서비스 간의 연동이 필요할 때
  - 데이터 품질 관리, 재실행, 에러 알림, 스케줄링 등 운영 자동화가 필요할 때
  

## Architecture

<details>
<summary>Dagster 아키텍처</summary>



### 핵심 컴포넌트

1. **Webserver (Dagit)**
   - 사용자 인터페이스를 제공하는 웹 서버
   - 파이프라인 실행, 모니터링, 디버깅 등 관리 기능 제공

2. **Dagster Instance**
   - 모든 메타데이터를 저장하는 중앙 저장소
   - 실행 기록, 이벤트 로그, 스케줄 상태 등 저장

3. **Dagster Daemon**
   - 스케줄과 센서를 실행하는 백그라운드 프로세스

4. **Run Launcher**
   - 작업 실행을 위한 컴퓨팅 리소스 관리
   - 로컬, Docker, Kubernetes 등 다양한 환경 지원

5. **Code Location Server**
   - Python 코드(자산, 작업 등)를 실행하는 gRPC 서버
   - 각 코드 위치(저장소)별로 별도의 서버 프로세스 실행
   - 코드 정의(자산, 작업)를 Dagster 시스템에 제공

6. **Executor**
   - 각 런(run) 내에서 개별 step을 어떻게 실행할지 결정
   - 로컬 프로세스, 멀티프로세스, Celery, K8s 등 다양한 실행 모드 지원

### 데이터 흐름

#### 실행 요청 과정

```
사용자 → Webserver → Instance → Daemon → Run Launcher → Code Location Server → 실행 환경
```

1. 사용자가 UI에서 작업 실행 요청
2. Webserver가 실행 정보를 Instance에 기록
3. Daemon이 Instance를 주기적으로 확인하여 실행 요청 감지
4. Daemon이 Run Launcher를 통해 실행 환경(로컬, Docker, K8s 등)에서 작업 시작
5. 실행 환경이 Code Location Server에서 파이프라인 정의를 가져와 실행
6. Executor가 각 작업 step을 실행 전략에 따라 처리

이러한 모듈화된 아키텍처 덕분에 Dagster는 복잡한 데이터 워크플로우를 안정적으로 관리하고 실행할 수 있습니다.
</details>

## 핵심 컴포넌트

<details>
<summary><h3>Dagster Daemon</h3></summary>

Dagster daemon은 Dagster의 핵심 기능을 지원하는 장기 실행 프로세스입니다. 이 프로세스는 배포 환경에서 백그라운드에서 실행되며, 다음과 같은 역할을 합니다:

- 활성화된 스케줄에서 런(run)을 생성
- 활성화된 센서에서 런을 생성
- 런 큐에 쌓인 작업을 실행
- 런 워커 실패를 감지 및 처리

즉, Dagster daemon이 있어야 스케줄이나 센서 기반 자동 실행, 런 큐 관리 등 자동화된 워크플로우가 제대로 동작합니다.

**실행 방법:**
```bash
# 로컬에서 daemon과 웹서버 함께 실행
dagster dev

# daemon만 단독으로 실행
dagster-daemon run
```

**주의사항:** 배포 환경에서는 반드시 하나의 dagster-daemon 프로세스만 실행해야 합니다.
</details>

<details>
<summary><h3>Code Location Server</h3></summary>

Code Location Server는 Dagster 정의(assets, jobs, schedules 등)가 포함된 Python 코드를 실행하는 gRPC 서버입니다.

**주요 특징:**
- 각 코드 로케이션별로 별도의 프로세스로 실행됨
- Dagster 시스템과 gRPC로 통신
- 파이프라인 정의와 메타데이터를 Dagster 시스템에 제공
- 코드 변경 시 Dagster 웹서버 재시작 불필요

코드 로케이션을 사용하면 여러 팀이 독립적으로 개발하면서도 하나의 UI에서 모니터링할 수 있고, 팀별로 서로 다른 Python 버전과 라이브러리를 사용할 수 있습니다.
</details>

<details>
<summary><h3>Executor</h3></summary>

Executor는 각 job 실행(run) 내에서 개별 step(작업 단위)을 어떻게 실행할지 결정하는 컴포넌트입니다. job의 실행 계획을 받아서 각 step을 어떤 방식으로 실행할지 관리합니다.

**주요 executor 종류:**

1. **in_process_executor**: 순차적 실행 (개발/디버깅 환경용)
2. **multiprocess_executor**: 병렬 처리 (CPU 바운드 작업)
3. **dask_executor**: Dask 클러스터 활용 (대규모 분산 처리)
4. **celery_executor**: Celery 기반 분산 실행
5. **docker_executor**: Docker 컨테이너로 실행
6. **k8s_job_executor**: Kubernetes Job으로 실행

**사용 예시:**
```python
@dg.job(executor_def=dg.multiprocess_executor)
def my_job():
    ...

# 또는 Definitions에 기본 executor 지정
defs = dg.Definitions(
    assets=[...],
    jobs=[...],
    executor=dg.multiprocess_executor
)
```

적절한 executor를 선택하면 파이프라인 성능과 리소스 활용도를 크게 향상시킬 수 있습니다.
</details>

## Code Locations

<details>
<summary>Code Location</summary>

### Code Location이란?

* Code Location은 Definitions(assets, jobs, schedules 등)를 포함하는 Python 모듈/파일과 이를 로드할 수 있는 환경의 조합
* 각 코드 로케이션은 별도 프로세스에서 실행되어 Dagster 시스템과 gRPC로 통신합니다.
* 장점ㅈ:
  - 코드 변경 시 Dagster 웹서버 재시작 불필요
  - 여러 팀이 독립적으로 개발하면서도 하나의 UI에서 모니터링 가능
  - 팀별로 서로 다른 Python 버전과 라이브러리 사용 가능

### 구현 방법 비교

#### 1. definitions.py 방식 (로컬 개발)

```python
# team_a/definitions.py
import dagster as dg

@dg.asset
def asset_a():
    return "A"

defs = dg.Definitions(assets=[asset_a])
```

```bash
# 실행 방법
dagster dev -f team_a/definitions.py -f team_b/definitions.py
```

* 제한: 모든 코드 로케이션이 동일한 Python 환경에서 실행됨

#### 2. workspace.yaml 방식 (프로덕션)

```yaml
# workspace.yaml
load_from:
  - python_file:
      relative_path: team_a/definitions.py
      location_name: team_a
      executable_path: /venvs/team_a/bin/python
  - python_file:
      relative_path: team_b/definitions.py
      location_name: team_b
      executable_path: /venvs/team_b/bin/python
```

```bash
# 실행 방법
dagster dev -w workspace.yaml
```

* 장점: 코드 로케이션별로 다른 Python 환경 사용 가능

### 여러 팀 환경 공존 방법

#### 1. 코드 로케이션 간 자산 의존성 설정

```python
# 첫 번째 코드 로케이션
@dg.asset
def code_location_1_asset():
    with open("/tmp/data.json", "w+") as f:
        json.dump(5, f)

# 두 번째 코드 로케이션
@dg.asset(deps=["code_location_1_asset"])  # 문자열로 다른 코드 로케이션 자산 참조
def code_location_2_asset():
    with open("/tmp/data.json") as f:
        x = json.load(f)
    return x + 10
```

* 코드 로케이션 간 자산 의존성은 문자열로 asset key를 지정하여 설정합니다.



## Key Concepts

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

#### Asset 유형

**1. @asset 데코레이터**

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

**2. @multi_asset**

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

**3. @graph_asset**

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

**4. @graph_multi_asset**

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
</details>

## Asset 간 dependency 설정

<details>
<summary><h3>Basic dependency</h3></summary>

deps 파라미터를 사용하여 두 자산(upstream-downstream) 간 의존성을 명시할 수 있음

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

deps를 사용하면, 지정된 자산(sugary_cereals)이 먼저 실행된 후 현재 자산(shopping_list)이 실행됩니다. 또한 의존하는 자산의 출력이 자동으로 현재 자산 함수의 파라미터로 전달됩니다.
</details>

<details>
<summary><h3>Different code locations</h3></summary>

서로 다른 코드 로케이션에 있는 자산 간에도 의존성을 설정할 수 있습니다.

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



### deps vs AssetIn 비교


- **deps**
  ```python
  @dg.asset
  def upstream_asset():
      return {"data": "value"}
      
  @dg.asset(deps=[upstream_asset])
  def downstream_asset(upstream_asset):
      # upstream_asset의 출력값 사용 가능
  ```

- **AssetIn**: 커스텀 파라미터 이름 지정 가능
  ```python
  @dg.asset
  def upstream_asset():
      return {"data": "value"}
      
  @dg.asset(
      ins={"custom_name": dg.AssetIn("upstream_asset")}
  )
  def downstream_asset(custom_name):  # 커스텀 이름으로 파라미터 받음
      # upstream_asset의 출력값을 custom_name으로 사용
  ```

**AssetIn이 필요한 경우:**
- 함수 인자 이름과 자산 이름이 다를 때
- 입력 자산의 키를 명시적으로 지정하고 싶을 때
- multi-asset의 특정 출력만 선택할 때

</details>


## 메타데이터 관리

<details>
<summary><h3>메타데이터 및 태그</h3></summary>

자산 정의 시 메타데이터와 태그를 추가할 수 있으며, 실행 시 메타데이터를 생성할 수도 있습니다.

**1. 자산 정의 시 메타데이터 및 태그**
```python
@dg.asset(
    metadata={
        "description": "레시피 파일에서 재료 정보를 추출합니다.",
        "owner": "ingredients_team",
        "documentation": dg.MetadataValue.md("## 재료 추출 프로세스\n1. 파일 로드\n2. 파싱\n3. 정규화")
    },
    tags={"category": "input", "data_type": "json", "priority": "high"}
)
def ingredients():
    # 구현 코드...
```

**2. MaterializeResult 사용하여 메타데이터 추가**
```python
@dg.asset
def ingredients():
    # 구현 코드...
    return dg.MaterializeResult(
        metadata={
            "recipe_count": len(ingredients_list),
            "output_file": dg.MetadataValue.path(output_file),
            "recipes": dg.MetadataValue.json([r["name"] for r in ingredients_list]),
            "table_schema": dg.MetadataValue.table_schema({
                "recipe_id": "INTEGER",
                "name": "TEXT",
                "ingredients": "JSONB"
            })
        }
    )
```

메타데이터와 태그는 Dagit UI에 표시되어 자산의 특성, 출처, 구조 등을 문서화하는 데 도움이 됩니다. 또한 메타데이터를 통해 데이터 품질, 처리 시간, 결과물의 특성 등을 추적할 수 있습니다.
</details>

<details>


## 자동화

<details>
<summary><h3>스케줄 (Schedule)</h3></summary>

정기적인 시간 간격으로 작업을 실행하는 스케줄을 정의할 수 있습니다.

**예제:**
```python
# 케이크 생산 작업 정의
cake_production_job = dg.define_asset_job(
    name="cake_production_job",
    selection="*",
    description="전체 케이크 생산 파이프라인을 실행합니다."
)

# 스케줄 정의
cake_production_schedule = dg.ScheduleDefinition(
    name="daily_cake_production",
    cron_schedule="0 9 * * *",  # 매일 오전 9시
    job=cake_production_job,
    description="매일 오전 9시에 케이크 생산 파이프라인을 실행합니다."
)

# Dagster 정의에 스케줄 등록
defs = dg.Definitions(
    assets=[ingredients, mixed_dough, baked_cakes, cake_analysis],
    jobs=[cake_production_job],
    schedules=[cake_production_schedule]
)
```

스케줄은 cron 표현식을 사용하여 정의되며, 지정된 시간에 작업을 자동으로 실행합니다. Dagster에서는 스케줄 실행 기록을 추적하고, 실패한 실행을 재시도하는 기능도 제공합니다.
</details>

<details>
<summary><h3>센서 (Sensor)</h3></summary>

특정 조건이 충족될 때 작업을 트리거하는 센서를 정의할 수 있습니다.

**예제:**
```python
@dg.sensor(
    job=process_new_recipes_job,
    minimum_interval_seconds=60
)
def new_recipe_sensor(context):
    # 새 레시피 파일 확인
    files = glob.glob("new_recipes/*.json")
    if not files:
        return dg.SensorResult(skip_reason="새 레시피 파일이 없습니다.")
    
    # 새 파일이 발견되면 작업 실행
    newest_file = max(files, key=os.path.getctime)
    return dg.RunRequest(
        run_key=os.path.basename(newest_file),
        run_config={"ops": {"process_recipe": {"config": {"file_path": newest_file}}}}
    )
```

센서는 정기적으로 실행되어 특정 조건(파일 존재, 데이터 변경 등)을 확인하고, 조건이 충족되면 작업을 실행합니다. 이는 이벤트 기반 파이프라인을 구현하는 데 유용합니다.
</details>

## Artifact

<details>
<summary><h3>아티팩트 저장 및 관리</h3></summary>

Dagster에서 아티팩트는 자산 실행의 결과로 생성되는 모든 유형의 파일이나 데이터입니다. 아티팩트의 저장 및 관리는 Dagster 인스턴스 설정(dagster.yaml)과 밀접하게 관련되어 있습니다.

### 1. 작업 중인 데이터와 최종 산출물의 저장 위치

* Dagster에서 자산(asset)의 실제 데이터(artifacts)는 IO Manager가 지정한 위치에 저장됩니다.
* 기본적으로 FilesystemIOManager를 사용하면 로컬 파일 시스템에 저장됩니다.
* dagster.yaml의 `local_artifact_storage` 항목이 이 경로를 지정합니다:

```yaml
local_artifact_storage:
  module: dagster.core.storage.root
  class: LocalArtifactStorage
  config:
    base_dir: "/path/to/dir"
```

* S3, GCS 등 외부 스토리지를 사용하려면, 별도의 IO Manager를 지정해야 합니다.

### 2. 메타데이터, materialization 이벤트, 로그의 저장 위치

* 자산의 materialization 이벤트, 실행 로그, 에러 등은 Dagster 인스턴스의 event log storage에 저장됩니다.
* dagster.yaml의 `storage` 항목이 이 메타데이터의 저장 위치를 지정합니다:

```yaml
storage:
  sqlite:
    base_dir: /path/to/dir
```

또는

```yaml
storage:
  postgres:
    postgres_db:
      username: ...
      password: ...
      hostname: ...
      db_name: ...
      port: 5432
```



## 케이크 제작 파이프라인 예제

<details>
<summary>케이크 제작 파이프라인 구조 및 구현</summary>

케이크 제작 파이프라인은 다음 단계로 구성됩니다:

1. **ingredients**: 레시피 파일에서 재료 정보 추출
2. **mixed_dough**: 추출된 재료 정보로 반죽 만들기
3. **baked_cakes**: 반죽으로 케이크 굽기
4. **cake_reports**: 케이크 생산량 분석 (multi_asset 사용)

각 단계는 이전 단계의 출력을 입력으로 받아 처리하고, 결과를 파일 시스템에 저장합니다. 또한 각 단계에서는 메타데이터를 생성하여 처리 결과를 문서화합니다.

**예제 코드:**
```python

```

# Airflow ETL SDK 技术设计文档

> 版本：v0.1（Draft）  
> 作者：Nathan + GPT-5.1  
> 目标读者：平台团队 / 数据工程团队 / 架构评审

---

## 1. 背景与目标

### 1.1 背景

公司内部已有 Airflow 作为统一的工作流调度平台，但当前数据工程师仍需要直接编写 Airflow DAG 与 Operator 来搭建 ETL 流程，存在以下问题：

- 对 Airflow 细节高度耦合，不利于平台演进（迁移到 EMR on EKS、Spark on K8s、Flink 等）。
- 同一个业务流程中混杂 **业务逻辑（Transformation）** 与 **基础设施逻辑（调度、资源、重试、日志等）**，可维护性差。
- 很难做统一的元数据治理、血缘分析、可观测性。

参考 Uber Piper 等实践，我们希望在 Airflow 之上封装一层 **企业级 ETL SDK**，提供统一的声明式 Python API，让用户只关注：

> **数据从哪里来 → 做什么转换 → 输出到哪里**

### 1.2 设计目标

1. **声明式 Python DSL**  
   - 用户通过 `@pipeline + source/transform/sink` 定义数据管道，不直接接触 Airflow DAG/Operator。

2. **IR（中间表示）+ 编译器**  
   - Pipeline 定义先转为内部 `PipelineIR`，用于元数据、治理与编译。  
   - `PipelineIR -> Airflow DAG` 由统一 compiler 负责。

3. **插件化能力（Registry Layer）**  
   - Source / Sink / Engine / Source Reader 通过注册表实现可插拔。  
   - 平台可按环境、业务线启用或禁用特定插件。

4. **统一元数据与可观测性**  
   - 所有 Pipeline 定义与运行信息写入 Metadata Store。  
   - 通过结构化日志与 metrics 暴露运行状态，方便接入现有监控系统。

5. **多租户 / 多环境支持**  
   - Pipeline 层面携带 `team / env / project / cost_center` 信息，便于授权与成本归属。

### 1.3 非目标（本版本）

- 不实现 UI 拖拽式 Authoring（后续可在 IR 之上构建）。
- 不实现流式作业全生命周期管理（先支持 batch，Flink 等实时引擎留未来扩展）。
- 不替代 Airflow，只在上层提供抽象。

---

## 2. 总体架构

### 2.1 分层视图

整体架构分为三层：

1. **Authoring / SDK 层（用户视角）**
   - 用户编写 Python 文件，使用：
     - `@pipeline` 装饰器
     - `p.read(source.xxx(...))`
     - `.transform(engine="python"|"spark"|...)`
     - `p.write(sink.xxx(...))`

2. **Compiler / Metadata 层（平台视角）**
   - `PipelineBuilder` 将 DSL 调用构建为 `PipelineIR`（节点 + 边 + 参数 + 资源）。  
   - `MetadataStore` 存储 Pipeline 定义与运行记录。  
   - `compile_pipeline_to_airflow_dag(PipelineIR) -> DAG` 将 IR 编译为 Airflow DAG。

3. **Execution 层（执行视角）**
   - Airflow Scheduler/Worker 运行由 SDK 生成的 DAG。  
   - Runtime 函数 `run_source_node / run_spark_transform / run_python_transform / run_sink_node` 在各引擎上执行任务。  
   - Source Reader 插件负责在各 Engine 上将 Source 配置 materialize 为可操作的对象（如 Spark DataFrame）。

### 2.2 目录结构（建议）

```text
repo_root/
  aiflow_sdk/
    __init__.py
    ir.py               # PipelineIR / NodeIR
    builder.py          # PipelineBuilder / DatasetHandle
    dsl.py              # 用户 DSL API: pipeline/source/sink/resources
    plugins.py          # registry & plugin 接口（source/sink/engine/source_reader）
    builtin_plugins.py  # onelake/onestream/feature_store/spark/python 等官方插件
    discovery.py        # 从 user_repo 发现所有 pipeline
    compiler.py         # IR -> Airflow DAG
    runtime.py          # runtime 执行逻辑（Spark/Python/Sink）
    metadata.py         # MetadataStore 接口与默认实现
    observability.py    # logging + metrics
    tenancy.py          # 多租户信息处理

  user_repo/
    pipelines/
      user_features.py  # 用户 pipeline 定义
    etl_jobs/
      enrich_orders.py  # Spark transform 逻辑
      build_user_features.py

  airflow_dags/
    generated_pipelines.py  # Airflow DAG 入口，使用 discovery + compiler
```

---

## 3. 数据模型：IR 结构

### 3.1 NodeIR & PipelineIR（`ir.py`）

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Dict, List, Any, Optional, Tuple

@dataclass
class NodeIR:
    id: str
    type: str                 # "source" | "transform" | "sink"
    engine: Optional[str]     # "python" | "spark" | "flink" | None
    params: Dict[str, Any] = field(default_factory=dict)
    resources: Dict[str, Any] = field(default_factory=dict)
    input_ids: List[str] = field(default_factory=list)
    output_schema: Optional[Dict[str, Any]] = None  # 预留 schema 管理


@dataclass
class PipelineIR:
    id: str
    schedule: str
    owner: str
    sla_minutes: int
    tags: List[str] = field(default_factory=list)
    extra: Dict[str, Any] = field(default_factory=dict)  # team/env/project 等

    nodes: Dict[str, NodeIR] = field(default_factory=dict)
    edges: List[Tuple[str, str]] = field(default_factory=list)

    def add_node(self, node: NodeIR):
        if node.id in self.nodes:
            raise ValueError(f"Duplicate node id: {node.id}")
        self.nodes[node.id] = node

    def add_edge(self, src_id: str, dst_id: str):
        self.edges.append((src_id, dst_id))
```

---

## 4. Authoring 层：DSL 与 Builder

### 4.1 PipelineBuilder & DatasetHandle（`builder.py`）

```python
from __future__ import annotations
from typing import Dict, Any, List
from .ir import PipelineIR, NodeIR


class DatasetHandle:
    """逻辑数据集句柄，用于链式 transform / write。"""

    def __init__(self, pipeline_builder: "PipelineBuilder", node_id: str):
        self._builder = pipeline_builder
        self._node_id = node_id

    @property
    def node_id(self) -> str:
        return self._node_id

    def transform(self,
                  id: str,
                  engine: str,
                  entrypoint: str | None = None,
                  fn: str | None = None,
                  resources: Dict[str, Any] | None = None,
                  **kwargs) -> "DatasetHandle":
        resources = resources or {}
        node = NodeIR(
            id=id,
            type="transform",
            engine=engine,
            params={
                "entrypoint": entrypoint,
                "fn": fn,
                **kwargs,
            },
            resources=resources,
            input_ids=[self._node_id],
        )
        self._builder._pipeline_ir.add_node(node)
        self._builder._pipeline_ir.add_edge(self._node_id, id)
        return DatasetHandle(self._builder, id)


class PipelineBuilder:
    """在 pipeline 函数执行期间，收集拓扑和节点信息。"""

    def __init__(self,
                 pipeline_id: str,
                 schedule: str,
                 owner: str,
                 sla_minutes: int,
                 tags: List[str] | None = None,
                 extra: Dict[str, Any] | None = None):
        self._pipeline_ir = PipelineIR(
            id=pipeline_id,
            schedule=schedule,
            owner=owner,
            sla_minutes=sla_minutes,
            tags=tags or [],
            extra=extra or {},
        )
        self.vars: Dict[str, Any] = {}

    @property
    def ir(self) -> PipelineIR:
        return self._pipeline_ir

    def read(self, source_def: Dict[str, Any]) -> DatasetHandle:
        node_id = source_def["id"]
        node = NodeIR(
            id=node_id,
            type="source",
            engine=None,
            params=source_def,
        )
        self._pipeline_ir.add_node(node)
        return DatasetHandle(self, node_id)

    def write(self, dataset: DatasetHandle, sink_def: Dict[str, Any]):
        node_id = sink_def["id"]
        node = NodeIR(
            id=node_id,
            type="sink",
            engine=None,
            params=sink_def,
            input_ids=[dataset.node_id],
        )
        self._pipeline_ir.add_node(node)
        self._pipeline_ir.add_edge(dataset.node_id, node_id)
```

### 4.2 DSL API（`dsl.py`）

#### 4.2.1 插件化的 source/sink/resources

```python
from __future__ import annotations
from typing import Callable, Any, Dict
from .builder import PipelineBuilder
from .plugins import get_source_builder, get_sink_builder, get_engine_builder
from . import builtin_plugins  # noqa: F401
from .tenancy import normalize_tenant_tags


class source:
    @staticmethod
    def onelake(**kwargs):
        return get_source_builder("onelake")(**kwargs)

    @staticmethod
    def onestream(**kwargs):
        return get_source_builder("onestream")(**kwargs)


class sink:
    @staticmethod
    def onelake(**kwargs):
        return get_sink_builder("onelake")(**kwargs)

    @staticmethod
    def feature_store(**kwargs):
        return get_sink_builder("feature_store")(**kwargs)


class resources:
    @staticmethod
    def spark(**kwargs):
        return get_engine_builder("spark")(**kwargs)

    @staticmethod
    def python(**kwargs):
        return get_engine_builder("python")(**kwargs)
```

#### 4.2.2 pipeline 装饰器

```python
def pipeline(id: str,
             schedule: str,
             owner: str,
             sla_minutes: int = 60,
             tags: list[str] | None = None,
             team: str | None = None,
             env: str | None = None,
             project: str | None = None,
             cost_center: str | None = None,
             **extra):

    def decorator(fn: Callable[[PipelineBuilder], Any]):
        def wrapper() -> "PipelineIR":
            builder = PipelineBuilder(
                pipeline_id=id,
                schedule=schedule,
                owner=owner,
                sla_minutes=sla_minutes,
                tags=tags,
                extra={
                    "team": team,
                    "env": env,
                    "project": project,
                    "cost_center": cost_center,
                    **extra,
                },
            )
            fn(builder)
            ir = builder.ir
            normalize_tenant_tags(ir)
            return ir

        wrapper._pipeline_id = id
        wrapper._pipeline_fn = fn
        return wrapper

    return decorator
```

### 4.3 用户示例：定义一个批处理 Feature Pipeline

```python
# user_repo/pipelines/user_features.py
from aiflow_sdk.dsl import pipeline, source, sink, resources


@pipeline(
    id="daily_user_feature_pipeline",
    schedule="0 3 * * *",
    owner="team_feature",
    team="feature-platform",
    env="prod",
    project="feature-store",
)
def daily_user_feature(p):
    raw_orders = p.read(
        source.onelake(
            id="orders_onelake",
            table="raw.sales_orders",
            format="delta",
            partitions={"dt": "{{ ds }}"},
        )
    )

    enriched_orders = raw_orders.transform(
        id="enrich_orders",
        engine="spark",
        entrypoint="etl_jobs.enrich_orders:main",
        resources=resources.spark(
            cluster="emr-feature-prod",
            driver_memory="4g",
            executor_memory="8g",
            executor_cores=4,
        ),
        output_table="tmp.enriched_orders_daily",
    )

    user_features = enriched_orders.transform(
        id="build_user_features",
        engine="python",
        fn="etl_jobs.build_user_features:main",
        resources=resources.python(),
    )

    p.write(
        user_features,
        sink.onelake(
            id="user_features_onelake",
            table="feature_store.user_features_daily",
            mode="overwrite",
            partition_by=["dt"],
        ),
    )
```

---

## 5. Registry Layer：插件化设计

### 5.1 Plugin 注册表（`plugins.py`）

```python
from __future__ import annotations
from typing import Callable, Dict, Any
from pyspark.sql import SparkSession, DataFrame

# --- Source / Sink / Engine builder ---

SourceBuilder = Callable[..., Dict[str, Any]]
SinkBuilder = Callable[..., Dict[str, Any]]
EngineConfigBuilder = Callable[..., Dict[str, Any]]

SOURCE_REGISTRY: Dict[str, SourceBuilder] = {}
SINK_REGISTRY: Dict[str, SinkBuilder] = {}
ENGINE_REGISTRY: Dict[str, EngineConfigBuilder] = {}


def register_source(name: str):
    def decorator(fn: SourceBuilder) -> SourceBuilder:
        if name in SOURCE_REGISTRY:
            raise ValueError(f"Source '{name}' already registered")
        SOURCE_REGISTRY[name] = fn
        return fn
    return decorator


def register_sink(name: str):
    def decorator(fn: SinkBuilder) -> SinkBuilder:
        if name in SINK_REGISTRY:
            raise ValueError(f"Sink '{name}' already registered")
        SINK_REGISTRY[name] = fn
        return fn
    return decorator


def register_engine(name: str):
    def decorator(fn: EngineConfigBuilder) -> EngineConfigBuilder:
        if name in ENGINE_REGISTRY:
            raise ValueError(f"Engine '{name}' already registered")
        ENGINE_REGISTRY[name] = fn
        return fn
    return decorator


def get_source_builder(name: str) -> SourceBuilder:
    return SOURCE_REGISTRY[name]


def get_sink_builder(name: str) -> SinkBuilder:
    return SINK_REGISTRY[name]


def get_engine_builder(name: str) -> EngineConfigBuilder:
    return ENGINE_REGISTRY[name]


# --- Spark Source Reader registry ---

SparkSourceReader = Callable[[SparkSession, Dict[str, Any], Dict[str, Any]], DataFrame]
SPARK_SOURCE_READERS: Dict[tuple[str, str], SparkSourceReader] = {}


def register_spark_source_reader(kind: str):
    def decorator(fn: SparkSourceReader) -> SparkSourceReader:
        key = ("spark", kind)
        if key in SPARK_SOURCE_READERS:
            raise ValueError(f"Spark reader for source kind '{kind}' already registered")
        SPARK_SOURCE_READERS[key] = fn
        return fn
    return decorator


def get_spark_source_reader(kind: str) -> SparkSourceReader:
    key = ("spark", kind)
    return SPARK_SOURCE_READERS[key]
```

### 5.2 内置插件示例（`builtin_plugins.py`）

```python
from __future__ import annotations
from typing import Any, Dict, List
from pyspark.sql import SparkSession, DataFrame
from .plugins import (
    register_source,
    register_sink,
    register_engine,
    register_spark_source_reader,
)


@register_source("onelake")
def onelake_source(
    id: str,
    table: str,
    format: str = "delta",
    partitions: Dict[str, Any] | None = None,
    **kwargs,
) -> Dict[str, Any]:
    return {
        "id": id,
        "kind": "onelake",
        "table": table,
        "format": format,
        "partitions": partitions or {},
        "extra": kwargs,
    }


@register_sink("onelake")
def onelake_sink(
    id: str,
    table: str,
    mode: str = "overwrite",
    partition_by: List[str] | None = None,
    **kwargs,
) -> Dict[str, Any]:
    return {
        "id": id,
        "kind": "onelake",
        "table": table,
        "mode": mode,
        "partition_by": partition_by or [],
        "extra": kwargs,
    }


@register_sink("feature_store")
def feature_store_sink(
    id: str,
    feature_group: str,
    mode: str = "upsert",
    **kwargs,
) -> Dict[str, Any]:
    return {
        "id": id,
        "kind": "feature_store",
        "feature_group": feature_group,
        "mode": mode,
        "extra": kwargs,
    }


@register_engine("spark")
def spark_engine(
    cluster: str,
    driver_memory: str = "2g",
    executor_memory: str = "4g",
    executor_cores: int = 2,
    **kwargs,
) -> Dict[str, Any]:
    return {
        "engine": "spark",
        "cluster": cluster,
        "driver_memory": driver_memory,
        "executor_memory": executor_memory,
        "executor_cores": executor_cores,
        "extra": kwargs,
    }


@register_engine("python")
def python_engine(**kwargs) -> Dict[str, Any]:
    return {"engine": "python"}


@register_spark_source_reader("onelake")
def read_onelake_source(
    spark: SparkSession,
    params: Dict[str, Any],
    context: Dict[str, Any],
) -> DataFrame:
    table = params["table"]
    partitions = params.get("partitions") or {}
    execution_date = context.get("ds") or context.get("execution_date")

    df = spark.table(table)
    for k, v in partitions.items():
        if isinstance(v, str) and "{{ ds }}" in v and execution_date:
            value = str(execution_date)[:10]
        else:
            value = v
        df = df.where(f"{k} = '{value}'")
    return df
```

### 5.3 用户自定义 Source 的扩展路径

用户希望新增一个内部 Kafka Source：

```python
# my_team_plugins.py
from typing import Dict, Any
from pyspark.sql import SparkSession, DataFrame
from aiflow_sdk.plugins import register_source, register_spark_source_reader


@register_source("internal_kafka")
def internal_kafka_source(
    id: str,
    topic: str,
    bootstrap_servers: str,
    starting_offsets: str = "latest",
    **kwargs,
) -> Dict[str, Any]:
    return {
        "id": id,
        "kind": "internal_kafka",
        "topic": topic,
        "bootstrap_servers": bootstrap_servers,
        "starting_offsets": starting_offsets,
        "extra": kwargs,
    }


@register_spark_source_reader("internal_kafka")
def read_internal_kafka(
    spark: SparkSession,
    params: Dict[str, Any],
    context: Dict[str, Any],
) -> DataFrame:
    topic = params["topic"]
    bootstrap = params["bootstrap_servers"]
    starting_offsets = params.get("starting_offsets", "latest")

    df = (
        spark.read
        .format("kafka")
        .option("kafka.bootstrap.servers", bootstrap)
        .option("subscribe", topic)
        .option("startingOffsets", starting_offsets)
        .load()
    )
    return df
```

在用户的 pipeline 中即可直接引用 `kind="internal_kafka"` 的 source，并由 Spark transform 自动通过 registry 调用正确的 reader。

---

## 6. Compiler 层：IR -> Airflow DAG

### 6.1 Discovery：发现所有 Pipeline（`discovery.py`）

```python
import importlib
import pkgutil
import pathlib
from typing import List
from .ir import PipelineIR
from .metadata import MetadataStore, InMemoryMetadataStore

METADATA_STORE: MetadataStore = InMemoryMetadataStore()


def discover_pipelines(package_name: str) -> List[PipelineIR]:
    pkg = importlib.import_module(package_name)
    package_path = pathlib.Path(pkg.__file__).parent

    pipeline_irs: List[PipelineIR] = []

    for _, module_name, _ in pkgutil.iter_modules([str(package_path)]):
        full_name = f"{package_name}.{module_name}"
        module = importlib.import_module(full_name)

        for attr_name in dir(module):
            attr = getattr(module, attr_name)
            if callable(attr) and hasattr(attr, "_pipeline_id"):
                ir = attr()  # wrapper -> PipelineIR
                pipeline_irs.append(ir)
                METADATA_STORE.register_or_update_pipeline(ir)

    return pipeline_irs
```

### 6.2 Compiler：从 IR 生成 DAG（`compiler.py`）

```python
from __future__ import annotations
from datetime import datetime, timedelta
from typing import Dict
from airflow import DAG
from airflow.operators.python import PythonOperator

from .ir import PipelineIR, NodeIR
from .runtime import (
    run_source_node,
    run_python_transform,
    run_spark_transform,
    run_sink_node,
)


DEFAULT_START_DATE = datetime(2024, 1, 1)


def _make_default_args(pipeline_ir: PipelineIR) -> Dict:
    return {
        "owner": pipeline_ir.owner,
        "retries": 1,
        "retry_delay": timedelta(minutes=5),
        "depends_on_past": False,
    }


def compile_pipeline_to_airflow_dag(pipeline_ir: PipelineIR) -> DAG:
    dag = DAG(
        dag_id=pipeline_ir.id,
        schedule_interval=pipeline_ir.schedule,
        start_date=DEFAULT_START_DATE,
        catchup=False,
        default_args=_make_default_args(pipeline_ir),
        tags=pipeline_ir.tags,
    )

    node_to_task: Dict[str, PythonOperator] = {}

    for node_id, node in pipeline_ir.nodes.items():
        if node.type == "source":
            task = PythonOperator(
                task_id=node_id,
                python_callable=run_source_node,
                op_kwargs={"node": node},
                dag=dag,
            )
        elif node.type == "transform":
            if node.engine == "spark":
                input_nodes = [pipeline_ir.nodes[i] for i in node.input_ids]
                task = PythonOperator(
                    task_id=node_id,
                    python_callable=run_spark_transform,
                    op_kwargs={"node": node, "input_nodes": input_nodes},
                    dag=dag,
                )
            elif node.engine == "python":
                task = PythonOperator(
                    task_id=node_id,
                    python_callable=run_python_transform,
                    op_kwargs={"node": node},
                    dag=dag,
                )
            else:
                raise ValueError(f"Unknown transform engine {node.engine}")
        elif node.type == "sink":
            task = PythonOperator(
                task_id=node_id,
                python_callable=run_sink_node,
                op_kwargs={"node": node},
                dag=dag,
            )
        else:
            raise ValueError(f"Unknown node type {node.type}")

        node_to_task[node_id] = task

    for src_id, dst_id in pipeline_ir.edges:
        node_to_task[src_id] >> node_to_task[dst_id]

    return dag
```

### 6.3 Airflow DAG 入口

```python
# airflow_dags/generated_pipelines.py
from aiflow_sdk.discovery import discover_pipelines
from aiflow_sdk.compiler import compile_pipeline_to_airflow_dag

for pipeline_ir in discover_pipelines("user_repo.pipelines"):
    dag = compile_pipeline_to_airflow_dag(pipeline_ir)
    globals()[dag.dag_id] = dag
```

---

## 7. Runtime：Spark Transform & Source Reader

### 7.1 可观测性与 Metadata 集成（`observability.py` + `metadata.py`）

略去部分实现，核心点：

- `structured_log(level, msg, **fields)` 输出 JSON 结构日志。  
- `MetricsClient` 提供 `incr/observe` 方法，后续可接 Prometheus 等。  
- `MetadataStore` 记录 pipeline 定义与 node run 状态。

### 7.2 Spark Transform Runtime（`runtime.py`）

```python
from __future__ import annotations
from typing import Any, Dict, List
from pyspark.sql import SparkSession, DataFrame

from .ir import NodeIR
from .plugins import get_spark_source_reader
from .metadata import MetadataStore, InMemoryMetadataStore
from .observability import structured_log, track_duration

METADATA_STORE: MetadataStore = InMemoryMetadataStore()


def _import_entrypoint(path: str):
    module_path, fn_name = path.split(":")
    mod = __import__(module_path, fromlist=[fn_name])
    return getattr(mod, fn_name)


def _get_pipeline_and_run(context: Dict[str, Any]) -> tuple[str, str]:
    dag_id = context.get("dag").dag_id if context.get("dag") else context.get("dag_id")
    ti = context.get("ti")
    run_id = ti.run_id if ti else context.get("run_id")
    return dag_id, run_id


def _build_input_dataframes(
    spark: SparkSession,
    input_nodes: List[NodeIR],
    context: Dict[str, Any],
) -> Dict[str, DataFrame]:
    dfs: Dict[str, DataFrame] = {}

    for upstream in input_nodes:
        if upstream.type != "source":
            continue
        kind = upstream.params.get("kind")
        reader = get_spark_source_reader(kind)
        df = reader(spark, upstream.params, context)
        dfs[upstream.id] = df

    return dfs


def run_spark_transform(node: NodeIR, input_nodes: List[NodeIR], **context: Any):
    pipeline_id, run_id = _get_pipeline_and_run(context)

    METADATA_STORE.record_node_run_start(
        pipeline_id=pipeline_id,
        run_id=run_id,
        node_id=node.id,
        extra={"engine": node.engine, "params": node.params},
    )

    structured_log(
        "info",
        "spark_transform_start",
        pipeline_id=pipeline_id,
        run_id=run_id,
        node_id=node.id,
        engine=node.engine,
    )

    with track_duration(
        "node_runtime_seconds",
        pipeline_id=pipeline_id,
        node_id=node.id,
        engine=node.engine or "unknown",
    ):
        try:
            spark = (
                SparkSession.builder
                .appName(f"{pipeline_id}:{node.id}")
                .getOrCreate()
            )

            inputs = _build_input_dataframes(spark, input_nodes, context)

            entrypoint = node.params.get("entrypoint")
            if not entrypoint:
                raise ValueError(f"Spark transform node {node.id} missing 'entrypoint'")

            user_fn = _import_entrypoint(entrypoint)

            result_df: DataFrame = user_fn(
                spark=spark,
                inputs=inputs,
                params=node.params,
            )

            output_table = node.params.get("output_table")
            output_path = node.params.get("output_path")

            if output_table:
                result_df.write.mode("overwrite").saveAsTable(output_table)
            elif output_path:
                result_df.write.mode("overwrite").format(
                    node.params.get("output_format", "delta")
                ).save(output_path)

            structured_log(
                "info",
                "spark_transform_success",
                pipeline_id=pipeline_id,
                run_id=run_id,
                node_id=node.id,
            )
            METADATA_STORE.record_node_run_end(
                pipeline_id=pipeline_id,
                run_id=run_id,
                node_id=node.id,
                status="SUCCESS",
            )
        except Exception as e:
            structured_log(
                "error",
                "spark_transform_failed",
                pipeline_id=pipeline_id,
                run_id=run_id,
                node_id=node.id,
                error=str(e),
            )
            METADATA_STORE.record_node_run_end(
                pipeline_id=pipeline_id,
                run_id=run_id,
                node_id=node.id,
                status="FAILED",
                error=str(e),
            )
            raise
        finally:
            try:
                spark.stop()
            except Exception:
                pass
```

### 7.3 用户侧 Spark Transform 函数示例

```python
# user_repo/etl_jobs/enrich_orders.py
from typing import Dict
from pyspark.sql import SparkSession, DataFrame
from pyspark.sql import functions as F


def main(
    spark: SparkSession,
    inputs: Dict[str, DataFrame],
    params: Dict,
) -> DataFrame:
    orders_df = inputs["orders_onelake"]

    result_df = (
        orders_df
        .groupBy("user_id")
        .agg(
            F.count("*").alias("order_cnt_30d"),
            F.sum("amount").alias("order_amt_30d"),
        )
    )

    return result_df
```

---

## 8. 多租户与治理

- Pipeline 层面通过 `extra` 携带 `team/env/project/cost_center` 等信息。  
- Tenancy 工具将这些转为 tag（如 `team:feature-platform`），方便 Airflow UI filter 与监控系统按维度聚合。  
- MetadataStore 可以根据 tenant 字段实现权限过滤与成本归属统计。


---

## 9. 未来扩展方向

1. **UI Authoring & Graph Visualization**  
   - 在 IR 基础上提供 UI 画 DAG，生成 Python DSL 或直接 IR。

2. **Flink / Streaming 支持**  
   - 引入 `engine="flink"`，并设计对应的 Source Reader / Sink 插件。

3. **更强大的 Metadata & Lineage 集成**  
   - 将 `PipelineIR` 与数据 catalog（如 Exchange / DataHub / OpenLineage）打通。

4. **安全与权限模型**  
   - 基于 tenant 信息控制哪些 team 可以使用哪些 Source/Engine 插件。

5. **测试与本地运行工具**  
   - 提供 `aiflow local run`/`aiflow dry-run` 命令，在本地模拟运行单节点或子图，提升开发体验。

---

## 10. 总结

该 Airflow ETL SDK 通过：

- **声明式 Python DSL** 把用户从 Airflow DAG 细节中解耦；
- **IR + Compiler** 允许平台统一治理与演进；
- **Registry Layer** 支撑可插拔的 Source / Engine / Sink / Reader；
- **Metadata + Observability** 提供企业级可观测与合规能力；
- **多租户字段** 支持跨团队、跨环境、跨项目的统一平台运营；

为公司构建一个可扩展、可治理、面向未来的企业级 ETL 平台。


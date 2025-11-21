## 1. 总体架构（借鉴 Uber Piper 的思路）

参考 Uber：他们最终收敛到 **Airflow-based 中央多租户平台 + Python DSL + 元数据序列化组件**，把“元数据表示”和“可执行 workflow”分离出来，用于提高可靠性和可扩展性。[Uber+1](https://www.uber.com/blog/managing-data-workflows-at-scale/?utm_source=chatgpt.com)

你可以设计成三层：

1. **Authoring / SDK 层（用户看到的）**
    - 提供 Python 包：`aiflow_sdk`（名字随便）
    - 用户只写：
        - `pipeline`（调度 + tag + owner）
        - 一个或多个 `source()`（OneLake/OneStream/table/stream）
        - 若干 `transform()`（Python / Spark on EMR / Flink job）
        - 一个或多个 `sink()`（OneLake / OneStream / Feature Store / 其他）
    - 所有资源、重试、并发、通知、tag 都是参数化字段，不暴露 Airflow Operator 细节。
2. **Compiler / Metadata 层（你写的控制面组件）**
    - 负责：
        - import 用户的 pipeline 函数，build 出一个**内部 IR**（节点 + 边 + I/O schema + 资源需求）。
        - 把 IR：
            - 一方面 **序列化存入元数据服务**（做审计/治理/可视化）
            - 一方面 **编译成 Airflow DAG**（真正执行）。
    - 这个层类似 Uber 文章里提到的 “workflow serializer”，把 DSL 拆成：
        - “元数据表示”——给 scheduler/UI 用
        - “可执行 workflow”——给 worker 执行。[Uber](https://www.uber.com/blog/managing-data-workflows-at-scale/)
3. **Execution 层（Airflow + 计算集群）**
    - Airflow Scheduler + Workers（部署在 EKS / VM）
    - Worker task 内部才启动：
        - Python 本地任务
        - Spark on EMR / Spark on K8s
        - Flink job（提交到 Flink cluster）
    - 所有这些对用户都通过 SDK 的 `engine="spark"` / `engine="flink"` 抽象掉。

> 关键原则：系统代码和用户代码隔离，scheduler / web 不直接跑用户代码，只消费你的 IR/metadata（借鉴 Piper 的经验教训）。Uber
> 

---

## 2. 用户体验 & 声明式 Python API 设计

先给一个“纯 UX” 示例，用户眼里就这么点东西：

```python
# user_repo/pipelines/user_features.py
from aiflow_sdk import pipeline, source, transform, sink, resources

@pipeline(
    id="daily_user_feature_pipeline",
    schedule="0 3 * * *",
    owner="team_feature",
    sla_minutes=60,
    tags=["feature-store", "batch"],
)
def daily_user_feature(p):
    # 1. 数据从哪里来
    raw_orders = p.read(
        source.onelake(
            id="orders_onelake",
            table="raw.sales_orders",
            format="delta",
            partitions={"dt": p.vars["run_date"]},
        )
    )

    # 2. 做什么转换（EMR 上跑 Spark）
    enriched_orders = raw_orders.transform(
        id="enrich_orders",
        engine="spark",
        entrypoint="etl_jobs.enrich_orders",   # module:function
        resources=resources.spark(
            cluster="emr-feature-prod",
            driver_memory="4g",
            executor_memory="8g",
            executor_cores=4,
        ),
    )

    # 3. 再做一层轻量 Python 转换（在 Airflow Worker 上跑）
    user_features = enriched_orders.transform(
        id="build_user_features",
        engine="python",
        fn="etl_jobs.build_user_features_local",
    )

    # 4. 输出到哪里（OneLake feature table）
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

### 2.1 核心概念

对用户暴露的概念尽量少：

- `pipeline(...)`：一个 ETL pipeline + 调度信息。
- `source.xxx(...)`：数据源（OneLake、OneStream、JDBC…）。
- `Dataset` 对象：`p.read(...)` 返回的逻辑数据集，只能做 `transform` / `write`，不让用户随便乱搞。
- `.transform(...)`：在不同执行引擎上做转换：
    - `engine="python"` → 轻量任务，直接在 Airflow worker 上跑。
    - `engine="spark"` → 提交 Spark job（EMR / Spark on K8s）。
    - `engine="flink"` → 提交 Flink job（实时场景）。
- `sink.xxx(...)`：输出到 OneLake / OneStream / Feature Store / 其他下游。

用户说的需求完全对应：

- **“数据从哪里来”** → `source.*`
- **“怎么做转换”** → `transform(...)`
- **“输出去哪里”** → `sink.*`
- **“可扩展”** → 提供插件式 API（下面讲）。

### 2.2 “声明式”的含义

虽然写的是 Python，但你可以限制 DSL 让它更“声明式”：

- 不让用户在 pipeline 函数里随便 `if/for`；如果一定要动态生成节点，可提供工具：
    
    ```python
    for region in ["us", "eu", "apac"]:
        p.branch_for_region(region, ...)
    
    ```
    
- 禁止用户直接操作 Airflow DAG / Operator。
- 所有转换必须通过 `.transform()` 声明，所有 IO 必须通过 `source` / `sink` 声明。
- 这些调用背后仅仅是在构建一棵 **有向图 IR**，而不是立刻执行任务。

---

## 3. SDK 在 Airflow 上的实现思路

### 3.1 IR 结构（内部，不暴露给用户）

你可以在 SDK 里定义一个简单 IR：

```python
class PipelineIR:
    id: str
    schedule: str
    owner: str
    tags: list[str]
    nodes: dict[str, NodeIR]   # 节点：source / transform / sink
    edges: list[tuple[str, str]]  # 邻接关系
    metadata: dict              # 审计用

class NodeIR:
    id: str
    type: str   # "source" | "transform" | "sink"
    engine: str | None
    params: dict
    resources: dict
    input_ids: list[str]
    output_schema: dict | None

```

`pipeline` decorator 做的事情：

1. 包装原始函数：在执行时传入一个 `PipelineBuilder`。
2. 用户写的 `p.read/transform/write` 只是往 `PipelineBuilder` 里 append 节点。
3. 函数执行完，Builder 里就有完整的 `PipelineIR`。

### 3.2 从 IR 编译到 Airflow DAG

写一个 compiler，例如：

```python
# aiflow_sdk/compiler.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.amazon.aws.operators.emr import EmrAddStepsOperator
# ... 各种 operator

def compile_pipeline_to_airflow_dag(pipeline_ir: PipelineIR) -> DAG:
    dag = DAG(
        dag_id=pipeline_ir.id,
        schedule_interval=pipeline_ir.schedule,
        default_args={...},
        tags=pipeline_ir.tags,
    )

    node_to_task = {}

    for node_id, node in pipeline_ir.nodes.items():
        if node.type == "source":
            # 源一般不需要真正 Operator，可以用 Dummy + XCom/meta
            task = PythonOperator(
                task_id=node_id,
                python_callable=_materialize_source,
                op_kwargs={"node": node},
                dag=dag,
            )
        elif node.type == "transform" and node.engine == "spark":
            task = EmrAddStepsOperator(
                task_id=node_id,
                job_flow_id=node.resources["cluster"],
                steps=_build_emr_steps(node),
                dag=dag,
            )
        elif node.type == "transform" and node.engine == "python":
            task = PythonOperator(
                task_id=node_id,
                python_callable=_run_python_transform,
                op_kwargs={"node": node},
                dag=dag,
            )
        elif node.type == "sink":
            task = PythonOperator(
                task_id=node_id,
                python_callable=_write_sink,
                op_kwargs={"node": node},
                dag=dag,
            )
        else:
            raise ValueError(f"Unknown node type {node.type}")

        node_to_task[node_id] = task

    # 建立依赖关系
    for src_id, dst_id in pipeline_ir.edges:
        node_to_task[src_id] >> node_to_task[dst_id]

    return dag

```

常见部署方式：

- 在 Airflow 安装里有一个固定的 “DAG 生成入口”：
    
    ```python
    # airflow_dags/autogen.py
    from aiflow_sdk.discovery import discover_pipelines
    from aiflow_sdk.compiler import compile_pipeline_to_airflow_dag
    
    for pipeline_ir in discover_pipelines("/user_repo"):
        globals()[pipeline_ir.id] = compile_pipeline_to_airflow_dag(pipeline_ir)
    
    ```
    
- Airflow 启动/重载 DAG 时，会 import 这个文件 → 自动生成所有 DAG。

### 3.3 “没有 spec 层”的实现方式

- 对用户来说：**只有 Python DSL**（`pipeline` 函数 + `source/transform/sink`）。
- 内部你还是会有一个 IR / 元数据（可以存在 DB / S3 / OneLake），但不要求用户接触或编辑。
- 这和 Uber 把 DSL 拆成“序列化元数据 + fully instantiated workflow”的模式非常像，只不过你把“元数据”视为完全内部实现细节。[Uber](https://www.uber.com/blog/managing-data-workflows-at-scale/)

---

## 4. 企业级特性 & 可扩展设计

### 4.1 插件体系（Source / Transform / Sink 可扩展）

设计一套注册机制：

```python
# aiflow_sdk/plugins.py
SOURCE_REGISTRY = {}
SINK_REGISTRY = {}
ENGINE_REGISTRY = {}

def register_source(name):
    ...

def register_engine(name):
    ...

# 用户 / 平台团队都可以扩展：
@register_source("onelake")
def onelake_source(**kwargs) -> SourceDefinition:
    ...

@register_engine("spark")
def spark_engine(node: NodeIR):
    # 返回适配 EMR or Spark on K8s 的执行信息
    ...

```

这样以后你要支持：

- 新的数据源（内部某个 REST service、Snowflake…）
- 新的执行引擎（Flink、EMR Serverless…）

都可以通过新增插件实现，不改 DSL 形态。

### 4.2 元数据 / 审计 / 治理

借鉴 Uber 的做法，把所有东西写进 metadata store：[Uber](https://www.uber.com/blog/managing-data-workflows-at-scale/)

- Pipeline 级别：
    - owner、tags、schedule、SLA
    - 版本（git commit）、创建人、变更记录
- 节点级别：
    - 输入/输出数据集（source table / sink table）
    - engine、资源配置、schema 信息
- 运行级别（run-level）：
    - 状态（成功/失败）、开始/结束时间、耗时
    - 任务级别 log link、错误码
- 这些信息可以用来：
    - 做 data lineage（从源表到 sink / feature store）
    - 权限控制（哪些 team 可读/可写某 pipeline / 数据）
    - 审计（谁改了连接配置/变量）

### 4.3 多租户 & 中央部署

也可以直接借用 Piper 的经验：

- **中央多租户 Airflow 平台**，而不是每个组自己搭一套（便于升级 & 运维）。[Uber](https://www.uber.com/blog/managing-data-workflows-at-scale/?utm_source=chatgpt.com)
- pipeline 上挂 `team`、`project`、`env` 等 tag，用于：
    - 权限隔离（UI 上只能看到自己 team 的 pipeline）
    - 成本归属（EMR/Flink 花费打到 team 上）
- Audit log：所有 UI/CLI 操作（启停 pipeline，改 schedule，更改连接）都记日志。

### 4.4 可观测性

- 所有执行路径都通过统一的 runtime lib（比如 `aiflow_sdk.runtime`）：
    - 打 structured log + trace ID（pipeline_id / run_id / node_id）
    - 发 metrics（成功率、延迟、数据量、异常条数）
- 为 heavy engine（Spark / Flink）封装 adapter：
    - 统一采集 job 状态
    - 把外部系统的 job id 关联回 pipeline run id，便于排错。

---

## 5. 小结（帮你捋一下关键设计点）

1. **用户视角：**
    - 只需要关心：`source → transform → sink`，用一个声明式 Python DSL 表达。
    - 不碰 Airflow DAG、不碰 Operator、不写 YAML/JSON spec。
2. **SDK 视角：**
    - `pipeline` decorator + `PipelineBuilder` 构建一棵 IR。
    - IR 被：
        - 序列化存入元数据服务（审计/治理/可视化）。
        - 编译成 Airflow DAG（真正执行）。
3. **平台视角：**
    - 中央多租户 Airflow 部署。
    - 插件式 Source / Engine / Sink，支持 OneLake、OneStream、EMR、Flink 等。
    - 元数据 + 可观测性 + 审计 + backfill + UI（可以以后再加 visual authoring，像 Uber 那样拖拽生成 DSL）。[Uber](https://www.uber.com/blog/managing-data-workflows-at-scale/)

---

如果你愿意，下一步我可以帮你**把上面这个 DSL 固化成一个最小可行的 Python 包结构**（`aiflow_sdk/` 下的模块划分 + 更完整的代码骨架），再顺手写一段“PRD 里的 UX section 描述”。

### You said:

好

### ChatGPT said:

那我直接帮你把“最小可行版 SDK”长什么样写出来，你可以拿去改造。

下面分 3 部分：

1. 包结构怎么分
2. 关键代码骨架（DSL / IR / Compiler / Runtime）
3. 写在 PRD 里的 UX 描述文案

---

## 1. 包结构设计（`aiflow_sdk`）

建议先搞一个干净的 Python 包：

```
aiflow_sdk/
  __init__.py

  dsl.py          # 用户直接 import 的 API: pipeline/source/transform/sink/resources
  builder.py      # PipelineBuilder / DatasetHandle, 负责把 DSL 调用变成 IR
  ir.py           # PipelineIR / NodeIR 等数据结构
  plugins.py      # source / sink / engine 的注册与实现
  compiler.py     # IR -> Airflow DAG
  runtime.py      # 真正 task 运行时函数（PythonTransform / SparkTransform / SinkWriter）
  discovery.py    # 扫描用户 repo，发现所有 pipeline 并返回 IR
  config.py       # 全局配置（元数据存哪，log 怎打等）

```

用户侧项目结构可以这样：

```
user_repo/
  pipelines/
    user_features.py     # 写 pipeline 的地方（用你的 DSL）
  etl_jobs/
    enrich_orders.py
    build_user_features.py

```

Airflow 的 DAG 目录再放一个统一入口：

```
airflow_dags/
  generated_pipelines.py

```

---

## 2. 关键代码骨架

### 2.1 IR 定义（`ir.py`）

```python
# aiflow_sdk/ir.py
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Dict, List, Any, Optional, Tuple

@dataclass
class NodeIR:
    id: str
    type: str                # "source" | "transform" | "sink"
    engine: Optional[str]    # "python" | "spark" | "flink" | None (source/sink 可 None)
    params: Dict[str, Any] = field(default_factory=dict)
    resources: Dict[str, Any] = field(default_factory=dict)
    input_ids: List[str] = field(default_factory=list)
    output_schema: Optional[Dict[str, Any]] = None   # 预留做 schema 管理

@dataclass
class PipelineIR:
    id: str
    schedule: str
    owner: str
    sla_minutes: int
    tags: List[str] = field(default_factory=list)
    extra: Dict[str, Any] = field(default_factory=dict)   # 比如 env, team, cost_center

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

### 2.2 Builder / Dataset 抽象（`builder.py`）

```python
# aiflow_sdk/builder.py
from __future__ import annotations
from typing import Dict, Any, List
from .ir import PipelineIR, NodeIR

class DatasetHandle:
    """
    逻辑上的“数据集句柄”，用户看不到 DataFrame，只能拿这个对象做 transform / write。
    """
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
        """
        声明一个 transform 节点。engine 决定由谁执行（python/spark/flink...）。
        entrypoint / fn 用字符串引用具体用户代码，runtime 再 import。
        """
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
    """
    在 pipeline 函数执行时注入的对象，用于收集 Source/Transform/Sink 调用，构建 PipelineIR。
    """
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
        # 提供一个简单的变量空间，比如 run_date 等
        self.vars: Dict[str, Any] = {}

    @property
    def ir(self) -> PipelineIR:
        return self._pipeline_ir

    # === Source / Sink 抽象 ===
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

---

### 2.3 DSL：用户用的 API（`dsl.py`）

```python
# aiflow_sdk/dsl.py
from __future__ import annotations
from typing import Callable, Any, Dict
from .builder import PipelineBuilder

# 简单的 source/sink/resources 工厂，这里先用 dict，后面可以改成 dataclass
class source:
    @staticmethod
    def onelake(id: str,
                table: str,
                format: str = "delta",
                partitions: Dict[str, Any] | None = None,
                **kwargs) -> Dict[str, Any]:
        return {
            "id": id,
            "kind": "onelake",
            "table": table,
            "format": format,
            "partitions": partitions or {},
            "extra": kwargs,
        }

    @staticmethod
    def onestream(id: str, stream: str, **kwargs) -> Dict[str, Any]:
        return {
            "id": id,
            "kind": "onestream",
            "stream": stream,
            "extra": kwargs,
        }

class sink:
    @staticmethod
    def onelake(id: str,
                table: str,
                mode: str = "overwrite",
                partition_by: list[str] | None = None,
                **kwargs) -> Dict[str, Any]:
        return {
            "id": id,
            "kind": "onelake",
            "table": table,
            "mode": mode,
            "partition_by": partition_by or [],
            "extra": kwargs,
        }

    @staticmethod
    def feature_store(id: str,
                      feature_group: str,
                      mode: str = "upsert",
                      **kwargs) -> Dict[str, Any]:
        return {
            "id": id,
            "kind": "feature_store",
            "feature_group": feature_group,
            "mode": mode,
            "extra": kwargs,
        }

class resources:
    @staticmethod
    def spark(cluster: str,
              driver_memory: str = "2g",
              executor_memory: str = "4g",
              executor_cores: int = 2,
              **kwargs) -> Dict[str, Any]:
        return {
            "engine": "spark",
            "cluster": cluster,
            "driver_memory": driver_memory,
            "executor_memory": executor_memory,
            "executor_cores": executor_cores,
            "extra": kwargs,
        }

    @staticmethod
    def python(request_cpu: str = "500m",
               request_memory: str = "1Gi",
               **kwargs) -> Dict[str, Any]:
        return {
            "engine": "python",
            "request_cpu": request_cpu,
            "request_memory": request_memory,
            "extra": kwargs,
        }

def pipeline(id: str,
             schedule: str,
             owner: str,
             sla_minutes: int = 60,
             tags: list[str] | None = None,
             **extra):
    """
    装饰器：把一个“声明式编排函数”变成一个能返回 PipelineIR 的工厂。
    用户写的函数接受一个 PipelineBuilder。
    """

    def decorator(fn: Callable[[PipelineBuilder], Any]):
        def wrapper() -> "PipelineIR":
            builder = PipelineBuilder(
                pipeline_id=id,
                schedule=schedule,
                owner=owner,
                sla_minutes=sla_minutes,
                tags=tags,
                extra=extra,
            )
            # 这里可以填一些 runtime vars（比如 run_date），后面与 Airflow execution_date 绑定
            # builder.vars["run_date"] = ...

            fn(builder)  # 用户的 pipeline 函数里只能用 builder.read/transform/write
            return builder.ir

        # 把元数据挂载在函数对象上，方便 discovery
        wrapper._pipeline_id = id
        wrapper._pipeline_fn = fn
        return wrapper

    return decorator

```

---

### 2.4 用户写 pipeline 的样子（`user_repo/pipelines/user_features.py`）

```python
# user_repo/pipelines/user_features.py
from aiflow_sdk.dsl import pipeline, source, sink, resources

@pipeline(
    id="daily_user_feature_pipeline",
    schedule="0 3 * * *",
    owner="team_feature",
    sla_minutes=60,
    tags=["feature-store", "batch"],
)
def daily_user_feature(p):
    # 1. 数据从哪里来
    raw_orders = p.read(
        source.onelake(
            id="orders_onelake",
            table="raw.sales_orders",
            format="delta",
            partitions={"dt": "{{ ds }}"},  # 也可以用 p.vars 方式
        )
    )

    # 2. Spark 上做重 transform
    enriched_orders = raw_orders.transform(
        id="enrich_orders",
        engine="spark",
        entrypoint="etl_jobs.enrich_orders:main",  # module:function 格式
        resources=resources.spark(
            cluster="emr-feature-prod",
            driver_memory="4g",
            executor_memory="8g",
            executor_cores=4,
        ),
    )

    # 3. Python 上做轻量 transform
    user_features = enriched_orders.transform(
        id="build_user_features",
        engine="python",
        fn="etl_jobs.build_user_features:main",
        resources=resources.python(),
    )

    # 4. 输出到 OneLake 的 feature 表
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

你后面要做可视化的时候，这个 pipeline 对应的 IR 里已经有：

- nodes：`orders_onelake` → `enrich_orders` → `build_user_features` → `user_features_onelake`
- edges：线性链路
- 每个 node 的 engine / resources / params 都在。

---

### 2.5 Discovery：从用户 repo 里发现 pipeline（`discovery.py`）

```python
# aiflow_sdk/discovery.py
import importlib
import pkgutil
import pathlib
from typing import List
from .ir import PipelineIR

def discover_pipelines(package_name: str) -> List[PipelineIR]:
    """
    扫描某个 package（比如 user_repo.pipelines），自动 import 其中所有模块，
    找到带有 _pipeline_id 的 wrapper 函数，并调用它生成 PipelineIR。
    """
    pkg = importlib.import_module(package_name)
    package_path = pathlib.Path(pkg.__file__).parent

    pipeline_irs: List[PipelineIR] = []

    for _, module_name, _ in pkgutil.iter_modules([str(package_path)]):
        full_name = f"{package_name}.{module_name}"
        module = importlib.import_module(full_name)

        for attr_name in dir(module):
            attr = getattr(module, attr_name)
            if callable(attr) and hasattr(attr, "_pipeline_id"):
                ir = attr()  # 调用 wrapper -> 返回 PipelineIR
                pipeline_irs.append(ir)

    return pipeline_irs

```

---

### 2.6 Compiler：IR → Airflow DAG（`compiler.py`）

```python
# aiflow_sdk/compiler.py
from __future__ import annotations
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta
from typing import Dict
from .ir import PipelineIR, NodeIR
from .runtime import (
    run_source_node,
    run_python_transform,
    run_spark_transform,
    run_sink_node,
)

DEFAULT_START_DATE = datetime(2024, 1, 1)  # 示例，实际可做成可配置

def _make_default_args(pipeline_ir: PipelineIR) -> Dict:
    return {
        "owner": pipeline_ir.owner,
        "retries": 1,
        "retry_delay": timedelta(minutes=5),
        "depends_on_past": False,
        "email_on_failure": True,
        "email": [f"{pipeline_ir.owner}@example.com"],  # TODO: 映射成真实邮箱
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
                task = PythonOperator(
                    task_id=node_id,
                    python_callable=run_spark_transform,
                    op_kwargs={"node": node},
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

---

### 2.7 Runtime：真正跑任务的地方（`runtime.py`）

这里只写骨架，具体逻辑你可以对接 OneLake/EMR/Flink：

```python
# aiflow_sdk/runtime.py
from __future__ import annotations
from typing import Any
from .ir import NodeIR

def run_source_node(node: NodeIR, **context: Any):
    """
    实际上可以只记录 metadata / 校验连通性，真正的数据读取在 Spark/下游 job 做。
    也可以在这里去 materialize 一些抽样数据，用于 debug。
    """
    # TODO: 检查 OneLake/OneStream connection 是否可用
    print(f"[SOURCE] {node.id} kind={node.params.get('kind')} table={node.params.get('table')}")

def _import_entrypoint(path: str):
    """
    "package.module:function" -> function 对象。
    """
    module_path, fn_name = path.split(":")
    mod = __import__(module_path, fromlist=[fn_name])
    return getattr(mod, fn_name)

def run_python_transform(node: NodeIR, **context: Any):
    fn_path = node.params.get("fn")
    if not fn_path:
        raise ValueError(f"Python transform {node.id} missing 'fn' param")
    fn = _import_entrypoint(fn_path)
    # 这里你可以决定传什么：可以是 (context, config)，也可以读上游输出路径
    return fn(context)

def run_spark_transform(node: NodeIR, **context: Any):
    """
    这里可以统一封装提交 EMR / Spark on K8s job 的方式：
    - 选择 cluster
    - 构造 spark-submit 命令
    - 等待 job 结束
    - 记录状态到 metadata store
    """
    entrypoint = node.params.get("entrypoint")
    cluster = node.resources.get("cluster")
    print(f"[SPARK] submitting job node={node.id} entrypoint={entrypoint} cluster={cluster}")
    # TODO: 调用 boto3 / k8s client / internal API 提交真正 job

def run_sink_node(node: NodeIR, **context: Any):
    """
    把结果写到 OneLake / Feature Store 等。
    """
    kind = node.params.get("kind")
    if kind == "onelake":
        table = node.params.get("table")
        print(f"[SINK] writing to OneLake table={table}")
    elif kind == "feature_store":
        fg = node.params.get("feature_group")
        print(f"[SINK] writing to Feature Group={fg}")
    else:
        raise ValueError(f"Unknown sink kind {kind}")

```

---

### 2.8 Airflow 入口：统一加载所有 pipeline（`airflow_dags/generated_pipelines.py`）

```python
# airflow_dags/generated_pipelines.py
from aiflow_sdk.discovery import discover_pipelines
from aiflow_sdk.compiler import compile_pipeline_to_airflow_dag

# 假设用户的 pipeline 都放在 user_repo.pipelines 这个 package 下面
for pipeline_ir in discover_pipelines("user_repo.pipelines"):
    dag = compile_pipeline_to_airflow_dag(pipeline_ir)
    globals()[dag.dag_id] = dag

```

这样一来：

- Airflow 只加载这一份 `generated_pipelines.py`。
- 你在 `user_repo/pipelines` 里新增/修改 pipeline，Airflow 重载 DAG 后就能拿到新的。

---

## 3. PRD 里可以写的 “User Experience” 描述（英文草稿）

你可以直接把这段塞进 PRD 的 **User Interface / Authoring Layer** 部分（稍微按你公司风格改一下就行）：

> Authoring Experience
> 
> 
> Data engineers author ETL pipelines using a **declarative Python DSL** on top of Airflow, instead of writing Airflow DAGs and operators directly.
> 
> A pipeline is defined in terms of three core concepts:
> 
> - **Sources** – where the data comes from (OneLake tables, streams, JDBC sources, etc.).
> - **Transforms** – how the data is processed (Python, Spark on EMR, Flink, …).
> - **Sinks** – where the data is written to (OneLake tables, feature store, streams, etc.).
> 
> For example:
> 
> ```python
> @pipeline(
>     id="daily_user_feature_pipeline",
>     schedule="0 3 * * *",
>     owner="team_feature",
>     tags=["feature-store", "batch"],
> )
> def daily_user_feature(p):
>     raw_orders = p.read(
>         source.onelake(
>             id="orders_onelake",
>             table="raw.sales_orders",
>             partitions={"dt": "{{ ds }}"},
>         )
>     )
> 
>     enriched_orders = raw_orders.transform(
>         id="enrich_orders",
>         engine="spark",
>         entrypoint="etl_jobs.enrich_orders:main",
>         resources=resources.spark(
>             cluster="emr-feature-prod",
>             executor_memory="8g",
>             executor_cores=4,
>         ),
>     )
> 
>     user_features = enriched_orders.transform(
>         id="build_user_features",
>         engine="python",
>         fn="etl_jobs.build_user_features:main",
>     )
> 
>     p.write(
>         user_features,
>         sink.onelake(
>             id="user_features_onelake",
>             table="feature_store.user_features_daily",
>             mode="overwrite",
>             partition_by=["dt"],
>         ),
>     )
> 
> ```
> 
> Under the hood, the SDK translates this declarative Python definition into:
> 
> - A **pipeline metadata representation** (nodes, edges, inputs/outputs, resources), which is stored in the metadata service for lineage, governance, and observability.
> - A **fully instantiated Airflow DAG**, which is deployed to the central Airflow cluster and uses the appropriate execution engines (Python workers, Spark on EMR, Flink, etc.) to run each transform.
> 
> This design allows users to focus on **“where the data comes from, how it is transformed, and where it goes”**, while the platform team owns the system code for scheduling, resource management, logging, and observability.
> 

---

如果你接下来想深入某一块（比如：

- **Spark / Flink plugin 设计**，或者
- **元数据 schema / lineage 模型**，或者
- **如何加一个 UI 做 drag-and-drop 再生成这个 Python DSL**），
    
    可以直接指定一个方向，我就按那个方向把细节铺开。
    

### You said:

Run spark transform那个函数实现一下给我看看，怎么从用户定义德souce 哪数据并做转换

### ChatGPT said:

Thought for 44s

好，我们把这个问题“收紧”成一个非常具体的例子：

> 目标：在 run_spark_transform 里，从用户通过 DSL 定义的 source 配置中读数据，用 Spark 做转换，再写出去。
> 

我会做两件事：

1. 稍微改一下 `compiler`，把上游的 `source` 节点也传到 `run_spark_transform`。
2. 给出一个完整的 `run_spark_transform` 实现 + 用户侧 `enrich_orders` 函数长什么样。

---

## 1. 编译时：把上游 source 节点一起传给 runtime

原来 `compile_pipeline_to_airflow_dag` 里 Spark transform 那块是这样的：

```python
elif node.type == "transform":
    if node.engine == "spark":
        task = PythonOperator(
            task_id=node_id,
            python_callable=run_spark_transform,
            op_kwargs={"node": node},
            dag=dag,
        )

```

我们改成**把上游节点（尤其是 source）也传进去**：

```python
elif node.type == "transform":
    if node.engine == "spark":
        input_nodes = [pipeline_ir.nodes[i] for i in node.input_ids]

        task = PythonOperator(
            task_id=node_id,
            python_callable=run_spark_transform,
            op_kwargs={
                "node": node,
                "input_nodes": input_nodes,
            },
            dag=dag,
        )

```

这样 `run_spark_transform` 的签名就可以是：

```python
def run_spark_transform(node: NodeIR, input_nodes: list[NodeIR], **context):
    ...

```

`input_nodes` 里至少有一个是你 DSL 定义的 `source.onelake(...)` 对应的节点，它的 `params` 里有 `table` / `partitions` 等信息。

---

## 2. 运行时：在 `run_spark_transform` 里读 source + 调用用户代码

下面是一个相对完整、但仍然简化过的实现示例：

```python
# aiflow_sdk/runtime.py
from __future__ import annotations
from typing import Any, Dict, List
from pyspark.sql import SparkSession, DataFrame
from .ir import NodeIR

def _import_entrypoint(path: str):
    """
    "package.module:function" -> function 对象。
    """
    module_path, fn_name = path.split(":")
    mod = __import__(module_path, fromlist=[fn_name])
    return getattr(mod, fn_name)

def _build_input_dataframes(
    spark: SparkSession,
    input_nodes: List[NodeIR],
    context: Dict[str, Any],
) -> Dict[str, DataFrame]:
    """
    根据上游 source 节点的配置，构造 Spark DataFrame。
    返回 dict: { node_id -> DataFrame }。
    """
    dfs: Dict[str, DataFrame] = {}

    for upstream in input_nodes:
        if upstream.type != "source":
            # 这里先只处理 source 节点，后面可以扩展支持 transform 作为输入
            continue

        kind = upstream.params.get("kind")

        # 示例：OneLake table source
        if kind == "onelake":
            table = upstream.params["table"]
            partitions = upstream.params.get("partitions") or {}

            df = spark.table(table)   # 假设已经通过 external catalog 注册了

            # 简单地用 where 过滤 partition
            # 实际实现里你可以把 "{{ ds }}" 等模板替换成 execution_date
            # context 里可以拿到 execution_date / ds
            execution_date = context.get("ds") or context.get("execution_date")

            for k, v in partitions.items():
                # 非常粗暴的模板替换示例
                if isinstance(v, str) and "{{ ds }}" in v and execution_date:
                    value = str(execution_date)[:10]
                else:
                    value = v
                df = df.where(f"{k} = '{value}'")

            dfs[upstream.id] = df

        # 你也可以加别的 source 类型，比如 onestream / jdbc：
        # elif kind == "onestream":
        #     stream = upstream.params["stream"]
        #     df = spark.read.format("kafka").option(...).load()
        #     dfs[upstream.id] = df

        else:
            raise ValueError(f"Unknown source kind: {kind}")

    return dfs

def run_spark_transform(
    node: NodeIR,
    input_nodes: List[NodeIR],
    **context: Any,
):
    """
    在 Spark 里把上游 source 的数据读出来，调用用户定义的 transform 函数。

    约定：
    - node.params["entrypoint"] 是 "package.module:function" 格式
    - 用户函数签名类似：
        def main(spark: SparkSession, inputs: dict[str, DataFrame], params: dict) -> DataFrame:
            ...
    - 写输出的策略可以有几种：
        1）transform 直接写到最终表（output_table），sink 只做 metadata；
        2）transform 写到中间路径，sink task 再读中间路径写最终表。
      这里先演示第 1 种，以示意为主。
    """
    # 1. 建 spark session
    spark = (
        SparkSession.builder
        .appName(f"{node.id}")
        # 实际上你可以在这里配置 master / dynamic allocation 等，
        # 或者这个函数本身就是在 EMR / Spark-on-K8s driver 里跑。
        .getOrCreate()
    )

    try:
        # 2. 把上游 source 解析成 DataFrame
        inputs = _build_input_dataframes(spark, input_nodes, context)

        # 3. import 用户的 transform 函数
        entrypoint = node.params.get("entrypoint")
        if not entrypoint:
            raise ValueError(f"Spark transform node {node.id} missing 'entrypoint'")

        user_fn = _import_entrypoint(entrypoint)

        # 4. 调用用户函数：传 spark + inputs + 当前节点的 params
        #    你也可以再传一点 pipeline_id / run_id 进去做 logging
        result_df: DataFrame = user_fn(
            spark=spark,
            inputs=inputs,
            params=node.params,
        )

        # 5. 写结果（示例：直接写到 output_table）
        output_table = node.params.get("output_table")
        output_path = node.params.get("output_path")

        if output_table:
            # 保存为 (managed/external) table，具体策略看你底层 catalog
            (
                result_df
                .write
                .mode("overwrite")   # 可以由 params 决定是 append/overwrite
                .saveAsTable(output_table)
            )
        elif output_path:
            (
                result_df
                .write
                .mode("overwrite")
                .format(node.params.get("output_format", "delta"))
                .save(output_path)
            )
        else:
            # 也可以什么都不写，只是由用户函数自己写
            # 这里保留一个 log
            print(f"[SPARK] node {node.id} did not specify output_table/output_path, "
                  "assuming user_fn handled the output writing")

    finally:
        spark.stop()

```

> ⚠️ 真实生产里你很可能不会在 Airflow worker 上直接跑 SparkSession.builder.getOrCreate()，
> 
> 
> 而是会：
> 
> - 使用 `spark-submit` 提交到 EMR / Spark-on-K8s
> - 或者让这个函数只负责构造 job config 交给 “Spark job launcher”
>     
>     但“如何根据 source 节点读数据并做 transform”的逻辑基本就是上面这套——只是换了执行环境。
>     

---

## 3. 用户侧 Spark transform 函数长什么样？

假设用户在 DSL 里这样写：

```python
# user_repo/pipelines/user_features.py
from aiflow_sdk.dsl import pipeline, source, sink, resources

@pipeline(
    id="daily_user_feature_pipeline",
    schedule="0 3 * * *",
    owner="team_feature",
)
def daily_user_feature(p):
    raw_orders = p.read(
        source.onelake(
            id="orders_onelake",
            table="raw.sales_orders",
            partitions={"dt": "{{ ds }}"},
        )
    )

    enriched_orders = raw_orders.transform(
        id="enrich_orders",
        engine="spark",
        entrypoint="etl_jobs.enrich_orders:main",
        resources=resources.spark(cluster="emr-feature-prod"),
        output_table="tmp.enriched_orders_daily",   # 交给 Spark 写到这个表
    )

    # 后面你可以再接一个 python transform / sink ...

```

对应的用户代码（真正的 Spark 逻辑）可以是这样：

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
    """
    inputs: { "orders_onelake": <DataFrame> }
    params: 就是 node.params（包括 output_table / 自定义参数等）
    """
    orders_df = inputs["orders_onelake"]

    # 举个例子：按 user 级别聚合一些 feature
    result_df = (
        orders_df
        .groupBy("user_id")
        .agg(
            F.count("*").alias("order_cnt_30d"),
            F.sum("amount").alias("order_amt_30d"),
        )
    )

    # 这里可以不写输出，交给 runtime 的 run_spark_transform 写；
    # 也可以自己写，然后在 params 里标记 output 已经完成。
    # 下面留给 runtime 写，所以直接 return。
    return result_df

```

这样端到端的流程就是：

1. 用户 DSL 里 `source.onelake(...)` → `NodeIR(type="source", params={"kind":"onelake", "table":...})`
2. 编译时把 `NodeIR` 和 `input_nodes` 传给 `run_spark_transform`
3. `run_spark_transform` 里用 `SparkSession` + 上游 `source` 的 `table`/`partitions` 信息构造 `DataFrame`
4. 调用用户的 `main(spark, inputs, params)` 做真正业务逻辑
5. runtime 再按约定写结果（写到 `output_table` 或 `output_path`）

---

如果你想走更“EMR 提交 job”的路线，我可以再帮你把 `run_spark_transform` 改成：

- 不直接 `getOrCreate()`，而是
- `subprocess.run(["spark-submit", "--conf", "...", "your_driver.py", "--config", json.dumps(job_config)])`

然后在 `your_driver.py` 里做上面那一套 `SparkSession -> 读 source -> 调用用户函数 -> 写输出`。

逻辑是一样的，只是 **执行位置** 和 **参数传递方式** 不同。
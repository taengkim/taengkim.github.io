---
title: "Airflow KubernetesPodOperator vs KubernetesExecutor — 무엇이 다르고 언제 쓰나"
date: 2026-06-10
categories: [Airflow, 운영]
tags: [airflow, kubernetes, pod-operator, kubernetes-executor, 운영, 인프라]
---

> **TL;DR** — `KubernetesExecutor`는 Airflow **실행 엔진** 계층이고, `KubernetesPodOperator`는 **오퍼레이터** 계층입니다. 서로 다른 계층이라 조합해서 쓸 수 있습니다. Airflow 3.x에서는 **멀티 익스큐터**가 공식 지원되어, DAG 안에서 태스크별로 CeleryExecutor와 KubernetesExecutor를 동시에 지정하는 것도 가능합니다.

---

## 1. 한 줄 정의

| | 한 줄 정의 |
|---|---|
| **KubernetesExecutor** | Airflow 스케줄러가 **모든 태스크**를 K8s Pod로 실행하는 실행 엔진 |
| **KubernetesPodOperator** | **특정 태스크** 하나가 외부 K8s Pod를 띄워 작업을 위임하는 오퍼레이터 |
| **멀티 익스큐터 (3.x)** | 하나의 Airflow 클러스터에서 **복수의 익스큐터**를 등록하고 태스크별로 선택하는 기능 |

핵심은 **계층이 다르다**는 점입니다. KubernetesExecutor는 "Airflow 자체가 어떻게 태스크를 실행하느냐"를 결정하고, KubernetesPodOperator는 "이 태스크가 무엇을 실행하느냐"를 결정합니다.

---

## 2. 작동 방식 비교

### KubernetesExecutor — 익스큐터 계층

스케줄러가 태스크마다 직접 K8s Pod를 생성합니다. Pod 이미지는 기본적으로 Airflow Worker 이미지입니다.

```
[Airflow Scheduler]
    │  Pod 생성 요청 (K8s API)
    ▼
[Kubernetes API Server]
    │
    ├── [Worker Pod] ──▶ task_a (airflow 이미지로 실행)
    ├── [Worker Pod] ──▶ task_b (airflow 이미지로 실행)
    └── [Worker Pod] ──▶ task_c (airflow 이미지로 실행)
```

Pod는 Airflow가 알아서 생성·삭제합니다. DAG 코드는 Worker Pod 안의 Airflow Task SDK가 실행합니다.

### KubernetesPodOperator — 오퍼레이터 계층

Worker(또는 스케줄러) 안에서 실행 중인 KPO 코드가 K8s API를 호출해 **별도 Pod**를 띄웁니다.

```
[Airflow Worker or Scheduler]
    │  KPO 코드 실행 중
    │  Pod 생성 요청 (K8s API)
    ▼
[Kubernetes API Server]
    │
    └── [Task Pod] ──▶ my-spark-image:3.5 실행
                       (Airflow와 무관한 이미지)
```

KPO가 띄운 Pod는 Airflow와 완전히 다른 이미지를 쓸 수 있습니다. Airflow는 Pod의 완료 여부만 감시합니다.

### 두 계층의 중첩

KubernetesExecutor + KubernetesPodOperator를 함께 쓰면 Pod가 Pod를 생성하는 구조가 됩니다.

```
[Scheduler] ──▶ [Worker Pod (Airflow 이미지)]
                    │  KPO 코드 실행
                    └──▶ [Task Pod (spark 이미지)]
```

이 구조는 완전히 유효하지만, Pod 2개 분의 기동 시간과 리소스가 필요합니다.

---

## 3. 핵심 차이점 비교

| 항목 | KubernetesExecutor | KubernetesPodOperator |
|---|---|---|
| **계층** | 실행 엔진 (Executor) | 오퍼레이터 (Operator) |
| **적용 범위** | 모든 태스크 | 이 오퍼레이터를 쓰는 태스크만 |
| **Pod 생성 주체** | Airflow 스케줄러 | 태스크 코드 (KPO) |
| **기본 이미지** | Airflow Worker 이미지 | 사용자 지정 (완전 자유) |
| **이미지 다양성** | 태스크별 오버라이드 가능 | 태스크마다 완전히 다른 이미지 |
| **로그 수집** | Pod 삭제 전 원격 스토리지 설정 필요 | KPO가 로그 fetch 후 Airflow에 전달 |
| **Airflow 의존성** | Worker Pod에 Airflow 설치 필요 | Task Pod에 Airflow 불필요 |
| **K8s 클러스터 요구** | 필수 | 필수 (단, Executor와 무관) |
| **Airflow 3.x 변경** | 멀티 익스큐터로 태스크별 지정 가능 | deferrable=True가 기본 권장 |

---

## 4. Airflow 3.x 멀티 익스큐터

Airflow 3.0부터 하나의 클러스터에서 **여러 익스큐터를 동시에 등록**하고, 태스크 단위로 어느 익스큐터로 실행할지 지정할 수 있습니다.

### 설정

```ini
# airflow.cfg
[core]
executor = CeleryExecutor,KubernetesExecutor
```

```bash
AIRFLOW__CORE__EXECUTOR=CeleryExecutor,KubernetesExecutor
```

첫 번째로 선언된 익스큐터가 기본값이 됩니다. 위 예시에서 `executor` 파라미터를 지정하지 않은 태스크는 CeleryExecutor로 실행됩니다.

### 태스크별 익스큐터 지정

```python
from airflow.sdk import dag, task
import pendulum

@dag(
    schedule="0 2 * * *",
    start_date=pendulum.datetime(2026, 1, 1, tz="UTC"),
    catchup=False,
)
def pipeline():

    @task(executor="CeleryExecutor")
    def extract():
        # 빠른 추출 — Celery Worker에서 실행
        ...

    @task(executor="CeleryExecutor")
    def transform():
        # 가벼운 변환 — Celery Worker에서 실행
        ...

    @task(executor="KubernetesExecutor")
    def train_model():
        # GPU 필요, 무거운 작업 — K8s Pod에서 실행
        ...

    extract() >> transform() >> train_model()


pipeline()
```

같은 DAG 안에서 경량 태스크는 Celery(빠른 기동), 무거운 태스크는 K8s(격리·GPU)로 자연스럽게 분리됩니다.

### KubernetesExecutor에서 이미지 오버라이드

특정 태스크에 다른 이미지나 리소스를 지정할 수 있습니다.

```python
@task(
    executor="KubernetesExecutor",
    executor_config={
        "KubernetesExecutor": {
            "image": "my-registry/custom-airflow:gpu",
            "resources": {
                "requests": {"memory": "8Gi", "cpu": "4", "nvidia.com/gpu": "1"},
                "limits":   {"memory": "16Gi", "cpu": "8", "nvidia.com/gpu": "1"},
            },
            "tolerations": [
                {"key": "nvidia.com/gpu", "operator": "Exists", "effect": "NoSchedule"}
            ],
        }
    },
)
def train_model():
    ...
```

---

## 5. KubernetesExecutor를 써야 할 때

- 모든(또는 대부분의) 태스크를 **격리된 Pod**에서 실행하고 싶다
- 상시 워커 비용을 없애고 싶다 (유휴 Pod 없음)
- 태스크가 다른 Python 패키지 환경을 필요로 한다
- K8s 클러스터를 이미 운영 중이고 인프라 팀이 있다
- Airflow 3.x 멀티 익스큐터로 익스큐터를 태스크별로 분리하려 한다

```ini
[core]
executor = KubernetesExecutor

[kubernetes]
namespace = airflow
in_cluster = True
worker_container_repository = my-registry/airflow
worker_container_tag = 3.2.0
```

---

## 6. KubernetesPodOperator를 써야 할 때

- **특정 태스크만** Airflow와 완전히 다른 환경(이미지)에서 실행해야 한다
  - 예: Spark 작업, dbt 실행, ML 학습, 레거시 Java/Go 도구
- Airflow Worker에 해당 의존성을 설치하고 싶지 않다
- Task Pod의 리소스(CPU/메모리/GPU)를 태스크별로 세밀하게 제어해야 한다
- Executor 종류에 관계없이 K8s Pod 실행이 필요하다 (CeleryExecutor 환경도 포함)

```python
from airflow.providers.cncf.kubernetes.operators.pod import KubernetesPodOperator
from airflow.sdk import dag
import pendulum

@dag(
    schedule="0 3 * * *",
    start_date=pendulum.datetime(2026, 1, 1, tz="UTC"),
    catchup=False,
)
def dbt_pipeline():

    run_dbt = KubernetesPodOperator(
        task_id="run_dbt",
        name="dbt-run",
        namespace="airflow",
        image="ghcr.io/dbt-labs/dbt-bigquery:1.8.0",
        cmds=["dbt"],
        arguments=["run", "--project-dir", "/dbt", "--profiles-dir", "/dbt"],
        env_vars={"DBT_PROFILES_DIR": "/dbt"},
        resources={
            "request_memory": "2Gi",
            "request_cpu": "1",
        },
        deferrable=True,   # Airflow 3.x 권장 — 폴링 대신 트리거 기반 대기
        get_logs=True,
        is_delete_operator_pod=True,
    )


dbt_pipeline()
```

### deferrable=True (Airflow 3.x 권장)

`deferrable=True`를 쓰면 KPO가 Pod 완료를 폴링하는 대신 **Triggerer**에게 감시를 위임합니다. Worker 슬롯을 점유하지 않아 대규모 병렬 실행에 유리합니다. Airflow 3.x에서는 기본값으로 설정하는 것을 권장합니다.

---

## 7. 조합 패턴 3가지

### 패턴 1: CeleryExecutor + KubernetesPodOperator

Celery Worker 위에서 KPO가 K8s Pod를 띄웁니다. K8s는 KPO용 클러스터만 있으면 되고, Airflow 자체는 VM/Docker 환경에서 돌릴 수 있습니다.

```
Celery Worker ──▶ KubernetesPodOperator 코드 실행
                    └──▶ [Task Pod] spark-image:3.5
```

**적합한 상황**: Airflow는 기존 Celery 환경 유지 + 일부 헤비 태스크만 K8s로 분리하고 싶을 때.

### 패턴 2: KubernetesExecutor + KubernetesPodOperator

Worker Pod 안에서 KPO가 또 다른 Pod를 생성합니다. Pod-in-Pod 구조입니다.

```
[Scheduler] ──▶ [Worker Pod (airflow)]
                    └──▶ [Task Pod (spark)]
```

**적합한 상황**: 모든 태스크를 K8s에서 격리 실행하되, 일부 태스크는 완전히 다른 이미지·환경이 필요할 때.

**주의**: Pod 2개 분의 기동 시간이 더해집니다. 짧은 태스크라면 오버헤드가 부담됩니다.

### 패턴 3: 멀티 익스큐터 + KubernetesPodOperator (Airflow 3.x)

가장 유연한 조합입니다. 경량 태스크는 Celery, 무거운 태스크는 K8s, 특수 환경 태스크는 KPO로 명시적으로 분리합니다.

```python
@dag(...)
def pipeline():

    @task(executor="CeleryExecutor")          # 빠른 추출
    def extract(): ...

    @task(executor="KubernetesExecutor")      # 격리 필요한 변환
    def transform(): ...

    run_spark = KubernetesPodOperator(        # 완전히 다른 이미지
        task_id="run_spark",
        image="apache/spark:3.5",
        ...
        deferrable=True,
    )

    extract() >> transform() >> run_spark
```

---

## 8. 함정 / 흔한 혼동

### "KPO를 쓰려면 KubernetesExecutor가 필요하다"

**오해입니다.** KPO는 Executor와 무관합니다. LocalExecutor, CeleryExecutor 환경에서도 KPO는 동작합니다. KPO가 K8s API에 직접 접근할 수만 있으면 됩니다(`kubeconfig` 또는 `in_cluster` 설정).

### "KubernetesExecutor를 쓰면 이미지를 마음대로 바꿀 수 있다"

기본 이미지는 Airflow Worker 이미지입니다. `executor_config`로 오버라이드할 수 있지만, **그 이미지에도 Airflow가 설치돼 있어야** 합니다(Task SDK가 태스크를 실행해야 하므로). 완전히 다른 런타임(spark, dbt 등)이 필요하다면 KPO가 맞습니다.

### KPO 로그 소실

`is_delete_operator_pod=True`(기본값)이면 Pod 종료와 동시에 로그가 사라집니다. 반드시 `get_logs=True`와 함께 원격 로그 스토리지(S3, GCS 등)를 설정하세요. 그렇지 않으면 실패한 태스크의 로그를 볼 수 없습니다.

```python
KubernetesPodOperator(
    ...
    get_logs=True,
    is_delete_operator_pod=True,  # 로그 fetch 완료 후 삭제
)
```

### 멀티 익스큐터에서 첫 번째 선언이 기본값

`executor = CeleryExecutor,KubernetesExecutor`로 설정하면 `executor` 파라미터를 생략한 모든 태스크는 CeleryExecutor로 실행됩니다. 의도치 않게 모든 태스크가 Celery로만 가는 상황을 주의하세요.

---

## 9. 의사결정 체크리스트

**KubernetesExecutor 선택 신호**
- [ ] 모든(또는 대부분) 태스크를 격리된 환경에서 실행해야 한다
- [ ] 상시 Worker 비용을 없애고 싶다
- [ ] Airflow 3.x 멀티 익스큐터로 일부 태스크만 K8s 격리하고 싶다
- [ ] K8s 클러스터와 운영 역량이 이미 있다

**KubernetesPodOperator 선택 신호**
- [ ] 특정 태스크만 Airflow와 완전히 다른 이미지(Spark, dbt, ML 등)로 실행해야 한다
- [ ] Task Pod에 Airflow를 설치하고 싶지 않다
- [ ] Executor 종류와 무관하게 K8s Pod 실행이 필요하다
- [ ] 태스크별로 리소스(CPU/메모리/GPU)를 개별 제어해야 한다

**멀티 익스큐터 선택 신호 (Airflow 3.x)**
- [ ] 하나의 DAG에 경량 태스크(Celery)와 무거운 태스크(K8s)가 섞여 있다
- [ ] CeleryKubernetesExecutor(Airflow 2.x 방식) 대신 더 명시적인 제어를 원한다
- [ ] 태스크마다 최적의 실행 환경을 선택하고 싶다

---

## 마무리

**KubernetesExecutor**는 Airflow 전체의 실행 방식을 바꾸고, **KubernetesPodOperator**는 특정 태스크 하나의 실행 환경을 바꿉니다. 이 둘은 충돌하지 않고 조합됩니다.

Airflow 3.x의 멀티 익스큐터는 이 그림을 더 명확하게 만들어 줍니다. "이 태스크는 Celery로, 저 태스크는 K8s로"를 DAG 코드에 명시할 수 있어 암묵적인 queue·pool 기반 라우팅보다 의도가 훨씬 뚜렷합니다.

헷갈릴 때는 이 질문 하나로 정리하세요:

> "모든 태스크의 실행 방식을 바꾸고 싶나?" → **KubernetesExecutor**
> "이 태스크만 다른 이미지·환경에서 실행하고 싶나?" → **KubernetesPodOperator**

---

더 궁금한 점은 [GitHub](https://github.com/taengkim)에서 이슈로 남겨주세요!

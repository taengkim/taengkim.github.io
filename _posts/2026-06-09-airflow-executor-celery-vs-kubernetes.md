---
title: "Airflow Celery Executor vs Kubernetes Executor 선택 가이드"
date: 2026-06-09
categories: [Airflow, 운영]
tags: [airflow, celery-executor, kubernetes-executor, 운영, 인프라]
---

> **TL;DR** — 정답은 없습니다. **팀 인프라 성숙도**, **DAG 태스크 특성**, **의존성 격리 필요 여부** 세 가지 기준으로 결정하면 됩니다. K8s를 이미 운영 중이고 태스크별 환경이 달라야 한다면 KubernetesExecutor, 그렇지 않다면 CeleryExecutor가 낮은 복잡도로 더 빠르게 구축됩니다.

---

## 1. 두 Executor의 작동 방식

### CeleryExecutor — 상주형 워커

스케줄러가 태스크를 **메시지 브로커**(Redis / RabbitMQ)에 넣으면, 항상 떠 있는 Celery Worker 프로세스가 꺼내 실행합니다.

```mermaid
flowchart TD
    Scheduler["Scheduler"]
    Broker["Redis / RabbitMQ\n브로커 (필수)"]
    W1["Celery Worker 1\nTask A 실행"]
    W2["Celery Worker 2\nTask B 실행"]
    W3["Celery Worker 3\nTask C 실행"]
    Scheduler -->|"태스크 enqueue"| Broker
    Broker -->|"태스크 dequeue"| W1 & W2 & W3
```

- Worker는 **항상 실행 중** (유휴 상태에도 리소스 점유)
- 모든 Worker가 **같은 Python 환경** 공유
- 태스크 기동 지연 거의 없음 (프로세스 포크)

### KubernetesExecutor — 일회용 Pod

스케줄러가 직접 **Kubernetes API 서버**에 Pod 생성 요청을 보내고, 태스크가 끝나면 Pod를 삭제합니다.

```mermaid
flowchart TD
    Scheduler["Scheduler"]
    K8s["Kubernetes API Server"]
    P1["Node A\nTask Pod → Task A 실행 후 삭제"]
    P2["Node B\nTask Pod → Task B 실행 후 삭제"]
    P3["Node C\nTask Pod → Task C 실행 후 삭제"]
    Scheduler -->|"Pod 생성 요청"| K8s
    K8s -->|"Pod 스케줄링"| P1 & P2 & P3
```

- Pod는 **태스크 기간만 존재** (유휴 비용 없음)
- 태스크마다 **다른 Docker 이미지** 지정 가능
- Pod 생성 오버헤드: 수 초 ~ 수십 초

---

## 2. 핵심 차이점 비교

| 항목 | CeleryExecutor | KubernetesExecutor |
|---|---|---|
| **실행 단위** | Worker 프로세스 | Kubernetes Pod |
| **브로커 필요** | 필수 (Redis / RabbitMQ) | 불필요 |
| **태스크 기동 속도** | 빠름 (프로세스 포크) | 느림 (Pod 생성: 10~60초) |
| **의존성 격리** | 워커 전체 공유 | 태스크마다 완전 격리 가능 |
| **리소스 격리** | 없음 (공유 프로세스) | Pod 단위 CPU/메모리 제한 |
| **스케일링 방식** | Worker 수 수동/자동 조정 | K8s 클러스터 리소스 내 자동 |
| **유휴 비용** | 높음 (워커 상시 실행) | 낮음 (실행 중 Pod만 과금) |
| **운영 복잡도** | 중간 (브로커 + 워커 관리) | 높음 (K8s 클러스터 필요) |
| **인프라 요구사항** | VM / 컨테이너 (K8s 불필요) | Kubernetes 클러스터 필수 |
| **로그 접근** | 워커 파일시스템 직접 | Pod 로그 → 외부 스토리지 권장 |

---

## 3. CeleryExecutor를 선택해야 할 때

다음 중 해당 사항이 많다면 CeleryExecutor가 적합합니다.

**인프라 환경**
- Kubernetes 클러스터를 운영하지 않거나, 도입 계획이 없다
- VM이나 베어메탈 서버 기반 환경이다
- Redis나 RabbitMQ를 이미 운영 중이다

**DAG/태스크 특성**
- 태스크 실행 시간이 짧다 (수 초 ~ 수 분)
- 모든 DAG이 동일한 Python 패키지 환경을 사용한다
- 태스크 수가 많고 빈번하게 실행된다 (Pod 기동 지연이 병목이 될 수 있음)

**팀 역량**
- Python/Celery에 익숙하지만 K8s 운영 경험이 없다
- 빠르게 구축하고 운영해야 한다

**설정 예시:**

```ini
# airflow.cfg
[core]
executor = CeleryExecutor

[celery]
broker_url = redis://redis:6379/0
result_backend = db+postgresql://airflow:airflow@postgres/airflow
worker_concurrency = 16
```

```bash
# 환경변수로 설정
AIRFLOW__CORE__EXECUTOR=CeleryExecutor
AIRFLOW__CELERY__BROKER_URL=redis://redis:6379/0
AIRFLOW__CELERY__RESULT_BACKEND=db+postgresql://airflow:airflow@postgres/airflow
```

워커 수 확장은 단순히 Celery Worker 프로세스를 추가로 띄우면 됩니다.

```bash
airflow celery worker --concurrency 32  # 워커 1개 추가 기동
```

---

## 4. KubernetesExecutor를 선택해야 할 때

다음 중 해당 사항이 많다면 KubernetesExecutor가 적합합니다.

**인프라 환경**
- 이미 Kubernetes 클러스터를 운영 중이다 (EKS, GKE, AKS 등)
- 인프라 팀에 K8s 운영 경험이 있다
- 비용 효율화가 중요하다 (유휴 워커 비용을 없애고 싶다)

**DAG/태스크 특성**
- 태스크 실행 시간이 길다 (수 분 ~ 수 시간) — Pod 기동 오버헤드가 상대적으로 무시할 만큼 작다
- DAG마다 또는 태스크마다 **다른 패키지 환경**이 필요하다 (예: TensorFlow DAG, Spark DAG, dbt DAG)
- GPU 같은 특수 리소스를 특정 태스크에만 할당해야 한다
- 태스크 간 완전한 리소스 격리가 필요하다 (태스크 A가 메모리를 폭식해도 태스크 B에 영향 없음)

**설정 예시:**

```ini
# airflow.cfg
[core]
executor = KubernetesExecutor

[kubernetes]
namespace = airflow
worker_container_repository = my-registry/airflow
worker_container_tag = latest
in_cluster = True
```

```bash
AIRFLOW__CORE__EXECUTOR=KubernetesExecutor
AIRFLOW__KUBERNETES__NAMESPACE=airflow
AIRFLOW__KUBERNETES__IN_CLUSTER=True
```

태스크별로 다른 이미지를 지정하려면 DAG에서 `executor_config`를 씁니다.

```python
@task(
    executor_config={
        "KubernetesExecutor": {
            "image": "my-registry/spark-airflow:3.5",
            "resources": {
                "requests": {"memory": "4Gi", "cpu": "2"},
                "limits":   {"memory": "8Gi", "cpu": "4"},
            },
        }
    }
)
def run_spark_job():
    ...
```

---

## 5. 하이브리드: CeleryKubernetesExecutor

두 Executor의 장점을 모두 취하고 싶을 때 쓰는 옵션입니다. **같은 Airflow 클러스터에서 두 Executor를 동시에 운영**하며, 태스크 단위로 어느 쪽으로 실행할지 선택합니다.

```ini
[core]
executor = CeleryKubernetesExecutor
```

- 기본 태스크: CeleryExecutor (빠른 기동)
- 특수 태스크: KubernetesExecutor (격리·GPU 등)

태스크에 `queue="kubernetes"`를 지정하면 K8s로 보냅니다.

```python
@task(queue="kubernetes")
def heavy_ml_task():
    ...
```

단, 두 시스템을 동시에 운영하므로 **복잡도는 두 배**입니다. 명확한 필요 없이 선택하면 운영 부담만 늘어납니다.

---

## 6. 함정 / 흔한 실수

### "K8s가 최신 기술이니까 무조건 좋다"

KubernetesExecutor는 K8s 클러스터 운영 역량이 없으면 오히려 독입니다.

- Pod가 `Pending` 상태에서 안 뜨는 이유를 디버깅하는 데 K8s 지식이 필요합니다.
- 로그가 Pod 삭제와 함께 사라지므로 원격 로그 스토리지(S3, GCS 등) 설정이 필수입니다. 이걸 빠뜨리면 실패한 태스크 로그를 볼 수 없습니다.
- `airflow.cfg` 하나로 끝나지 않고, PVC, ServiceAccount, RBAC, NetworkPolicy 등 K8s 리소스 설정이 뒤따릅니다.

### "CeleryExecutor는 격리가 없어서 위험하다"

워커가 Python 환경을 공유하는 건 맞지만, DAG가 모두 같은 환경을 써도 된다면 문제가 없습니다. 불필요한 격리를 위해 복잡도를 올리지 마세요.

### "Pod 기동 지연이 얼마 안 되겠지"

환경에 따라 Pod 기동에 10~60초가 걸립니다. 태스크 실행 시간이 30초짜리 DAG에서 KubernetesExecutor를 쓰면 기동 시간이 실행 시간을 넘어서는 경우도 생깁니다.

### "CeleryKubernetesExecutor가 절충안이니 항상 좋다"

두 시스템을 모두 운영해야 하므로 장애 포인트도 두 배입니다. 특별한 이유 없이 선택하지 마세요.

---

## 7. 의사결정 체크리스트

다음 질문에 답하면서 선택하세요.

**인프라 환경 확인**
- [ ] Kubernetes 클러스터를 이미 운영 중이다 → **K8s 쪽으로 +1**
- [ ] K8s 운영 경험이 있는 인프라/SRE 팀이 있다 → **K8s 쪽으로 +1**
- [ ] Redis 또는 RabbitMQ를 이미 운영 중이다 → **Celery 쪽으로 +1**
- [ ] VM/베어메탈 환경이고 컨테이너 오케스트레이션이 없다 → **Celery 쪽으로 +2**

**DAG 특성 확인**
- [ ] 태스크 실행 시간이 평균 5분 이상이다 → **K8s 쪽으로 +1**
- [ ] DAG마다 필요한 Python 패키지가 다르다 → **K8s 쪽으로 +2**
- [ ] GPU 같은 특수 리소스가 특정 태스크에만 필요하다 → **K8s 쪽으로 +2**
- [ ] 태스크 수가 많고 실행 주기가 짧다 (분 단위 이하) → **Celery 쪽으로 +1**

**운영 요구사항 확인**
- [ ] 유휴 상태 비용 절감이 중요하다 → **K8s 쪽으로 +1**
- [ ] 빠르게 구축하고 안정적으로 운영해야 한다 → **Celery 쪽으로 +1**
- [ ] 태스크 간 완전한 리소스 격리가 필요하다 → **K8s 쪽으로 +1**

**결과 해석:** K8s가 더 높으면 KubernetesExecutor, Celery가 더 높거나 같으면 CeleryExecutor로 시작하세요.

---

## 마무리

두 Executor의 기술적 우열을 가리는 게 아니라, **내 팀과 인프라에 맞는 것을 고르는 것**이 핵심입니다.

- K8s 없는 환경에서 K8s Executor를 억지로 도입하면 운영 복잡도만 올라갑니다.
- 반대로 태스크별 환경이 달라야 하는 상황에서 CeleryExecutor를 고집하면 의존성 충돌과 싸우게 됩니다.

확실하지 않다면 **CeleryExecutor로 시작**하세요. 단순하고 검증됐습니다. 나중에 K8s 인프라가 갖춰지고 격리 필요성이 생기면 그때 전환해도 늦지 않습니다.

---

더 궁금한 점은 [GitHub](https://github.com/taengkim)에서 이슈로 남겨주세요!

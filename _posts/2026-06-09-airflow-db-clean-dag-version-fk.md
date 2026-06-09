---
title: "Airflow 3.x db clean이 dag_version에서 막히는 이유와 PostgreSQL 자동화 해법"
date: 2026-06-09
categories: [Airflow, 운영]
tags: [airflow, postgresql, db-clean, dag-version, maintenance, metadb]
---

> **TL;DR** — Airflow 3.x에서 `airflow db clean`이 `task_instance_dag_version_id_fkey` 제약 위반(`ForeignKeyViolation`)으로 실패하는 건 설정 실수가 아니라 **알려진 미해결 버그**입니다. `db clean`이 FK 관계를 무시하고 테이블별 나이 기준으로만 삭제하기 때문입니다. FK를 없애는 건 해법이 아닙니다(데이터 손상·마이그레이션 붕괴를 부릅니다). 제약을 *위반*하지 말고 *존중*하는 방향 — "참조가 사라진 고아 `dag_version`만 삭제" — 으로 가면 되고, 이건 유지보수 DAG 한 개로 자동화됩니다.

---

## 1. 증상

`airflow db clean`을 보존 기간 정리 용도로 돌리면 어느 순간 다음처럼 터집니다.

```
Checking table dag_version
Found 46 rows meeting deletion criteria.
Performing Delete...
...
psycopg2.errors.ForeignKeyViolation: update or delete on table "dag_version"
violates foreign key constraint "task_instance_dag_version_id_fkey" on table "task_instance"
DETAIL: Key (id)=(0199ce36-3943-7f32-af3b-3976a2646dc8) is still referenced from table "task_instance".
```

`dag_version` 차례에서만 죽습니다. `task_instance`, `xcom`, `log` 등 나머지 테이블은 멀쩡히 지워집니다.

관련 이슈(모두 open):

- [#56192](https://github.com/apache/airflow/issues/56192) — 3.1.0 (2025-09)
- [#59474](https://github.com/apache/airflow/issues/59474) — 3.1.3 (2025-12)
- [#61390](https://github.com/apache/airflow/issues/61390) — 3.1.6 (2026-02)

`dag_version` 테이블은 Airflow 3.0의 DAG 버저닝(AIP-66)과 함께 도입됐고, 수정 PR이 머지되지 않았으므로 **3.0 / 3.1 / 3.2가 동일하게 영향**받습니다.

---

## 2. 원인: 제약은 맞고, 삭제 로직이 틀렸다

### 2.1 문제의 FK

마이그레이션 리비전 `3ac9e5732b1f`가 다음 제약을 겁니다.

```
task_instance.dag_version_id  ──(FK, ON DELETE RESTRICT)──▶  dag_version.id
```

`ON DELETE RESTRICT`는 "자식이 아직 참조 중이면 부모를 못 지운다"는 뜻입니다. 즉 어떤 `task_instance`가 특정 `dag_version`을 가리키고 있으면, 그 `dag_version`은 삭제 불가입니다. 이건 의미상 **올바른** 제약입니다. TI가 실행된 DAG의 직렬화 버전·번들 정보를 복원하려면 해당 `dag_version`이 살아 있어야 하니까요.

### 2.2 db clean의 삭제 방식

문제는 `db clean`이 **테이블 간 FK 관계를 전혀 고려하지 않고**, 각 테이블을 독립적으로 "나이 기준"으로만 지운다는 점입니다. 그런데 테이블마다 나이를 재는 컬럼이 다릅니다.

| 테이블 | 나이 판정 컬럼 |
|---|---|
| `task_instance` | `start_date` / `updated_at` |
| `dag_version` | `created_at` / `last_updated` |

### 2.3 데드락 같은 조합

다음 상황을 생각해 봅시다.

- **자주 실행되지만 코드가 안 바뀐 DAG**: `dag_version` 행은 오래전(`last_updated`가 과거)에 만들어졌지만, 그걸 참조하는 `task_instance`는 어제도 돌았습니다(`updated_at`이 최근).

그러면 cutoff 시점이 "최근 run 이전 + 마지막 DAG 변경 이후"에 떨어질 때:

```
오래된 dag_version  →  삭제 대상 O  (last_updated가 cutoff보다 과거)
최근 task_instance →  삭제 대상 X  (updated_at이 cutoff보다 최근)
```

`db clean`은 자식(TI)을 안 지운 채 부모(`dag_version`)를 지우려 들고, `RESTRICT`에 그대로 부딪힙니다. **"오래된 버전 + 최근 실행"** 조합이 정확히 이 상태를 만듭니다. 실제로 이 조건이면 100% 재현됩니다.

---

## 3. 그럼 키를 없애면 안 되나? — 안 된다

가장 먼저 떠오르는 발상이 "DB 구조를 바꿔서 그냥 지워지게 하자"인데, 세 가지 변형 모두 함정입니다.

### 3.1 FK를 완전히 drop

`db clean`은 통과합니다. 하지만 삭제된 `dag_version`을 가리키는 `task_instance.dag_version_id`가 dangling 상태로 남습니다(고아 행). 결과:

- ORM 모델엔 `dag_version` relationship이 그대로라, 해당 TI 조회 시 join이 `None` → 과거 run의 버전/번들 복원 실패(UI grid·code view 등).
- **결정적으로** 다음 `airflow db migrate`가 이 제약을 재생성할 때, 기존 고아 행을 검증하다 실패합니다. 한 번 drop하고 정리 안 된 채 두면 업그레이드가 막힙니다.
- 공식 스키마에서 이탈 → 미지원 상태로 이후 버전 업이 계속 꼬입니다.

### 3.2 RESTRICT → SET NULL

불가능합니다. 메인테이너들이 [#46362](https://github.com/apache/airflow/issues/46362)에서 DR/TI의 `dag_version_id`를 **non-nullable**로 의도적으로 바꿨습니다(DagVersion 없는 TI가 존재하지 않도록). NOT NULL 컬럼엔 `SET NULL`을 걸 수 없습니다.

### 3.3 RESTRICT → CASCADE

흥미롭게도 ORM 모델 선언 자체는 `ForeignKey("dag_version.id", ondelete="CASCADE")`인데, 실제 DB엔 마이그레이션이 `RESTRICT`로 깔아놨습니다(모델 ↔ DDL 불일치). 그렇다고 `CASCADE`로 맞추면 **더 위험**합니다: 오래된 `dag_version`을 지우는 순간 그걸 참조하던 **최근 `task_instance`까지 cascade로 동반 삭제**됩니다. 보존하려던 최근 run 이력이 날아갑니다 — cleanup이 막히는 것보다 나쁜 결과입니다.

**결론**: 제약은 올바릅니다. 버그는 스키마가 아니라 `db clean`의 삭제 순서/범위 로직에 있습니다. 키 제거는 "시끄러운 실패"를 "조용한 데이터 손상 + 업그레이드 붕괴"로 바꿉니다.

---

## 4. 올바른 해법: 제약을 존중하는 삭제

핵심 통찰은 단순합니다. **어떤 자식도 참조하지 않는 `dag_version`(고아)만 지우면, RESTRICT를 위반할 일이 없습니다.** 나이 기준 대신 "참조 여부" 기준으로 삭제하는 것입니다.

`dag_version.id`를 참조하는 자식은 세 군데입니다.

- `task_instance.dag_version_id`
- `task_instance_history.dag_version_id`
- `dag_run.created_dag_version_id`

이걸 하드코딩하기보단 PG 카탈로그에서 동적으로 뽑으면 3.x 버전 간 스키마 변화에도 안 깨집니다.

```sql
-- dag_version.id를 참조하는 모든 FK 자식 (테이블, 컬럼)
SELECT conrelid::regclass::text AS child_table,
       att.attname             AS child_column
FROM pg_constraint c
JOIN pg_attribute att
  ON att.attrelid = c.conrelid
 AND att.attnum   = ANY (c.conkey)
WHERE c.contype = 'f'
  AND c.confrelid = 'dag_version'::regclass;
```

이 결과로 `NOT EXISTS` 가드를 만들어 고아만 삭제합니다.

```sql
DELETE FROM dag_version dv
WHERE dv.created_at < :cutoff
  AND NOT EXISTS (SELECT 1 FROM task_instance         ch WHERE ch.dag_version_id        = dv.id)
  AND NOT EXISTS (SELECT 1 FROM task_instance_history ch WHERE ch.dag_version_id        = dv.id)
  AND NOT EXISTS (SELECT 1 FROM dag_run               ch WHERE ch.created_dag_version_id = dv.id);
```

두 가지 안전장치가 있습니다.

- `created_at < cutoff`: 방금 파싱돼 아직 run이 안 붙은 **신규 버전**을 보호합니다(`created_at`은 갱신되지 않는 생성 시각이라 `last_updated`보다 가드로 적합합니다).
- **RESTRICT 자체가 백스톱**: 삭제와 동시에 새 run이 그 버전을 참조하는 레이스가 나도, `DELETE` 문이 FK 검사에서 실패해 트랜잭션만 롤백됩니다. 손상 없이 다음 실행에서 재시도될 뿐입니다.

---

## 5. 자동화: 유지보수 DAG

전체 흐름은 2단계입니다.

1. **`dag_version`을 제외한** 모든 테이블에 Airflow 표준 `db clean` 실행. 오래된 `task_instance`/`dag_run`이 먼저 비워지면, 그만큼 `dag_version`이 고아가 되어 2단계 회수율이 올라갑니다.
2. 참조 없는 `dag_version`만 위 SQL로 삭제.

```python
"""metadb_cleanup_pg — Airflow 3.2 (PostgreSQL) 메타DB 자동 정리 DAG."""
from __future__ import annotations

import logging
import pendulum
from airflow.sdk import dag, task  # Airflow 3 Task SDK

log = logging.getLogger(__name__)

RETENTION_DAYS = 30
SCHEDULE = "0 3 * * *"   # 매일 03:00 (Asia/Seoul)
SKIP_ARCHIVE = True       # False면 _airflow_deleted__ 아카이브 테이블 생성


@dag(
    dag_id="metadb_cleanup_pg",
    schedule=SCHEDULE,
    start_date=pendulum.datetime(2026, 1, 1, tz="Asia/Seoul"),
    catchup=False,
    max_active_runs=1,
    tags=["maintenance", "metadb"],
    default_args={"retries": 1},
)
def metadb_cleanup_pg():

    @task
    def clean_standard_tables() -> str:
        """dag_version을 제외한 모든 테이블에 표준 db clean 실행."""
        from airflow.utils.db_cleanup import config_dict, run_cleanup

        cutoff = pendulum.now("UTC").subtract(days=RETENTION_DAYS)
        tables = [t for t in config_dict if t != "dag_version"]
        run_cleanup(
            clean_before_timestamp=cutoff,
            table_names=tables,
            confirm=False,
            skip_archive=SKIP_ARCHIVE,
        )
        return cutoff.isoformat()

    @task
    def clean_orphan_dag_versions(cutoff_iso: str) -> int:
        """참조 없는(고아) dag_version만 FK-aware하게 삭제."""
        from sqlalchemy import text
        from airflow.utils.session import create_session

        cutoff = pendulum.parse(cutoff_iso)
        find_children = text(
            """
            SELECT conrelid::regclass::text AS child_table,
                   att.attname             AS child_column
            FROM pg_constraint c
            JOIN pg_attribute att
              ON att.attrelid = c.conrelid AND att.attnum = ANY (c.conkey)
            WHERE c.contype = 'f' AND c.confrelid = 'dag_version'::regclass
            """
        )
        with create_session() as session:
            children = session.execute(find_children).fetchall()
            guards = " ".join(
                f'AND NOT EXISTS (SELECT 1 FROM {tbl} ch WHERE ch."{col}" = dv.id)'
                for tbl, col in children
            )
            delete_sql = text(
                f"DELETE FROM dag_version dv WHERE dv.created_at < :cutoff {guards}"
            )
            result = session.execute(delete_sql, {"cutoff": cutoff})
            session.commit()
            deleted = result.rowcount or 0
            log.info("Deleted %d orphaned dag_version rows", deleted)
            return deleted

    cutoff = clean_standard_tables()
    clean_orphan_dag_versions(cutoff)


metadb_cleanup_pg()
```

`config_dict`(테이블명 → 설정 매핑)에서 `dag_version`만 빼고 `run_cleanup`에 넘기는 게 핵심입니다. 자식 가드는 카탈로그 조회 결과(우리 스키마, 신뢰 가능한 출처)로 만들므로 인젝션 우려는 없습니다.

---

## 6. 대안: K8s CronJob

위 DAG는 **워커가 메타DB(`sql_alchemy_conn`)에 직접 접근 가능한 배포**를 전제합니다. Celery / Kubernetes executor는 보통 해당되지만, 완전 격리형 remote/edge worker라면 `create_session()`이 동작하지 않습니다.

그 경우엔 DB 크리덴셜을 가진 파드에서 도는 **CronJob**으로 빼는 게 깔끔합니다. 스케줄러/워커 격리 정책과 무관하게 동작합니다.

```bash
# 1) dag_version 제외 표준 정리
airflow db clean \
  --clean-before-timestamp "$(date -u -d '30 days ago' '+%Y-%m-%d %H:%M:%S+00:00')" \
  --tables "task_instance,task_instance_history,dag_run,xcom,log,job,..." \
  --skip-archive -y

# 2) 고아 dag_version 정리
psql "$AIRFLOW__DATABASE__SQL_ALCHEMY_CONN" -v cutoff="'$(date -u -d '30 days ago' '+%Y-%m-%d')'" -f cleanup_orphan_dag_version.sql
```

---

## 7. 운영 체크리스트

- [ ] 처음엔 `RETENTION_DAYS`를 크게 잡고 `run_cleanup(..., dry_run=True)`로 영향 범위 확인
- [ ] 보수적으로 가려면 고아 삭제 전 `INSERT INTO ... SELECT`로 별도 백업 후 삭제
- [ ] `--error-on-cleanup-failure`로 돌려 실패를 묵살하지 말고 태스크 실패로 노출(기본은 에러를 삼키고 exit 0)
- [ ] `_airflow_deleted__*` 아카이브 테이블이 쌓이면 `airflow db drop-archived`로 주기적 정리
- [ ] 운영 DB에 강한 락이 걸리지 않도록 `batch_size` 조정, 트래픽 낮은 시간대로 스케줄

---

## 마무리

이 이슈의 본질은 "스키마가 잘못됐다"가 아니라 "정리 도구가 참조 관계를 모른 채 나이만 본다"는 점입니다. 따라서 **키를 손대지 말고**, 정리 로직 쪽에서 참조를 인지하도록 우회하는 게 정공법입니다. 공식 패치가 머지되기 전까지는 위 2단계 패턴이 가장 안전하고 재현 가능한 운영 해법입니다.

---

더 궁금한 점은 [GitHub](https://github.com/taengkim)에서 이슈로 남겨주세요!

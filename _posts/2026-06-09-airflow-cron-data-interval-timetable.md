---
title: "Airflow 3에서 data_interval_start/end가 같아지는 이유와 CronDataIntervalTimetable로 되돌리기"
date: 2026-06-09
categories: [Airflow, 운영]
tags: [airflow, timetable, cron, data-interval, 마이그레이션]
---

> **TL;DR** — Airflow 3.x에서 `schedule="0 0 * * *"` 같은 맨 cron 문자열을 쓰면 `data_interval_start == data_interval_end == logical_date`가 되어 윈도우 기반 증분 추출이 깨집니다. 이건 버그가 아니라 **기본 타임테이블이 바뀐 것**입니다: 2.x는 `CronDataIntervalTimetable`, 3.x는 `CronTriggerTimetable`이 기본입니다. 되돌리려면 타임테이블을 `CronDataIntervalTimetable`로 명시하거나, `[scheduler] create_cron_data_intervals=True`로 전역 설정합니다.

---

## 1. 증상

Airflow 2.x → 3.x 업그레이드 후, 일/주 단위로 도는 DAG에서 데이터 윈도우가 사라집니다.

```python
@dag(schedule="10 0 * * 7", start_date=datetime(2025, 1, 5))  # 매주 일요일 00:10
def my_dag():
    @task
    def extract(**ctx):
        print(ctx["data_interval_start"], ctx["data_interval_end"])
```

2.x에서는 `start = 지난주 일요일`, `end = 이번주 일요일`(1주 간격)로 나왔는데, 3.x에서는:

```
data_interval_start == data_interval_end == logical_date   # 둘 다 트리거 시각
```

UI의 DAG details에서도, 태스크 컨텍스트·템플릿에서도 동일하게 폭이 0인 구간으로 나옵니다. 실제로 `affected_version:3.0`, `priority:high`로 여러 건 리포트됐습니다([#50172](https://github.com/apache/airflow/issues/50172), [#52145](https://github.com/apache/airflow/issues/52145), [Discussion #51371](https://github.com/apache/airflow/discussions/51371)).

---

## 2. 원인: 버그가 아니라 기본 타임테이블 교체

### 2.1 두 종류의 cron 타임테이블

Airflow에는 cron을 해석하는 타임테이블이 두 가지 있습니다.

| | `CronDataIntervalTimetable` | `CronTriggerTimetable` |
|---|---|---|
| 모듈 | `airflow.timetables.interval` | `airflow.timetables.trigger` |
| 한 run이 의미하는 것 | 직전 cron ~ 이번 cron **구간**을 처리 | 이번 cron 시점에 **트리거**만 |
| `data_interval_start` | 구간 시작 (직전 cron) | 트리거 시각 |
| `data_interval_end` | 구간 끝 (이번 cron = 트리거 시각) | 트리거 시각 |
| start vs end | **다름** (폭 있음) | **같음** (폭 0) |

예를 들어 `@daily`(`0 0 * * *`)를 1월 31일 오후 3시에 활성화하면 둘 다 2월 1일 0시에 첫 run을 만들지만:

- `CronTriggerTimetable`: `data_interval_start == data_interval_end == 2월 1일 0시`
- `CronDataIntervalTimetable`: `start = 1월 31일 0시`, `end = 2월 1일 0시`

### 2.2 무엇이 바뀌었나

맨 cron 문자열(`schedule="..."`)이 어느 타임테이블로 해석되는지를 제어하는 설정이 `[scheduler] create_cron_data_intervals`입니다. 이 기본값이 버전 경계에서 뒤집혔습니다.

| | `create_cron_data_intervals` 기본값 | 맨 cron 문자열 → 타임테이블 |
|---|---|---|
| Airflow 2.x | `True` | `CronDataIntervalTimetable` |
| **Airflow 3.x** | **`False`** | **`CronTriggerTimetable`** |

같은 `schedule="0 0 * * *"`라도 2.x에서는 구간 의미를 갖던 게 3.x에서는 트리거 의미로 바뀌어 `start == end`가 됩니다.

### 2.3 `logical_date` 재정의도 함께 바뀜

3.x의 또 다른 변경점: `logical_date`의 정의가 바뀌었습니다.

- 2.x: `logical_date == data_interval_start`
- 3.x: `logical_date == run_after` (트리거 시각)

게다가 3.0부터 `logical_date`는 nullable이 됐습니다(asset 트리거·수동 run 등 "구간" 개념이 없는 run 때문). 이 모든 변화가 "data interval은 더 이상 1급 시민이 아니다"라는 방향을 가리킵니다.

---

## 3. 왜 바꿨나

데이터 구간(run이 `[직전, 현재]`를 처리한다는 모델)은 처음 보는 사람에게 직관적이지 않습니다. "오늘 0시 run인데 어제 데이터를 처리한다"가 혼란의 원천이었고, 이벤트·asset 기반 스케줄링이 들어오면서 "트리거 시각 = logical_date"가 더 단순하고 일관된 모델이라 기본값을 트리거 쪽으로 옮긴 것입니다.

문제는 **데이터 엔지니어링의 전형적 패턴 — `WHERE updated_at >= data_interval_start AND < data_interval_end` 같은 윈도우 증분 추출 — 이 정확히 이 구간 의미에 의존**한다는 점입니다. ETL 파이프라인 입장에서 기본값 변경이 회귀처럼 느껴지는 이유입니다.

---

## 4. 해결책 A — 타임테이블 명시 (권장)

DAG 단위로 의도를 명확히 박는 방법입니다. 가장 명확하고 부작용이 작습니다.

```python
from airflow.timetables.interval import CronDataIntervalTimetable

@dag(
    schedule=CronDataIntervalTimetable("0 0 * * *", timezone="UTC"),
    start_date=datetime(2025, 1, 1),
    catchup=False,
)
def my_dag():
    ...
```

`data_interval_start`(직전 cron)과 `data_interval_end`(이번 cron)가 다시 구간으로 채워집니다.

---

## 5. 해결책 B — 전역 설정으로 2.x 동작 복원

모든 DAG의 맨 cron 문자열 동작을 한 번에 되돌리고 싶을 때 사용합니다.

```bash
AIRFLOW__SCHEDULER__CREATE_CRON_DATA_INTERVALS=True
```

또는 `airflow.cfg`:

```ini
[scheduler]
create_cron_data_intervals = True
```

이 플래그는 최소 Airflow 4까지 유지된다고 메인테이너가 확인했으므로, 마이그레이션 과도기 대응으로 안전합니다. 단 "맨 cron 문자열을 쓰는 모든 DAG"의 해석을 바꾸므로 영향 범위가 넓습니다. 일부 DAG만 구간이 필요하다면 해결책 A가 낫습니다.

---

## 6. 해결책 C — 트리거 시각은 유지하되 구간만 부여

`logical_date`는 cron 시각 그대로 두고 데이터 윈도우만 필요하다면 `CronTriggerTimetable`에 `interval`을 줄 수 있습니다.

```python
from datetime import timedelta
from airflow.timetables.trigger import CronTriggerTimetable

schedule = CronTriggerTimetable(
    "0 0 * * *",
    timezone="UTC",
    interval=timedelta(days=1),  # 트리거 시각에서 거꾸로 1일 구간 부여
)
```

버전에 따라 `interval` 인자 시그니처가 다를 수 있으니, 설치된 버전의 `CronTriggerTimetable` 시그니처를 먼저 확인하세요.

---

## 7. 함정 / 주의점

- **Cross-DAG 트리거 전파 안 됨**: `TriggerDagRunOperator`로 다른 DAG을 띄울 때, 부모의 data interval이 자식에게 전파되지 않습니다([#51371](https://github.com/apache/airflow/discussions/51371)). 자식에서 구간이 필요하면 `conf`로 명시 전달하는 게 안전합니다.
- **수동 트리거의 구간 추론**: 수동 run은 `infer_manual_data_interval`로 구간을 역산합니다. 과거 logical_date를 찍어 수동 트리거하면 스케줄 run과 구간이 어긋나 보일 수 있습니다(주/시간 단위 cron에서 특히 주의).
- **문서와 직관의 괴리**: cron 문서만 보면 "문자열 cron이면 당연히 구간"으로 읽히지만, 3.x 기본값은 그렇지 않습니다. 팀 온보딩 문서에 이 내용을 명시해두는 게 사고 예방에 가장 효과적입니다.

---

## 8. 마이그레이션 체크리스트 (2.x → 3.x)

- [ ] `data_interval_start`/`data_interval_end`(또는 deprecated `execution_date`)에 의존하는 DAG 목록화
- [ ] 윈도우 증분 추출을 하는 DAG는 **해결책 A**(`CronDataIntervalTimetable`)로 명시 전환
- [ ] 단기 호환이 급하면 **해결책 B** 플래그로 전역 복원 후, DAG별로 점진 이관
- [ ] `logical_date == run_after`로 바뀐 점을 고려해, `run_id` 파싱·외부 조인 로직 점검
- [ ] `TriggerDagRunOperator` 체인은 `conf`로 구간 명시 전달하도록 수정
- [ ] 전환 후 UI DAG details에서 `Data interval start ≠ end` 확인

---

## 마무리

핵심은 "버그를 고친다"가 아니라 **"기본 스케줄링 모델이 구간(interval)에서 트리거(trigger)로 바뀌었다"**는 사실을 인지하는 것입니다. 데이터 파이프라인이 구간 의미에 의존한다면, 맨 cron 문자열을 믿지 말고 `CronDataIntervalTimetable`로 의도를 명시하세요. 그게 3.x에서 가장 견고하고 자기설명적인 방식입니다.

---

### 참고

- Timetables — Airflow 3.2.2 공식 문서 (`create_cron_data_intervals` 기본값: 3.x=False, 2.x=True)
- Discussion [#51371](https://github.com/apache/airflow/discussions/51371) — `CronDataIntervalTimetable` 해법 및 설정 플래그 유지 정책
- Issue [#50172](https://github.com/apache/airflow/issues/50172), [#52145](https://github.com/apache/airflow/issues/52145) — 3.0.x에서 start==end 리포트

더 궁금한 점은 [GitHub](https://github.com/taengkim)에서 이슈로 남겨주세요!

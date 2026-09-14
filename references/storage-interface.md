# 저장 인터페이스 계약 v0.2

이 문서는 Notion 또는 이후 별도 DB와 주고받을 공통 데이터 계약이다. v0.2에서는 연결된 Notion을 기본 저장소로 사용하며, 연결되지 않았을 때는 사용자 제공 스냅샷과 미저장 변경안을 다룬다. Notion의 실제 속성 매핑은 [Notion 연결 규칙](notion-connection.md)을 따른다.

## 읽는 순서와 원본

매번 전체 대화를 불러오지 않고 요청과 관련된 활성 목표·프로젝트, 계획 기간의 작업·루틴, 미완료·기한 초과 작업, 선행 작업, 고정 일정, 최근 실행 기록만 선택한다. 주간 회고의 기본 기간은 최근 7일이다. 필요한 선행 작업과 부모 ID는 기간 밖이어도 포함한다.

연결 상태는 `disconnected / connected / error`로 구분한다. 읽기 결과에는 `as_of`, `revision`, `coverage`를 붙인다. `coverage`는 전체 또는 일부인지, 어느 날짜 범위인지, 어떤 필터인지 설명한다. 필요한 범위가 불완전하면 전체 진척률이나 충돌 없음 판정을 내리지 않는다.

원본은 마지막으로 저장 성공이 확인된 외부 상태다. 현재 사용자 보고는 그 위에 적용할 변경이며 사용자의 정정이 과거 기록과 다르면 정정안을 만든다. 오래된 채팅 발언으로 최신 상태를 덮어쓰지 않는다. 저장되지 않은 변경안은 다음 대화에 별도로 제공되지 않으면 복구를 보장할 수 없다.

## 공통 표기

- 날짜: `YYYY-MM-DD`. 시각: UTC 오프셋을 포함한 ISO 8601. 시간대는 사용자·환경에서 확인하고 없으면 미정으로 둔다.
- 모르는 값은 `null`, 연결 목록이 없는 것은 `[]`다. 변경안에서 필드 생략은 유지, `null`은 명시적 비우기다.
- 기존 ID는 유지한다. 신규 ID는 변경안 내에서 `tmp:g1`, `tmp:t1` 등으로 유일하게 만들고, 미래 저장기가 영구 ID로 매핑한다. 제목을 ID로 쓰지 않는다.
- 값이 추정이면 `assumptions`에 대상 필드, 제안값, 근거를 적는다. 원래 사용자 표현은 짧은 `source_note`로 보존한다. 대화 전문이나 불필요한 민감정보는 저장하지 않는다.

## 데이터 구조

단일 항목 모델을 쓰되 유형별 속성을 나눈다. 다음 이름은 향후 저장소와 교환할 필드 이름이다.

| 대상 | 필드와 규칙 |
|---|---|
| 모든 항목 | `id`, `type`(Goal/Project/Task/Routine/Idea), `title`, `status`, `goal_id`, `project_id`, `source_note`, `created_at`, `updated_at`, `revision` |
| Goal | `outcome`, `success_criteria`, `why`, `due_date`, `target_date`, `metric`(선택: 현재값·목표값·단위·측정일). 상태 `draft/active/paused/achieved/cancelled` |
| Project | `deliverable`, `success_criteria`, `due_date`, `target_date`. 상태 `draft/active/paused/done/cancelled` |
| Task | `done_when`, `estimate_minutes`, `remaining_minutes`, `effort`(low/medium/high), `importance`(high/medium/low/미정), `due_date`, `target_date`, `depends_on`, `completed_at`. 상태 `todo/in_progress/blocked/done/cancelled` |
| Routine | `frequency`, `start_date`, `end_date`, `timezone`, `cue`, `standard_minutes`, `minimum_minutes`, `success_criteria`. 상태 `draft/active/paused/retired` |
| Idea | `note`, `review_date`(선택), `converted_to_ids`. 상태 `inbox/parked/converted/archived`. 실행 일정은 없음 |

추가 레코드:

- `PlanSlot`: `id`, `item_id`, `local_date`, `start_at`, `end_at`, `planned_minutes`, `commitment`(proposed/confirmed), `rationale`. 정확한 시각이 없으면 시각 필드는 null. Task와 Routine을 배치한다. Task 여러 슬롯은 같은 Task의 분량을 나누며 Task를 중복 생성하지 않는다.
- `RoutineOccurrence`: `id`, `routine_id`, `local_date`, `slot_key`, `status`(planned/done/minimum/partial/skipped/unknown), `actual_minutes`. 중복 판별키는 `(routine_id, local_date, slot_key)`다. 하루 1회면 `slot_key=default`이며 여러 번이면 `morning/evening`처럼 구분한다.
- `CheckIn`: `id`, `item_id`, `occurrence_id`(루틴이면 지정), `reported_at`, `performed_on`, `actual_minutes`, `result`, `remaining_note`, `blocker`, `source`(user_report), `correction_of`(선택). 완료 보고의 근거와 정정을 남긴다.
- `Reflection`: `id`, `period_start`, `period_end`, `item_ids`, `user_note`, `facts`, `suggested_adjustments`. 소감과 AI 해석을 분리한다.
- `PlanningContext`: `timezone`, 날짜별 `available_minutes`, `fixed_events`, `capacity_basis`(고정 일정 차감 전/후), `preferences`, `as_of`. 시간 계산의 중복 차감을 막는다.

새 항목의 관련 없는 속성은 생략한다. 아직 저장되지 않은 항목의 저장 시각·revision은 null로 둔다. Goal/Project의 진행률은 연결 항목에서 계산하며 Task처럼 직접 증가시키지 않는다. 사용자 확인 없이 Idea를 active 상태로 전환하지 않는다.

## 논리적 저장 인터페이스 — 미래 구현용

| 연산 | 입력 | 결과 |
|---|---|---|
| `read_context` | 기간, 대상 ID, 상태 필터 | `connection`, `as_of`, `revision`, `coverage`, 관련 레코드와 계획 맥락 |
| `propose_changes` | 읽은 기준 revision, 사용자 입력, 변경 목록 | 아래 ChangeSet. 외부 쓰기 없음 |
| `commit_changes` | 허용된 ChangeSet, `request_id`, 기준 revision | 변경별 성공 여부, 영구 ID 매핑, 새 revision 또는 최신 편집 시각, 저장 시각, 충돌·오류 |

v0.2에서는 사용자의 변경 의도가 명확하고 Notion이 연결된 경우에만 `commit_changes`에 해당하는 Notion 생성·수정을 수행한다. 조회·제안 요청은 쓰지 않는다. 자동 알림은 별도 스케줄러와 사용자 설정이 필요하며 이 인터페이스에 포함되지 않는다.

## ChangeSet: 미저장 변경안

필수 필드:

- `schema_version`: `0.2`
- `request_id`: 변경 묶음의 유일한 ID. 같은 저장 요청 재시도 시 유지한다.
- `storage_status`: `proposed / saved / partial / not_saved / error`
- `base_revision`: 사용한 스냅샷 revision, 없으면 null
- `coverage`: 이 변경안이 다루는 데이터 범위
- `assumptions`: 확정되지 않은 제안 목록
- `changes`: `op`(create/update/append), `entity`, `id`, `fields`, `reason`을 갖는 목록

`create`는 새 항목·슬롯, `update`는 기존 항목의 바뀐 필드만, `append`는 체크인·회고 이력 추가다. 물리적 삭제 대신 cancelled/archived 등을 제안한다. 실제 사용자 지시가 삭제라면 후속 저장 구현의 권한·보존 정책을 따르고 임의로 영구 보존을 강제하지 않는다.

예: “내일 책 반납해야 해”를 사용자 입력 시점이 2026-09-12, 시간대가 Asia/Seoul인 예로 처리한다. 신규 Task 하나이며 기존 일정 조회는 안 된 상황이다. 반납 방법과 시간은 모른다.

```json
{
  "schema_version": "0.2",
  "request_id": "example-return-book-01",
  "storage_status": "not_saved",
  "base_revision": null,
  "coverage": {"complete": false, "scope": "사용자가 방금 입력한 항목만"},
  "assumptions": [],
  "changes": [
    {
      "op": "create",
      "entity": "Task",
      "id": "tmp:t1",
      "fields": {
        "type": "Task",
        "title": "책 반납하기",
        "status": "todo",
        "goal_id": null,
        "project_id": null,
        "source_note": "내일 책 반납해야 해",
        "created_at": null,
        "updated_at": null,
        "revision": null,
        "done_when": "반납 접수가 완료됨",
        "estimate_minutes": null,
        "remaining_minutes": null,
        "effort": null,
        "importance": null,
        "due_date": "2026-09-13",
        "target_date": null,
        "depends_on": [],
        "completed_at": null
      },
      "reason": "사용자가 명시한 일회성 할 일과 기한"
    }
  ]
}
```

예시 날짜와 ID는 실제 처리 때 재사용하지 않는다. 간단한 대화에서는 위 필드를 표로 요약해도 된다. 누락 없는 교환·인계에는 JSON 내보내기를 사용한다.

## 미래 저장기의 일관성 규칙

- 같은 `request_id`의 재시도는 중복 생성하지 않는다. 별도 요청 ID의 반복 입력도 기존 항목과 실행 기록을 조회해 중복 여부를 확인한다.
- 읽은 revision과 현재 revision이 다르면 자동 덮어쓰기 대신 다시 읽고 관련 변경만 재계산한다.
- 여러 변경 중 일부만 성공하면 항목별 결과를 보고하고 성공분을 다시 생성하지 않는다. 응답이 불명확하면 상태를 조회하기 전 재전송하지 않는다.
- 임시 ID의 부모·선행 작업·실행 기록 참조도 영구 ID로 함께 치환한다. 자기 참조와 선행 관계 순환은 거부한다.
- 완료 시 `status=done`과 `completed_at`을 함께 반영한다. 정정으로 다시 열면 완료 시각을 비우고 정정 이력을 남긴다.
- 저장 성공은 반환된 ID·revision 또는 재조회로 확인한다. 실패하면 미저장 변경안을 유지하며 사용자에게 재입력을 요구하기 전에 복구 가능한 내용을 제공한다.

## 새 대화로 인계할 최소 묶음

1. `schema_version`, 외부 저장 위치(있으면), `as_of`, `revision`, `coverage`.
2. 현재 활성 목표와 관련 Project/Task/Routine/Idea의 ID·상태·기한·완료 기준.
3. 계획 기간 슬롯·고정 일정·가용 시간 및 관련 선행 작업.
4. 최근 체크인·회고 요약, 아직 미저장인 ChangeSet.

스냅샷과 변경안을 혼동하지 않는다. 사용자가 변경안을 채택해 수동 스냅샷 파일을 저장한 경우에만 그 파일을 다음 대화의 입력으로 사용할 수 있다. 파일 경로를 알고 있다는 사실만으로 내용이 최신이라고 가정하지 않는다.

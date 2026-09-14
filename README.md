# 개인 실행관리 AI

`personal-execution-manager`는 막연하게 말한 목표와 할 일을 실행 가능한 계획으로 바꾸고, 실행 결과에 따라 다음 계획을 조정하는 Codex Agent Skill입니다.

현재 버전은 **v0.2.1**이며 Notion의 통합 실행 데이터베이스를 장기 저장소로 사용합니다.

## 주요 기능

- 자연어 입력을 `Goal`, `Project`, `Task`, `Routine`, `Idea`로 분류
- 꼭 필요한 정보만 질문하고 목표를 측정 가능한 형태로 구체화
- 작업을 15~45분 정도의 실행 가능한 단위로 분해
- 중요도, 마감, 노력도, 선행 작업, 사용 가능한 시간을 고려해 일정 제안
- 오늘 할 일과 루틴의 실행률 및 목표·프로젝트 진행 상황 관리
- 완료, 부분 실행, 건너뜀, 회고를 구분해 기록
- 실제 실행 시간과 피드백을 바탕으로 다음 계획 재조정
- 채팅 기록 대신 Notion을 장기 상태의 원본으로 사용

## 동작 흐름

```mermaid
flowchart LR
    A[자연어로 목표와 할 일 입력] --> B[Skill이 유형 분류]
    B --> C[목표 구체화와 Task 분해]
    C --> D[우선순위와 일정 제안]
    D --> E[Notion 통합 실행 DB]
    E --> F[대시보드와 Notion Calendar]
    F --> G[실행 결과와 회고 입력]
    G --> D
```

## 파일 구조

```text
personal-execution-manager/
├── SKILL.md
├── agents/
│   └── openai.yaml
└── references/
    ├── notion-connection.md
    └── storage-interface.md
```

| 파일 | 역할 |
|---|---|
| `SKILL.md` | 분류, 목표 설정, Task 분해, 일정 제안, 체크인, 재계획의 핵심 규칙 |
| `agents/openai.yaml` | Codex 화면에 표시되는 이름, 설명, 기본 실행 문구 |
| `references/notion-connection.md` | Notion 데이터베이스 위치, 속성 연결, 저장·조회 규칙 |
| `references/storage-interface.md` | Notion 또는 다른 저장소와 교환할 공통 데이터 구조 |

## 설치 전 준비

다음 환경이 필요합니다.

1. 로컬 Skill을 사용할 수 있는 Codex
2. Notion 저장 기능을 사용할 경우 연결된 Notion 앱 또는 커넥터
3. `references/notion-connection.md`와 호환되는 Notion 데이터베이스

Notion이 연결되지 않아도 목표 분류와 계획 제안은 사용할 수 있습니다. 이 경우 결과는 Notion에 저장되지 않고 `미저장 변경안`으로 제공됩니다.

## 설치 방법

### 새 컴퓨터에 설치

터미널에서 다음 명령을 실행합니다.

```bash
git clone \
  https://github.com/YunjeeJung/personal-execution-manager.git \
  ~/.codex/skills/personal-execution-manager
```

비공개 저장소라면 GitHub 로그인이 필요합니다. 설치 후 Codex를 다시 시작하거나 새 작업을 열어 스킬 목록을 갱신합니다.

설치가 끝나면 다음과 같이 호출할 수 있습니다.

```text
$personal-execution-manager 이번 달 안에 포트폴리오를 만들고 싶은데 무엇부터 해야 할지 모르겠어.
```

### 이미 설치된 스킬 업데이트

GitHub 저장소를 `~/.codex/skills/personal-execution-manager`에 직접 복제했다면 다음 명령으로 업데이트합니다.

```bash
cd ~/.codex/skills/personal-execution-manager
git pull
```

## Notion 연결 설정

현재 저장소는 개인 대시보드에 맞춘 설정을 포함합니다. 같은 Notion 워크스페이스에서 사용할 때는 Codex에서 Notion 연결을 활성화하면 됩니다.

다른 사람이나 다른 Notion 워크스페이스에서 사용하려면 다음 값을 자신의 환경에 맞게 수정해야 합니다.

- `references/notion-connection.md`의 워크스페이스 이름
- 대시보드 페이지 이름과 ID
- 통합 실행 데이터베이스 URL
- 통합 실행 데이터 소스 ID
- `agents/openai.yaml`의 기본 실행 문구
- 필요하면 `SKILL.md`의 기본 Notion 대시보드 이름

실제 Notion 구조가 다르면 `references/notion-connection.md`의 속성 매핑도 함께 수정합니다.

## 사용 예시

### 목표와 계획 만들기

```text
$personal-execution-manager 10월 말까지 토익스피킹 AL을 받고 싶어. 지금 점수는 150점이고 평일 이동 시간을 활용할 수 있어.
```

### 오늘 계획 확인하기

```text
$personal-execution-manager Notion을 확인해서 오늘 해야 할 일과 루틴을 정리해줘.
```

### 새로운 할 일 저장하기

```text
$personal-execution-manager 이번 주 토요일까지 포트폴리오 프로젝트 설명 초안을 작성해야 해. 계획을 나누고 Notion에 저장해줘.
```

### 실행 결과 기록하기

```text
$personal-execution-manager 파이썬 복습을 30분 했지만 예외 처리 부분은 끝내지 못했어. 기록하고 다음 계획을 조정해줘.
```

### 회고 반영하기

```text
$personal-execution-manager 이번 주에는 저녁 집중력이 낮았어. 실행 기록을 보고 다음 주 계획을 조정해줘.
```

## 조회와 저장 기준

- `확인해줘`, `보여줘`, `계획을 짜줘`는 Notion을 읽고 제안만 작성합니다.
- `저장해줘`, `반영해줘`, `완료했어`, `미뤄줘`는 대상이 분명할 때 Notion을 수정합니다.
- 실제 마감 변경, 목표 포기, 여러 일정에 영향을 주는 큰 변경은 영향을 먼저 보여줍니다.
- 요청 없이 Notion 페이지를 물리적으로 삭제하지 않습니다.
- 저장 후에는 다시 조회하여 실제 반영 여부를 확인합니다.

## 개인 정보와 공개 배포

현재 `references/notion-connection.md`에는 개인 Notion 페이지와 데이터베이스 식별자가 포함되어 있습니다. 저장소를 공개하기 전에 이 값을 예시값으로 교체하고 개인 설정 파일을 별도로 분리해야 합니다.

Notion 페이지 ID 자체는 로그인 자격 증명이 아니지만 개인 워크스페이스 구조를 드러낼 수 있습니다. API 키, 비밀번호, 환경 변수 파일은 저장소에 추가하지 마세요.

## 버전

- `0.2.1`: Notion 통합 실행 DB, 날짜별 루틴 기록, 주간 일정, Time Log 및 Notion Calendar 연결 규칙


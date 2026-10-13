# 태아보험 PV KPI 일일 리포트 (Slack 자동화 + 대시보드)

마미톡 `ce:태아보험 PV` 이벤트의 일별 순방문자(Unique Users, KST)를 목표 **일 평균 5,000명**과 비교해 보는 리포트입니다.
두 부분으로 되어 있습니다.

| 구성 | 하는 일 | 실행 위치 |
|---|---|---|
| **Slack 자동 리포트** | 평일 아침 전날(월요일은 금·토·일) 수치를 요약해 Slack DM으로 발송 | claude.ai 예약 작업 |
| **대시보드** | 기준일 카드, 최근 60일 차트, 14일 동요일 비교표, 월별 요약·일자별 달력 | claude.ai Dashboard 아티팩트 |

> 이 문서는 2026-10-13 13:48 KST 기준으로 **직접 조회해서 확인한 내용만** 적었습니다.
> 확인할 수 없는 항목은 **⚠️ 확인 불가**로 따로 표시했습니다.
> 문서를 만드는 동안 운영 중인 예약 작업과 대시보드는 읽기만 했고, 수정하거나 중단하지 않았습니다.

---

## 0. 리뷰 전에 알아야 할 점: 이 자동화는 일반적인 코드 프로젝트가 아닙니다

- **Slack 자동 리포트에는 소스코드 파일이 없습니다.** 실행 정의는 자연어 지시문 하나뿐입니다(`automation/scheduled-task-prompt.md`). 지시문은 claude.ai 예약 작업에 저장되어 있고, 매 실행마다 새 클라우드 세션의 Claude가 이 지시문을 읽어 Amplitude를 조회하고, 계산하고, Slack으로 보냅니다.
- 계산할 때 Claude가 Python 같은 코드를 즉석에서 작성해 실행하도록 지시되어 있습니다. 하지만 **그 코드는 실행마다 새로 만들어지고 저장되지 않습니다.**
  ⚠️ 확인 불가: 지금까지 실행에서 실제로 생성된 코드의 내용.
- **대시보드 코드(`dashboard/index.html`)는 실제 파일입니다.** 다만 claude.ai Dashboard 아티팩트 런타임(`dash` API, `d3` 전역, CSS 토큰) 안에서만 동작하므로 **이 파일만으로는 브라우저에서 단독 실행되지 않습니다.**

따라서 이 저장소는 **현재 동작의 기록과 리뷰용 사본**입니다. 이 저장소를 바꿔도 운영 중인 자동화에는 반영되지 않습니다.
회사 인프라(GitHub Actions 등)에서 직접 돌리려면 재구현이 필요합니다(7절).

---

## 1. 전체 소스코드와 파일명

| 파일 | 내용 | 원본 |
|---|---|---|
| `automation/scheduled-task-prompt.md` | Slack 자동 리포트의 실행 지시문 원문 | 예약 작업 `trig_01CgFVZ3sABLiFqU935s29it`의 prompt |
| `automation/schedule.json` | 스케줄·커넥터·최근 실행 상태 등 비밀이 아닌 설정 사본 | 같은 예약 작업의 설정값 |
| `dashboard/index.html` | 대시보드 페이지 전체(HTML + CSS + JS, 약 26.6KB) | 아티팩트 저장소 `files/index.html` (version 3) |
| `dashboard/datasets/daily.json` | 대시보드 데이터 소스 정의(Amplitude 쿼리 인자) | 아티팩트 저장소 `datasets/daily` (version 2) |
| `dashboard/meta.json` | 대시보드 제목 | 아티팩트 저장소 `dash/meta` (version 1) |

`dashboard/index.html`의 주요 로직은 다음과 같습니다.

- `dash.loader('daily')`: Amplitude 응답(`data.jsonResponse.xValuesForTimeSeries` / `timeSeries[0]`)을 `{day, uv}` 행으로 변환합니다.
- `dash.calc('compare')`: 날짜별로 전주 동요일(D−7), 전월 동요일 평균(직전 달의 같은 요일 전체 평균), 목표 달성률을 계산합니다.
- `dash.calc('summary')`: 기준일(가장 최근 마감일)의 요약과 이번 달 일평균을 계산합니다.
- `dash.calc('monthly')`: 월별 집계일수, 일평균, 달성률, 전월 대비, 목표 달성일, 최고·최저값을 계산합니다. 365일 범위 때문에 중간에 잘린 첫 달은 제외합니다.
- 그리기 함수:
  - `drawCards`: 상단 카드
  - `drawChart`: d3 막대 차트
  - `drawTable`: 14일 비교표
  - `drawMonthly`, `drawCalendar`: 월별 표와 일자별 달력

같은 수치를 Slack 리포트(지시문)와 대시보드(`compare`/`summary` calc)가 **각자 따로 계산합니다.**
리뷰할 때 두 정의가 일치하는지 봐 주세요. 현재 차이는 다음과 같습니다.

- 조회 범위: 지시문 90일, 대시보드 365일
- 전월 평균 반올림: 지시문은 0.5 올림, 대시보드는 `Math.round`

이 차이 때문에 결과값이 달라지지는 않습니다.

---

## 2. 실제 코드가 저장된 위치

| 대상 | 저장 위치 | 비고 |
|---|---|---|
| Slack 자동 리포트 지시문·스케줄 | claude.ai 계정의 예약 작업 `태아보험 PV 데일리 요약` (ID `trig_01CgFVZ3sABLiFqU935s29it`) | Anthropic 서버에 저장. Claude 앱의 예약 작업 화면에서 보고 수정할 수 있음 |
| 대시보드 페이지·데이터 정의 | claude.ai 아티팩트 `태아보험 PV 데일리` (https://claude.ai/artifact/BrTwuT7BctMDoeVQuN93kL)의 내부 저장소 | 페이지는 `files/index.html`, 소스는 `datasets/daily`, 제목은 `dash/meta` |
| 대시보드 런타임(`app.js`, `app.css`, `SKILL.md` 등) | Anthropic이 제공하는 Dashboard 아티팩트 타입 | 우리 코드가 아님. 업로드 대상 아님 |
| 작업 중 임시 사본 | 이 대화의 일회성 클라우드 작업공간(scratchpad) | 세션이 끝나면 삭제됨. 원본이 아님 |

- 사용자 컴퓨터, 사내 서버, Git 저장소에 저장된 사본은 **없습니다**. 이 폴더가 첫 사본입니다.
- ⚠️ 확인 불가: 예약 작업 실행 세션 안에서 만들어진 임시 파일. 실행 세션은 매번 새로 생성되고 끝나면 사라집니다.

---

## 3. 실행에 필요한 라이브러리 및 환경 설정

### Slack 자동 리포트

| 항목 | 값 |
|---|---|
| 실행 환경 | claude.ai 예약 작업(클라우드 세션, `persist_session: false`) |
| 모델 | `claude-opus-5-5` |
| 승인 방식 | `auto`: 실행 중 승인 요청 없이 진행 |
| 사용 커넥터 | **Amplitude**(`query_amplitude_data`), **Slack**(`slack_send_message`) |
| 연결만 된 커넥터 | Google Drive, Claude Docs, Claude Code Remote, Figma, Notion. 지시문에서는 쓰지 않음(계정 기본 연결이 함께 붙음) |
| 인증 | 두 커넥터 모두 claude.ai에 연결된 **사용자 계정 OAuth**로 동작. 별도 키·토큰 설정 없음 |
| 외부 패키지 | 명시된 것 없음 |

- ⚠️ 확인 불가: 실행 세션의 Python 버전과 설치된 패키지 목록. 지시문은 "Python 등 코드로 계산"만 요구하며, 어떤 런타임이 쓰였는지는 기록이 남지 않습니다.

### 대시보드

| 항목 | 값 |
|---|---|
| 런타임 | claude.ai Dashboard 아티팩트 타입(계약 버전 `0.2.67`) |
| 라이브러리 | `d3` v7. 런타임이 전역으로 제공하며 별도 설치나 `<script>` 로드 없음 |
| 런타임 API | `dash.data`, `dash.onData`, `dash.calc`, `dash.loader`, `dash.params`, `dash.setParams`, `dash.colors` |
| 스타일 | 런타임 CSS 토큰(`--cds-*`, `--color-*`, `--font-anthropic-*`) |
| 빌드 | 없음(npm·번들러 미사용) |
| 데이터 인증 | 대시보드를 **여는 사람 각자의** Amplitude 커넥터 권한으로 조회 |

### 데이터 정의

| 항목 | 값 |
|---|---|
| Amplitude 프로젝트 | `309235` |
| 이벤트 | `ce:태아보험 PV` (사용자 정의 이벤트) |
| 지표 | `unique_users`(일별 순방문자), interval `day`, timezone `Asia/Seoul` |
| 미완료 데이터 | `excludeIncompleteDatapoints: true` (오늘 제외) |
| 조회 범위 | Slack 리포트 `Last 90 Days`, 대시보드 `Last 365 Days` |
| 목표값 | 5,000명/일. 지시문과 `index.html`에 상수로 들어 있음 |

---

## 4. 자동화 실행 방식과 스케줄

| 항목 | 값 |
|---|---|
| 스케줄 | `CRON_TZ=Asia/Seoul 48 8 * * 1-5` → **평일 08:48 KST**(정각 혼잡을 피하려 9시 직전으로 설정) |
| 수신처 | 작업 소유자 본인의 Slack DM(user ID `U090G62SX9B`) |
| 보고 대상일 | 화~금은 어제 하루, 월요일은 지난 금·토·일 3일 |
| 실패 처리 | Amplitude 조회가 실패하면 메시지를 보내지 않고 실행 결과에만 실패 사유를 남김 |
| 알림 | 푸시·이메일 꺼짐 |
| 최근 실행 | 2026-10-13 08:48:39 KST 시작, 08:49:12 종료, **SUCCEEDED** |
| 다음 실행 | 2026-10-14 08:48 KST |

1회 실행 흐름:

1. 스케줄 시각에 새 클라우드 세션 시작
2. Amplitude `query_amplitude_data` 호출(최근 90일)
3. 지시문 규칙대로 계산
4. 정해진 형식의 메시지 1건을 Slack DM으로 발송
5. 세션 종료

변경 이력(대화 기록 기준):

- 2026-10-12 16:04 KST: `#3_fh_광고_보험파트` 채널 발송으로 생성
- 이 채널은 Slack Connect(외부 공유) 채널이라 커넥터로 직접 발송할 수 없었음
  - 실행 전에 일시 중지
  - 같은 날 16:22에 수신처를 본인 DM으로 바꾸고 다시 켬
- 해당 채널에는 수동 발송용 임시 저장(Draft) 메시지 1건이 생성됨. ⚠️ 확인 불가: 이후 전송 또는 삭제 여부

대시보드는 스케줄 없이 **열거나 새로고침할 때마다** 실시간으로 조회합니다.

---

## 5. GitHub에 업로드할 파일 목록

```
fetal-insurance-pv-report/
├── README.md                          # 이 문서
├── .gitignore
├── automation/
│   ├── scheduled-task-prompt.md       # 예약 작업 지시문 원문
│   └── schedule.json                  # 스케줄·커넥터·상태(비밀 아님)
└── dashboard/
    ├── index.html                     # 대시보드 페이지 원본
    ├── meta.json                      # 대시보드 제목
    └── datasets/
        └── daily.json                 # Amplitude 쿼리 정의
```

**업로드하지 않는 것**

- Dashboard 타입 런타임 파일(`SKILL.md`, `artifact-type/*`): Anthropic 제공 파일이며 우리 코드가 아님
- 작업 중 임시 파일(`ds.json`, `page.json`, `chk.js`): 각각 이전 버전 소스 정의, 저장용 래퍼, 문법 검사용 임시본이라 원본이 아님

---

## 6. 보안: 업로드하면 안 되는 정보

### 이 자동화에서 확인된 것

- 모든 파일과 지시문에서 다음 패턴을 검사했고 **발견되지 않았습니다**:
  - API 키
  - Slack Webhook URL(`hooks.slack.com`)
  - Slack 토큰(`xoxb-`/`xoxp-`)
  - Bearer 토큰, 비밀번호, 개인키(`PRIVATE KEY`)
- 인증은 claude.ai 커넥터 OAuth가 처리하므로 **이 프로젝트가 직접 가진 비밀정보는 없습니다.**
- ⚠️ 확인 불가: 커넥터 OAuth 토큰 값. claude.ai가 보관하며 조회하거나 내보낼 수 없습니다. 저장소에 넣을 일도 없습니다.

### 비밀은 아니지만 공개 저장소라면 가릴지 검토할 식별자

| 값 | 위치 | 설명 |
|---|---|---|
| `U090G62SX9B` | 지시문, schedule.json | 작업 소유자의 Slack user ID |
| `309235` | 지시문, daily.json | Amplitude 프로젝트 ID |
| `https://claude.ai/artifact/BrTwuT7BctMDoeVQuN93kL` | 지시문 | 대시보드 주소. 공유 설정한 사람만 열 수 있어 인증정보는 아님 |

사내 private 저장소라면 그대로 두어도 무방합니다.

### 같은 계정의 다른 예약 작업에서 발견된 비밀정보 (이번 범위 밖)

- 확인 과정에서 같은 계정의 **`Notifly daily dashboard`** 예약 작업 지시문 본문에 다음이 **평문으로** 들어 있는 것을 확인했습니다.
  - Notifly Access Key / Secret Key
  - Google 서비스 계정 개인키(base64 인코딩)
- 이 작업은 이번 이관 대상이 아니라 이 폴더에 포함하지 않았고, 수정도 하지 않았습니다.
- 해당 작업을 GitHub로 옮길 때는 지시문을 그대로 커밋하지 말고, 키를 GitHub Secrets 같은 비밀 저장소로 분리해야 합니다.
- 이미 평문으로 노출된 상태이므로 키 재발급(rotation)도 검토해 주세요.
- 참고: `마케팅 KPI 데일리 리포트 (#2_fh_mkt)` 예약 작업도 태아보험 PV(`fetal_pv`)를 조회합니다. 다만 코드가 다른 아티팩트의 `pipeline/` 파일에 있는 **별개 자동화**라 이 폴더에는 포함하지 않았습니다.

---

## 7. (참고) Claude 밖에서 운영하려면

아래 항목은 **현재 존재하지 않는** 것들로, 재구현할 때 새로 만들어야 합니다.

- Amplitude API 키·시크릿(Dashboard REST API 등): 현재는 claude.ai 커넥터 OAuth를 써서 키가 없습니다.
- Slack 봇 토큰 또는 Incoming Webhook: 현재는 커넥터가 사용자 계정으로 발송합니다.
  - 봇으로 바꾸면 Slack Connect 채널(`#3_fh_광고_보험파트`)에도 발송할 수 있는지는 워크스페이스 정책에 따라 달라 ⚠️ 확인 필요합니다.
- 실행 스크립트(계산 로직은 `automation/scheduled-task-prompt.md` 2~5절과 `dashboard/index.html`의 `compare`·`summary` calc를 기준으로 구현)
- 스케줄러(GitHub Actions `schedule` 등). GitHub Actions cron은 UTC 기준이라 평일 08:48 KST는 `48 23 * * 0-4`입니다.

새로 만든 키는 GitHub Secrets에 넣고, `.gitignore`에 이미 등록된 `.env` 같은 파일로만 로컬에서 다뤄 주세요.

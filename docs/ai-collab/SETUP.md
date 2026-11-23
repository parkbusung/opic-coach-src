# Codex → Claude 순차 자동 협업 설정

Codex가 PR에 리뷰를 남기면 GitHub Actions가 조건을 확인하고 Claude 예약 작업(Routine)을 PR당 한 번 호출한다. Claude는 지적을 독립적으로 검증해 확인된 문제만 기능 브랜치에서 고친다. main 병합과 배포는 사람이 한다.

## 흐름

1. 기능 브랜치에서 main 대상 PR을 연다(사람 또는 Claude).
2. Codex 자동 리뷰(ChatGPT Plus)가 PR에 리뷰를 제출한다. 문제가 없으면 보통 리뷰 대신 짧은 코멘트만 남기며, 리뷰가 와도 인라인 지적이 없으면 다음 단계로 가지 않는다.
3. `.github/workflows/codex-to-claude.yml`이 리뷰를 검증하고 `ai-autofix-requested` 라벨을 붙인 뒤 Routine Fire URL을 한 번 호출한다.
4. Claude Routine이 리뷰를 검증하고 확인된 문제만 고쳐 테스트한 뒤 원래 브랜치에 push한다. push할 수 없으면 원래 브랜치를 대상으로 Draft PR을 만든다.
5. 사람이 결과를 보고 main 병합 여부를 정한다.

## 비용: 추가 비용 0원 조건

| 구성 요소 | 과금 기준 | 0원을 지키는 조건 |
| --- | --- | --- |
| GitHub Actions | 공개 저장소의 표준 GitHub 호스티드 러너는 무료 | 저장소를 공개로 유지하고 `ubuntu-latest`만 사용. 비공개로 바꾸면 월 무료 사용량 안에서 돌고, 결제 수단이 없으면 소진 시 실행이 막힌다 |
| Codex 자동 리뷰 | ChatGPT Plus에 포함, 사용량 한도 있음 | Codex 추가 크레딧을 구매하지 않는다 |
| Claude Routine | Pro 구독 사용량에서 차감. API 호출은 작업당 시간당 30회, 계정당 100회 | claude.ai/settings/usage의 사용량 크레딧(한도 초과 시 종량 과금)을 켜지 않는다 |
| API 키 | 사용하지 않음 | OpenAI·Anthropic Console API 키를 저장소나 Secrets에 넣지 않는다. Routine 토큰(`sk-ant-oat01-`)은 이 작업 호출 전용이다 |

## 직접 설정할 항목

이 파일들이 main에 병합돼 있어야 워크플로가 동작한다. PR은 GitHub의 PR 화면에서 **Merge pull request** → **Confirm merge**로 병합한다.

### 1. Claude 예약 작업: 먼저 진행

Claude 채팅이 아니라 **Claude Code의 예약 작업**(Scheduled tasks, 이전 이름 Routines)에서 설정한다. 웹에서 [claude.ai/code/routines](https://claude.ai/code/routines)를 열거나, 데스크톱 앱 **Code** 탭 → 왼쪽 사이드바 **Scheduled**(안 보이면 **More** 메뉴 안)에서 연다. 한국어 화면에서는 아래 메뉴 이름이 번역돼 보일 수 있다.

1. [Claude GitHub App](https://github.com/apps/claude)을 이 저장소에 설치한다. Repository access는 **Only select repositories**로 두고 `opic-coach-src`만 고른다. 설치가 없으면 작업이 저장소를 읽을 수는 있어도 push와 Draft PR 생성은 거부된다.
2. 작업을 만들거나 연다.
   - 이미 `codex-autofix` 작업이 있으면: 목록에서 작업을 누르고 이름 옆 메뉴 → **Edit**.
   - 새로 만들면: **New task**(또는 **New routine**). **Create with Claude / Set up manually** 메뉴가 뜨면 **Set up manually**를, 데스크톱 앱에서 **Local / Cloud** 메뉴가 뜨면 **Cloud**를 고른다. Local 작업은 PC가 켜져 있어야 돌고 저장소·API 트리거 설정이 없다.
3. 이름은 `codex-autofix`, 프롬프트(Instructions)에는 [`claude-routine-prompt.md`](claude-routine-prompt.md)의 프롬프트 블록 전체를 넣는다.
4. 저장소와 실행 환경을 고른다.
   - 대화 상자: 아래쪽 **Advanced settings**를 펼친다 → **General** 탭 → **Repository**에서 `parkbusung/opic-coach-src` → 그 아래 나타나는 **Cloud environment**는 **Default**(Trusted 네트워크, 테스트용 패키지 설치 가능).
   - 전체 화면 양식: **Instructions** 입력란 안의 저장소·환경 선택 버튼에서 고른다.
5. 모델은 프롬프트 입력란의 모델 선택기에서 고른다. 사용량을 아끼려면 Sonnet 계열, 상관없으면 그대로 둔다.
6. **Advanced settings** → **Connectors** 탭(전체 화면 양식은 **Connectors** 항목)에서 목록의 커넥터를 모두 뺀다. 연결된 커넥터가 기본으로 전부 들어가며, GitHub 작업은 커넥터 없이 내장 연결로 처리된다.
7. API 트리거와 토큰을 만든다.
   - 새로 만들 때: **Trigger** 행에서 **API**를 고르고 **Create** → 뜨는 **API token** 창에서 토큰을 바로 복사한다(한 번만 표시).
   - 기존 작업: **Edit** 창의 **Trigger** 행 끝 **+** → **API** → 나타난 패널에서 **Fire URL**을 복사하고 **Generate token** → **Token generated** 창의 토큰을 복사한다. 토큰은 만드는 즉시 적용된다.
   - 저장소·환경·커넥터를 바꿨다면 마지막에 **Save changes**를 누른다.
8. 웹이나 데스크톱 앱의 **설정 → 사용**(claude.ai/settings/usage)에서 **사용 크레딧**이 비활성인지, **자동 재충전**이 꺼져 있는지 확인한다. 꺼져 있으면 구독 한도에 닿았을 때 추가 요금 없이 실행이 멈춘다.

### 2. GitHub (저장소 Settings)

1. Secrets and variables → Actions → **New repository secret**
   - `CLAUDE_ROUTINE_FIRE_URL`: Fire URL (`https://api.anthropic.com/v1/claude_code/routines/trig_…/fire`)
   - `CLAUDE_ROUTINE_FIRE_TOKEN`: 토큰 (`sk-ant-oat01-…`)
2. (필요할 때만) Variables → `CODEX_BOT_LOGIN`: Codex 리뷰 작성자 계정이 `chatgpt-codex-connector[bot]`이 아니면 실제 계정명으로 설정한다. Actions 실행 요약의 "리뷰 작성자(...)" 문구에서 확인할 수 있다.
3. (권장) Rules → Rulesets → New branch ruleset: 대상 Default branch, Restrict deletions, Block force pushes, Require a pull request before merging(필요 승인 0).
4. 라벨 `ai-autofix-requested`는 워크플로가 처음 실행될 때 만든다.

### 3. Codex (chatgpt.com)

1. Codex에 GitHub를 연결하고 이 저장소에 ChatGPT Codex Connector 접근을 허용한다.
2. [Code review 설정](https://chatgpt.com/settings/code-review)에서 저장소를 고르고 **Review code → Automatic review**를 켠다. 저장소 설정이 개인 설정을 따르게 되어 있으면 **Personal preferences → Automatic review**도 켠다.
3. Review trigger는 PR을 열 때만 리뷰하도록 두기를 권장한다. 매 push마다 리뷰하게 해도 자동 수정은 PR당 1회만 실행된다.
4. 리뷰 기준은 저장소 루트 `AGENTS.md`의 `## Code Review Rules`를 따른다.

## 반복·중복 실행 방지

| 상황 | 막는 장치 |
| --- | --- |
| Claude의 수정 push 뒤 Codex가 다시 리뷰 | `ai-autofix-requested` 라벨이나 요청 기록 코멘트가 있으면 건너뜀 |
| 같은 PR에 리뷰가 거의 동시에 두 번 | PR별 concurrency로 하나씩 처리하고, API로 라벨을 다시 확인 |
| 이전 커밋에 대한 리뷰 | 리뷰 commit_id와 PR 최신 HEAD가 다르면 건너뜀 |
| Claude가 만든 Draft PR에 Codex 리뷰 | Draft PR은 main 대상이 아니고 라벨도 붙어 있어 건너뜀 |
| 워크플로가 붙인 라벨·코멘트 | 다른 이벤트 종류이고, GITHUB_TOKEN이 만든 이벤트는 워크플로를 다시 실행하지 않음 |
| Routine 호출 실패·시간 초과 | 재시도하지 않음. 라벨이 남아 다시 실행되지 않고 실패 코멘트만 남김 |
| Routine이 같은 리뷰로 두 번 실행됨 | 프롬프트가 `claude-autofix:done` 기록을 보고 종료 |
| 토큰 유출 후 임의 호출 | Routine이 저장소·라벨·Codex 리뷰를 GitHub에서 직접 확인한 뒤에만 작업 |
| Claude 코멘트가 Codex를 다시 부름 | Claude는 리뷰를 제출하지 않고 `@codex`를 쓰지 않음 |
| Codex가 아닌 계정의 리뷰 | 로그인과 Bot 유형이 Codex 봇과 다르면 건너뜀 |

## 다시 실행하기

- 자동 경로는 PR당 1회다. 다시 돌리려면 Claude 웹의 작업 상세 페이지에서 **Run now**에 같은 형식의 텍스트를 넣는다.
- 호출이 실패했던 PR은 `ai-autofix-requested` 라벨을 지우고 Codex 리뷰를 다시 받으면 한 번 더 요청된다. 성공 기록 코멘트(`<!-- codex-to-claude:fired -->`)가 있는 PR은 자동으로 다시 요청되지 않는다.

## 문제 해결

| 증상 | 확인할 것 |
| --- | --- |
| 실행이 "건너뜀"으로 끝남 | Actions 실행 요약의 사유 |
| 작업이 skipped로 표시됨 | 리뷰 작성자가 봇이 아니거나, main 대상·같은 저장소 PR이 아님 |
| HTTP 401 | 토큰 불일치. Routine에서 토큰을 다시 만들고 Secret을 갱신 |
| HTTP 400 | Routine이 꺼져 있거나 요청 헤더 문제. 작업이 켜져 있는지 확인 |
| HTTP 429 | 시간당 호출 한도. 재시도하지 않으므로 Run now로 수동 실행 |
| Routine이 push나 Draft PR 생성에 실패 | Claude GitHub App이 이 저장소에 설치돼 있는지 확인 |

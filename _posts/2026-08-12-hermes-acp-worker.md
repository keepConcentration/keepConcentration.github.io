---
layout: post
title: "Hermes ACP 워커를 고쳤지만 PR은 닫았다"
date: 2026-08-12 00:03:16 +0900
categories: [Hermes Agent, Open Source]
tags: [hermes, open-source, acp, python, github]
---

Discord에서 Hermes의 코딩 워커에게 파일 수정을 맡겼다. 파일을 읽고 원인을 찾는 데까지는 문제가 없었다. 근데 정작 수정하려는 순간 모든 작업이 막혔다.

```text
User refused permission to run tool
Tool permission request failed: Error: Tool use aborted
```

처음에는 프로필 설정이 잘못됐나 싶었다. 몇 번 다시 실행해 봤지만 결과는 똑같았다. `git status`나 파일 읽기는 되는데, `Edit`, `Write`, `git fetch`, 파일을 쓰는 쉘 명령은 죄다 거부됐다.

문제는 작업 내용이 아니라 ACP 권한 처리 쪽에 있었다.

## 답할 사람이 없는 승인 요청

Hermes의 `copilot-acp` provider는 Copilot CLI나 Claude 계열 ACP agent와 JSON-RPC로 통신한다. Agent가 Bash나 파일 수정 도구를 쓰려고 하면 `session/request_permission`을 보낸다.

대화형 환경에서는 사용자가 승인 버튼 누르면 끝이다. 근데 Discord gateway, cron, Kanban worker 같은 headless 환경에는 승인 창을 눌러 줄 사람이 없다.

<img src="/assets/acp-permission-flow.png" alt="ACP Permission Flow" width="600" />

당시 `agent/copilot_acp_client.py`의 처리는 단순했다.

```python
if method == "session/request_permission":
    response = _permission_denied(message_id)
```

어떤 옵션이 들어와도 결과는 `cancelled`. 그래서 worker는 코드를 읽고 분석할 수는 있어도 실제 수정은 못 했다.

### 먼저 온 사람들이 있었다

코드부터 고치기 전에 GitHub 이슈를 찾아봤다. Copilot ACP permission 관련해서 이미 #17284가 열려 있었다. Hermes가 무조건 거절하고, Copilot CLI는 답변 없이 턴을 끝내고, Hermes는 빈 응답을 받아 무한 재시도한 끝에 `(empty)`를 반환한다는 내용이었다. 내가 겪은 현상 그대로더라.

#17284에 연결된 PR도 두 개 있었다.

- **#17285**: 빈 응답을 받으면 follow-up prompt를 다시 보내는 방식. 이미 **Closed** 상태였다.
- **#17604**: reasoning을 텍스트 fallback으로 돌려주는 방식. Open 상태였지만 수개월째 stale이었다. 리뷰에서 `end_turn` 시그널 누락, thought 텍스트 노출 위험 같은 지적이 달려 있었다.

둘 다 접근 방식이 비슷했다. **"거절당한 뒤에 어떻게 복구할까"** 를 고민하고 있었다. 내 경우는 상황이 좀 달랐다. headless 환경이라 애초에 거절 자체가 의미가 없었다. 승인 UI 없이 permission request를 날리는 구조 자체를 우회할 방법이 필요했다.

## 이미 등록된 환경변수가 연결되지 않았다

코드를 따라가다 보니 `HERMES_ACP_AUTO_APPROVE`가 이미 환경변수 로딩 대상에 들어가 있었다.

```python
_PROFILE_MANAGED_ENV_KEYS = frozenset({
    "HERMES_ACP_AUTH_METHOD",
    "HERMES_ACP_AUTO_APPROVE",
    "HERMES_COPILOT_ACP_COMMAND",
    "HERMES_COPILOT_ACP_ARGS",
    "COPILOT_CLI_PATH",
    "COPILOT_ACP_BASE_URL",
})
```

`hermes_cli/config.py`에도 같은 키가 등록돼 있었다. 환경변수는 읽을 준비가 돼 있었는데, 실제 permission handler에서는 그 값을 쓰지 않고 있었다. 선언만 되고 미배선 상태였던 거다.

수정 방향은 명확했다.

```text
HERMES_ACP_AUTO_APPROVE가 꺼져 있음
  → permission request를 cancelled로 처리 (기존 동작 그대로)

HERMES_ACP_AUTO_APPROVE가 켜져 있음
  → ACP agent가 제공한 허용 옵션을 선택
```

기본 동작은 그대로 두고, 운영자가 명시적으로 설정한 경우에만 자동 승인하게 했다.

## ACP가 제공한 옵션 안에서만 고르기

처음에는 `allow_always`를 선택하면 충분하다고 생각했다. 근데 ACP permission 응답은 임의의 값을 반환하는 구조가 아니더라.

요청에는 선택 가능한 옵션이 함께 들어온다.

```json
{
  "options": [
    {
      "kind": "allow_always",
      "name": "Always allow Bash",
      "optionId": "allow_always"
    },
    {
      "kind": "allow_once",
      "name": "Allow",
      "optionId": "allow"
    }
  ]
}
```

응답은 제공된 `optionId` 중 하나를 골라야 한다.

```json
{
  "outcome": {
    "outcome": "selected",
    "optionId": "allow_always"
  }
}
```

Plan mode를 빠져나오는 요청처럼 `kind`는 `allow_always`지만 `optionId`가 `bypassPermissions`인 경우도 있었다. 그래서 문자열 하나를 하드코딩하지 않고 요청에 포함된 옵션을 해석하는 쪽으로 갔다.

최종 선택 규칙:

1. 제공된 `allow_always` 옵션을 먼저 고른다
2. `allow_always`가 없으면 제공된 `allow_once`를 고른다
3. 허용 옵션이 하나도 없으면 `cancelled`를 반환한다
4. 요청에 없는 `optionId`는 절대 만들지 않는다

마지막 규칙이 중요했다. 초기 구현에는 허용 옵션을 찾지 못했을 때 문자열 `allow_always`를 반환하는 fallback이 있었다. 그런데 요청에 없는 옵션을 만들어 응답하면 ACP 규격에 맞지 않는다. 테스트를 먼저 실패시킨 뒤, 허용 옵션이 없으면 그냥 닫힌 상태로 실패하게 고쳤다.

핵심 코드는 이렇게 정리됐다.

```python
def _permission_response(message_id, params):
    if _acp_auto_approve_enabled():
        option_id = _pick_auto_approve_option_id(params)
        if option_id is not None:
            return _permission_selected(message_id, option_id)
    return _permission_denied(message_id)
```

## 자동 승인이라도 기본값은 거절이다

이 기능은 편리하지만 권한 경계를 바꾼다. 자동 승인을 켜면 ACP agent가 요청한 Bash와 파일 수정 같은 작업도 승인될 수 있다.

그래서 `HERMES_ACP_AUTO_APPROVE`는 opt-in으로 유지했다.

```python
def _acp_auto_approve_enabled() -> bool:
    raw = os.getenv("HERMES_ACP_AUTO_APPROVE", "").strip().lower()
    return raw in {"1", "true", "yes", "on"}
```

환경변수가 없거나 false 값이면 기존처럼 `cancelled`다. 신뢰할 수 있는 ACP provider를 운영자가 통제하는 환경에서만 켜야 한다. 일반 사용자 환경의 기본 권한 정책은 바뀌지 않는다.

문서에도 이 점을 같이 적었다. 기능 설명만 쓰고 위험을 숨기면 안 되는 종류의 설정이다.

## 테스트 환경도 실사용 Hermes와 분리했다

내 MacBook에는 실제 Discord gateway를 처리하는 Hermes가 계속 실행 중이었다. 오픈소스 기여용 테스트가 이걸 건드리면 안 됐다.

두 경로를 분리했다.

```text
실사용 Hermes
~/.hermes/hermes-agent

오픈소스 기여용 clone
~/Desktop/hermes-agent
```

기여용 가상환경도 별도로 만들었다.

```text
~/.hermes/venvs/hermes-contrib
```

저장소의 `.venv`는 이 가상환경을 가리키게 했다. Hermes 공식 테스트 runner는 로컬 `.venv`를 먼저 찾기 때문에 기여용 Python과 dependency만 사용한다.

테스트는 `pytest`를 직접 실행하지 않고 프로젝트의 공식 runner를 썼다.

```bash
scripts/run_tests.sh tests/agent/test_copilot_acp_client.py -q
```

최종 결과:

```text
11 tests passed, 0 failed
```

새 테스트는 기본 거절, 일반 `allow_always`, mode switch의 `bypassPermissions`, `allow_once` fallback, 허용 옵션이 없을 때의 거절을 확인한다.

전체 테스트도 작업 브랜치와 최신 `main`에서 각각 돌렸다. 두 환경 모두 같은 테스트 91개가 실패했고 collection error도 같았다. 외부 서비스와 하드웨어에 의존하는 기존 실패였고, 작업 브랜치에서만 새로 생긴 실패는 없었다.

## PR을 열고 중복을 발견했다

코드와 테스트를 마친 뒤 PR #83958을 열었다. 얼마 지나지 않아 먼저 열린 PR #83205와 핵심 구현이 겹친다는 댓글이 달렸다.

확인해 보니 두 PR 모두 `agent/copilot_acp_client.py`에서 기본값은 거절로 유지하고, 운영자가 명시적으로 켰을 때 ACP가 제공한 허용 옵션을 선택한다. #83205가 먼저 열린 구현이었으니 내 PR을 계속 유지할 이유는 없었다.

처음에 이슈 검색할 때 #17284에 연결된 복구 접근 PR들만 보고, 같은 자동 승인 구현을 놓친 거다. 검색했다고 체크박스 하나 채운 게 충분한 검색은 아니었던 셈이다.

그래서 #83958은 내가 직접 닫았다.

근데 작업한 내용이 전부 사라진 건 아니었다. 두 구현 사이에는 먼저 열린 PR에 전달할 만한 차이가 있었다.

첫째, Hermes에는 `HERMES_ACP_AUTO_APPROVE`가 이미 환경 로딩과 설정 코드에 등록돼 있었다. #83205는 별도의 `HERMES_COPILOT_ACP_AUTO_APPROVE`를 새로 추가하고 있었다. 같은 동작을 위한 환경변수를 하나 더 만들기보다 기존 값을 재사용하면 된다는 점을 댓글로 남겼다.

둘째, 내 PR에는 환경변수 문서가 들어 있었다. 기본값이 거절이라는 점과 신뢰할 수 있는 운영 환경에서만 켜야 한다는 보안 주의사항도 함께였다.

셋째, mode switch 요청을 다루는 테스트가 있었다. 이 요청은 `kind`가 `allow_always`여도 `optionId`가 `allow_always`가 아니라 `bypassPermissions`일 수 있다. 하드코딩한 문자열이 아니라 실제 제공된 `optionId`를 선택해야 한다는 걸 보여주는 사례다.

이 세 가지를 #83205에 전달하고 내 PR을 닫았다. 머지되는 커밋은 안 남았지만, 먼저 열린 구현을 보완할 정보는 남길 수 있었다.

## 이번에 배운 것

처음에는 permission error 몇 줄만 보고 프로필 설정을 의심했다. 실제 원인은 환경변수가 없어서가 아니라, 이미 등록된 환경변수와 실제 동작 사이의 연결이 빠져 있었던 거다.

또 하나. 프로토콜 값을 함부로 만들어내면 안 된다. `AUTO_APPROVE`라는 이름만 보면 `allow_always`를 반환하면 될 것 같지만, ACP는 요청이 제공한 옵션 중 하나를 선택하게 정해져 있다. `allow_once`만 제공하는 agent도 있을 수 있고, mode switch처럼 option ID가 전혀 다를 수도 있다.

그리고 이슈 트래커 검색. #17284랑 연결된 PR만 보고 "이거랑 다른 접근이 필요하네"라고 판단한 건 맞았지만, 더 넓게 검색했으면 #83205를 먼저 발견했을 거다. 검색 범위를 타이틀이나 연결된 이슈만 보지 말고 코드 변경 자체로도 찾아봤어야 했다.

이번 기여는 코드 고치는 시간보다 경계를 확인하는 시간이 더 길었다. 실사용 환경과 테스트 환경을 나누고, 최신 main과 실패 결과를 비교하고, 기존 이슈와 PR을 읽고, 자동 승인의 보안 의미를 문서에 적었다. permission handler 하나 바꾸는 일도 실제 운영 환경에 연결되면 확인할 게 꽤 많더라.

그래도 출발점은 단순했다. Discord에서 코딩 worker가 파일을 읽기만 하고 고치지는 못했다. 그 불편을 따라가다 보니 재현 가능한 문제와 테스트, 문서가 남았다.

## 관련 링크

- [Hermes Agent](https://github.com/NousResearch/hermes-agent)
- [Issue #17284 — permission request gets cancelled](https://github.com/NousResearch/hermes-agent/issues/17284)
- [PR #17285 — follow-up prompt 재전송 (Closed)](https://github.com/NousResearch/hermes-agent/pull/17285)
- [PR #17604 — reasoning text fallback (Stale)](https://github.com/NousResearch/hermes-agent/pull/17604)
- [먼저 열린 PR #83205](https://github.com/NousResearch/hermes-agent/pull/83205)
- [내가 닫은 PR #83958](https://github.com/NousResearch/hermes-agent/pull/83958)
- [ACP Tool Calls and Permission Requests](https://agentclientprotocol.com/protocol/v1/tool-calls)
---
layout: post
title: "Hermes Agent에 기여하기: 프로필 환경변수 누수 버그를 고치기까지"
date: 2026-08-02 18:00:00 +0900
tags: [hermes, open-source, contribution, python, debugging]
---

## 시작: 운영 중 발생한 의문의 버그

헤르메스라는 AI 에이전트를 다중 프로필로 운영하고 있었다. `all-rounder` 프로필은 Cursor ACP 기반, `coder` 프로필은 Claude Code ACP 기반으로 작동하도록 설정한 상태. 그런데 `coder` 프로필의 Discord 봇으로 메시지를 보내면 이런 응답이 돌아왔다.

> The model provider failed after retries.

게이트웨이 로그를 까보니 원인은 이거였다.

```
Copilot ACP authenticate failed: Internal error
```

이상했다. `coder` 프로필은 Claude Code CLI를 사용하도록 설정했고, Claude Code ACP 서버는 `authMethods: []`를 반환하는데 `authenticate` 호출이 발생할 이유가 없었다.

## 원인 추적: 상속된 환경변수

로그를 따라가면서 발견한 건, `coder` 프로필의 `os.environ`에 **내가 작성하지 않은 환경변수**가 들어 있다는 사실이었다.

```
HERMES_ACP_AUTH_METHOD=cursor_login
```

이 값은 `all-rounder` 프로필의 `.env`에만 설정한 값이다. 그런데 `coder` 프로필을 재시작할 때 **부모 프로세스로부터 환경변수를 그대로 상속**받으면서, `all-rounder`의 ACP 설정이 `coder`로 새어 들어간 것이다.

헤르메스는 **프로필마다 격리된 `.env`를 제공**한다고 문서화되어 있었다. 실제로 코드를 보니 핫 리로드(`reload_env()`)에서는 `.env`에 없는 known Hermes 키를 `os.environ`에서 삭제하는 로직이 있었다. 그런데 **최초 구동 시점(`load_hermes_dotenv()`)** 에는 이 삭제 로직이 빠져 있었다. `.env`에 정의된 키만 override하고, `.env`에 없는데 부모로부터 상속된 키는 그대로 남겨둔 것이다.

## 이슈 제보

레포를 뒤져보니 비슷한 문제를 다룬 이슈가 세 개 있었다. [#68367](https://github.com/NousResearch/hermes-agent/issues/68367)은 Desktop spawn 환경변수 상속, [#66930](https://github.com/NousResearch/hermes-agent/issues/66930)은 bare 프로필에 Matrix credential 상속, [#61046](https://github.com/NousResearch/hermes-agent/issues/61046)은 multi-profile gateway에서 `.env`를 아예 못 읽는 문제였다. 전부 "격리 누수"라는 공통점은 있지만, 내가 겪은 건 CLI/게이트웨이 재시작 시 ACP 설정이 새는 케이스라 기존 이슈들과는 결이 달랐다.

게다가 ACP 관련 키들은 `OPTIONAL_ENV_VARS`에도 등록이 안 되어 있어서, 기존 클린업 로직으로는 아예 잡히지도 않는 상태였다. 새 이슈로 파기로 하고 [#75141](https://github.com/NousResearch/hermes-agent/issues/75141)을 열었다.

이슈에 포함한 내용:
- 버그 설명과 기존 이슈와의 차이점
- 재현 단계
- 기대 동작 vs 실제 동작
- 게이트웨이, 에이전트, ACP 서버 3계층에 걸친 디버그 로그
- 코드 레벨 근본 원인 분석
- 임시 해결책 (`HERMES_ACP_AUTH_METHOD=` 빈 값 할당)

## PR 작성: 구현부터 테스트까지

이슈 제보 후 바로 코드를 작성했다.

### 변경 내용

**1. `hermes_cli/config.py`** ACP 관련 키 6개를 `_EXTRA_ENV_KEYS`에 등록해서 known-key 클린업 대상에 포함:

```python
"HERMES_ACP_AUTH_METHOD",
"HERMES_ACP_AUTO_APPROVE",
"HERMES_COPILOT_ACP_COMMAND",
"HERMES_COPILOT_ACP_ARGS",
"COPILOT_CLI_PATH",
"COPILOT_ACP_BASE_URL",
```

**2. `hermes_cli/env_loader.py`** `_clear_known_keys_missing_from_dotenv()` 함수 구현. `.env` 로드 후, `.env`에 정의되지 않은 known Hermes 키를 `os.environ`에서 삭제:

```python
def _clear_known_keys_missing_from_dotenv(
    dotenv_keys: set[str],
    *,
    optional_env_vars: list[str] | None = None,
    extra_env_keys: list[str] | None = None,
) -> None:
    # ...
```

**3. `tests/hermes_cli/test_env_loader.py`** 회귀 테스트 4개 추가:
- 상속된 키가 클린업되는지
- `KEY=` 빈 값 할당도 정상 동작하는지
- bare 프로필(no `.env`)은 건드리지 않는지
- 명시적 override는 유지되는지

### 테스트 결과

헤르메스는 `pytest`를 bare로 돌리면 파일 간 mock 오염으로 **16,000개 가짜 실패**가 발생하는 특이한 구조라, 공식 러너인 `scripts/run_tests.sh`를 사용해야 한다. (CI와 동일한 파일별 서브프로세스 격리 환경)

```
$ scripts/run_tests.sh tests/hermes_cli/test_env_loader.py -q
9/9 passed (신규 4개 포함)

$ scripts/run_tests.sh tests/hermes_cli/test_web_server.py::TestReloadEnv -q
2/2 passed (hot-reload 패리티 유지)
```

전체 테스트: 22,783 passed / 72 failed (99.7%). 72개 실패는 Daytona, FAL, voice, systemd 등 환경 의존적 테스트로 `main` 브랜치에서도 동일하게 실패하는 것들.

## 리뷰: hermes-sweeper와 메인테이너의 피드백

PR을 올린 지 몇 시간 만에 리뷰가 달렸다. hermes-sweeper(자동화된 리뷰 봇)와 메인테이너의 리뷰였다.

### 지적 1: `export ` 접두어 파싱 버그

```bash
# .env 파일
export HERMES_ACP_AUTH_METHOD=cursor_login
```

내 코드는 `.env` 파일을 `KEY=VALUE`로 파싱할 때 `=` 앞까지를 키 이름으로 잘랐다. 그런데 `export KEY=VALUE` 문법에서는 `export ` 접두어가 포함된 채로 키 이름이 인식되는 버그가 있었다.

헤르메스의 기존 파서는 이 접두어를 스트립하고 있었고, 심지어 전용 테스트 파일(`test_env_export_prefix.py`)까지 존재했다.

**수정:** `line[7:]` 스트립 추가 (기존 파서 관례와 일치). 회귀 테스트 `test_export_prefixed_known_key_in_user_env_is_kept` 추가.

```python
if line.startswith("export "):
    line = line[7:]
```

### 지적 2 (내가 발견): 클린업 범위가 너무 넓다

전체 테스트 스위트를 돌리다가 `test_dump_flags_shell_only_key_not_in_dotenv`가 실패하는 걸 발견했다. 내 코드가 **모든 known Hermes 키**(`OPTIONAL_ENV_VARS` 전체)를 대상으로 클린업을 수행하면서, 사용자가 셸에서 의도적으로 export한 API 키(`FIRECRAWL_API_KEY`, `OPENAI_API_KEY` 등)까지 삭제해버린 것. 이는 `hermes debug`의 "shell only" 계약을 위반하는 동작이었다.

리뷰어에게 질문 코멘트를 남겼다:

> Should I narrow the startup cleanup to just the profile/ACP-managed keys (`HERMES_ACP_*`, `HERMES_COPILOT_ACP_*`, `COPILOT_*`) instead of the full `OPTIONAL_ENV_VARS` set?

## 통합 머지: PR 3개가 하나로

며칠 뒤, 메인테이너가 새로운 PR을 열었다. [#76462](https://github.com/NousResearch/hermes-agent/pull/76462) "fix(secrets): finish profile secret-scope migration."

이 PR은 **같은 문제 영역(프로필 credential 격리)을 다루던 3개의 PR을 한데 모아 통합**하는 PR이었다:

| 원본 PR | 작성자 | 내용 |
|---------|--------|------|
| [#67065](https://github.com/NousResearch/hermes-agent/pull/67065) | @webtecnica | `get_env_value()` scope-aware |
| [#51604](https://github.com/NousResearch/hermes-agent/pull/51604) | @JoaoMarcos44 | Anthropic adapter credential isolation |
| **#75197** | **@keepConcentration** | **ACP env scrub** |

내 코드는 **cherry-pick**되어 #76462에 그대로 포함됐다. 커밋 히스토리에 내 계정이 co-author로 찍혀 있고, PR 본문에도 "Cherry-picked from #75197 (@keepConcentration)"라고 명시되어 있다.

그리고 내가 리뷰에서 제기했던 클린업 범위 문제도 이 PR에서 해결됐다. 원래 `OPTIONAL_ENV_VARS` 전체를 대상으로 하던 클린업을 **프로필 관리 키(`_PROFILE_MANAGED_ENV_KEYS`)로만 제한**하고, 셸 export 키는 건드리지 않도록 수정한 것. 내 질문이 반영된 셈이다.

```
fix(env): narrow startup env scrub to profile-managed ACP keys

The salvaged cleanup (#75197) scrubbed every known Hermes key absent from
the profile .env — deleting user-shell-exported credentials
(export OPENAI_API_KEY=...) on every hermes invocation, a documented flow
the author's own failing test_dump_flags_shell_only_key_not_in_dotenv
confirmed. A child process cannot distinguish shell exports from
parent-process leakage, so the scrub now covers ONLY
_PROFILE_MANAGED_ENV_KEYS (ACP routing keys: HERMES_ACP_*,
HERMES_COPILOT_ACP_*, COPILOT_CLI_PATH, COPILOT_ACP_BASE_URL) —
the vector from #75141.
```

## 회고

지금까지 오픈소스 기여는 주로 good first issue를 찾거나, 커뮤니티에서 가이드를 받아서 진행했었다. 그런데 이번에는 달랐다. 내가 직접 운영하는 환경에서 마주친 버그를 스스로 분석하고, 이슈를 제보하고, 코드를 작성해서 PR까지 올린, 처음부터 끝까지 내 손으로 완주한 첫 기여였다.

전체 과정을 돌아보면:

1. **버그 발견** 프로덕션 환경에서 마주친 실제 문제
2. **근본 원인 분석** `load_hermes_dotenv()`와 `reload_env()`의 비대칭성을 코드 레벨에서 추적
3. **이슈 차별화** 기존 이슈 3건을 조사해 새 이슈의 정당성을 입증
4. **구현** `env_loader.py`, `config.py`, 테스트 파일 3건 수정
5. **리뷰 대응** `export ` 접두어 파싱 버그 수정 + 회귀 테스트 추가
6. **자체 QA** 전체 테스트 스위트에서 셸 export 키 삭제라는 부작용 발견 및 보고
7. **업스트림 통합** 코드가 cherry-pick으로 머지되고, 저자권 보존 + Credits 명시

228k star를 가진 대형 오픈소스 프로젝트에, 그것도 프로필 격리라는 보안 경계를 건드리는 변경을 기여했다는 점이 개인적으로 의미 있었다. CI/CD, 컨벤션, 리뷰 문화, 테스트 철학까지 짧은 기간이었지만 실제 오픈소스 메인테이닝의 워크플로우를 체험할 수 있었다.

무엇보다 이슈를 분석한 내용이 그대로 코드가 되어, 다른 기여자들의 PR과 함께 통합되고, 내 계정이 커밋 히스토리에 남았다는 사실이 뿌듯하다.

**관련 링크:**

- [Issue #75141 — Profile `.env` does not clear inherited Hermes env vars](https://github.com/NousResearch/hermes-agent/issues/75141)
- [PR #75197 — fix(env): clear inherited Hermes keys missing from profile `.env` (ACP leak)](https://github.com/NousResearch/hermes-agent/pull/75197)
- [PR #76462 — fix(secrets): finish profile secret-scope migration](https://github.com/NousResearch/hermes-agent/pull/76462)
- [Hermes Agent Repository](https://github.com/NousResearch/hermes-agent)
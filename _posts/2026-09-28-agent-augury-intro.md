---
layout: post
title: "AgentRadio 까보고 나서, 내가 만든 agent-augury"
date: 2026-09-28 17:21:00 +0900
categories: [agent, ai]
tags: [agent-augury, multi-agent, local-ai, python, hitl]
---

<!--
제목 대안:
1. 로컬에서 에이전트 팀을 돌리고 싶어서 만든 agent-augury
2. 모델 가리지 않는 로컬 멀티에이전트 런타임, agent-augury
-->

친구가 AgentRadio를 추천했다. 까봤다. 그 이야기는 [이전 글](/ai/tool/2026/08/16/agentradio-review.html)에 썼다.

그 글에서는 Hermes 워크플로우랑 안 맞아서 도입을 접었다고 했다. 근데 그걸로 끝이 아니었다. 까보면서 느낀 불편이 따로 남았다.

Claude 모델에 묶인 느낌이 강했다. 논문 결과물 같은 인상도 있었다. 돌리려면 깔아야 할 프로그램도 많았다. 범용으로 쓰긴 어렵겠다는 생각이 들었다.

그래서 직접 만들기로 했다. 그게 **agent-augury**다.

## agent-augury가 뭔가

한 줄로 말하면, 로컬 Python에서 LLM 에이전트 여러 명을 팀으로 돌리는 런타임이다.

에이전트들이 공유 스레드에서 `@mention`으로 말하고, 역할(role)을 갖고, 사람이 중간에 끼어들 수 있다. 모델은 특정 벤더에 묶이지 않는다. OpenAI 호환 API, Nous Portal(API 키/OAuth) 같은 백엔드를 에이전트마다 섞을 수 있다.

설치는 pip 한 줄이다.

```bash
pip install agent-augury
```

YAML로 에이전트를 짜거나, 위저드로 세션을 만든 뒤 터미널에서 돌린다. 대화형 UI는 Ink Surface다. Node.js가 필요하고, 첫 실행 때 Ink 프론트가 캐시에 풀린다. Discord/Slack 미러도 붙일 수 있다. 헤드리스로만 돌리는 옵션도 있다.

## 일하면서 듣게 하고 싶었다

멀티에이전트에서 제일 짜증 나는 패턴이 있다. 동료 메시지를 받으려고 일을 멈추는 거다.

agent-augury는 그걸 피하려고 수신을 전경 대기가 아니라 inbox push로 둔다. 메시지는 대상 에이전트 inbox에 들어가고, 다음 `step()`에서 컨텍스트로 흡수된다. 에이전트는 멈추지 않고 계속 일한다.

통신 명령은 세 가지다.

- `create_thread` — 이름 있는 스레드를 연다
- `send_message` — 스레드에 메시지를 붙이고 바로 돌아온다 (`mentions`가 비면 방송)
- `read_resource` — 필요할 때 스레드/메시지 스냅샷을 본다

진실은 외부 채팅 앱이 아니다. 내부 메시지 서버가 SSOT다. Discord나 Slack은 그 위를 비추는 표면이다. 프로토콜 상태를 채널 규약에 욱여넣지 않으려고 이렇게 나눴다.

## 역할과 사람

에이전트마다 페르소나를 붙일 수 있다. config의 `roles`에 프롬프트를 두고, 에이전트에 `role: orchestrator`처럼 걸면 된다. 인라인 `role_custom`도 된다. 에이전트 id는 고유해야 하고, `human`은 예약어다.

사람은 1급 참여자다. 에이전트가 `ask_user`로 물어볼 수 있고, Ink에서 `@agent-id`로 중간에 끼어들 수도 있다. 사람 말도 에이전트 메시지와 같은 inbox 경로로 들어간다.

구조화된 팀 작업이 필요하면 P1~P5 프로토콜을 켤 수 있다. 탐색, 분할, 실행, 교차검토, 제출이다. 게이트는 `PROPOSE:` / `APPROVE:` 같은 명시 신호로만 열린다. 자유 형식 세션에서는 그냥 끄면 된다.

## 왜 이런 모양으로 만들었나

설계 원칙은 README에 적혀 있다. 내가 중요하게 본 건 네 개다.

첫째, 들으면서도 계속 일하게 한다. 동료 트래픽 때문에 blocking wait를 강제하지 않는다.

둘째, 에이전트 한 명이 진실이 아니다. 공유 상태는 메시지 서버가 가진다.

셋째, 표면은 뷰다. Ink, Discord, Slack은 관측하거나 상호작용하는 창이지, 프로토콜 상태를 소유하지 않는다.

넷째, 모델은 갈아끼울 수 있다. 통신 규칙은 런타임에 두고, 벤더 SDK에 얹지 않는다.

AgentRadio에서 느낀 한계랑 대비하면 이렇게 정리된다. 모델 한정은 model-agnostic 백엔드로, 연구용 스택/의존성 무게는 로컬 Python 패키지 하나로, 장수명 협업 전용 느낌은 범용 멀티에이전트 세션(스레드, 비차단 전달, role, HITL)으로 풀었다.

논문 벤치마크를 재현하려는 코드가 아니다. 실제로 팀처럼 돌릴 도구를 목표로 했다.

## 그래서 나는 이걸 뭐에 쓰나

에이전트 여러 명이 역할을 나눠 토론하고, 내가 중간에 끼어들고, 모델은 그때그때 바꾸고 싶을 때. 그 장면을 로컬에서 재현하고 싶었다.

아직 베타(PyPI 기준 0.7.x)다. 완벽하다고 말하진 않겠다. 다만 "까보고 아쉽던 지점"을 내 손으로 메운 결과물이긴 하다.

코드와 문서는 여기 있다.

[https://github.com/keepConcentration/agent-augury](https://github.com/keepConcentration/agent-augury)

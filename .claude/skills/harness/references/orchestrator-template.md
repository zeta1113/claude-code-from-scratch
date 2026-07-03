# 오케스트레이터 스킬 템플릿

에이전트 팀 또는 서브에이전트를 조율하는 오케스트레이터 스킬의 세 가지 구현 패턴.

---

## 실행 모드 선택

```
에이전트 간 실시간 통신이 필요한가?
  ├─ YES → 템플릿 A (에이전트 팀 모드)
  └─ NO: 에이전트가 결과를 독립적으로 반환하는가?
        ├─ YES → 템플릿 B (서브 에이전트 모드)
        └─ Phase마다 다름 → 템플릿 C (하이브리드 모드)
```

**기본값: 템플릿 A** — 2명 이상이고 통신이 필요하면 에이전트 팀이 기본이다.

---

## 템플릿 A: 에이전트 팀 모드

팀원들이 SendMessage로 직접 통신하며 TaskCreate/TaskUpdate로 작업을 공유 관리.

```markdown
---
name: {orchestrator-name}
description: "{도메인} 에이전트 팀을 조율. {핵심 기능 설명}. {트리거 상황}.
  재실행, 보완, 업데이트 시에도 이 스킬을 사용할 것."
model: opus
---

# {Orchestrator Name}

당신은 {도메인} 에이전트 팀의 오케스트레이터입니다.

## 실행 모드: 에이전트 팀

TeamCreate로 팀을 구성하고 SendMessage + TaskCreate로 조율한다.

## Phase 0: 초기화

`_workspace/` 디렉토리를 확인한다:
- 없음 → 신규 실행: `_workspace/` 생성
- 있음 + 완료 표시 → 새 실행: 이전 결과 보존, 새 서브디렉토리 생성
- 있음 + 미완료 → 재개: 미완료 Phase부터 재시작

## Phase 1: 입력 분석

입력을 분석하고 `_workspace/01_analysis.md`에 저장:
- 핵심 목표
- 예상 산출물
- 팀 구성 결정

## Phase 2: 팀 구성

TeamCreate로 팀을 구성한다:

```
TeamCreate({
  members: [
    { agentType: "{agent-1}", role: "{역할 설명}" },
    { agentType: "{agent-2}", role: "{역할 설명}" },
    { agentType: "qa-inspector", role: "품질 검증" }
  ]
})
```

TaskCreate로 초기 작업 목록을 생성한다.

## Phase 3: 주요 작업

팀원들에게 SendMessage로 작업을 배분한다. 팀원들은:
- TaskUpdate로 진행 상황을 업데이트
- SendMessage로 서로 직접 통신하며 조율
- 발견 사항을 공유하여 작업 방향 조정

## Phase 4: 통합 및 검증

- 모든 팀원의 산출물 수집
- QA 에이전트에게 검증 요청
- `_workspace/04_integrated.md`에 통합 결과 저장

## Phase 5: 완료

`_workspace/DONE` 파일 생성. 사용자에게 최종 결과 보고.

## 에러 핸들링

팀원 실패 시: 해당 작업을 TaskUpdate로 재배분하거나 직접 처리.
```

---

## 템플릿 B: 서브 에이전트 모드

에이전트 간 통신 불필요. Agent 도구로 병렬 호출, 결과를 메인이 수집.

```markdown
---
name: {orchestrator-name}
description: "{도메인} 서브에이전트들을 조율. {핵심 기능 설명}. {트리거 상황}.
  재실행, 보완, 수정 요청에도 이 스킬을 사용할 것."
model: opus
---

# {Orchestrator Name}

당신은 {도메인} 작업을 서브에이전트들에게 분배하고 결과를 통합하는 오케스트레이터입니다.

## 실행 모드: 서브 에이전트

Agent 도구로 직접 호출. 에이전트들은 독립적으로 실행하고 결과를 반환.

## Phase 0: 초기화

`_workspace/` 확인 및 준비 (템플릿 A와 동일).

## Phase 1: 입력 분석

입력을 분석하고 서브에이전트별 작업 분배 계획을 수립.
`_workspace/01_plan.md`에 저장.

## Phase 2: 병렬 실행

독립적인 작업들을 동시에 Agent 도구로 호출:

```
# 동시 실행 (병렬)
Agent({ subagent_type: "{agent-1}", prompt: "..." })
Agent({ subagent_type: "{agent-2}", prompt: "..." })
Agent({ subagent_type: "{agent-3}", prompt: "..." })
```

순차적으로 실행해야 하는 경우 이전 결과를 다음 프롬프트에 포함.

## Phase 3: 결과 수집 및 통합

각 에이전트의 반환값을 수집하여 `_workspace/03_results/`에 저장.
결과를 통합하여 최종 산출물 생성.

## Phase 4: 검증

QA 서브에이전트로 최종 산출물 검증:

```
Agent({ subagent_type: "qa-inspector", prompt: "검증 요청..." })
```

## Phase 5: 완료

`_workspace/DONE` 생성. 결과 보고.

## 에러 핸들링

에이전트 실패 시 해당 작업만 재시도. 치명적 실패 시 사용자에게 보고.
```

---

## 템플릿 C: 하이브리드 모드

Phase별로 팀 모드와 서브 에이전트 모드를 혼합.

```markdown
---
name: {orchestrator-name}
description: "{도메인} 하이브리드 실행 조율. {핵심 기능 설명}. {트리거 상황}.
  재실행, 보완, 업데이트, 수정 요청에도 이 스킬을 사용할 것."
model: opus
---

# {Orchestrator Name}

Phase에 따라 팀 모드와 서브 에이전트 모드를 혼합하여 실행.

## 실행 모드: 하이브리드

| Phase | 모드 | 이유 |
|-------|------|------|
| 분석/계획 | 서브 에이전트 | 독립적, 통신 불필요 |
| 핵심 작업 | 에이전트 팀 | 상호 조율 필요 |
| 검증 | 서브 에이전트 | 독립 검증 |

## Phase 0: 초기화 (공통)

`_workspace/` 확인 및 준비.

## Phase 1: 분석 (서브 에이전트)

```
Agent({ subagent_type: "Explore", prompt: "코드베이스 분석..." })
```

`_workspace/01_analysis.md`에 저장.

## Phase 2: 핵심 작업 (에이전트 팀)

TeamCreate로 팀 구성. Phase 1 분석 결과를 팀원들에게 전달.

## Phase 3: 통합 및 검증 (서브 에이전트)

```
Agent({ subagent_type: "qa-inspector", prompt: "검증..." })
```

## Phase 4: 완료

결과 보고 및 `_workspace/DONE` 생성.
```

---

## Description 작성 주의사항

오케스트레이터 description은 **후속 작업 키워드를 반드시 포함**해야 한다.

**나쁜 예:**
```yaml
description: "리서치 팀을 조율하여 종합 보고서를 생성."
```
→ 첫 실행 후 재실행 요청("이전 리서치 업데이트해줘")에 트리거되지 않는다.

**좋은 예:**
```yaml
description: "리서치 팀을 조율하여 종합 보고서를 생성. 재조사, 업데이트,
  추가 조사, 수정, 보완 요청에도 이 스킬을 사용할 것."
```

후속 작업 키워드 없이 초기 트리거 키워드만 있으면 **첫 실행 후 하네스가 사실상 죽은 코드**가 된다.

---

## 공통 설계 원칙

1. **_workspace/ 보존** — 절대 삭제하지 않는다. 감사 추적 및 재개를 위해 필요.
2. **Phase 단계 명시** — 현재 실행 중인 Phase를 로그로 출력한다.
3. **재개 가능 설계** — 중단된 지점부터 재시작할 수 있도록 각 Phase의 산출물을 파일로 저장.
4. **QA 포함** — 모든 오케스트레이터는 검증 단계를 포함한다.

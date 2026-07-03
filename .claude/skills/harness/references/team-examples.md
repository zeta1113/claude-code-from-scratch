# 에이전트 팀 예시 모음

Claude Agent SDK를 활용한 실제 팀 아키텍처 5가지 예시.

---

## 예시 1: 리서치 팀 (Fan-out/Fan-in)

**구조:** 4명의 전문 리서처가 병렬로 조사 후 통합 보고서 생성.

```markdown
---
name: research-orchestrator
description: "종합 리서치 팀을 조율하여 다각도 조사 보고서를 생성.
  시장 조사, 경쟁사 분석, 기술 조사, 트렌드 리서치 요청 시 사용.
  재조사, 업데이트, 보완 요청에도 이 스킬을 사용할 것."
model: opus
---
```

**팀 구성:**
- `official-researcher`: 공식 문서, 보도자료, 기업 홈페이지
- `media-researcher`: 뉴스, 언론 보도, 미디어 분석
- `community-researcher`: 커뮤니티, 포럼, 소셜 반응
- `background-researcher`: 역사적 맥락, 배경 정보

**핵심 특징:**
- 팀원들이 SendMessage로 발견 사항을 실시간 공유
- 한 팀원의 발견이 다른 팀원의 조사 방향을 조정
- 리더를 거치지 않은 직접 통신으로 효율 향상

**에이전트 통신 프로토콜 예시 (official-researcher.md):**
```markdown
## 팀 통신 프로토콜
- 공식 자료에서 예상치 못한 발견 시 → 모든 팀원에게 SendMessage
- 미디어 리서처가 공식 입장과 다른 보도를 발견했다고 알리면 → 재확인 후 검증
- 리더에게: 공식 자료 조사 완료 시 결과 요약 전달
```

---

## 예시 2: SF 소설 집필 팀 (Pipeline + Fan-out 복합)

**구조:** Phase 1 병렬 세계관/캐릭터/플롯 설계 → Phase 2 단독 집필 → Phase 3 검토.

**팀 구성:**

Phase 1 (병렬):
- `worldbuilder`: 세계관, 과학적 설정, 사회 구조
- `character-designer`: 주인공/조연 설정, 심리, 관계도
- `plot-architect`: 3막 구조, 주요 사건, 복선

Phase 2 (단독):
- `prose-stylist`: Phase 1 결과물을 받아 실제 집필

Phase 3 (병렬):
- `science-reviewer`: 과학적 설정의 일관성 검증
- `continuity-reviewer`: 캐릭터/플롯 연속성 검증

**오케스트레이터 실행 흐름:**
```
1. TeamCreate(worldbuilder, character-designer, plot-architect)
2. 3명 병렬 작업 → _workspace/phase1/ 결과 저장
3. prose-stylist에게 phase1 결과 전달 → 집필
4. TeamCreate(science-reviewer, continuity-reviewer)
5. 검토 후 prose-stylist에게 피드백 → 수정
```

---

## 예시 3: 웹툰 제작 (Producer-Reviewer, 서브 에이전트 모드)

**구조:** 아티스트가 생성 → 리뷰어가 품질 게이트 → 최대 2회 루프.

**실행 모드:** 서브 에이전트 (통신 불필요, 결과만 반환)

```markdown
---
name: webtoon-orchestrator
description: "웹툰 스크립트를 받아 패널 설명과 대사를 생성.
  웹툰, 만화, 그래픽 노블 제작 요청 시 사용."
model: opus
---

## 실행 플로우

1. Agent({ subagent_type: "webtoon-artist", prompt: 스크립트 })
2. Agent({ subagent_type: "webtoon-reviewer", prompt: 결과물 })
3. 리뷰 결과에 따라:
   - PASS → 완료
   - FIX → artist에게 수정 요청 (1회)
   - REDO → artist에게 전면 재작성 요청 (1회)
4. 최대 2회 루프 후 강제 완료 (무한 루프 방지)
```

**리뷰어 판정 기준:**
- `PASS`: 품질 기준 충족
- `FIX`: 부분 수정 필요 (구체적 항목 명시)
- `REDO`: 전면 재작성 필요 (이유 명시)

**주의:** 재시도는 최대 2~3회로 제한. 무한 개선 루프는 토큰 낭비다.

---

## 예시 4: 코드 리뷰 팀 (Fan-out + 팀 모드 직접 통신)

**구조:** 보안/성능/테스트 리뷰어가 각자 분석 후 팀 내 직접 소통으로 통합 리뷰 생성.

**팀 구성:**
- `security-reviewer`: 취약점, 인젝션, 인증 이슈
- `performance-reviewer`: 시간복잡도, 메모리, N+1 쿼리
- `test-reviewer`: 테스트 커버리지, 엣지 케이스, 테스트 가능성

**팀 모드의 핵심 가치:**
- 보안 리뷰어가 발견한 취약점이 성능 리뷰어에게 영향 → 직접 통신으로 즉시 공유
- 팀원 간 교차 발견으로 단독 리뷰 대비 품질 향상
- 리더를 거치지 않아 지연 없음

**에이전트 정의 예시 (security-reviewer.md):**
```markdown
---
name: security-reviewer
description: "코드의 보안 취약점을 분석하는 전문가."
model: opus
---

# Security Reviewer

## 핵심 역할
- OWASP Top 10 취약점 스캔
- 인증/인가 로직 검토
- 입력 검증 및 SQL 인젝션 확인

## 팀 통신 프로토콜
- 성능에 영향을 미치는 보안 이슈 발견 시 → performance-reviewer에게 SendMessage
- 테스트가 없는 보안 로직 발견 시 → test-reviewer에게 알림
- 리더에게: 보안 리뷰 완료 + 심각도별 이슈 목록

## 출력
_workspace/security-review.md에 저장:
- 심각도: Critical/High/Medium/Low
- 이슈 설명 + 코드 위치 + 수정 방법
```

---

## 예시 5: 코드 마이그레이션 (Supervisor 패턴)

**구조:** 감독자가 코드베이스를 분석하고 워커를 동적으로 배치.

**Fan-out과의 차이:** 작업이 사전에 고정되지 않고 런타임에 동적으로 할당.

```markdown
---
name: migration-supervisor
description: "대규모 코드 마이그레이션을 감독. 레거시 코드 분석 후
  파일별 마이그레이션 워커를 동적으로 배치. 코드베이스 업그레이드,
  프레임워크 마이그레이션, 리팩토링 요청 시 사용."
model: opus
---

## Phase 1: 분석

코드베이스를 스캔하여 마이그레이션 대상 파일 목록 생성.
복잡도와 의존성에 따라 우선순위 결정.

## Phase 2: 동적 배치

TaskCreate로 파일별 작업 생성.
사용 가능한 워커를 확인하여 작업 배분:

```
TaskCreate({ title: "src/auth.ts 마이그레이션", status: "pending" })
TaskCreate({ title: "src/db.ts 마이그레이션", status: "pending" })
...

# 워커들이 자기 요청(claim) 패턴으로 작업 선택
SendMessage(worker-1, "auth.ts 작업 시작해")
SendMessage(worker-2, "db.ts 작업 시작해")
```

## Phase 3: 모니터링

TaskUpdate 신호를 수신하여 완료된 작업 파악.
블로킹 파일 발견 시 재배분.
의존성 오류 발생 시 관련 워커들에게 알림.

## Phase 4: 검증

QA 에이전트에게 전체 마이그레이션 결과 검증 요청.
```

**Supervisor 패턴의 핵심:** 감독자는 전체 상황을 보면서 동적으로 조율. 워커들은 작업을 완료하면 TaskUpdate로 신호를 보내고 다음 작업을 받는다.

---

## 공통 설계 원칙

모든 예시에서 반복되는 패턴:

1. **`_workspace/` 보존** — 팀 산출물은 항상 파일로 저장, 재개 가능
2. **에이전트 정의 파일 필수** — `.claude/agents/{name}.md`에 역할, 원칙, I/O, 통신 프로토콜
3. **팀 통신 프로토콜 명시** — 누구에게 무엇을 언제 보내는지
4. **실행 모드 선택 기준** — 통신 필요 → 팀 모드, 독립 처리 → 서브 에이전트
5. **루프 제한** — Producer-Reviewer 패턴에서 최대 2~3회 재시도

# 도메인 분석: 레포 문서 한국어 번역

## 핵심 도메인
claude-code-from-scratch 튜토리얼 문서 한국어 로컬라이제이션

## 번역 대상
- 소스: en/docs/ (영어, 15개 파일)
- 타겟: ko/docs/ (신규 생성)

## 파일 목록
- 00-introduction.md
- 01-agent-loop.md
- 02-tools.md
- 03-system-prompt.md
- 04-cli-session.md
- 05-streaming.md
- 06-permissions.md
- 07-context.md
- 08-memory.md
- 09-skills.md
- 10-plan-mode.md
- 11-multi-agent.md
- 12-mcp.md
- 13-whats-next.md
- 14-testing.md

## 아키텍처
- 패턴: Fan-out (서브 에이전트 모드)
- 에이전트: doc-translator
- 병렬 처리로 속도 최적화

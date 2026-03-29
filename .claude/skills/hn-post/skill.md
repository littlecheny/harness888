---
name: hn-post
description: "Hacker News 'Show HN' 게시글을 작성하는 스킬. 프로젝트 분석, HN 커뮤니티 패턴 연구, 초안 작성, 톤 검수까지 전 과정을 에이전트 팀으로 수행. 'HN에 올릴 글', 'Show HN 작성', 'Hacker News 소개글', 'HN 포스트' 등의 요청 시 사용."
---

# Show HN Post Writer

Hacker News "Show HN" 게시글을 에이전트 팀으로 작성한다.

## 실행 모드
에이전트 팀 (파이프라인 패턴)

## 팀 구성

| 에이전트 | 파일 | 타입 | 모델 | 역할 |
|---------|------|------|------|------|
| hn-researcher | `.claude/agents/hn-researcher.md` | general-purpose | opus | HN 패턴 분석 + 프로젝트 앵글 추출 |
| hn-writer | `.claude/agents/hn-writer.md` | general-purpose | opus | 초안 작성 |
| hn-reviewer | `.claude/agents/hn-reviewer.md` | general-purpose | opus | HN 톤 검수 + 최종본 확정 |

## 워크플로우

### Phase 1: 연구 (hn-researcher)
1. HN "Show HN" 성공 패턴 웹 검색 및 분석
2. 프로젝트 코드베이스 탐색 (README, 랜딩페이지, 핵심 스킬 파일)
3. HN 커뮤니티가 반응할 앵글 도출
4. 산출물: `_workspace/01_researcher_hn_patterns.md`, `_workspace/01_researcher_project_angles.md`

### Phase 2: 작성 (hn-writer)
- 의존: Phase 1 완료
1. 연구 결과 읽기
2. 제목 후보 3~5개 생성
3. 본문 초안 작성 (What → Why → How → Links)
4. 산출물: `_workspace/02_writer_draft.md`

### Phase 3: 검수 (hn-reviewer)
- 의존: Phase 2 완료
1. HN 안티패턴 체크 (과대광고, 마케팅 용어, 길이)
2. 기술적 정확성 교차 검증
3. 톤/스타일 최종 조정
4. 산출물: `_workspace/03_reviewer_final.md`

## 데이터 전달
- 파일 기반: `_workspace/{phase}_{agent}_{artifact}.md`
- 메시지 기반: 단계 완료 시 다음 에이전트에게 SendMessage

## 에러 핸들링
- 웹 검색 실패: 내부 지식 기반으로 HN 패턴 분석 수행
- 초안 품질 미달: reviewer → writer 재작성 요청 (1회 재시도)
- 재시도 후에도 미달: reviewer가 직접 수정하여 최종본 확정

## Show HN 핵심 규칙

### 제목 포맷
```
Show HN: [Name] – [What it does in ≤10 words]
```
- 60자 이내
- 기술적이고 구체적으로
- "revolutionary", "AI-powered" 같은 버즈워드 금지

### 본문 구조
1. **What**: 한 문장으로 무엇인지 설명
2. **Why**: 왜 만들었는지 (개인적 동기, 기존 도구의 문제)
3. **How**: 기술적으로 어떻게 작동하는지 (간결하게)
4. **Differentiator**: 기존 대안 대비 차별점
5. **Links**: GitHub, 데모, 문서
6. **Ask**: 피드백 요청 (선택)

### 톤 가이드
- 1인칭 사용 ("I built...", "I've been working on...")
- 겸손하되 자신감 있게
- 기술적 깊이를 보여주되 장황하지 않게
- 마케팅 느낌 배제 — 엔지니어가 엔지니어에게 말하는 톤
- 전체 본문 300단어 이내 권장

## 테스트 시나리오

### 정상 흐름
1. 사용자가 "HN에 소개글 작성해줘" 요청
2. researcher가 HN 패턴 + 프로젝트 분석 완료
3. writer가 초안 작성
4. reviewer가 검수 후 최종본 확정
5. `_workspace/03_reviewer_final.md`에 최종 게시글 출력

### 에러 흐름
1. writer 초안에 마케팅 용어 다수 포함
2. reviewer가 피드백과 함께 writer에게 재작성 요청
3. writer가 수정본 작성
4. reviewer가 최종 승인

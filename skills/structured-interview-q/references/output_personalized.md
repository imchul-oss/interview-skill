# output_personalized.md — 후보자별 맞춤 가이드 .md 변수 치환 템플릿 (v0.2.0 슬림형)

## 원칙

면접관이 면접 중·후에 **실제로 쓰는 정보만** 출력한다. 군더더기·중복·예측 평가는 모두 제거한다.

표를 우선 사용해 가독성을 높인다. 예상 점수·예측 평가는 출력하지 않는다.

검증 우선순위 라벨(High/Medium/Critical)을 부여하지 않는다. 매칭 강도(🟢🟡🟠🔴)로 일원화한다.

직무 모듈에 anchor 질문 1~2개를 강제 포함한다. MVC 5가치는 모두 평가한다 (선별 금지, 가치당 시간만 압축).

6역량 + MVC 5가치 외 추가 자유 영역을 생성하지 않는다.

**슬림 원칙 (v0.2.0)**: 후보자 사실확인 내용을 단락마다 길게 나열하지 않는다. 매칭 매트릭스 1개 표로 핵심만 요약하고, 개별 질문의 근거는 **그 질문 탭 안에 "이유" 1줄**로만 보여준다. 강점·갭·Red Flag를 별도 섹션으로 펼치지 않는다.

**기법 다양화 (v0.2.0)**: 질문은 STAR/BEI/SIT에 국한하지 않는다. 각 질문에 기법 라벨과 **관찰 포인트 1줄**을 부착하고, 갭 역량에는 면접관 옵션으로 실기·워크샘플 제안을 1줄 덧붙인다 (§7).

**레벨별 BARS (v0.2.0)**: 트랙(senior/mid/junior)이 명시되면 해당 레벨 앵커를 렌더한다. DB에 레벨별 앵커가 없으면 단일 앵커로 폴백하고 그 사실을 주석으로 명시한다 (§8). 임의 생성 금지.

### 제거 대상 (운영 데이터 기반)

| 제거 항목 | 사유 |
|---|---|
| 학술 근거 frontmatter | 면접 중 안 봄, SKILL.md에 위치 |
| 데이터 무결성·CV 정책 선언 박스 | 1회 읽으면 불필요, SKILL.md 정책 |
| §강점·§갭·§사실확인 별도 단락 | 매칭 매트릭스 1표 + 질문 탭 "이유"로 통합 |
| 각 질문의 메타 표·검증 의도 라벨 | 질문 탭 "이유" 1줄로 대체 |
| STAR 4단계 전부 표 (질문마다) | 핵심 꼬리질문 3개 bullet로 압축 |
| 검증 우선순위 라벨 (High/Medium/Critical) | 매칭 강도와 중복 |
| 큰 누적시간 체크표 | 소형 시간 표 1개로 축소 |
| 변경 이력 알림 / 자유 영역 | 운영 중 불필요 / 평가 기준 부동 |

---

## 1. 변수 정의

| 변수 | 출처 |
|---|---|
| `{{candidate_name}}` | CV 식별자 (안전 변환) |
| `{{position_name}}` / `{{job_title}}` | positions |
| `{{stage}}` / `{{minutes}}` / `{{track}}` | 사용자 입력 (track 미입력 시 unspecified) |
| `{{one_line_descriptor}}` | CV 사실 기반 한 줄 (예: "Win/macOS/Linux 시스템 프로그래밍 8.7년, 통신계층 강점") |
| `{{competency_matches[]}}` | competency_matching.md (강도만, 우선순위 라벨 없음) |
| `{{prioritized_questions[]}}` | priority_algorithm.md (anchor 포함, 시간 알고리즘 적용) |
| `{{question.technique}}` | 질문 기법 라벨 (BEI/SIT/관찰/케이스/워크샘플 — §7) |
| `{{question.why_one_line}}` | 이 질문을 하는 이유 1줄 (CV 강점/갭 연결, 간략) |
| `{{question.observation_cue}}` | 관찰 포인트 1줄 (답변 중 면접관이 볼 행동 신호) |
| `{{question.bars}}` | 트랙 레벨 BARS (없으면 단일 폴백 — §8) |
| `{{mvc_questions[]}}` | 5가치 모두 (선별 금지) + 직무 친화 컨텍스트 |

예상 점수·우선순위 라벨·자유 영역은 변수에서 제외한다.

---

## 2. 채팅 응답 표 패턴

채팅창에는 핵심만 표로 출력하고 상세는 .md 링크로 제공한다.

```markdown
✅ {{candidate_name}} 후보자 · {{job_title}} {{stage}} {{minutes}}분 가이드 생성 완료.

| 항목 | 내용 |
|---|---|
| 후보자 한 줄 | "{{one_line_descriptor}}" |
| 트랙 | {{track_or_unspecified}} |
| 공통 anchor | {{anchor_question_keys}} |
| 직무역량 / MVC 시간 | {{job_minutes}}분 / {{mvc_minutes}}분 |

**📁 상세 가이드**: [{{filename}}](computer://{{path}})

예상 평가 점수·검증 우선순위 라벨은 포함하지 않습니다.
```

---

## 3. .md 가이드 — 전체 구조 (슬림형)

frontmatter는 운영 메타만 유지한다. 학술 근거·정책 선언은 SKILL.md에 위치한다.

```markdown
---
candidate: {{candidate_name}}
position: {{position_name}} ({{job_title}})
stage: {{stage}}
minutes: {{minutes}}
track: {{track_or_unspecified}}
generated_at: {{generated_at}}
generated_by: structured-interview-q v{{skill_version}}
---

# {{candidate_name}} — {{job_title}} {{stage}} 면접 가이드

## 1. 후보자 한눈에

| 항목 | 내용 |
|---|---|
| 한 줄 | "{{one_line_descriptor}}" |
| 경력 / 현재 역할 | {{experience_years}}년 / {{current_role}} |
| 핵심 기술 | {{top_skills}} |
| 트랙 | {{track_or_unspecified}} |

### 역량 매칭 (핵심만)

| # | 역량 | 가중치 | 강도 | 핵심 근거 (CV) | Anchor |
|---|---|---:|:---:|---|:---:|
{{#each competency_matches}}
| {{key_short}} | {{name}} | {{weight_pct}}% | {{strength_emoji}} | {{evidence_one_phrase}} | {{anchor_badge}} |
{{/each}}

> 확인 필요: {{red_flag_inline_or_none}}

매칭 강도는 CV 명시 사실 기반 매핑이며 평가 점수 예측이 아니다. 상세 근거는 각 질문 탭의 "이유"에서 간략히 다룬다.

## 2. 진행 시간 (간략)

| 블록 | 시간 |
|---|---:|
| 오프닝 | {{opening_min}}분 |
| 직무역량 ({{competency_count}}역량, 가중치 비례) | {{job_minutes}}분 |
| MVC 5가치 | {{mvc_minutes}}분 |
| 클로징·후보자 질문 | {{closing_min}}분 |

시간은 priority_algorithm.md 결정적 공식으로 산정된다. MVC 5가치는 시간과 무관하게 모두 평가한다.

## 3. 직무역량 질문

> **면접관 빠른 참조** — 🔴 필수 2개는 모든 후보 공통(반드시 진행) · ⚪ 선택 2~3개는 시간·상황에 맞게. 각 카드: 질문 → 이유 → 관찰 포인트 → 꼬리 → BARS.

### 🔴 필수 질문 (2 · 모든 후보 공통)

모든 후보자·세션이 공통으로 받는 anchor 2개. 후보자 간 직접 비교의 기준이 된다.

{{#each must_ask_questions}}
#### 🔴 필수 {{idx}}. {{competency_name}} · [{{technique}}] 🔗 Anchor

> {{standard_question_text}}

- **이유**: {{why_one_line}}
- **관찰 포인트**: {{observation_cue}}
{{#if cv_specific_addendum}}- **CV 안내** (이어서 발화): {{cv_specific_addendum}} (L{{cv_lines}}){{/if}}
{{#if worksample_suggestion}}- **실기 제안(옵션)**: {{worksample_suggestion}}{{/if}}

꼬리질문: {{probe_1}} / {{probe_2}} / {{probe_3}}

| BARS{{bars_level_suffix}} | 5 | 4 | 3 | 2 | 1 |
|---|---|---|---|---|---|
|  | {{bars.5}} | {{bars.4}} | {{bars.3}} | {{bars.2}} | {{bars.1}} |

---

{{/each}}

### ⚪ 선택 질문 (2~3 · 면접관 재량)

시간·후보 상황에 맞게 취사한다. priority_score 순으로 정렬되어 있다.

{{#each optional_questions}}
#### ⚪ 선택 {{idx}}. {{competency_name}} · [{{technique}}]

> {{standard_question_text}}

- **이유**: {{why_one_line}}
- **관찰 포인트**: {{observation_cue}}
{{#if cv_specific_addendum}}- **CV 안내** (이어서 발화): {{cv_specific_addendum}} (L{{cv_lines}}){{/if}}
{{#if worksample_suggestion}}- **실기 제안(옵션)**: {{worksample_suggestion}}{{/if}}

꼬리질문: {{probe_1}} / {{probe_2}} / {{probe_3}}

| BARS{{bars_level_suffix}} | 5 | 4 | 3 | 2 | 1 |
|---|---|---|---|---|---|
|  | {{bars.5}} | {{bars.4}} | {{bars.3}} | {{bars.2}} | {{bars.1}} |

---

{{/each}}

## 4. MVC 평가 ({{mvc_minutes}}분 — 5가치 모두)

시간 제약과 무관하게 5가치 모두 평가한다 (선별 금지, Levashina 2014 동일 질문 원칙).

{{#each mvc_questions}}
### 4.{{order}} {{value_name}}

> {{question_text}}

- **{{position_name}} 컨텍스트** (이어서 발화): {{position_context}}
- **관찰 포인트**: {{observation_cue}}

꼬리질문: {{probe_1}} / {{probe_2}} / {{probe_3}}

| Go/No-Go | 🚩 No-Go 신호 | ✅ Go 신호 |
|---|---|---|
| {{value_key}} | {{nogo_signal}} | {{go_signal}} |

---

{{/each}}

{{#if is_stage_2}}
## 4.5 추가 검증 포인트 (2차 전용 · 면접관 입력 기반)

면접관이 입력한 "추가 검증 희망 영역"을 표준 질문·관찰과 연결한다. 표준 질문 텍스트는 수정하지 않는다. 입력이 없으면 본 섹션을 생략한다.

| 검증 영역 | 연결 질문 / 접근 | 관찰 포인트 |
|---|---|---|
{{#each extra_verification_points}}
| {{area}} | {{linked_question_or_probe}} | {{observation_cue}} |
{{/each}}

## 4.6 레퍼런스 체크 포인트 (2차 전용 · 평판조회용)

면접·CV로 검증이 어려운 항목을 평판조회 질문 형태로 정리한다. 입력이 없으면 본 섹션을 생략한다.

| 확인 항목 | 평판조회 질문 (예) | 비고 |
|---|---|---|
{{#each reference_check_points}}
| {{item}} | {{ref_question}} | 면접·CV로 직접 검증 불가 |
{{/each}}

{{/if}}
## 5. 종합 (면접 후 면접관 작성)

| 판정 | 직무 가중평균 | MVC | 결정 |
|---|---:|---|---|
| Strong Hire | 4.0+ | All Go·평균 4.0+ | 즉시 오퍼 |
| Hire | 3.5+ | All Go·평균 3.0+ | 오퍼 진행 |
| Conditional | 3.0~3.5 | All Go·평균 3.0~3.5 | 추가 면접/과제 |
| No Hire | < 3.0 | No-Go 1개+ | 불합격 |

점수는 면접 후 직접 입력한다. 본 가이드는 점수를 미리 산정하지 않는다.

## 6. 안내

CV 데이터는 세션 종료 시 폐기된다. 본 .md에는 CV 발췌만 포함된다. 외부 공유 시 후보자 동의가 필요하다.
```

---

## 4. 파일명·저장 위치

파일명: `{{candidate_name_safe}}_{{position_name_safe}}_{{stage}}_{{YYYYMMDD}}.md`
저장 위치: 사용자 폴더 (예: `<workspace>/면접/`)

---

## 5. 검증 체크리스트 (Render 직전)

- [ ] 표준 질문 텍스트 변경 0건
- [ ] 가중 점수·예상 평가 0건 / 검증 우선순위 라벨 0건
- [ ] 6역량 + MVC 5가치 외 자유 영역 0건
- [ ] MVC 5가치 모두 포함 (5건 미만 시 오류)
- [ ] 직무역량 = 🔴 필수 정확히 2개 + ⚪ 선택 2~3개로 이원화 (필수는 anchor)
- [ ] 각 질문에 이유 1줄 + 관찰 포인트 1줄 부착
- [ ] 강점·갭·사실확인 **별도 단락 0건** (매칭 매트릭스 1표 + 질문 탭 이유로 통합)
- [ ] 시간 표 1개로 간략화 (누적 체크표 미사용)
- [ ] 트랙 명시 시 해당 레벨 BARS, 미존재 시 단일 폴백 + 주석 (§8)
- [ ] 모든 CV 인용에 cv_lines 메타 부착
- [ ] 시인성·인쇄 규칙 (§10) 적용

---

## 6. 환각 가드

CV 인용에 cv_lines가 없는 항목은 즉시 삭제한다.
addendum이 CV에 없는 표현을 사용하면 addendum만 삭제하고 표준 질문은 유지한다.
시간 배분 합계 불일치 시 priority_algorithm을 재호출한다.
MVC가 5건이 아니면 즉시 보강한다.
레벨별 BARS가 DB에 없으면 단일 앵커로 폴백하고 임의 생성하지 않는다.

---

## 7. 질문 기법 + 관찰 포인트 렌더 규칙 (v0.2.0 신규)

표준 질문 자체는 immutable이다. 본 절은 **렌더 시 부착하는 메타**(기법 라벨·관찰 포인트·실기 제안)의 규칙이며 질문 텍스트를 바꾸지 않는다. 기법 카탈로그의 정의는 companion `structured-interview/references/question_generation.md §7`을 따른다.

### 7.1 기법 라벨

각 질문에 다음 중 하나를 라벨로 표시한다.

| 라벨 | 의미 | 출처 |
|---|---|---|
| BEI | 과거 행동 사건 (McClelland 1973) | interview_questions.question_type |
| SIT | 가상 상황 판단 (Latham 1980) | interview_questions.question_type |
| 관찰 | 답변 중 행동·사고 과정 관찰 (reasoning-aloud) | 모든 질문에 관찰 포인트로 부착 |
| 케이스 | 시나리오 심화 (SIT 변형) | SIT 질문에 한해 |
| 워크샘플 | 실기·코드 워크스루·화이트보드 (면접관 옵션) | 갭 역량에 제안 |
| 대조 | 두 과거 결정 비교 (contrastive) | BEI 꼬리질문 변형 |

### 7.2 관찰 포인트 (모든 질문 1줄)

답변 중 면접관이 관찰할 **행동·사고 신호**를 1줄로 제시한다. 평가 점수가 아니라 "무엇을 볼지"의 안내다.

예) "'우리/나'의 구분이 자연스러운가, 정량 지표를 자발적으로 제시하는가."

### 7.3 실기·워크샘플 제안 (갭 역량 옵션)

매칭 강도 🟠 weak / 🔴 none 인 핵심 역량(가중치 상위)에는 면접관 **옵션**으로 짧은 실기 제안을 1줄 덧붙인다. 강제 아님.

예) "옵션: 화이트보드로 TLS 핸드셰이크 복호화 흐름을 5분간 스케치 요청 → 개념 이해 깊이 확인."

실기 제안은 직무 역량 키워드에 기반하며, CV에 없는 사실을 단정하지 않는다.

---

## 8. 레벨별 BARS 렌더 + 폴백 (v0.2.0 신규, Critical)

트랙 입력에 따라 BARS 앵커를 결정적으로 선택한다 (SKILL.md §5.4와 동일).

| 트랙 | `bars_anchors_by_level` 존재 | 렌더 |
|---|:---:|---|
| senior/mid/junior | ✅ | `bars_anchors_by_level[track]` 5점 앵커. BARS 행 라벨에 `(senior)` 등 표기 |
| senior/mid/junior | ❌ | 단일 `bars_anchors` + 표 아래 주석: "ℹ️ 레벨별 앵커 미작성 직무 — 단일 기준 적용" |
| unspecified | — | 단일 `bars_anchors`(또는 mid) + 주석: "ℹ️ 트랙 미입력 — 단일/기준 레벨 적용" |

레벨별 앵커가 없을 때 LLM이 레벨별 앵커를 만들어내지 않는다. 앵커 작성 규칙·스키마는 `structured-interview/references/bars_anchoring.md §7`.

---

## 10. 시인성·인쇄 친화 작성 규칙 (v0.1.5, Critical)

운영 데이터 분석 결과 .md 가이드의 단락 구분이 약해 VS Code Preview·A4 인쇄에서 가독성이 떨어졌다. 다음 규칙을 출력 시 강제한다.

### 10.1 단락 구분 7원칙

1. 모든 `##`·`###` 헤더 **위·아래 빈 줄 1개** 강제.
2. 모든 표 **위·아래 빈 줄 1개** 강제 (VS Code Preview 렌더링 안정).
3. 표 직후 다른 표 시작 금지 — 사이에 한 문장 설명 또는 빈 줄 2개.
4. 한 문단 **≤ 5 문장** (초과 시 분리).
5. 인용 블록(`>`) 다음 빈 줄 1개.
6. 헤더 직후 한 줄 안내문 후 표 시작 (시각적 도입).
7. 각 질문/가치 카드 사이 `---` 가로 구분선 삽입 (프리뷰에서 깔끔하게 분리). A4 강제 페이지 분리가 필요하면 PDF 변환 시 처리한다 (§10.3).

### 10.2 VS Code Preview (`Ctrl+Shift+V`) 친화

- CommonMark + GFM 표준 준수 (표·체크박스·코드블록 표준 문법).
- 헤더 위계 h1·h2·h3만 사용 (h4 이상 금지).
- 표 컬럼 ≤ 6, 한 줄 폭 ≤ 80자 (가로 스크롤 방지).
- 이모지 의미 매핑 일관 (🟢🟡🟠🔴 강도 · 🔗 anchor · ☐ 체크 · 🚩 red · ✅ green · ℹ️ 주석).

### 10.3 A4 보고서 친화

- 카드 사이는 `---` 구분선. A4 강제 페이지 분리가 필요하면 PDF 변환 도구에서 카드 단위로 설정한다 (마크다운 본문에 page-break div를 넣지 않는다 — 프리뷰 노출 방지).
- 표 폭 A4 portrait 80자 이내, 본문 11pt 기준, 여백 상하·좌우 20mm.
- PDF 변환 시 `Markdown PDF` 확장 (A4 portrait, 헤더·푸터에 후보자명·날짜·페이지) 권장.

# output_template.md — 출력 .md 변수 치환 템플릿 (v0.1.1 표 중심)

**원칙**: 어떤 직무가 입력되든 동일 구조로 출력. 변수만 치환되며, 역량 개수·가중치는 가변. **모든 보고는 표(table) 우선** — 산문 대신 표를 사용하여 가독성 극대화.

---

## 0. 채팅 응답 표 패턴 (스킬 작업 직후 사용자에게 보고)

각 시나리오별 표 우선 응답 형식:

### 0.1 INSERT 완료 보고 (첫 호출, 질문 생성·INSERT 직후)

> 직무역량 질문 수 = `competency_count × 2` (역량당 BEI 1 + SIT 1). 역량 수는 JD에 따라 4~7개로 가변이므로 **숫자를 고정하지 않는다** (competency_extraction.md). 총합 = 직무역량 + MVC 15.

```markdown
✅ {{job_title}} 표준 면접 질문 {{total_question_count}}개 INSERT 완료.

**📊 INSERT 결과**

| 모듈 | 행수 | 가중치 합 | 상태 |
|---|---:|---:|:---:|
| 직무역량 ({{position_name}}) | {{job_question_count}} | 100% | ✅ |
| MVC (자동 부착) | 15 | — | ✅ |
| **합계** | **{{total_question_count}}** | | **모두 sync** |

**🎯 추출 역량 (MECE)**

| # | 역량 | 가중치 | 유형 | JD 출처 |
|---|---|---:|---|---|
{{#each competencies}}
| {{key}} | {{name}} | {{weight_pct}}% | {{type}} | {{source_jd_lines}} |
{{/each}}

**🔗 추적성**

| section | hash |
|---|---|
| responsibilities | {{jd_hashes.responsibilities | first8}} |
| qualifications | {{jd_hashes.qualifications | first8}} |
| preferred | {{jd_hashes.preferred | first8}} |

**다음 단계**: 후보자 면접 시 `/structured-interview-q` 호출 → CV 매칭 + 맞춤 가이드.
```

### 0.2 JD 변경 자동 감지 (T1/T2/T3) 보고

```markdown
⚠️ {{position_name}} JD 변경 감지.

**🔍 분류**

| 항목 | 값 |
|---|---|
| Tier | **{{tier}}** ({{tier_name}}) |
| Levenshtein 유사도 | {{similarity}} |
| 변경 요약 | {{summary}} |
| 영향 받는 질문 | {{affected_count}}개 |
| 처리 정책 | {{action}} |

**📝 액션**

| # | 작업 | 상태 |
|---|---|:---:|
{{#each actions}}
| {{step}} | {{description}} | {{status}} |
{{/each}}
```

### 0.3 가드레일 작동 (직무 미등록·JD 누락) 보고

```markdown
❌ 생성 거부 — 가드레일 작동.

| 검증 단계 | 결과 |
|---|:---:|
| positions 검색 | {{positions_status}} |
| position_content 검증 | {{content_status}} |

**📋 등록된 직무 목록**

| name | job_title | status |
|---|---|:---:|
{{#each available_positions}}
| {{name}} | {{job_title}} | {{status}} |
{{/each}}

**다음 단계 옵션**:
- (A) 위 목록에서 선택 후 재호출
- (B) 신규 직무 등록 (`new_position_workflow.md` 진행)
- (C) 직무명 오타 — 정확한 이름 알려주세요
```

---

## 1. 변수 정의

| 변수 | 타입 | 설명 | 출처 |
|---|---|---|---|
| `{{position_name}}` | string | snake_case | positions.name |
| `{{job_title}}` | string | 표시명 | positions.job_title |
| `{{lang}}` | string | "ko" \| "en" | 사용자 요청 |
| `{{generated_at}}` | date | 생성 시각 | NOW() |
| `{{candidate_or_purpose}}` | string | 후보자명 또는 면접 목적 | 사용자 입력 |
| `{{competencies[]}}` | array | 역량 N개 | interview_questions JOIN |
| `{{mvc[]}}` | array | 5 가치 | mvc/_core_values.md (모든 직무 동일) |
| `{{position_context_map}}` | object | 직무 친화 컨텍스트 | mvc/_core_values.md §4 |
| `{{jd_hashes}}` | object | derived_from_jd_hash | INSERT 시점 |
| `{{academic_refs}}` | array | 학술 근거 | static |

---

## 2. 출력 .md 구조 (변수 치환 대상)

```markdown
---
position: {{position_name}}
job_title: {{job_title}}
candidate_or_purpose: {{candidate_or_purpose}}
generated_at: {{generated_at}}
generated_by: structured-interview v{{skill_version}}
source: supabase://<YOUR_PROJECT_REF>
jd_hashes: {{jd_hashes}}
academic_basis: {{academic_refs}}
---

# {{job_title}} 구조화 면접 가이드

## 1. 직무 요약
{{jd_summary_3_to_5_lines}}

## 2. 평가 역량 모델 (가중치 합 100%)

| # | 역량 | 가중치 | 유형 | JD 출처 |
|---|---|---:|---|---|
{{#each competencies}}
| {{key}} | {{name}} | {{weight_pct}}% | {{type}} | {{source_jd_lines | join(", ")}} |
{{/each}}

## 3. 직무역량 면접 질문 세트

{{#each competencies}}
### {{competency_name}} ({{weight_pct}}%)

#### {{question_type_BEI}} 질문
> {{bei.question_text}}

- **꼬리질문 (STAR 단계별)**:
{{#each bei.probing_scripts}}
  - {{this}}
{{/each}}
- 🚩 **Red Flag**: {{bei.red_flags | join(" / ")}}
- ✅ **Green Flag**: {{bei.green_flags | join(" / ")}}

#### {{question_type_SIT}} 질문
> {{sit.question_text}}

- **꼬리질문**:
{{#each sit.probing_scripts}}
  - {{this}}
{{/each}}
- 🚩 **Red Flag**: {{sit.red_flags | join(" / ")}}
- ✅ **Green Flag**: {{sit.green_flags | join(" / ")}}

#### BARS 5점 앵커
{{#if bars_anchors_by_level}}
> 레벨별 차등 앵커 (v0.2.0). 면접 시 후보 트랙에 해당하는 행만 사용.

| 점수 | junior | mid | senior |
|---:|---|---|---|
| 5 | {{bars.junior.5}} | {{bars.mid.5}} | {{bars.senior.5}} |
| 4 | {{bars.junior.4}} | {{bars.mid.4}} | {{bars.senior.4}} |
| 3 | {{bars.junior.3}} | {{bars.mid.3}} | {{bars.senior.3}} |
| 2 | {{bars.junior.2}} | {{bars.mid.2}} | {{bars.senior.2}} |
| 1 | {{bars.junior.1}} | {{bars.mid.1}} | {{bars.senior.1}} |
{{else}}
| 점수 | 행동 앵커 |
|---:|---|
| 5 | {{bars.5}} |
| 4 | {{bars.4}} |
| 3 | {{bars.3}} |
| 2 | {{bars.2}} |
| 1 | {{bars.1}} |

> ℹ️ 단일 기준 (레벨별 앵커 미작성). 레벨 차등은 `bars_anchoring.md §7` 적용 + DB 등록 후 활성.
{{/if}}

{{/each}}

## 4. MVC 조직문화 면접 모듈 (자동 부착)

{{#each mvc}}
### {{value_name}}

#### 정의
{{value_definition}}

#### 직무 친화 질문 (Universal Stem + Position Context)

**Q-{{stem_id}} [BEI · Universal]**
> {{stem.bei_1}}

> **+ {{position_name}} 컨텍스트**: {{position_context_map[value_key]}}

**Q-{{stem_id}} [BEI · Universal]**
> {{stem.bei_2}}

**Q-{{stem_id}} [SIT · Universal]**
> {{stem.sit_1}}

#### 꼬리질문
{{#each value.probing_scripts}}
- {{this}}
{{/each}}

#### Red/Green Flag
- 🚩 {{value.red_flags | join(" / ")}}
- ✅ {{value.green_flags | join(" / ")}}

#### BARS
{{value.bars_table}}

#### Go/No-Go
{{value.go_nogo_criterion}}

{{/each}}

## 5. 시간 배분 (1차 면접 권장)

| 블록 | 시간 |
|---|---:|
| 오프닝 | 5분 |
| 직무역량 ({{competency_count}} 역량 × 평균) | {{job_block_minutes}}분 |
| MVC 5가치 크로스체크 | 15분 |
| 클로징·후보자 질문 | 5분 |
| **총** | **{{total_minutes}}분** |

## 6. 평가 의사결정 매트릭스

| 판정 | 직무 점수 | MVC | 결정 |
|---|---:|---|---|
| Strong Hire | 4.0+ | All Go + 평균 4.0+ | 즉시 오퍼 |
| Hire | 3.5+ | All Go + 평균 3.0+ | 오퍼 진행 |
| Conditional | 3.0~3.5 | All Go + 평균 3.0~3.5 | 추가 면접/과제 |
| No Hire | < 3.0 | No-Go 1개 이상 | 불합격 |

## 7. 면접관 가이드

### 편향 방지 체크리스트
- [ ] 후광효과 / 유사성 / 확인 / 앵커링 / 편안함 5종

### 꼬리질문 운용
- Result 단계 정량 지표 1회 강제
- 모호 답변 시 "구체 사례 1개" 1회 재요청
- "우리"/"나" 분리 질문

## 8. 출처 및 추적성

- **JD 데이터**: Supabase `position_content` WHERE position_name='{{position_name}}' AND lang='{{lang}}'
  - responsibilities hash: `{{jd_hashes.responsibilities}}`
  - qualifications hash: `{{jd_hashes.qualifications}}`
  - preferred hash: `{{jd_hashes.preferred}}`
- **interview_questions ID**: {{question_ids[]}}
- **MVC 모듈**: `mvc/_core_values.md` v{{mvc_version}}
- **학술 근거**: {{academic_refs}}

## 9. 변경 이력 알림 (있을 시)

{{#if jd_change_log_recent}}
> ⚠️ 최근 30일 내 이 직무 JD 변경 감지: {{jd_change_log_recent | summarize}}
{{/if}}

---
*본 문서는 structured-interview v{{skill_version}} 스킬에 의해 {{generated_at}}에 생성되었습니다.*
*동일 직무 호출 시 동일한 질문이 fetch됩니다 (구조화 일관성 원칙).*
```

---

## 3. 변수 검증 (Render 직전)

- [ ] `competency_count` = `competencies.length` 일치
- [ ] 모든 역량의 `weight_pct` 합 = 100
- [ ] `jd_hashes`가 현재 `position_content` MD5와 일치
- [ ] `mvc[]`가 정확히 5개
- [ ] `position_context_map`에 본 직무 컨텍스트 존재 (없으면 fallback: "...직무 사례를 중심으로 말씀해 주세요.")
- [ ] `academic_refs` 5개 이상

---

## 4. 출력 파일명 규칙

```
{{position_name}}_{{candidate_or_purpose}}_{{YYYYMMDD}}.md
```

예시 (직무 무관):
- `{position}_김후보_20260426.md`
- `{position}_2차면접용_20260426.md`

---

## 5. xlsx 평가시트 (옵션)

xlsx 산출 시 별도 템플릿 (sheets):
1. 메타·후보자
2. 직무역량 평가 (역량별 BARS + 가중점수 자동)
3. 레벨별 BARS (v0.2 이후)
4. 레벨 전용 질문 (v0.2 이후)
5. MVC 평가
6. 종합 의사결정

xlsx 생성 코드는 `scripts/generate_xlsx.py` (별도, 본 v0.1 범위 외).

---

## 6. 부록: examples/ 와의 차이

`examples/frontend_engineer_sample.md`은 본 템플릿으로 **frontend 변수를 치환한 결과**의 한 예시. 다른 직무는 같은 템플릿 + 다른 변수로 다르게 생성. examples는 LLM 컨텍스트에 주입되지 않음.

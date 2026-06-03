# jd_diff_analysis.md — JD 변경 3-Tier 분류 + 처리 알고리즘

**원칙**: JD가 변경되어도 **검증된 질문은 가능한 한 보존** (구조화 일관성). 단, 능력 요구가 실질 변경되면 재생성.

---

## 1. 3-Tier 분류 (T1/T2/T3)

| Tier | 변경 유형 | 처리 |
|---|---|---|
| **T1** Cosmetic | 표현·어순·오타·동의어 치환 | 자동, 질문 유지, hash만 갱신 |
| **T2** Substantive Equivalent | 도구·기술명 변경 (동일 카테고리 내) | 자동, 새 INSERT + 이전 archive |
| **T3** Capability Change | 역량 추가·삭제·가중치 큰 변화 | **사용자 승인 게이트** + archive + 새 INSERT |

---

## 2. 분류 알고리즘 (LLM 기반)

### Step 1: 텍스트 유사도 1차 검사

```
levenshtein_similarity = 1 - (levenshtein(old, new) / max(len(old), len(new)))
```

- **≥ 0.90** → T1 후보 (cosmetic 가능성 높음)
- **0.70 ≤ s < 0.90** → T2 후보 (substantive equivalent 가능성)
- **< 0.70** → T3 후보 (capability change 가능성 높음)

### Step 2: LLM 의미 분석

다음 프롬프트로 LLM 호출:

```
TASK: 두 JD 버전의 변경 성격을 분류하세요.

OLD JD:
{old_content}

NEW JD:
{new_content}

분류 기준:
- T1 (cosmetic): 의미 동일, 표현·어순·오타만 변경
- T2 (substantive equivalent): 도구·기술명 변경, 평가 역량은 동일
- T3 (capability change): 평가 역량 추가/삭제/가중치 큰 변화

OUTPUT (JSON):
{
  "tier": "T1" | "T2" | "T3",
  "summary": "한 단락 요약",
  "changed_competencies": [{"name": "...", "change": "added|removed|modified"}],
  "recommendation": "keep" | "regenerate_auto" | "regenerate_with_approval"
}
```

### Step 3: Levenshtein과 LLM 결과 정합성 검증

| Levenshtein | LLM | 처리 |
|---|---|---|
| ≥ 0.90 | T1 | T1 확정 |
| ≥ 0.90 | T2 또는 T3 | LLM 우선 (의미적 변경이 표현보다 중요) |
| < 0.90 | T1 | 재검토 (텍스트 변경 큰데 의미 동일이라는 LLM 답변 의심) |
| < 0.70 | T1 또는 T2 | 재검토 |
| 모든 구간 | T3 | T3 확정 + 사용자 게이트 |

---

## 3. Tier별 처리 흐름

### 3.1 T1 처리 (자동, 질문 유지)

```sql
-- 1) 기존 질문의 derived_from_jd_hash 갱신 (UPDATE 차단됨)
-- → 우회: 새 row INSERT + 이전 row archive
INSERT INTO public.interview_questions (
  ...all columns same as old...,
  derived_from_jd_hash = $new_hash,
  version = old.version + 1,
  created_by = 'structured-interview-vX.X-T1-auto'
);

UPDATE public.interview_questions
SET is_active = FALSE,
    archive_reason = 'jd_t1_cosmetic_hash_renewed',
    superseded_by = $new_id
WHERE id = $old_id;

-- 2) jd_change_log 갱신
UPDATE public.jd_change_log
SET diff_tier = 'T1',
    diff_summary = $summary,
    questions_action = 'kept_t1'
WHERE id = $log_id;
```

**알림**: "T1(cosmetic) 변경이 자동 처리되었습니다. 질문은 그대로 유지됩니다."

### 3.2 T2 처리 (자동 재생성)

```
1. competency_extraction 재실행 (새 JD로)
2. 새 역량·가중치가 이전과 동일하면 질문 텍스트 재생성 (도구명 등 업데이트)
3. 새 InterviewQuestions INSERT
4. 이전 활성 질문 archive (archive_reason='jd_t2_substantive')
5. jd_change_log.questions_action = 'regenerated_t2'
6. 사용자에게 변경 보고
```

**알림**: "T2(도구·기술명 변경)으로 자동 재생성되었습니다. 변경 내역: {summary}"

### 3.3 T3 처리 (사용자 승인 게이트)

```
1. 사용자에게 변경 요약 + 영향 분석 보고
2. 옵션 제시:
   (A) 재생성 승인 → 새 INSERT + archive
   (B) 거절 → 기존 질문 유지 (단, jd_change_log.questions_action='user_overrode' 기록)
   (C) 일부만 재생성 (변경된 역량만)
3. 승인 시 처리:
   - competency_extraction 전면 재실행 (역량 추가·삭제 포함)
   - 가중치 재산정 (이전 가중치 승계 금지)
   - 새 질문 세트 INSERT
   - 이전 활성 질문 모두 archive (archive_reason='jd_t3_capability_change')
```

**알림**: "T3(능력 변경) 감지. 재생성 여부를 결정해 주세요."

---

## 4. JSONB 스키마

### 4.1 derived_from_jd_hash

```json
{
  "responsibilities": "<md5>",
  "qualifications": "<md5>",
  "preferred": "<md5>"
}
```

### 4.2 jd_change_log.diff_summary 예시

```
T2 분석 결과: qualifications에서 'Webpack' → 'Vite'로 도구명이 변경됨.
역량 ''번들러 운영''의 본질은 동일하므로 자동 재생성. 영향 받는 질문: C2 BEI/SIT 2개.
가중치·역량 구조 변경 없음.
```

---

## 5. 자동 처리 vs 게이트 정책

본 v0.1의 게이트 정책 (사용자 결정에 따름):

- **Q1=B** (이전 결정): T1·T2 자동, T3만 게이트 (효율 우선)
- **Q2=Y**: Levenshtein ≥ 0.90 → T1
- **Q3=P**: T3 시 가중치 재산정

향후 변경 시 본 문서와 SKILL.md `정책 변경` 섹션 동시 갱신.

---

## 6. 사후 검토 (월 1회 권장)

T2 자동 처리분 검토:

```sql
SELECT id, position_name, section_key, diff_summary, detected_at
FROM public.jd_change_log
WHERE diff_tier = 'T2'
  AND reviewed_by IS NULL
  AND detected_at > NOW() - INTERVAL '30 days'
ORDER BY detected_at;

-- 검토 후
UPDATE public.jd_change_log
SET reviewed_by = $reviewer, reviewed_at = NOW()
WHERE id = $log_id;
```

검토 시 필요하면 T2 자동 처리 결과를 T3로 재분류·archive·재생성 가능.

---

## 7. 환각 방지 (Critical)

T1·T2·T3 분류 시:
- LLM 응답이 모호하면 **항상 더 보수적인 tier로 분류** (T2 의심 → T3 처리)
- 사용자 승인 없이 T3 처리 절대 금지
- 변경 요약 작성 시 JD 실제 텍스트 인용 (변경 부분만)

---

## 8. 학술 근거

- Cizek (2012) *Setting Performance Standards* — 평가 도구 변경 시 archive + supersede 패턴
- Levashina et al. (2014) — 구조화 일관성의 학술적 정당성
- Anthropic Trust & Safety guidelines — 환각 방지·사용자 승인 우선

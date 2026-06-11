# structured-interview Workflow (5 Steps)

본 워크플로우는 직무 중립적이며, 어떤 직무가 입력되든 동일하게 적용됩니다.

---

## Step 1. PARSE — 입력 정규화

**목적**: 사용자의 자연어 입력에서 `position_name` (snake_case)을 추출.

**입력 예시 → 정규화**:
| 사용자 입력 | 정규화 결과 |
|---|---|
| "프론트엔드 면접 준비" | `frontend_engineer` |
| "Frontend Engineer 면접" | `frontend_engineer` |
| "frontend_engineer 면접 질문" | `frontend_engineer` |
| "백엔드 면접 만들어줘" | `backend_engineer` |
| "B2B 영업 인터뷰" | `b2b_sales` |

**규칙**:
1. 영문 직무명 → `snake_case` (Frontend Engineer → `frontend_engineer`)
2. 한글 직무명 → positions 테이블의 `job_title` 또는 `name` 검색 후 매핑
3. 모호하면 사용자에게 정확한 직무명 확인

---

## Step 2. VALIDATE — 가드레일 검증

**목적**: 환각 방지 + 데이터 진실성 보장.

```sql
-- 검증 1: positions 존재
SELECT name, status FROM public.positions
WHERE (name = $1 OR job_title ILIKE $1) AND status='active';

-- 검증 2: position_content JD 존재 (3섹션 모두)
SELECT section_key, LENGTH(content) AS len
FROM public.position_content
WHERE position_name = $1 AND lang = $2;
```

**분기 처리**:

| 검증 결과 | 처리 |
|---|---|
| positions에 미존재 | → `references/new_position_workflow.md` Step A |
| position_content에 0행 | → `references/new_position_workflow.md` Step B |
| 일부 섹션만 존재 | 누락 섹션을 사용자에게 명시 + 입력 요청 |
| 모두 존재 (responsibilities + qualifications + preferred) | → Step 3 진행 |
| **Supabase MCP 끊김** | → `references/jd_fetch.md §7.1` 복구 (ToolSearch 재로드 → 재시도 1회 → 실패 시 중단·보고, 추측 금지) |

### 2.1 다국어(lang) 매칭 규칙

`positions.lang`과 `position_content.lang`이 다를 수 있습니다(예: positions.lang='en'인데 JD는 ko/en 모두 존재). 다음 규칙을 적용:

1. **사용자 요청 lang 우선**: 사용자가 "한글 면접" 요청 → `position_content.lang='ko'` 사용
2. **fallback 순서**: 요청 lang → ko → en 순으로 시도
3. **positions.lang은 등록 메타정보**: 면접 질문 lang은 `position_content.lang` 기준
4. **lang 불일치 알림**: positions와 content lang이 다르면 사용자에게 1회 명시:
   > "이 직무는 positions에 lang='{positions_lang}'으로 등록되어 있고, JD 콘텐츠는 lang='{content_lang}'으로 사용됩니다."

### 2.2 직무명 특수문자 처리

`research_scientist(Evaluation)`처럼 괄호·공백 등 특수문자가 포함된 직무명도 그대로 사용. 단:
- SQL literal에서 ' 이스케이프 (`''`)
- 파일명 사용 시 OS-safe로 변환 (예: `research_scientist_Evaluation_`)

**환각 방지 핵심**: 이 단계에서 0행이 나오면 **JD 내용을 절대 추측·생성하지 않음**. 사용자에게 입력 요청만.

---

## Step 3. FETCH — 기존 질문 + hash 비교

**목적**: 재현성 보장 + JD 변경 감지.

```sql
-- 활성 질문 조회 (sync_status 포함)
SELECT * FROM public.v_active_interview_questions
WHERE position_name = $1
ORDER BY competency_key, question_order;

-- 현재 JD hash ($2 = Step 2.1에서 결정된 lang, 요청 lang → ko → en 폴백)
SELECT section_key, MD5(content) AS hash
FROM public.position_content
WHERE position_name = $1 AND lang = $2
ORDER BY section_key;
```

**hash 비교 결과 분류** (3-tier 분류는 `references/jd_diff_analysis.md` 참조):

| sync_status | 의미 | 다음 단계 |
|---|---|---|
| (no rows) | 첫 호출 | Step 4 → GENERATE |
| `in_sync` | hash 일치 = 재현성 보장 | Step 5 → ASSEMBLE (그대로 사용) |
| `jd_changed_review_needed` | hash 불일치 → diff 분석 필요 | Step 4 → DIFF 분석 |
| `no_hash` | (이상 케이스, 마이그레이션 데이터 등) | Step 4 → REGENERATE |

---

## Step 4. DECIDE — 생성·재사용·재생성 분기

### 4A. 첫 호출 (활성 질문 0행)

```
1. references/competency_extraction.md → JD에서 역량 N개 + 가중치 도출
2. references/question_generation.md → 역량별 BEI 1 + SIT 1 페어링
3. references/bars_anchoring.md → 5점 BARS 앵커
4. references/probing_patterns.md → 꼬리질문 5+개
5. Red Flag 4+개, Green Flag 4+개 도출
6. interview_questions에 INSERT (with derived_from_jd_hash, created_by='structured-interview-vX.X')
7. Step 5 → ASSEMBLE
```

### 4B. 재호출 (in_sync)

→ 그대로 fetch한 질문 사용. 추가 생성 불가 (immutable). Step 5 → ASSEMBLE.

### 4C. 재호출 (jd_changed_review_needed) → 3-tier diff 분석

`references/jd_diff_analysis.md` 알고리즘 호출:

| Tier | 처리 |
|---|---|
| **T1 (cosmetic)** | 자동: 질문 내용은 그대로 둔 채 새 hash로 재-INSERT(version+1) + 이전 행 archive (`jd_diff_analysis.md §3.1` — UPDATE는 트리거가 차단) + jd_change_log.diff_tier='T1' 기록 |
| **T2 (substantive equivalent)** | 자동: 새 질문 INSERT + 이전 행 archive(superseded_by 링크) + jd_change_log 기록 + 사용자에게 변경 알림 |
| **T3 (capability change)** | **사용자 승인 게이트** — 변경 요약 보고 → 승인 시 새 INSERT + archive |

```sql
-- T1/T2/T3 archive 처리 (immutability 트리거는 is_active·archive_reason·superseded_by 변경만 허용)
UPDATE public.interview_questions
SET is_active = FALSE,
    archive_reason = 'jd_t2_substantive' OR 'jd_t3_capability_change',
    superseded_by = $new_id
WHERE id = $old_id;
```

### 4D. 환각 자기 점검 (Critical)

INSERT 직전:
1. `derived_from_jd_hash` ≡ `MD5(position_content.content)` 동일성 검증
2. `source_jd_lines`가 실제 JD 라인을 가리키는지 확인
3. 위 둘 중 하나라도 실패하면 **INSERT 중단** + 사용자 보고

---

## Step 5. ASSEMBLE — 산출물 생성

**모듈 결합**:

```
[활성 직무 질문 N개]   ← Step 3·4 결과
       +
[MVC 5가치 모듈]      ← Supabase mvc_* 테이블 fetch (mvc_schema.md §3) · 자동 부착
       +
[직무별 컨텍스트 매트릭스]  ← mvc_position_context WHERE position_name=$1
       ↓
references/output_template.md 변수 치환
       ↓
[산출물 .md 1개]
       (옵션: .xlsx 평가시트)
```

**MVC 출처 (v0.3.0)**: Supabase `mvc_*` 테이블에서 fetch한다 (편집 가능). DB 미이전·0행·MCP 끊김 시 `mvc/_core_values.md`(시드)로 폴백하고 출력에 "ℹ️ MVC 파일 시드 사용(DB 미이전)"을 표기한다. 어느 경우든 **5가치 전부** 포함을 확인한다(5개 미만이면 폴백·재확인, 선별 금지).

**저장 위치**:
- 기본: 사용자 작업 폴더 (예: `<workspace>/면접/{position_name}_{candidate}_{YYYYMMDD}.md`)
- 사용자 지정 가능

**파일 명명 규칙**: `{position_name}_{candidate_or_purpose}_{YYYYMMDD}.md`

---

## Step 6 (선택). 후속 검증

산출물 생성 후 권장 자체 점검:

- [ ] `derived_from_jd_hash`가 응답에 인용되어 추적 가능한가
- [ ] 모든 BARS 앵커가 5단계 차등이 명확한가
- [ ] 꼬리질문이 모두 완전 문장(면접관 그대로 사용 가능)인가
- [ ] Red Flag·Green Flag가 각각 4개 이상인가 (Step 4A 기준과 동일)
- [ ] MVC 모듈이 부착되었는가

---

## 부록: 운영 정책

### A. T2 자동 처리 사후 검토 (월 1회)

```sql
SELECT * FROM public.jd_change_log
WHERE diff_tier = 'T2'
  AND reviewed_by IS NULL
  AND detected_at > NOW() - INTERVAL '30 days';
```

→ 사용자가 검토 후 `reviewed_by`, `reviewed_at` 채움.

### B. 신규 직무 추가 절차

`references/new_position_workflow.md` 참조.

### C. 환각 사고 발생 시

`is_active=FALSE` + `archive_reason='hallucinated_*'`로 즉시 archive. 데이터 추적성 보존.

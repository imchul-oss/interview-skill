# new_position_workflow.md — 신규 직무 등록 흐름

**원칙**: 사용자가 미등록 직무를 호출했을 때, **거부만 하지 않고 등록 흐름으로 안내**합니다.

---

## 시나리오 분기

```
사용자: "{{새 직무}} 면접 준비"
       ↓
positions 미존재? → Step A
positions 있는데 position_content 미존재? → Step B
position_content 일부 섹션 누락? → Step C
모두 정상 → 일반 흐름으로 복귀
```

---

## Step A: 직무 자체 미등록

**메시지**:
```
"{{직무명}}"는 아직 positions 테이블에 등록되지 않았습니다.
현재 등록된 직무 {{N}}개:
  {{등록된 직무 목록 — positions에서 fetch한 name (job_title)}}

이 직무를 신규 등록하시겠습니까?
- (A) 예. 등록 진행 (이름·status·sort_order 입력 받음)
- (B) 아니요. 위 목록에서 다시 선택
- (C) 직무명 오타 — 정확한 이름 알려주세요
```

**A 선택 시**:
1. 사용자에게 다음 정보 요청:
   - `name` (snake_case): 예) `data_scientist`
   - `job_title` (표시명): 예) `Data Scientist`
   - `lang` (`ko` 또는 `en`)
   - `status` (`active` 또는 `draft`)
   - `sort_order` (정렬 순서, 정수)
2. INSERT:
   ```sql
   INSERT INTO public.positions (name, job_title, lang, status, sort_order)
   VALUES ($1, $2, $3, $4, $5);
   ```
3. → Step B로 자동 진행 (JD 입력 받기)

---

## Step B: positions 있으나 JD 미등록

**메시지**:
```
"{{직무명}}"는 등록되어 있지만 JD가 입력되지 않았습니다.
구조화 면접 질문을 생성하려면 JD가 필요합니다.

다음 3개 섹션을 입력해 주세요:
  1. responsibilities (주요 책임)
  2. qualifications (필수 자격요건 — 하드스킬 + 소프트스킬)
  3. preferred (우대 사항)

각 섹션은 100자 이상 권장.
```

**입력 받은 후**:
```sql
INSERT INTO public.position_content (position_name, lang, section_key, content)
VALUES
  ($1, 'ko', 'responsibilities', $2),
  ($1, 'ko', 'qualifications', $3),
  ($1, 'ko', 'preferred', $4);
```

→ Step 3 (FETCH)로 자동 진행

---

## Step C: 일부 섹션만 존재

**메시지**:
```
"{{직무명}}"의 JD가 일부만 등록되어 있습니다.
누락된 섹션: [responsibilities] / [qualifications] / [preferred]

누락된 섹션 내용을 입력해 주세요.
```

**입력 받은 후**:
```sql
INSERT INTO public.position_content (position_name, lang, section_key, content)
VALUES ($1, 'ko', $2, $3);
```

→ 모든 섹션 검증 후 Step 3 진행

---

## 환각 방지 가드

- 사용자가 JD 입력을 명시적으로 하기 전까지 **임의 JD 생성 금지**
- "일반적인 데이터 사이언티스트 JD를 만들어 드릴까요?" 같은 제안 **금지** (스킬의 본질 = 사용자 회사 JD 기반)
- JD 입력이 너무 간단하면 (50자 미만) 보강 요청

---

## 등록 완료 확인 (검증)

새 직무 등록 후 일반 흐름으로 진입하기 전 검증:

```sql
SELECT
  p.name,
  p.status,
  COUNT(pc.section_key) AS jd_sections
FROM public.positions p
LEFT JOIN public.position_content pc ON p.name = pc.position_name AND pc.lang = $2
WHERE p.name = $1
GROUP BY p.name, p.status;
```

- `status = 'active'` AND `jd_sections = 3` 일 때만 진행
- 그 외 → 사용자에게 누락 알림

---

## 자주 발생할 수 있는 케이스

### 케이스 1: 직무명 표기 차이
- 사용자가 "프론트엔드 엔지니어"라고 입력 → positions의 `frontend_engineer`와 매칭
- positions에 한글 별칭 컬럼이 없으므로, 다국어 `job_title` 또는 fuzzy match 활용

### 케이스 2: 같은 직무의 시니어/주니어 분리 요청
- v0.1에서는 단일 직무 = 단일 질문 세트 (level은 'all'로 고정)
- 시니어/주니어 분리는 v0.2 이후 기능

### 케이스 3: 직무 통합·분리
- positions에 새 직무 INSERT 후 기존 직무 archive (status='archived')
- 기존 interview_questions는 그대로 보존 (immutable)
- 새 직무에 대해 새 질문 생성

---

## 학술 근거

- *Levashina et al. (2014)* "직무분석 기반 질문" — JD 없이는 구조화 면접 성립 불가
- *Campion et al. (1997)* "Job-relatedness" — 채용 의사결정의 법적 방어 기반

---
name: structured-interview-q
display_name: 조직 면접 맞춤 가이드 (CV 기반)
description: |
  조직 후보자별 맞춤 면접 가이드 생성 스킬. structured-interview 스킬이 만든 표준 질문 세트(interview_questions)를 fetch하고, 사용자가 입력한 후보자 CV와 매칭하여 (1) 강점·약점·갭 분석, (2) BEI 질문에 CV 구체 인용 추가, (3) 면접 단계(1차/2차) + 시간 제약 내 질문 우선순위 산정 후 후보자별 맞춤 .md 가이드를 생성합니다. interview_questions는 SELECT만, 절대 변경하지 않아 구조화 일관성을 보존합니다. CV는 세션 내에서만 사용하고 DB·메모리에 영속 저장하지 않습니다.
version: 0.3.0
companion_skill: structured-interview (Question Set Builder)
data_source: supabase (config.json project_ref — structured-interview/references/setup_connection.md)
academic_basis:
  - Schmidt & Hunter (1998) — 구조화 면접 r=.51
  - Levashina et al. (2014) — 구조화 일관성 ("같은 직무 = 같은 표준 질문")
  - Janz (1982) — Patterned Behavior Description Interview
  - McClelland (1973) — Behavioral Event Interview
  - Campion et al. (1997) — 15 구조화 요소 중 ''follow-up standardized''
  - Dreyfus & Dreyfus (1980) — Novice→Expert 5단계 (레벨별 BARS 차등 근거)
---

# structured-interview-q — 조직 면접 맞춤 가이드 스킬

## 1. 스킬의 본질 (이름의 의미)

- **structured-interview**: 직무 → 표준 질문 세트 (Question Set Builder, **writer**)
- **structured-interview-q**: 후보자 → 맞춤 면접 가이드 (Personalizer, **reader**)

두 스킬은 **분리된 read/write 역할**을 가지며, 데이터 무결성·구조화 일관성을 함께 보존합니다.

## 2. 사용 시점 (When to use)

본 스킬은 **사용자가 슬래시 커맨드로 명시 발동**하는 모델입니다:

```
사용자: /structured-interview-q
   ↓
스킬 발동 → 사용자에게 4가지 입력 안내
```

또한 다음 발화 패턴도 트리거:

- "이 후보자에 맞춰 면접 준비해줘" (CV 첨부 시)
- "[후보자명] [직무] 면접 가이드"
- "[후보자명] CV로 면접 다듬어줘"

## 3. 스킬 발동 시 첫 안내 메시지 (Standard Prompt)

```
**조직 면접 맞춤 가이드** 발동되었습니다.

다음 4가지 정보를 입력해 주세요:

1. **직무명** (필수): 예) frontend_engineer / "프론트엔드 엔지니어"
2. **후보자 CV** (필수): 파일 업로드 또는 텍스트 붙여넣기
3. **면접 단계** (필수): 1차 / 2차
4. **면접 시간** (필수): 분 단위 (예: 60, 90, 120)

선택 입력:
5. **트랙** (선택): 시니어 / 미들 / 주니어. 명시 시 해당 레벨 BARS 앵커가 렌더됩니다.
   미입력 시 track=unspecified로 처리하며 LLM이 CV에서 시니어리티를 추정하지 않습니다.

직무를 먼저 알려주시지 않으면 CV에서 추출을 시도하되, 명확하지 않으면
직무명을 다시 한 번 확인합니다.
```

**2차 면접 선택 시 추가 질문 (v0.2.0)**: 면접 단계가 **2차**이면, 위 입력 시각화(질문지)에 다음 두 항목을 **추가로 노출**하여 면접관에게 묻는다. 입력된 내용은 가이드의 2차 전용 강화 섹션으로 반영된다 (§5.6).

```
(2차 한정 추가 입력 — 모두 선택)
6. **추가 검증 희망 영역**: 1차 결과·서류에서 더 파고들고 싶은 부분이 있나요?
   (예: "C1 TLS 복호화 실무 깊이", "리더십·팀 갈등 관리")
7. **레퍼런스 체크 추가 항목**: 평판조회에서 확인하고 싶은 항목이 있나요?
   (예: "실제 프로젝트 기여도", "퇴사 사유", "협업 태도")
```

## 4. 핵심 원칙 (Inviolable Principles)

1. **데이터 무결성**: `interview_questions` 테이블은 **SELECT만**. INSERT/UPDATE/DELETE 절대 금지
2. **CV 영속 금지**: 세션 종료 시 CV 데이터 즉시 폐기. DB·메모리·로그·`.md` 파일 어디에도 영속 저장 금지
3. **환각 금지**: CV에 명시된 사실만 인용. 추측·일반화·합성 금지 (예: "이 회사 출신은 보통..." ❌)
4. **추적성**: 모든 매칭·인용에 `source_cv_line` 또는 `cv_excerpt` 표시
5. **구조화 보존**: 표준 질문 텍스트는 절대 수정하지 않고, 별도 `cv_specific_addendum` 필드에 인용을 추가
6. **결정성** (v0.1.4): MVC 5가치 선별 금지, 6역량+5가치 외 자유 영역 신설 금지, 검증 우선순위 라벨 금지, 시간 배분은 `priority_algorithm.md §0` 결정적 알고리즘 사용
7. **레벨 명시성** (v0.2.0): BARS 레벨(junior/mid/senior)은 **사용자가 트랙을 명시 입력한 경우에만** 선택. LLM이 CV로 레벨을 추정하지 않으며, DB에 레벨별 앵커가 없으면 단일(`all`) BARS로 폴백 + 주석 명시

## 5. 가드레일 (Guard Rails)

| 검증 단계 | 미통과 시 동작 |
|---|---|
| 직무명 미등록 (positions에 없음) | 거부 + `structured-interview` 스킬로 안내 |
| 직무에 활성 질문 0행 (interview_questions) | 거부 + `structured-interview` 발동 안내 |
| CV 파일·텍스트 모두 미입력 | 입력 요청, 추측·생성 금지 |
| CV 길이 < 200자 | "CV 내용이 충분하지 않습니다. 다시 확인 부탁드립니다" |
| 면접 단계·시간 미입력 | 묻기 (질문 우선순위 산정에 필수) |
| 사용자가 "CV 없이 일반 가이드 만들어줘" | 거부 — structured-interview만 사용 권고 |
| Supabase MCP 연결 끊김 | §5.5 복구 절차 (ToolSearch 재로드 → 재시도 1회 → 그래도 실패 시 중단·보고) |

### 5.1 결정성 가드 (v0.1.4 추가, Critical)

운영 데이터 분석 결과 LLM의 자유 재량이 후보자 간 비교 가능성을 깨뜨림. 다음 자유를 절대 행사하지 않는다.

| 금지 | 사유 |
|---|---|
| MVC 5가치 중 일부만 선별 평가 | 후보자 간 가치 비교 불가능 |
| 6역량 + MVC 5가치 외 추가 자유 영역 신설 | 평가 기준 부동 (예: "AI 도메인 적응력" 같은 임의 추가) |
| 검증 우선순위 라벨 (High/Medium/Critical/★) 부여 | 매칭 강도와 중복, 임의 강조 |
| 시간 배분 자유 재할당 | `priority_algorithm.md §0` 결정적 공식만 사용 |
| 트랙 (시니어/미들/주니어) 임의 분류 | 사용자가 명시 입력한 경우에만 frontmatter 표기 |

### 5.2 트랙 명시 입력 정책

사용자가 면접 단계·시간과 함께 트랙(시니어/미들/주니어)을 명시 입력한 경우에만 frontmatter `track` 필드에 기록한다. 명시되지 않은 경우 `track: unspecified`로 처리하며, LLM이 CV에서 임의로 시니어리티를 추정·분류하지 않는다.

### 5.3 시인성·인쇄 가드 (v0.1.5 추가, Critical)

운영 가이드 .md가 VS Code Preview·A4 인쇄에서 가독성이 떨어진다는 피드백 기반. 출력 시 다음을 강제한다.

| 가드 | 적용 위치 |
|---|---|
| 모든 헤더(`##`·`###`) 위·아래 빈 줄 1개 강제 | 마크다운 단락 구분 |
| 모든 표 위·아래 빈 줄 1개 강제 | VS Code Preview 렌더링 |
| 표 직후 다른 표 시작 금지 (사이 설명 또는 빈 줄 2개) | 시각적 휴식 |
| 한 문단 ≤ 5 문장 (초과 시 두 문단으로 분리) | 인지 부하 감소 |
| 각 질문 카드 사이 `---` 가로 구분선 삽입 (page-break div 미사용) | 프리뷰 깔끔 분리, A4는 PDF 변환 설정 |
| 표 컬럼 수 ≤ 6, 폭 ≤ 80자 | A4 portrait 친화 |
| h1·h2·h3만 사용 (h4 이상 금지) | 목차 깊이 통제 |

세부 규칙은 `references/output_personalized.md §10` 참조.

### 5.4 트랙 → 레벨별 BARS 선택 정책 (v0.2.0 추가, Critical)

레벨별 차등 BARS(L2)를 다음 결정적 규칙으로 렌더한다. 같은 트랙·직무 입력은 같은 BARS를 보장한다 (재현성).

| 입력 트랙 | DB에 `bars_anchors_by_level` 존재 | 렌더되는 BARS |
|---|:---:|---|
| senior / mid / junior | ✅ | 해당 레벨의 5점 앵커 (`bars_anchors_by_level[track]`) |
| senior / mid / junior | ❌ (level='all'만) | 단일 `bars_anchors` + 주석: "이 직무는 레벨별 앵커 미작성, 단일 기준 사용" |
| unspecified | ✅ 또는 ❌ | 단일 `bars_anchors`(또는 mid 레벨)을 기준선으로 렌더 + 주석 |

폴백 원칙: **레벨별 앵커가 없다고 임의로 레벨별 앵커를 생성하지 않는다** (환각 금지). 단일 앵커로 폴백하고 그 사실을 출력에 명시한다. 레벨별 앵커 작성은 structured-interview(writer) + 시니어 패널 검토를 통해서만 DB에 등록된다 (`bars_anchoring.md §7`).

### 5.5 Supabase MCP 끊김 복구 절차 (v0.2.0 추가)

면접 운영 중 Supabase MCP 도구(`mcp__*__execute_sql` 등)가 연결 끊김 상태일 수 있다. 다음 순서로 복구한다.

1. ToolSearch로 `select:mcp__<supabase>__execute_sql` 등 스키마 재로드 시도
2. 재로드 성공 시 직전 SELECT 1회 재시도
3. 그래도 실패 시 **중단** + 사용자에게 정직 보고 ("Supabase 연결이 끊겨 표준 질문을 fetch할 수 없습니다. 재연결 후 재시도해 주세요")
4. **연결 실패를 메모리·추측으로 우회하지 않는다** — 표준 질문을 만들어내거나 캐시 추정으로 진행 금지

### 5.6 2차 면접 강화 elicitation (v0.2.0 추가)

면접 단계가 **2차**이면 입력 시각화(질문지)에 §3의 추가 입력 2개(추가 검증 희망 영역 · 레퍼런스 체크 추가 항목)를 노출하고, 입력 결과를 가이드에 **2차 전용 섹션**으로 반영한다.

| 입력 | 가이드 반영 | 규칙 |
|---|---|---|
| 추가 검증 희망 영역 | "추가 검증 포인트" 섹션 — 해당 영역을 표준 질문·꼬리질문과 연결, 관찰 포인트 명시 | 표준 질문 텍스트 미수정 (cv_specific_addendum/관찰로만 보강) |
| 레퍼런스 체크 추가 항목 | "레퍼런스 체크 포인트" 섹션 — 평판조회 시 확인할 질문 형태로 정리 | CV·면접 답변으로 검증 불가한 항목임을 명시 |

가드:
- 1차 면접에는 위 두 섹션을 **생성하지 않는다** (2차 한정).
- 입력이 비어 있으면 해당 섹션을 생략한다 (빈 섹션·임의 생성 금지, 환각 방지).
- 추가 검증/레퍼런스 항목을 빌미로 표준 질문을 수정하거나 새 직무 질문을 생성하지 않는다 (immutable 보존).

## 6. 워크플로우 (5 Steps)

세부는 [`workflow.md`](workflow.md) 참조.

```
Step 1. INITIATE   슬래시 커맨드 발동 + 4(+1)가지 입력 받기 (직무·CV·단계·시간·[트랙])
Step 2. PARSE&FETCH CV 파싱 + interview_questions SELECT (수정 안 함)
Step 3. MATCH      CV 역량 ↔ 직무 역량 매칭 (강점·약점·갭 분석)
Step 4. TAILOR     BEI 질문에 CV 구체 인용 추가 + 우선순위 산정 + 트랙 레벨 BARS 선택
Step 5. ASSEMBLE   맞춤 .md 산출 (1차/2차 + 시간 배분 + 레벨 BARS 반영) → 세션 종료 시 CV 폐기
```

## 7. References (LLM이 읽는 직무 중립 알고리즘)

| 파일 | 역할 |
|---|---|
| `references/cv_parsing.md` | CV 텍스트·파일 파싱 + 구조화 (개인정보 가드 포함) |
| `references/competency_matching.md` | CV 역량 ↔ 직무 역량 매칭 알고리즘 |
| `references/question_personalization.md` | BEI 질문에 CV 인용 추가 패턴 |
| `references/priority_algorithm.md` | 1차/2차 + 시간 제약 내 질문 우선순위 |
| `references/output_personalized.md` | 맞춤 .md 변수 치환 템플릿 + 레벨 BARS 렌더 + 시인성 규칙 |

레벨별 BARS 앵커 자체의 작성 규칙은 companion 스킬 `structured-interview/references/bars_anchoring.md §7`에 정의되어 있으며, 본 스킬은 DB에 저장된 앵커를 **읽어서 렌더만** 한다.

## 8. 외부 의존 (Companion Skill)

- **`structured-interview`** (writer): interview_questions의 데이터 출처
- **Supabase**: 연결 대상은 config.json(`structured-interview/references/setup_connection.md`)으로 결정 (하드코딩 아님). 테이블 `positions`·`position_content`·`interview_questions`·`v_active_interview_questions` + MVC `mvc_*`(v0.3.0, 편집 가능 — `mvc_schema.md`)
- **MVC 출처**: Supabase `mvc_*` fetch, 미이전·MCP끊김 시 `mvc/_core_values.md` 시드 폴백 + 주석

## 9. 산출물 (Output)

- **단일 파일**: `{candidate_name}_{position_name}_{stage}_{date}.md`
- **저장 위치**: `<workspace>/면접/` (사용자 폴더)
- **xlsx 평가시트** (선택, 로드맵 L1): `output_personalized.md` 참조 — 본 v0.2.0 범위 외

## 10. 개인정보 정책

CV는 **개인 식별 정보**를 포함합니다. 본 스킬은 다음을 준수:

1. CV 파일은 세션 내에서만 메모리에 로드, **세션 종료 시 즉시 폐기**
2. 출력 .md에 후보자 식별 정보 최소화 (이름·경력만, 주소·주민번호·전화번호 등 제외)
3. CV 원본 텍스트를 `.md` 출력에 포함하지 않음 (인용은 발췌만)
4. 메모리 시스템(`.claude/memory/`)에 CV 관련 정보 저장 금지
5. 다른 면접관과 공유 시 후보자 동의 가정 (스킬은 정책 준수만)

## 11. DO NOT (절대 금지)

- ❌ `interview_questions` UPDATE / INSERT / DELETE
- ❌ CV 텍스트를 메모리·DB·로그에 영속 저장
- ❌ CV에 없는 사실 추측·일반화·합성
- ❌ 표준 질문 텍스트 수정 (인용은 별도 필드에 추가)
- ❌ 레벨별 BARS 앵커를 임의 생성 (DB에 없으면 단일 앵커 폴백 + 주석)
- ❌ `<workspace>/_archive/structured_interview_v0.1_examples/` 참조 (격리됨)
- ❌ 직무·표준 질문 미존재 시 임의로 질문 생성

## 12. 변경 이력

| Version | 날짜 | 변경 |
|---|---|---|
| 0.1 | 2026-04-28 | 초기 패키징 (Phase 3 구현, structured-interview의 짝 스킬) |
| 0.1.4 | 2026-04-29 | 결정성 가드 (§5.1): MVC 5가치 강제·자유영역 금지·라벨 금지·결정적 시간 배분 |
| 0.1.5 | 2026-05-27 | 시인성·인쇄 가드 (§5.3): 단락 구분·페이지 분리·A4 친화 |
| 0.1.6 | 2026-06-02 | Supabase 연결정보 정합화 (companion writer와 동기화) |
| 0.2.0 | 2026-06-03 | 하드닝(손상 파일 §11~13 복구) + L2 레벨별 BARS 선택(§5.4) + MCP 끊김 복구(§5.5) |
| 0.3.0 | 2026-06-03 | Supabase 연결 config 기반(하드코딩 제거, setup_connection.md) + MVC를 DB(`mvc_*`)에서 fetch·편집 가능(파일 폴백) |

## 13. 미해결 (v0.2+)

- **레벨별 BARS DB 마이그레이션**: 스킬로직·렌더·폴백은 v0.2.0 완료. 실제 `bars_anchors_by_level` 스키마 적용 + level='all' 135행 분리는 시니어 패널 검토 후 별도 운영 작업 (`roadmap_v0.2_progress.md` 참조)
- 레퍼런스 체크 포인트 자동 생성 (사용자가 추가 요구 시)
- 후보자별 평가 기록 보존 (별도 candidate_evaluations 테이블, 동의 후 — 로드맵 L3)
- CV 다국어 지원 (현재 한·영 우선)

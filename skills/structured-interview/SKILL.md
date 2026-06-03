---
name: structured-interview
display_name: 조직 구조화 면접
description: |
  조직 구조화 면접 질문지 자동 생성·관리 스킬. 직무명을 입력하면 Supabase에 등록된 JD를 fetch하여 역량 추출 → BEI+SIT 질문 + BARS 5점 + 꼬리질문 + Red/Green Flag + MVC 5가치 평가 모듈을 자동 생성하고, interview_questions 테이블에 영속 저장합니다. 한 번 생성된 질문은 immutable로 보존되어 모든 면접관이 동일한 질문을 받습니다(구조화 면접 일관성 원칙). JD가 변경되면 변경 정도를 분류(T1/T2/T3)하여 재생성 여부를 결정합니다. 미등록 직무는 생성 거부 후 등록 워크플로우로 안내합니다.
version: 0.3.0
data_source: supabase (config.json project_ref — setup_connection.md)
academic_basis:
  - Schmidt & Hunter (1998) Psychological Bulletin — 구조화 면접 예측타당도 r=.51 / 워크샘플 r=.54
  - Levashina et al. (2014) Personnel Psychology — 6대 핵심 구조화 요소
  - Latham et al. (1980) Journal of Applied Psychology — Situational Interview
  - Smith & Kendall (1963) Journal of Applied Psychology — BARS
  - Kristof-Brown et al. (2005) — P-O Fit 메타분석
  - Dreyfus & Dreyfus (1980) — Novice→Expert 5단계 (레벨별 BARS 차등)
---

# structured-interview — 조직 구조화 면접 스킬

## 1. 사용 시점 (When to use)

이 스킬은 다음 입력 패턴에서 발동합니다:

- "[직무명] 면접 준비"
- "[직무명] 구조화 면접"
- "[직무명] 면접 질문"
- "[직무명] 인터뷰 만들어줘"
- "조직 면접 [직무명]"
- "구조화 면접 [직무명]"

직무명은 한글(예: "프론트엔드"), 영문(예: "frontend engineer"), snake_case(예: "frontend_engineer") 모두 허용. 스킬 내부에서 정규화.

## 2. 핵심 원칙 (Inviolable Principles)

1. **데이터 진실성**: Supabase `position_content`에 JD가 없으면 **절대 생성하지 않음** (환각 금지)
2. **구조화 일관성** (*Levashina et al. 2014*): 같은 직무 = 같은 질문. interview_questions는 immutable
3. **추적성**: 모든 INSERT는 `derived_from_jd_hash` + `created_at` 기록
4. **변경 통제**: JD 변경은 자동 감지(jd_change_log 트리거) + 3-tier 분류(T1/T2/T3)로 처리
5. **직무 중립**: 본 스킬과 references는 어떤 특정 직무에도 종속되지 않음

## 3. 가드레일 (Guard Rails)

```
사용자 입력 → 정규화 → positions 검증 → JD 검증
                  ↓ 미통과 시
        [신규 직무 등록 워크플로우]
        (references/new_position_workflow.md)
```

| 검증 단계 | 미통과 시 동작 |
|---|---|
| positions 테이블에 직무 미존재 | 등록된 직무 리스트 표시 + "이 직무를 신규 등록할까요?" 확인 |
| position_content에 JD 미존재 | "이 직무의 JD(responsibilities/qualifications/preferred)를 입력해 주세요" 안내 |
| JD 일부 섹션만 있음 | 누락 섹션 명시 + 입력 요청 |
| 사용자가 임의 텍스트로 JD 즉석 작성 | **거부** — Supabase에 정식 등록 후 재호출 안내 |

### 3.1 임의 JD 거부 정책 (Critical)

본 스킬은 **Supabase position_content 외 어떤 출처의 JD도 사용하지 않음**. 다음 케이스 모두 거부:

1. 사용자가 채팅창에 "이 JD로 만들어줘"라고 텍스트 직접 붙여넣기 → 거부, "Supabase에 등록 후 재호출"
2. "일반적인 {직무} JD를 만들어 봐" 요청 → 거부, JD 생성은 본 스킬의 책임 영역 아님
3. 부분 정보 + "나머지는 알아서" → 거부, 모든 섹션이 명시적으로 등록되어야 함
4. 임시 .docx/.pdf 파일 업로드 → 거부 또는 등록 가이드 안내

이 정책은 *Levashina et al. (2014) 직무분석 기반 질문* 원칙 + 환각 방지를 위함.

### 3.2 환각 방지 다층 가드

1. fetch 결과 row count 즉시 검증, 0행이면 즉시 중단
2. INSERT 전 `derived_from_jd_hash` ≡ `MD5(content)` 검증
3. JD 내용을 절대 추측·요약·합성하지 않음 (인용만 허용)
4. 환각 발견 시 즉시 archive(`is_active=FALSE` + `archive_reason='hallucinated_*'`)

### 3.3 examples/ 격리

본 스킬 폴더에는 examples/가 없으며, 사람 참고용 시범 산출물은 `<workspace>/_archive/structured_interview_v0.1_examples/`에 격리되어 있음. SKILL.md·workflow.md·references/ 어디서도 examples를 참조하지 않음.

## 4. 워크플로우 (5 Steps)

세부는 [`workflow.md`](workflow.md) 참조.

```
Step 0. SETUP     (최초 1회) Supabase 연결 config 확인·없으면 입력받아 저장 (setup_connection.md)
Step 1. PARSE     사용자 입력 → position_name 정규화
Step 2. VALIDATE  positions + position_content 검증 (가드레일)
Step 3. FETCH     interview_questions에서 활성 질문 조회 + JD hash 비교
Step 4. DECIDE    첫 호출 / in_sync / T1 / T2 / T3 → 처리 분기
Step 5. ASSEMBLE  직무 모듈 + MVC 모듈(Supabase fetch) 결합 → 산출물 생성 (.md, optional .xlsx)
```

Step 0: 모든 발동 시 먼저 `<워크스페이스>/.structured-interview/config.json`을 확인한다. 없으면 `references/setup_connection.md`의 SETUP을 발동하여 Supabase project URL·publishable key를 입력받아 저장한 뒤 진행한다. 하드코딩된 기본 프로젝트로 임의 진행하지 않는다.

## 5. References (LLM이 읽는 규칙·알고리즘)

본 스킬이 참조하는 references는 모두 **직무 중립**입니다. 어떤 특정 직무 사례도 박혀있지 않습니다.

| 파일 | 역할 |
|---|---|
| `workflow.md` | 5단계 상세 |
| `references/setup_connection.md` | (v0.3.0) Supabase 연결 최초 세팅 + config.json |
| `references/mvc_schema.md` | (v0.3.0) MVC Supabase 스키마 + 편집·마이그레이션 |
| `references/jd_fetch.md` | Supabase 쿼리 패턴 + hash 계산 |
| `references/competency_extraction.md` | JD → 역량 추출 + 가중치 산정 알고리즘 |
| `references/question_generation.md` | BEI/SIT 질문 작성 패턴 |
| `references/bars_anchoring.md` | BARS 1~5점 앵커 작성 규칙 (Smith & Kendall 1963) |
| `references/probing_patterns.md` | STAR 4단계 꼬리질문 카탈로그 |
| `references/jd_diff_analysis.md` | T1/T2/T3 분류 알고리즘 + 처리 |
| `references/output_template.md` | 출력 .md 변수 치환 템플릿 (역량 개수 가변) |
| `references/new_position_workflow.md` | 신규 직무 등록 흐름 |
| `mvc/_core_values.md` | MVC 5가치 모듈 — **시드·오프라인 폴백** (편집은 Supabase `mvc_*` 테이블, `mvc_schema.md`) |

## 6. References에 포함되지 않는 것 (DO NOT reference)

- `examples/` 폴더 — **사람이 결과 형태를 참고하는 용도**일 뿐, LLM 컨텍스트에 절대 주입되지 않음. examples의 frontend 구체 가중치(25/22/18/15/10/10) 등은 새 직무 호출 시 leak되면 안 됨.

## 7. 인접 시스템 (External Dependencies)

| 시스템 | 용도 |
|---|---|
| Supabase project (config.json의 project_ref; 기본 제안 `<YOUR_PROJECT_REF>`) | 단일 진실 원천 |
| 테이블 `positions`, `position_content` | JD 마스터 |
| 테이블 `interview_questions` | 면접 질문 마스터 (immutable) |
| 테이블 `jd_change_log` | JD 변경 자동 추적 (트리거 자동 INSERT) |
| 테이블 `mvc_values`/`mvc_questions`/`mvc_bars`/`mvc_flags`/`mvc_position_context` | (v0.3.0) MVC 편집 가능 마스터 — `mvc_schema.md` |
| 뷰 `v_active_interview_questions` | 활성 질문 + sync_status |

연결 대상 프로젝트는 `references/setup_connection.md`의 config.json으로 결정된다 (하드코딩 아님).

## 8. 산출물 (Outputs)

- **기본**: `.md` 파일 1개 (output_template.md 기준)
- **선택**: `.xlsx` 평가시트 (메타·직무역량·BARS·MVC·종합 의사결정)
- **저장 위치**: 사용자 지정 또는 `<workspace>/면접/{position_name}_{candidate}_{date}/`

## 9. 변경 이력

| Version | 날짜 | 변경 |
|---|---|---|
| 0.1 | 2026-04-26 | 초기 패키징 (8 직무 + MVC 데이터 INSERT 완료) |
| 0.1.6 | 2026-06-02 | jd_fetch.md §1 Supabase URL·publishable key 정합화 |
| 0.2.0 | 2026-06-03 | 하드닝(output_template 중복 §1 제거·질문 수 하드코딩 동적화·MCP 끊김 복구 §7.1) + L2 레벨별 BARS(§7) + 질문 기법 카탈로그(question_generation §6) |
| 0.3.0 | 2026-06-03 | Supabase 연결 하드코딩 제거 → 최초 setup 워크플로우·config.json(`setup_connection.md`, Step 0) + MVC를 Supabase `mvc_*`로 이전해 편집 가능화(`mvc_schema.md`, 파일은 시드·폴백). 조직/타인 배포 시 각자 프로젝트 연결. |

## 10. 미해결 항목 (Open Items)

- **레벨별 BARS DB 등록**: 작성 규칙·스키마·폴백은 v0.2.0 완료(`bars_anchoring.md §7`). `bars_anchors_by_level` 실제 등록 + 시니어 패널 검토는 별도 운영 작업 (`roadmap_v0.2_progress.md`)
- xlsx 자동 생성 스크립트 (로드맵 L1): 본 스킬 범위 외
- T2 자동 처리 후 월 1회 사후 검토 루틴 (운영 정책)
- (해소됨) research_* JD 미등록 — 2026-04 기준 positions 17직무 등록 완료, 본 항목 폐기

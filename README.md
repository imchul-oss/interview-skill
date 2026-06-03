# structured-interview

구조화 면접(Structured Interview) 자동화 플러그인 — **직무 중립 템플릿**. JD(직무기술서)에서 표준 면접 질문 세트를 생성·보존하고, 후보자 CV에 맞춘 면접 가이드를 만들어 줍니다. 면접관 간 평가 일관성과 비교 가능성을 학술 근거 위에서 확보하는 것을 목표로 합니다.

> 이 저장소는 특정 조직 색채를 제거한 **바닐라 버전**입니다. 핵심가치·Supabase 프로젝트·직무 컨텍스트를 여러분 환경에 맞게 채워 사용하세요.

---

## 1. 두 개의 스킬 (writer / reader)

| 스킬 | 역할 | 한 줄 |
|---|---|---|
| `structured-interview` | **Writer** | 직무 JD → 표준 질문 세트(BEI·SIT × 역량 N개 + 핵심가치 모듈)를 생성해 Supabase에 **immutable** 저장 |
| `structured-interview-q` | **Reader** | 표준 질문 세트 + 후보 CV 매칭 → 강점·약점·갭 분석 + 우선순위 + **후보자별 맞춤 가이드(.md)** 생성 |

writer는 "같은 직무 = 같은 질문"을 보장(구조화 일관성), reader는 표준 질문을 **수정하지 않고** CV 인용·관찰 포인트만 덧붙입니다.

학술 근거: Schmidt & Hunter (1998, r=.51 / 워크샘플 r=.54), Levashina et al. (2014), McClelland (1973, BEI), Latham et al. (1980, SIT), Smith & Kendall (1963, BARS), Kristof-Brown et al. (2005·2023, P-O Fit), Dreyfus & Dreyfus (1980, 레벨 차등).

---

## 2. 설치

1. 이 저장소를 `.plugin`(zip)으로 패키징하거나, Cowork/Claude Code 플러그인으로 로드합니다.
2. `.claude-plugin/plugin.json` + `skills/` 구조를 그대로 사용합니다.

```
.claude-plugin/plugin.json
skills/
  structured-interview/        # writer
    SKILL.md  workflow.md  mvc/_core_values.md  references/*.md
  structured-interview-q/      # reader
    SKILL.md  workflow.md  references/*.md
```

---

## 3. 최초 1회 — Supabase 연결 설정 (하드코딩 없음)

연결 정보는 코드에 박혀 있지 않습니다. 첫 사용 시 setup 워크플로우가 발동되어 입력받습니다.

1. Supabase 프로젝트의 **Project URL** (`https://<ref>.supabase.co`)
2. **Publishable key** (`sb_publishable_...`, Project Settings → API)
   - ⚠️ `sb_secret_...`(service_role)은 입력 금지 — 공개 가능 키만 사용(RLS 보호).
3. 입력값은 `<workspace>/.structured-interview/config.json`에 저장되어 재사용됩니다(이 파일은 `.gitignore` 처리, 커밋 금지).

상세: `skills/structured-interview/references/setup_connection.md`.

---

## 4. 데이터 모델 (Supabase)

| 테이블/뷰 | 용도 |
|---|---|
| `positions` | 직무 마스터 (name, job_title, status …) |
| `position_content` | JD 본문 (responsibilities / qualifications / preferred) |
| `interview_questions` | 표준 면접 질문 마스터 — **immutable** (버전·archive) |
| `jd_change_log` | JD 변경 자동 추적(T1/T2/T3) |
| `v_active_interview_questions` | 활성 질문 + sync_status 뷰 |
| `mvc_values` / `mvc_questions` / `mvc_bars` / `mvc_flags` / `mvc_position_context` | **핵심가치 모듈(편집 가능)** — `mvc_schema.md` |
| `mvc_change_log` | 핵심가치 편집 감사 로그 |

스키마 DDL·마이그레이션은 `references/mvc_schema.md`(핵심가치)와 각 reference에 포함되어 있습니다. **DB 스키마/INSERT는 사용자 승인 후 1회 수행**하며, 미실행 시 파일 시드로 폴백합니다.

---

## 5. 사용 흐름

### 표준 질문 생성 (직무당 1회)

```
"<직무명> 면접 준비"
→ structured-interview 발동 → 질문 INSERT(역량 N개×2 + 핵심가치 모듈) → in_sync 검증
```

### 후보자 맞춤 가이드 (매번)

```
"/structured-interview-q"
→ 직무·CV·단계(1차/2차)·시간(분)[·트랙] 입력
→ 매칭·우선순위·맞춤 .md 산출
```

---

## 6. 핵심 기능

- **역량 추출 동적화**: JD에서 역량 4~7개를 MECE로 추출, 가중치 자동 산정(고정 비율 없음).
- **질문 기법 다양화**: BEI/SIT 외 **관찰(observation)·워크샘플·케이스·대조·reasoning-aloud**. 질문마다 관찰 포인트 1줄, 갭 역량엔 실기 제안(옵션). (`question_generation.md §6`)
- **필수 2 / 선택 2~3 이원화**: 모든 후보 공통 anchor 2개(필수) + 면접관 재량 2~3개(선택). 시간 부족 시 선택부터 제거, 필수 보존. (`priority_algorithm.md §0.6`)
- **트랙별 레벨 BARS**: junior/mid/senior 5점 앵커 차등(Dreyfus). DB에 레벨 앵커 없으면 단일 기준 폴백 + 주석(임의 생성 금지). (`bars_anchoring.md §7`)
- **2차 면접 강화**: 2차 선택 시 "추가 검증 영역 + 레퍼런스 체크 항목"을 elicit → 가이드 전용 섹션.
- **슬림 출력**: 사실확인 단락 제거 → 매칭 1표 + 질문 탭 "이유" 1줄, 시간표 간략, 시인성·A4 친화.
- **JD 변경 추적**: T1(표현)/T2(도구 동등)/T3(역량 변경) 자동 분류, T3는 승인 게이트.

---

## 7. 커스터마이즈 (바닐라 → 우리 조직)

1. **핵심가치**: `skills/structured-interview/mvc/_core_values.md`의 VALUE_1~5와 정의·질문·BARS·직무 컨텍스트를 조직 값으로 교체(또는 `mvc_*` 테이블에서 편집).
2. **직무·JD**: Supabase `positions`/`position_content`에 등록(생성은 스킬이 거부 — 사용자 등록 후 호출, 환각 방지).
3. **Supabase**: §3 setup으로 자신의 프로젝트 연결.
4. **파일명/네임스페이스**: 스킬명 `structured-interview(-q)`를 원하는 이름으로 바꿀 수 있습니다(plugin.json + 폴더 + 상호참조 동기화).

---

## 8. 가드레일 (설계 원칙)

- **환각 금지**: JD/질문/CV에 없는 사실을 생성·추측·일반화하지 않음. 데이터 없으면 정직 보고·중단.
- **immutability**: `interview_questions`는 SELECT/INSERT/archive만, UPDATE 금지(구조화 일관성). 핵심가치(`mvc_*`)는 편집 가능하되 감사 로그·확인 게이트.
- **결정성**: 핵심가치 전부 평가(선별 금지), 자유 영역 신설 금지, 검증 우선순위 라벨 금지(매칭 강도로 일원화), 시간 배분은 결정적 공식.
- **개인정보**: CV는 세션 내에서만, 출력엔 발췌만, 영속 저장 금지.
- **MCP 끊김 복구**: 연결 끊김 시 재로드→재시도→실패 시 중단(추측 금지). (`jd_fetch.md §7.1`)

---

## 9. References 맵

| 파일 | 역할 |
|---|---|
| `structured-interview/references/setup_connection.md` | Supabase 최초 연결 설정 + config.json |
| `structured-interview/references/mvc_schema.md` | 핵심가치 Supabase 스키마 + 편집·마이그레이션 |
| `structured-interview/references/jd_fetch.md` | Supabase 쿼리·hash·오류/MCP 복구 |
| `structured-interview/references/competency_extraction.md` | JD→역량 추출·가중치·레벨 범위 |
| `structured-interview/references/question_generation.md` | BEI/SIT + 기법 카탈로그·관찰·워크샘플 |
| `structured-interview/references/bars_anchoring.md` | BARS 작성 + 레벨별 차등(§7) |
| `structured-interview/references/probing_patterns.md` | STAR 꼬리질문 카탈로그 |
| `structured-interview/references/jd_diff_analysis.md` | T1/T2/T3 분류 |
| `structured-interview/references/output_template.md` | 표준 가이드 출력 템플릿 |
| `structured-interview/references/new_position_workflow.md` | 신규 직무 등록 흐름 |
| `structured-interview-q/references/competency_matching.md` | CV↔역량 매칭(강도) |
| `structured-interview-q/references/cv_parsing.md` | CV 파싱 + 개인정보 가드 |
| `structured-interview-q/references/question_personalization.md` | 표준질문에 CV 인용 추가 |
| `structured-interview-q/references/priority_algorithm.md` | 시간 배분·필수/선택·anchor |
| `structured-interview-q/references/output_personalized.md` | 맞춤 가이드 템플릿·시인성 |

---

## 10. 버전

| Version | 변경 |
|---|---|
| 0.3.0 | Supabase 연결 config화(setup 워크플로우) + 핵심가치 DB 편집 가능화 + 직무 중립 바닐라 템플릿화 |
| 0.2.0 | 하드닝 + 레벨별 BARS + 출력 슬림 재설계 + 질문 기법 다양화 + 필수/선택 + 2차 강화 |
| 0.1.x | 초기 패키징 + 결정성·시인성 강화 |

---

## 11. 라이선스

`<원하는 라이선스를 지정하세요 (예: MIT)>`

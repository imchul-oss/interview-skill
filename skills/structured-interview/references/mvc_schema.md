# mvc_schema.md — MVC를 Supabase로 이전 + 편집 (v0.3.0 신규)

**목표**: MVC(핵심가치 모듈)를 파일(`mvc/_core_values.md`) 고정에서 **Supabase 편집 가능** 구조로 이전한다. 가치 정의·질문·BARS·Go/No-Go·직무 컨텍스트를 DB에서 수정하면 이후 모든 면접에 자동 반영된다.

**파일과의 관계**: `mvc/_core_values.md`는 **시드(seed) + 오프라인 폴백**으로 보존한다. DB에 MVC가 없거나 MCP가 끊기면 파일로 폴백하고 그 사실을 출력에 명시한다 (환각·임의생성 금지).

---

## 1. 스키마 (DDL)

```sql
-- 1) 가치 마스터
CREATE TABLE IF NOT EXISTS public.mvc_values (
  value_key        TEXT PRIMARY KEY,            -- VALUE_1 ~ VALUE_5
  name             TEXT NOT NULL,               -- 예) "VALUE_1 (<가치명>)"
  definition       TEXT NOT NULL,
  go_nogo_criterion TEXT NOT NULL,
  sort_order       INT  NOT NULL,
  is_active        BOOLEAN NOT NULL DEFAULT TRUE,
  updated_at       TIMESTAMPTZ DEFAULT NOW(),
  updated_by       TEXT
);

-- 2) 가치별 질문 (BEI 2 + SIT 1 = 가치당 3)
CREATE TABLE IF NOT EXISTS public.mvc_questions (
  id          BIGSERIAL PRIMARY KEY,
  value_key   TEXT NOT NULL REFERENCES public.mvc_values(value_key),
  qtype       TEXT NOT NULL CHECK (qtype IN ('BEI','SIT')),
  stem_text   TEXT NOT NULL,
  sort_order  INT  NOT NULL,
  is_active   BOOLEAN NOT NULL DEFAULT TRUE,
  updated_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_by  TEXT
);

-- 3) 가치별 BARS 5점 (단일) — 레벨별은 v0.2.0 by_level 패턴 준용(옵션)
CREATE TABLE IF NOT EXISTS public.mvc_bars (
  value_key  TEXT NOT NULL REFERENCES public.mvc_values(value_key),
  score      INT  NOT NULL CHECK (score BETWEEN 1 AND 5),
  anchor     TEXT NOT NULL,
  PRIMARY KEY (value_key, score)
);

-- 4) 가치별 Go/No-Go 신호 (flag)
CREATE TABLE IF NOT EXISTS public.mvc_flags (
  id         BIGSERIAL PRIMARY KEY,
  value_key  TEXT NOT NULL REFERENCES public.mvc_values(value_key),
  kind       TEXT NOT NULL CHECK (kind IN ('go','nogo')),
  signal     TEXT NOT NULL
);

-- 5) 직무 친화 컨텍스트 (가치 × 직무)
CREATE TABLE IF NOT EXISTS public.mvc_position_context (
  value_key      TEXT NOT NULL REFERENCES public.mvc_values(value_key),
  position_name  TEXT NOT NULL REFERENCES public.positions(name),
  context_text   TEXT NOT NULL,
  updated_at     TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (value_key, position_name)
);

-- 6) 편집 감사 로그 (MVC는 immutable 아님 — 편집 허용·추적)
CREATE TABLE IF NOT EXISTS public.mvc_change_log (
  id          BIGSERIAL PRIMARY KEY,
  table_name  TEXT, row_key TEXT,
  field       TEXT, old_value TEXT, new_value TEXT,
  changed_by  TEXT, changed_at TIMESTAMPTZ DEFAULT NOW()
);
```

**immutability 차이**: `interview_questions`(직무 질문)는 immutable이지만, **MVC는 편집 가능**하다. 단 모든 변경은 `updated_at/updated_by` + `mvc_change_log`로 추적한다. 과거 생성된 .md 가이드는 스냅샷이므로 영향받지 않는다.

---

## 2. 마이그레이션 (seed = _core_values.md → DB)

`mvc/_core_values.md`의 현재 내용을 1회 INSERT한다. 값 매핑:

| 출처 (_core_values.md) | 대상 테이블 |
|---|---|
| §3.x 가치 정의·Go/No-Go | mvc_values |
| §3.x 질문 풀 (Q-x1/x2 BEI, x3 SIT) | mvc_questions |
| §3.x BARS 5점 표 | mvc_bars |
| §3.x Red/Green Flag | mvc_flags |
| §4 직무 컨텍스트 매트릭스 (N직무×5가치) | mvc_position_context |

```sql
-- 예시 (VALUE_1)
INSERT INTO public.mvc_values(value_key,name,definition,go_nogo_criterion,sort_order)
VALUES ('VALUE_1','VALUE_1 (<가치명>)','<가치 정의 placeholder>',
        '''우리''보다 ''나''를 일관되게 우선시하는 태도가 답변 2개 이상에서 관찰될 때',1);
INSERT INTO public.mvc_questions(value_key,qtype,stem_text,sort_order) VALUES
 ('VALUE_1','BEI','팀 또는 부서를 넘어서 협력하여…',1),
 ('VALUE_1','BEI','의견이 크게 달랐던 동료 또는 팀과…',2),
 ('VALUE_1','SIT','[상황] 긴급한 고객 이슈로 다른 팀의 도움이…',3);
-- mvc_bars / mvc_flags / mvc_position_context 동일 패턴으로 5가치 전부
```

> 전체 INSERT는 `mvc/_core_values.md` 내용을 그대로 옮기며, **새 문구를 창작하지 않는다** (시드 충실 복제). 5가치 × (정의1·질문3·BARS5·flag다수) + 등록 직무 N개×5컨텍스트.

**실행 정책 (Critical)**: 실제 CREATE/INSERT는 **Supabase MCP 연결 + 사용자 명시 승인** 후 1회 수행한다. 스킬은 임의로 DB 스키마를 만들지 않는다. 미실행 상태에서는 파일 폴백으로 정상 동작한다.

---

## 3. Fetch (ASSEMBLE 시)

```sql
SELECT v.value_key, v.name, v.definition, v.go_nogo_criterion,
       q.qtype, q.stem_text, q.sort_order
FROM public.mvc_values v
JOIN public.mvc_questions q ON q.value_key=v.value_key AND q.is_active
WHERE v.is_active
ORDER BY v.sort_order, q.sort_order;
-- + mvc_bars, mvc_flags, mvc_position_context(WHERE position_name=$1) 조인/별도 조회
```

폴백: 위 쿼리가 0행이거나 MCP 끊김 → `mvc/_core_values.md` 사용 + 출력에 "ℹ️ MVC 파일 시드 사용(DB 미이전)" 주석.

가드: MVC 5가치 전부 fetch 확인(5개 미만이면 폴백). 선별 금지(결정성).

---

## 4. MVC 편집 워크플로우 ("수정 가능")

사용자가 "MVC 수정/편집"(예: "VALUE_1 정의 바꿔줘", "<직무> VALUE_3 컨텍스트 수정") 요청 시:

| 단계 | 동작 |
|---|---|
| 1 | 대상 식별 (value_key + 필드/질문/컨텍스트) |
| 2 | 변경 전후 값 사용자에게 확인 (diff 제시) |
| 3 | 승인 시 UPDATE + `updated_by`=config.configured_by, `mvc_change_log` INSERT |
| 4 | 변경 영향 안내: "이후 생성 면접부터 반영, 과거 .md는 불변" |

가드:
- MVC 편집은 허용하되 **항상 사용자 확인 게이트**를 거친다 (직무 질문 immutable과 구분).
- 5가치 자체를 삭제/4가치로 축소 금지 (결정성 — 평가 일관성). 신규 가치 추가는 평가 체계 변경이므로 별도 승인.
- 편집 후 `_core_values.md`(시드)와의 차이는 `mvc_change_log`로만 추적 (파일은 시드 기준선 유지).

---

## 5. 학술·정합성

- MVC 편집 가능화는 운영 유연성을 위함이나, *Levashina et al. (2014) 동일 질문 원칙*상 **빈번한 변경은 비교 가능성을 해친다**. 변경은 신중히, 분기·반기 단위 권장.
- 5가치 강제 평가·선별 금지(결정성 가드)는 편집 후에도 유지.

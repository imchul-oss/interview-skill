# jd_fetch.md — Supabase JD 조회 + Hash 계산

**원칙**: JD는 단일 진실 원천(Supabase). 외부에서 별도 캐싱·복제 금지.

---

## 1. Supabase 프로젝트 (config 기반, v0.3.0)

연결 정보는 **하드코딩하지 않는다.** 첫 사용 시 `references/setup_connection.md`의 SETUP으로 입력받아 `<워크스페이스>/.structured-interview/config.json`에 저장하고, 이후 그 값을 읽어 사용한다.

```
project_ref / url / publishable_key  ← config.json (setup_connection.md)
```

- config 없음 → `setup_connection.md §2 SETUP` 발동 (진행 중단, 임의 기본 프로젝트로 진행 금지).
- publishable key(`sb_publishable_*`)만 사용 (RLS 보호, 클라이언트 노출 안전). service_role(`sb_secret_*`)은 config·plugin·git 어디에도 저장 금지.
- 본 스킬은 일차적으로 Supabase **MCP** 연결로 인증한다. config의 url·publishable_key는 외부 클라이언트(자체 도구·xlsx 스크립트 등) 직접 접근 시 참조.

**기본 제안값 (조직 내부, 강제 아님)**: project_ref `<YOUR_PROJECT_REF>` · `https://<YOUR_PROJECT_REF>.supabase.co` (region ap-northeast-1, "<YOUR_PROJECT_NAME>"). 내부 사용자는 SETUP에서 이 값을 그대로 채택하면 된다.

## 2. 핵심 쿼리

### 2.1 직무 존재 검증

```sql
SELECT name, job_title, lang, status
FROM public.positions
WHERE (name = $1 OR job_title ILIKE $1)
  AND status = 'active';
```

- 0행 → `references/new_position_workflow.md` Step A
- 1+ 행 → 정확한 `name` 추출 후 Step 2.2

### 2.2 JD 조회 (3 섹션 모두)

```sql
SELECT
  section_key,
  content,
  MD5(content) AS hash
FROM public.position_content
WHERE position_name = $1
  AND lang = $2  -- 'ko' (기본) 또는 'en'
ORDER BY
  CASE section_key
    WHEN 'responsibilities' THEN 1
    WHEN 'qualifications' THEN 2
    WHEN 'preferred' THEN 3
  END;
```

**검증**:
- 행 수 = 3 (responsibilities + qualifications + preferred 모두 존재)
- 각 content 길이 > 50자 (너무 짧으면 의심)
- 0~2행 → `new_position_workflow.md` Step B

### 2.3 활성 면접 질문 조회

```sql
SELECT *
FROM public.v_active_interview_questions
WHERE position_name = $1
ORDER BY competency_key, question_order;
```

`sync_status` 분기:
- `in_sync`: hash 일치, 그대로 사용
- `jd_changed_review_needed`: diff 분석 필요
- (no rows): 첫 호출

### 2.4 JD hash 객체 생성

```python
# 의사 코드
jd_hash = {
    "responsibilities": MD5(jd_resp_content),
    "qualifications":   MD5(jd_qual_content),
    "preferred":        MD5(jd_pref_content)
}
# 이 객체가 derived_from_jd_hash JSONB로 INSERT됨
```

---

## 3. 환각 방지 검증

INSERT 직전:

```sql
-- 검증 쿼리: 우리가 사용한 hash가 현재 JD와 일치하는가
SELECT
  $position_name AS pos,
  $section_key AS section,
  $used_hash AS used,
  MD5(content) AS current,
  MD5(content) = $used_hash AS match
FROM public.position_content
WHERE position_name = $position_name AND lang = 'ko' AND section_key = $section_key;
```

- `match = true` 모든 섹션 → INSERT 진행
- `match = false` 어느 한 섹션이라도 → **INSERT 중단** + 사용자 보고

---

## 4. 다국어 정책

- **기본 lang**: `ko`
- positions의 lang과 position_content의 lang이 다를 수 있음 (예: marketing_lead = positions.lang='en' but position_content.lang='ko')
- 면접 질문은 사용자 요청 시 한글이 기본. 영문 요청 시 lang='en' fetch

---

## 5. JD 변경 감지 (자동)

`position_content` UPDATE 시 트리거 `tg_track_jd_changes`가 자동으로 `jd_change_log`에 INSERT. 본 스킬은 별도 조작 불필요.

```sql
-- 변경 이력 조회 (필요 시)
SELECT *
FROM public.jd_change_log
WHERE position_name = $1
  AND diff_tier = 'pending'
ORDER BY detected_at DESC;
```

---

## 6. 권한·RLS

본 스킬은 `service_role` 또는 `authenticated` 권한으로 호출 가정. RLS 정책상:
- `positions`, `position_content`: SELECT 모두 허용
- `interview_questions`, `jd_change_log`: SELECT/INSERT/UPDATE 허용

---

## 7. 오류 처리

| 오류 | 처리 |
|---|---|
| Supabase MCP 도구 미연결/끊김 | §7.1 복구 절차 (ToolSearch 재로드 → 재시도 1회 → 실패 시 중단) |
| Supabase 연결 실패 (네트워크) | 사용자에게 보고, 재시도 1회, 그래도 실패 시 중단 |
| 0행 응답 (예상 외) | "JD를 찾을 수 없습니다" 정직 보고, **만들어내지 않음** |
| hash 불일치 (검증 실패) | INSERT 중단, 사용자에게 데이터 정합성 알림 |

### 7.1 Supabase MCP 끊김 복구 절차 (v0.2.0)

운영 중 Supabase MCP 도구(`mcp__<supabase>__execute_sql`·`list_tables` 등)가 연결 끊김(deferred/disconnected) 상태일 수 있다. 다음 순서로 복구한다.

1. ToolSearch로 `select:mcp__<supabase>__execute_sql` (필요 시 `list_tables`도) 스키마 재로드 시도.
2. 재로드 성공 시 직전 쿼리 1회 재시도.
3. 그래도 실패 시 **중단** + 정직 보고: "Supabase 연결이 끊겨 JD/질문을 조회할 수 없습니다. 재연결 후 재시도해 주세요."
4. **연결 실패를 메모리·추측으로 우회하지 않는다** — JD·질문을 만들어내거나 캐시 추정으로 진행 금지 (환각 방지 §3).

> 실제 운영 사례(2026-04 backend_engineer): MCP 끊김 → 사용자 "수파베이스 연결됨 재시도" → ToolSearch 재로드 후 정상 fetch. 본 절차는 그 복구 흐름을 표준화한 것.

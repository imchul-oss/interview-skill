# setup_connection.md — Supabase 연결 최초 세팅 (v0.3.0 신규)

**원칙**: 플러그인은 특정 Supabase 프로젝트에 **하드코딩되지 않는다**. 첫 사용 시 1회 연결 정보를 입력받아 워크스페이스 config에 저장하고, 이후에는 config를 읽어 재사용한다. 조직·타인 배포 시 각자 자신의 프로젝트를 연결한다.

---

## 1. Config 위치·스키마

플러그인 설치 디렉터리는 읽기 전용이므로 config는 **사용자 워크스페이스**에 저장한다.

```
경로: <워크스페이스>/.structured-interview/config.json
예:   <workspace>/.structured-interview/config.json
```

```json
{
  "supabase": {
    "project_ref": "<YOUR_PROJECT_REF>",
    "url": "https://<YOUR_PROJECT_REF>.supabase.co",
    "publishable_key": "sb_publishable_xxx",
    "configured_at": "2026-06-03",
    "configured_by": "<your-email>"
  },
  "schema_version": "0.3.0"
}
```

- `publishable_key`만 저장한다 (`sb_publishable_*`, RLS 보호, 클라이언트 노출 안전). **`sb_secret_*` service_role은 절대 config·plugin·git에 저장 금지.**
- config 파일은 git에 커밋하지 않는다 (`.gitignore`에 `.structured-interview/` 추가). 키 노출 방지.

---

## 2. 최초 세팅 워크플로우 (First-run Setup)

스킬(structured-interview / structured-interview-q) 발동 시 **가장 먼저** config 존재를 확인한다.

```
config.json 있음 → 연결 정보 로드 → 정상 진행
config.json 없음 → [SETUP] 발동
```

### 2.1 SETUP 단계

사용자에게 다음을 입력받는다 (시각화 질문지 또는 텍스트).

```
**조직 면접 플러그인 — Supabase 최초 연결 설정**

이 플러그인은 직무 JD·면접 질문·MVC를 Supabase에 저장합니다.
처음 한 번만 연결 정보를 입력해 주세요 (이후 자동 사용).

1. **Project URL** (필수): 예) https://<project_ref>.supabase.co
2. **Publishable key** (필수): `sb_publishable_...` (Supabase → Project Settings → API)
   ⚠️ service_role(`sb_secret_...`)은 입력하지 마세요. 공개 가능 키만 사용합니다.

(조직 내부 사용자는 기존 '<YOUR_PROJECT_NAME>' 프로젝트 값을 그대로 쓰시면 됩니다.)
```

### 2.2 검증 (저장 전)

1. URL 형식 검증: `https://<ref>.supabase.co`. `project_ref` 추출.
2. key 형식 검증: `sb_publishable_`로 시작. `sb_secret_`이면 **거부** + 재입력 요청.
3. 연결 테스트: Supabase MCP로 가벼운 SELECT 1회 (예: `SELECT count(*) FROM public.positions`). 실패 시 사용자에게 보고하고 저장하지 않는다.

### 2.3 저장

검증 통과 시 `<워크스페이스>/.structured-interview/config.json`에 기록. 완료 메시지:

```
✅ Supabase 연결 저장 완료 (project_ref=<ref>). 이제 직무 면접 질문을 생성할 수 있습니다.
재설정하려면 "Supabase 연결 재설정" 또는 config.json 삭제 후 재발동하세요.
```

---

## 3. 재설정 / 변경

- 사용자가 "Supabase 연결 재설정" 요청 또는 config.json 삭제 → §2 SETUP 재실행.
- key 회전(rotation) 시에도 동일.

---

## 4. 로드 규칙 (모든 워크플로우 공통)

`jd_fetch.md` 등에서 Supabase 접근 전:

```
1. config.json 로드 → supabase.url / publishable_key / project_ref
2. config 없음 → setup_connection.md §2 SETUP 발동 (진행 중단, 추측 금지)
3. Supabase MCP는 위 project_ref 프로젝트에 연결되어 있어야 함 (MCP 끊김 시 jd_fetch §7.1)
```

---

## 5. 가드 (Critical)

- 연결 정보 미설정 상태에서 **임의의 기본 프로젝트로 진행하지 않는다** — 반드시 SETUP을 거친다 (잘못된 프로젝트 쓰기 방지).
- service_role/secret 키 저장·요청 금지.
- config는 워크스페이스에만, git·plugin 배포물에는 포함하지 않는다.
- 하드코딩된 예시 값(조직 `<YOUR_PROJECT_REF>`)은 **편의를 위한 기본 제안**일 뿐, 강제 연결 대상이 아니다.

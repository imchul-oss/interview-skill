# competency_extraction.md — JD → 역량 추출 + 가중치 산정 알고리즘

**원칙**: 역량 개수와 가중치는 **JD 내용에 의해 동적으로 결정**됩니다. 어떤 직무에도 고정된 N개·고정 비율이 없습니다.

---

## 1. 입력

```
{
  "position_name": "...",
  "responsibilities": "...",
  "qualifications": "...",
  "preferred": "..."
}
```

## 2. 알고리즘 (5 Step)

### Step 1: 역량 후보 추출

각 섹션을 별도로 처리:

**responsibilities** → 동작·책임 동사구 추출
- 예: "구축·운영", "설계·개발", "트러블슈팅" 등

**qualifications.하드스킬** → 명사구 추출
- 예: "Kubernetes 클러스터 운영", "TLS 복호화 구현"

**qualifications.소프트스킬** → 행동 패턴 추출
- 예: "장애 침착 분석·재발 방지", "자기주도성"

**preferred** → 가산점 항목 추출 (가중치 낮음)

### Step 2: MECE 그룹핑 (중복 병합)

비슷한 역량 후보를 1개로 묶음. *Mutually Exclusive, Collectively Exhaustive* 원칙:

- 중복 → 병합 (예: "K8s 운영" + "Kubernetes 클러스터" → 1개)
- 누락 검토 → 핵심 책임이 어느 역량에도 매핑되지 않으면 새 역량 추가

**최종 역량 수**: 일반적으로 **4~7개**. JD가 풍부하면 6~7, 단순하면 4~5. **6개 고정 아님**.

### Step 3: 가중치 산정

다음 신호를 종합:

| 신호 | 점수 |
|---|---|
| responsibilities 명시 | +3 (직무의 핵심) |
| qualifications "필수" 또는 "X년 이상" 명시 | +3 |
| qualifications에서 가장 처음 등장 (앞쪽) | +2 |
| preferred 명시 | +1 (가산점) |
| 소프트스킬 단독 | +1 (행동·태도 평가) |

**정규화**: 각 역량 점수를 합 100%로 정규화. 단 다음 제약:

- 단일 역량 가중치 **최대 30%** (한 역량 독점 방지)
- 단일 역량 가중치 **최소 5%** (의미 없는 역량은 통합·삭제)
- 합 = 100% (소수점 둘째 자리)

### Step 4: 역량 키 명명

각 역량에 다음 형식의 `competency_key` 부여:

```
C{N}_{snake_case_label}

예시:
- C1_kubernetes_operations
- C1_react_typescript
- C1_brand_positioning
- C2_api_design
```

`competency_name`은 한글 표시명 (예: "C1. Kubernetes 클러스터 운영").

### Step 5: 역량별 분류 태그

각 역량에 다음 태그 부여:

| 태그 | 설명 | 평가 비중 |
|---|---|---|
| `기술/필수` | 하드스킬 + 필수 | 직무역량 점수 핵심 |
| `기술/우대` | 하드스킬 + preferred | 가산점 |
| `행동/필수` | 소프트스킬 + 필수 | 행동 평가 |
| `행동/우대` | 소프트스킬 + preferred | 가산점 |

---

## 3. 출력 (JSON 예시 — 직무 무관)

```json
{
  "position_name": "{{any_position}}",
  "competency_count": 5,
  "competencies": [
    {
      "key": "C1_<extracted_label>",
      "name": "C1. <Korean Display>",
      "weight": 0.28,
      "type": "기술/필수",
      "source_jd_lines": [
        "responsibilities-1",
        "qualifications.하드스킬-2"
      ]
    },
    ...
  ],
  "weight_sum": 1.00
}
```

**검증**: `weight_sum = 1.00 ± 0.01` 아니면 재산정.

---

## 3.5 레벨 범위 추정 (v0.2.0, L2 연계)

JD의 연차·시니어리티 신호로 이 **직무가 포괄하는 레벨 범위**를 추정한다. 이는 레벨별 BARS 앵커 작성 대상을 정하기 위한 것이며, **후보자 개인의 레벨을 추정하는 것이 아니다** (후보 트랙은 사용자가 명시 입력 — structured-interview-q §5.2).

| JD 신호 | 추정 레벨 범위 |
|---|---|
| "0~2년", "신입/주니어" | junior (+ mid) |
| "3~6년", 연차 무명시 | mid (기본값) |
| "7년+", "시니어/리드/아키텍트" | senior (+ mid) |
| 범위 명시 (예: "3~8년") | 걸치는 레벨 모두 |

출력 JSON에 `level_scope: ["junior"|"mid"|"senior", ...]`를 포함한다. 레벨별 BARS 앵커는 이 범위에 한해 `bars_anchoring.md §7`로 작성하며, 범위 밖 레벨은 작성하지 않는다. 신호가 모호하면 `mid` 단일로 처리하고 단일 `bars_anchors` 폴백을 쓴다 (환각 방지).

---

## 4. 비-금지 사항 (Anti-patterns)

다음은 **하지 않는다**:

- ❌ 모든 직무에 6개 역량을 강요 (역량 수는 JD에 따라 동적)
- ❌ 25/22/18/15/10/10 같은 고정 비율 사용 (가중치는 점수 기반 동적)
- ❌ 다른 직무의 가중치 패턴 복사
- ❌ JD에 없는 역량을 "일반적으로 필요하다"는 이유로 추가 (환각)
- ❌ JD에 명시된 핵심 책임을 누락

---

## 5. 검증 체크리스트

INSERT 전 확인:

- [ ] 역량 합 가중치 = 1.00
- [ ] 모든 `source_jd_lines`가 실제 JD 라인 가리킴
- [ ] 단일 역량 ≤ 30%, ≥ 5%
- [ ] MECE: 중복 없음, 핵심 책임 누락 없음
- [ ] competency_key는 직무 내 unique
- [ ] 한글 representation은 명확함

---

## 6. 학술 근거

- *Levashina et al. (2014)* — 직무분석 기반 질문 (구조화 면접 6 핵심 요소 중 1번)
- *Campion et al. (1997)* — JD 기반 평가 차원 정의
- *Flanagan (1954)* — Critical Incident Technique (직무 핵심 사건 추출)

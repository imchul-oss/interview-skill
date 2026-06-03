---
module: core_values
title: 조직 핵심가치 면접 모듈 (Core Values Module) — 템플릿
version: 0.3.0
purpose: 모든 직무 면접에 자동 부착되는 조직문화·핵심가치 평가 모듈
question_count: 15  # 5 values × (BEI 2 + SIT 1) — 가치 수는 조직에 맞게 조정 가능
applies_to: all_positions
auto_attach: true
role: seed_and_fallback   # 편집 가능 마스터는 Supabase mvc_* (mvc_schema.md). 이 파일은 시드·폴백.
academic_basis:
  - Kristof-Brown et al. (2005) — P-O Fit 메타분석, 면접관 직접 평가 r=.61
  - Kristof-Brown et al. (2023) — Culture Fit vs Culture Add, 동질성 편향 경고
  - McClelland (1973) — Behavioral Event Interview
  - Latham et al. (1980) — Situational Interview
  - Smith & Kendall (1963) — BARS
last_updated: 2026-06-03
---

# 조직 핵심가치 면접 모듈 (Core Values) — 커스터마이즈 템플릿

> ⚠️ **이 파일은 제네릭 템플릿입니다.** 아래 5개 가치(VALUE_1~5)와 정의·질문·BARS·직무 컨텍스트를 **여러분 조직의 실제 핵심가치로 교체**하세요. 운영 편집은 Supabase `mvc_*` 테이블에서 합니다(`references/mvc_schema.md`). 본 파일은 시드·오프라인 폴백입니다.

## 1. 개요

본 모듈은 **모든 직무 면접에 자동 부착**되는 조직문화 평가 모듈입니다. 직무역량 모듈과 페어로 운용되어 면접관 간 일관성을 확보합니다.

### 1.1 핵심가치 정의 (예시 — 조직 값으로 교체)

| 구분 | 내용 (교체 대상) |
|---|---|
| Mission | `<조직의 미션 한 줄>` |
| Vision | `<조직의 비전 한 줄>` |
| Core Values | VALUE_1 · VALUE_2 · VALUE_3 · VALUE_4 · VALUE_5 |

### 1.2 의사결정 원칙

> **핵심가치 모두 충족 → 진행 / 하나라도 No-Go → 재검토.** 채용 의사결정에도 동일 적용: 가치 1개라도 No-Go면 직무역량과 무관하게 No Hire.

## 2. 평가 원칙

- **Go/No-Go**: 가치별 비타협 기준선(이진 판단).
- **BARS 5점**: Go 통과자 정량 평가(Smith & Kendall 1963).
- **질문 구조 — Universal Stem + Position Context**: 본 질문(stem)은 모든 후보 95% 동일하게 유지하고, 직무별 한 줄 컨텍스트만 덧붙인다(동일 질문 원칙 + 직무 친화).
- **5가치 전부 평가**(선별 금지). 시간 제약 시 가치당 시간만 압축.

## 3. 가치별 질문 풀 (예시 구조 — 내용 교체)

각 가치는 **BEI 2 + SIT 1 = 3문항**. 1차는 가치당 1문항, 2차는 2~3문항 선택.

### 3.N VALUE_N (가치명) — `<한 줄 정의>`

#### 정의
`<이 가치가 무엇이며 왜 중요한지 2~3문장>`

#### 행동 지표
- (+) `<긍정 행동 신호 2~3개>`
- (−) `<부정 행동 신호 2~3개>`

#### 질문 풀
**Q-N1 [BEI · Universal]** > `<과거 행동을 묻는 질문>`

**Q-N2 [BEI · Universal]** > `<다른 각도의 과거 행동 질문>`

**Q-N3 [SIT · Universal]** > `[상황] <가상 시나리오>. 어떻게 하시겠습니까?`

#### 꼬리질문 (STAR 4단계)
- S: `<상황 확인>` / T: `<역할·과제>` / A: `<구체 행동>` / R: `<정량 결과>`

#### Red/Green Flag
- 🚩 No-Go 신호: `<...>`  /  ✅ Go 신호: `<...>`

#### BARS (5점)

| 점수 | 행동 앵커 |
|---:|---|
| 5 | `<산업·조직 표준 수립 수준>` |
| 4 | `<독립 수행 + 팀 표준 기여>` |
| 3 | `<기본 기대 충족>` |
| 2 | `<일부 미흡, 지원 필요>` |
| 1 | `<기대 크게 미달>` |

#### Go/No-Go 기준
> **No-Go**: `<이 가치의 비타협 기준선이 답변 2개 이상에서 위반될 때>`

---

> 위 3.N 블록을 **VALUE_1~VALUE_5 각각에 대해 1개씩** 작성하세요(총 5블록). 가치 수를 늘리거나 줄이려면 `question_count`와 평가 로직을 함께 조정합니다.

## 4. 직무 친화 컨텍스트 매트릭스 (가치 × 직무)

각 Universal Stem 뒤에 직무별 **한 줄 컨텍스트**를 덧붙여 직무 특화 답변을 유도합니다. 본 질문은 동일하게 유지합니다.

| 가치 \ 직무 | `<role_a>` | `<role_b>` | … |
|---|---|---|---|
| VALUE_1 | "...특히 `<role_a>` 맥락의 사례 중심으로…" | "...`<role_b>` 맥락…" | |
| VALUE_2 | … | … | |

> 직무가 추가되면 컬럼을 추가합니다. 운영 시에는 `mvc_position_context` 테이블에서 관리합니다(`mvc_schema.md`).

## 5. 면접관 운용 가이드

- 1차: 가치당 1문항 ≈ 15~20분 / 2차: 가치당 2~3문항 ≈ 40~50분.
- 편향 방지 체크: 유사성·확인·후광·앵커링 + **Culture Add도 평가**(Kristof-Brown 2023, 동질성 편향 경계).
- STAR Result 단계 정량 지표 1회 강제, "우리/나" 분리 질문.

## 6. 종합 평가

- 가치별: Go/No-Go + BARS 점수 + 관찰 인용.
- 종합: `[Go × 전 가치] AND [BARS 평균 ≥ 3.0]` → 통과 / `No-Go 1개+` → 미통과(직무역량 무관 No Hire).

## 7. 학술 근거

- Kristof-Brown et al. (2005), (2023); McClelland (1973); Latham et al. (1980); Smith & Kendall (1963). (상세 인용은 SKILL.md)

---
*본 모듈은 조직 핵심가치를 모든 직무 면접에 일관되게 적용하기 위한 템플릿입니다. 가치 정의 변경 시 Supabase `mvc_*`(또는 본 파일)만 수정하면 모든 직무에 자동 반영됩니다.*

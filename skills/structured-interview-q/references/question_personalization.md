# question_personalization.md — BEI 질문에 CV 인용 추가 패턴

**원칙**: 표준 질문 텍스트는 **절대 변경하지 않음**. 별도 `cv_specific_addendum` 필드에 CV 인용을 추가하여 출력 .md에서 자연스럽게 합쳐짐.

---

## 1. 왜 표준 질문 텍스트를 변경하지 않는가

*Levashina et al. (2014)*: "All applicants asked the same questions" — 구조화 면접의 핵심.
- 표준 질문이 후보자별로 달라지면 비교 불가 → 비구조화로 회귀 → 예측타당도 추락
- structured-interview-q는 표준 질문에 **컨텍스트만 추가**하고, 본 질문은 **모든 후보자 동일** 보존

---

## 2. cv_specific_addendum 패턴

### 2.1 기본 구조

```
[표준 질문 — 모든 후보자 동일]
"{competency} 영역에서 가장 까다로웠던 문제와 해결 과정을 설명해 주세요."

[CV 기반 추가 — 후보자별 가변]
"{cv_specific_addendum}"
   = "이력서에 명시된 ''{프로젝트명}''에서의 경험을 중심으로 답변해 주시면 좋습니다."
```

면접관이 자연스럽게 합쳐 발화:
> "{competency} 영역에서 가장 까다로웠던 문제와 해결 과정을 설명해 주세요. 이력서에 명시된 ''{프로젝트명}''에서의 경험을 중심으로 답변해 주시면 좋습니다."

### 2.2 addendum 시드 카드 (직무 무관)

| 시드 | 용도 |
|---|---|
| **A. 강점 검증** | "이력서에 ''{프로젝트명}''이 명시되어 있는데, 그 사례를 중심으로 답변해 주시면 좋습니다." |
| **B. 정량 성과 검증** | "이력서의 ''{정량 지표}'' 결과가 어떻게 측정되었는지 함께 알려주시면 좋습니다." |
| **C. 기간·역할 검증** | "{기간} 동안 ''{역할}''로 일하셨는데, 그 시기의 경험을 중심으로 답변해 주시면 좋습니다." |
| **D. 갭 탐색 (약점)** | "이력서에는 {약점 영역} 경험이 명시되지 않았는데, 학습·도전 경험이 있다면 함께 알려주시면 좋습니다." |
| **E. 모순·확인** | "이력서에 ''{모호 표현}''으로 적혀 있는데, 구체적인 한 사례를 들어주시면 좋습니다." |
| **F. 트레이드오프** | "{프로젝트}에서 {기술 A}와 {기술 B} 사이의 결정이 있었다면 그 근거를 함께 알려주시면 좋습니다." |
| **G. 후속 영향** | "{프로젝트}가 끝난 이후의 후속 영향(인용·확장 등)을 함께 알려주시면 좋습니다." |

### 2.3 시드 → 직무·CV별 변환

각 BEI 질문에 대해 매칭 강도에 따라 시드 1개 선택:

| 매칭 강도 | 권장 시드 |
|---|---|
| 🟢 strong | A (강점 검증) + B (정량) |
| 🟡 moderate | A 또는 C (기간) + F (트레이드오프) |
| 🟠 weak | D (갭 탐색) + 학습 사례 요청 |
| 🔴 none | (BEI 대신 SIT 권장 — priority_algorithm.md) |

---

## 3. 변환 알고리즘

### Step 1: 매칭 결과에서 인용 발췌

```python
# 의사 코드
for question in standard_questions:
    competency = question.competency_key
    match = competency_matches[competency]
    
    if match.strength == "none":
        continue  # SIT으로 대체 (priority_algorithm.md)
    
    evidence = match.evidence[0]  # 가장 강한 증거
    
    addendum = generate_addendum(
        seed=select_seed(match.strength, question.type),
        cv_excerpt=evidence.cv_excerpt,
        project_name=evidence.name
    )
```

### Step 2: addendum 검증

생성된 addendum에 다음 검증:

- [ ] CV에 인용된 모든 표현이 실제 CV에 존재 (cv_lines 추적 가능)
- [ ] 추측·일반화 표현 없음 ("보통", "일반적으로" 등)
- [ ] 표준 질문의 의미를 좁히지 않음 (확장만)
- [ ] 한국어 자연스러움 (어색한 번역체 X)

검증 실패 시 → 그 질문은 addendum 없이 표준 그대로 사용.

---

## 4. SIT 질문 처리

SIT 질문은 가상 시나리오라 CV 인용이 자연스럽지 않음. 대신:

- 매칭 강도 🔴 none / 🟠 weak 일 때 SIT 우선 사용 (학습·잠재력 평가)
- SIT는 표준 그대로 사용 (addendum 없음)
- 단 후보자 배경에 따라 SIT 선택 — 예: 시니어 후보면 더 복잡한 SIT, 주니어면 기본 SIT

---

## 5. MVC 질문은 표준 그대로

MVC 모듈 (`_core_values.md`)의 질문은:
- Universal stem + 직무 친화 컨텍스트가 이미 부착됨
- CV 기반 addendum **추가하지 않음** (MVC는 모든 후보자에 동일 적용 = 가치 평가 일관성)

---

## 6. 환각 가드 (Critical)

### 6.1 CV에 없는 인용 절대 금지

```
❌ Bad:
addendum = "이력서에 ''대용량 트래픽 처리''가 명시되어 있는데..."
(CV 검색 결과: 명시되지 않음)

✅ Good:
match.strength == "none" → addendum 생략
```

### 6.2 추측·확장 금지

```
❌ Bad:
"이력서에 React가 있으니 ''컴포넌트 라이브러리''에 대한 질문도..."

✅ Good:
CV에 컴포넌트 라이브러리 명시 없으면 → 표준 질문 그대로
```

### 6.3 회사·학교 prestige 인용 금지

```
❌ Bad:
"{유명 회사}에서의 경험을 중심으로..."

✅ Good:
"이력서에 명시된 ''{프로젝트}''에서의 경험을 중심으로..."
(회사 이름 자체를 강조 안 함)
```

---

## 7. 출력 .md 표시 형식

출력 .md에서는 다음과 같이 표시:

```markdown
### {competency_name} - BEI 질문

> {standard_question_text}

**CV 기반 안내**: {cv_specific_addendum}

- 매칭 강도: 🟢 strong (가중치 25%)
- CV 인용: "{cv_excerpt}" (CV L23-L27)
- 검증 의도: 강점 진위·정량 검증
- 꼬리질문: ...
```

면접관은 표준 질문 + addendum을 자연스럽게 발화. 후보자에게는 **하나의 질문**으로 들리되, 본인 이력에 맞게 답변하기 쉬워짐.

---

## 8. 학술 근거

- *Levashina et al. (2014)* — 동일 질문 원칙
- *Janz (1982)* — Patterned BEI: 표준 질문 + 후속 질문 표준화
- *Campion et al. (1997) §15요소* — Question Standardization 보존

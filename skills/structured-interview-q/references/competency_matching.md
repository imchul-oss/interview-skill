# competency_matching.md — CV 역량 ↔ 직무 역량 매칭 알고리즘 (v0.1.4)

## 원칙

CV의 명시된 사실만으로 직무 역량과 매칭한다. 추측·일반화·합성을 금지한다.

매칭 결과는 **강도(strong/moderate/weak/none)** 로만 표현한다. **검증 우선순위 라벨(High/Medium/Low/Critical)을 부여하지 않는다.** 운영 데이터 분석 결과, 같은 갭 영역에 후보자마다 다른 라벨(예: 한 후보자는 "Critical", 다른 후보자는 "Medium")이 부여되어 면접관 혼란을 유발했다. v0.1.4부터 우선순위는 매칭 강도와 가중치만으로 결정한다.

다음 자유 라벨은 출력에 절대 포함하지 않는다.

- "검증 우선순위: High/Medium/Low/Critical"
- "★/★★" 강조 기호 (anchor 표시는 🔗로 한정)
- "검증 우선순위 ★ / ★★ / 보통" 등 모든 자유 라벨

매칭 강도 4단계만 출력하며, 면접관은 강도(🟢🟡🟠🔴) + 가중치로 직접 우선순위를 판단한다.

---

## 1. 입력

- **CV 구조화 데이터**: cv_parsing.md 출력
- **직무 역량 N개**: interview_questions에서 SELECT
  - competency_key, competency_name, weight, source_jd_lines

## 2. 알고리즘 (4 Step)

### Step 1: 직무 역량 키워드 확장

각 역량마다 매칭 키워드 풀 생성:
- `competency_name`의 명사구
- `source_jd_lines`가 가리키는 JD 부분의 명사구
- 동의어·하위 개념·도구명까지 확장

```
예시 (직무 무관, 알고리즘 패턴):
역량: "Container/Orchestration"
키워드: ["Docker", "Kubernetes", "K8s", "containerd", "Helm", "ArgoCD",
        "컨테이너", "오케스트레이션", "Pod", "Service Mesh", ...]
```

### Step 2: CV 매칭 검색

CV 구조에서 다음 위치를 검색:
- `skills` (모든 카테고리)
- `projects[].technologies`
- `projects[].description`
- `projects[].outcome`
- `current_role` / `previous_roles[]`
- `achievements[]`
- `publications`
- `certifications`

매칭마다 **반드시** `cv_lines` 또는 `cv_excerpt` 메타 함께 저장.

### Step 3: 강도 평가 (4단계)

| 강도 | 정의 | 기준 |
|---|---|---|
| 🟢 **strong** | 명확한 경험 + 정량 성과 + 충분 기간 | 3+ 매칭 + 정량 지표 + 1년+ 경력 |
| 🟡 **moderate** | 일부 경험, 정량 성과 일부 | 2~3 매칭 + 정성 설명 |
| 🟠 **weak** | 단편적 언급만, 짧은 노출 | 1 매칭 + 짧은 사용 |
| 🔴 **none** | CV에 전혀 없음 | 0 매칭 |

**중요**: 4단계는 **CV 명시 사실 기반**. CV에 없으면 **none**, 추측 금지.

### Step 4: 증거 인용 부착

각 강도 평가에 다음 메타 부착:

```json
{
  "competency_key": "C1_react_typescript",
  "match_strength": "strong",
  "evidence": [
    {
      "type": "project",
      "name": "{프로젝트명}",
      "summary": "...",
      "cv_lines": ["L23-L27"],
      "cv_excerpt": "..."
    },
    {
      "type": "skill_listing",
      "items": ["React 18", "TypeScript strict"],
      "cv_lines": ["L10"]
    }
  ],
  "verification_question": "면접에서 추가 검증할 측면",
  "potential_red_flags": "정량 성과 없음·기간 모호 등"
}
```

---

## 3. 출력 — 매칭 매트릭스

```json
{
  "candidate_name": "...",
  "position_name": "...",
  "competency_matches": [
    {"competency_key": "C1_...", "weight": 0.25, "match_strength": "strong", "evidence": [...]},
    {"competency_key": "C2_...", "weight": 0.22, "match_strength": "moderate", "evidence": [...]},
    {"competency_key": "C3_...", "weight": 0.18, "match_strength": "weak", "evidence": [...]},
    {"competency_key": "C4_...", "weight": 0.15, "match_strength": "none", "evidence": []},
    ...
  ],
  "weighted_strength_score": 3.4,   // 1~5 스케일 (strong=5, moderate=4, weak=3, none=1)
  "summary": {
    "top_strengths": ["..."],
    "identified_gaps": ["..."],
    "verification_priorities": ["..."]
  }
}
```

### 가중 강도 점수 산정

```
score = sum(weight × strength_value)
where strength_value: strong=5, moderate=4, weak=3, none=1
```

이 점수는 **참고 지표**. 채용 의사결정의 근거 아님 (실제 평가는 면접 수행 후).

---

## 4. Red Flag 후보 식별 (옵션)

CV에서 다음 패턴 발견 시 "Red Flag 후보"로 마킹 (확정 아님, 면접 검증 필요):

| 패턴 | 검증 방향 |
|---|---|
| 경력 공백 ≥ 6개월 | "그 기간에 무엇을 하셨나요?" |
| 잦은 이직 (1년 이내 이직 ≥ 3회) | "이직 사유 패턴" |
| 모호 표현 ("다양한", "여러" 등 정량 없음) | "구체 사례 1개" 강제 |
| 직무·기술 비일관 (CS 학위 없는데 ML 연구원 등) | "그 분야로 어떻게 진입하셨나요?" |
| 팀이 한 일을 본인이 한 것처럼 표현 | "본인이 직접 한 일은?" |
| 정량 지표 부재 ("성공적으로 마무리" 등) | "어떻게 측정하셨나요?" |

**중요**: 이는 **면접에서 확인할 가설**일 뿐. CV의 모순을 사실로 단정하지 않음.

---

## 5. 매칭 결과 → 면접 전략 매핑

| 강도 | 면접 전략 | 권장 질문 유형 |
|---|---|---|
| 🟢 strong | **검증** — 진위·깊이 확인 | BEI 우선, 정량·구체 인용 강제 |
| 🟡 moderate | **확장 검증** — 더 깊은 사례 | BEI + 보조 SIT |
| 🟠 weak | **갭 탐색** — 학습 의지·잠재력 | SIT 우선, BEI는 학습 사례 |
| 🔴 none | **잠재력 평가** — 적응력 | SIT, 학습 능력 BEI |

---

## 6. 환각 가드 (Critical)

### 6.1 CV 인용 없는 매칭은 거부

각 매칭에 `cv_lines` 또는 `cv_excerpt` 누락 → **즉시 strength="none" 처리**.

### 6.2 일반화 금지

- ❌ "삼성 출신 = K8s 익숙"
- ❌ "ML 박사 = Python 능숙"
- ✅ CV에 명시되지 않은 한 **none**

### 6.3 회사·학교 편향 차단

평가 시 다음 사용 금지:
- 회사 이름의 prestige
- 학교 ranking
- 출신 지역
- 추천인 (CV에 명시되지 않은 한)

---

## 7. 학술 근거

- *Highhouse (2008) Stubborn Reliance on Intuition* — 정량 매칭이 직관 평가보다 우수
- *Sackett et al. (2022)* — JD 직접 매칭이 일반 적성 검사보다 직무성과 예측력 우수
- *Kahneman (2011) Thinking Fast and Slow* — 일반화·앵커링 편향 차단의 가치

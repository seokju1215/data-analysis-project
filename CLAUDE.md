# 데이터마이닝 프로젝트 — 도서 SNS 앱 사용자 활동성 분석

## 행동 지침 (Claude 작업 방식)

각 분석 단계는 다음 순서로 진행한다. 코드 작성에서 멈추지 말 것.

1. **코드 작성 & 실행** — 분석 코드를 작성하고 `python_repl` 또는 Bash로 직접 실행
2. **결과 확인** — 실제 숫자, 분포, 통계량을 읽어낸다
3. **해석 제시** — "이 숫자가 이 앱의 맥락에서 무슨 의미인가"를 설명한다
   - 단순 수치 나열 금지. 반드시 "따라서 ~한 액션이 필요하다" 수준까지
4. **가설 연결** — 해당 결과가 H1~H5 중 어느 가설을 지지/반박하는지 명시
5. **발표 인사이트 도출** — 결과를 12~13분 발표의 어느 슬라이드에서 어떻게 쓸지 제안
6. **다음 단계 제안** — 이 결과를 보고 추가로 파봐야 할 것이 있으면 제안

> 한 단계가 끝날 때마다 사용자와 결과를 검토하고 다음 단계로 넘어간다.
> "코드 완성 = 완료"가 아니라 "인사이트 도출 = 완료"다.

### 노트북 작성 규칙

- **실행한 코드와 결과는 절대 지우지 않는다.** 누적 보존이 원칙.
- 결과를 확인한 뒤 바로 아래에 Markdown 셀로 **인사이트 요약**을 남긴다.
  ```
  ## 인사이트
  - 수치: ...
  - 해석: ...
  - 가설 연결: H? 지지/반박
  - 발표 활용: 슬라이드 X에서 ~로 활용
  ```
- 그 아래에 **다음 분석 코드**를 이어서 작성한다.
- 중간에 방향이 바뀌어도 이전 코드는 남기고, 새 셀에서 시작한다.
- 새로운 모델/기법을 적용하기 전에 반드시 **선택 근거 셀**을 먼저 작성한다.
  ```
  ## 모델 선택 근거
  - 선택: XXX
  - 대안: AAA (왜 안 쓰나), BBB (왜 안 쓰나)
  - 이 데이터/문제에 XXX가 적합한 이유: ...
  ```

---

## 프로젝트 개요

실제 운영 중인 도서 기반 소셜 네트워크 앱(누적 가입자 ~1,251명)의 Supabase PostgreSQL
데이터를 분석하여, 사용자 휴면 결정 요인을 발굴하고 앱 개선 전략을 도출한다.

**발표**: 12~13분 / **제출**: 발표 슬라이드 사전 제출

---

## 분석 스코프 (확정)

| 단계 | 분석 | 주요 기법 |
|------|------|-----------|
| 1 | 탐색적 분석 + 코호트 리텐션 | 기술통계, 코호트 분석 |
| 2 | 사용자 행동 군집화 (페르소나) | K-means, 계층 군집 |
| 3 | 휴면 예측 분류 | Logistic, RandomForest, XGBoost + SHAP |
| 4 | 소셜 네트워크 분석 | 중심성, Louvain, Link Prediction |
| 5 | 텍스트 마이닝 | LDA + BERTopic, 이탈 사유 분석 |
| 보조 | 생존 분석 | Kaplan-Meier, Cox |

> **단계 6 (랜딩페이지 동등성 검정) 제외** — 마케팅 데이터 미수집.
> 계획서에는 존재하나 발표에서는 한계점으로만 언급.

---

## 데이터 파일 (`data/` 디렉토리)

| 파일 | 내용 | 행 수 |
|------|------|--------|
| `01_users_cohort_20260524.csv` | 코호트·활성 상태 (user_id, signup_at, cohort_month, last_seen_at, activity_status, ad_cohort, job, followers, visits_received_total, total_books, total_reviews, archived_books, following_count) | 1,077 |
| `02_user_features_20260524.csv` | 전체 피처 20개 — 행동강도, 사회적 변수, 알림, 타이밍 | 1,077 |
| `03_dormancy_train_20260524.csv` | 가입 후 첫 7일 행동 + `is_dormant` 레이블 | 1,060 |
| `04a_follow_edges_20260524.csv` | 팔로우 그래프 엣지 (source, target, created_at, is_reciprocal) | 206 |
| `04b_visit_edges_20260524.csv` | 프로필 방문 그래프 엣지 (source, target, weight, first/last_visit_at) | 5,042 |
| `04c_nodes_20260524.csv` | 그래프 노드 메타 (user_id, job, followers, following, visitors, signup_at, last_seen_at, activity_status) | 1,077 |
| `05a_reviews_20260524.csv` | 리뷰 텍스트 (user_id, book_id, title, author, review_content, review_length, is_archived, activity_status) | 1,666 |
| `05b_delete_feedback_20260524.csv` | 이탈 피드백 (reason_index, reason_text, created_at) | 207 |

**reason_index 매핑 (앱 실제 정의)**
| index | 이탈 이유 | 응답 수 |
|-------|----------|---------|
| 0 | 원하는 책이 서비스에 없어서 | 22 |
| 1 | 인생 책을 9권밖에 설정할 수 없어서 | 9 |
| 2 | 앱이 느리거나 오류가 많아서 | 9 |
| 3 | 팔로우/댓글/좋아요 등 소통이 없어서 | 2 |
| 4 | 독서 기록용으로 쓰기 부적합해서 | 39 |
| 5 | 사용빈도가 낮아서 | 83 (1위, 40%) |
| 6 | 재밌는 콘텐츠가 없어서 | 21 |
| 7 | 기타 | 23 |
| `06_survival_20260524.csv` | 생존 분석용 (duration_days, event_dormant + 파생 피처) | 1,077 |

### 주요 관찰
- `activity_status`: `active` / `dormant` / `long_dormant` 3단계
- `ad_cohort`: `pre_ad` / `ad` 구분 (인스타그램 광고 시기)
- 방문 그래프(5,042)가 팔로우 그래프(206)보다 **25배 밀도** → 네트워크 분석의 핵심

---

## Python 환경

- **인터프리터**: `/Users/hongseogju/anaconda3/bin/python` (3.11.4)
- **Jupyter**: `/Users/hongseogju/anaconda3/bin/jupyter`
- 노트북 실행: `cd "Data Analysis Project" && /Users/hongseogju/anaconda3/bin/jupyter notebook`

### 설치된 주요 패키지 (확정 버전)
numpy 1.26.4 (**numpy<2 로 핀 — 올리면 sklearn/pandas 충돌**), pandas 2.3.3,
scikit-learn 1.3.0, xgboost 3.2.0, lifelines 0.30.3, shap 0.51.0,
networkx 3.1, gensim 4.3.0, scipy 1.10.1, matplotlib 3.7.1, seaborn 0.12.2,
plotly 5.9.0

### 한국어 NLP (단계 5, 필요 시)
```bash
/Users/hongseogju/anaconda3/bin/pip install bertopic
# konlpy + Mecab 은 별도 시스템 설치 필요 (https://konlpy.org/ko/latest/install/)
# 설치 안 되면 BERTopic multilingual 모델로 대체
```

---

## 노트북 구조

```
notebooks/
  01_eda_cohort.ipynb          # 단계 1: EDA + 코호트 리텐션
  02_clustering.ipynb          # 단계 2: K-means 페르소나
  03_dormancy_prediction.ipynb # 단계 3: 분류 + SHAP
  04_network.ipynb             # 단계 4: 네트워크 분석
  05_text_mining.ipynb         # 단계 5: 텍스트 마이닝
  06_survival.ipynb            # 보조: 생존 분석
```

---

## 핵심 결정 사항

- **활성 정의**: `days_since_last_seen < 30` → active, 30~90 → dormant, 90+ → long_dormant
- **훈련 피처**: 가입 후 첫 7일 행동(`03_dormancy_train`)으로 제한 → 조기 개입 시나리오
- **클래스 불균형 대응**: SMOTE 또는 class_weight='balanced' 비교
- **모델 평가**: AUC-ROC + F1 + Precision-Recall (목표: AUC 0.75↑)
- **군집 수 결정**: 실루엣 계수 + 엘보 방법 → 4~6개 예상
- **텍스트**: 리뷰는 한국어, Mecab 사용 불가 시 BERTopic(multilingual) 대체

## 검증할 핵심 가설

| 가설 | 검증 방법 |
|------|-----------|
| H1: 받은 방문 수 多 → 활성률 高 | 로지스틱 회귀 계수, Log-rank |
| H2: 가입 후 N일 내 첫 책 추가 → 휴면률 低 | 생존 분석, 변수 중요도 |
| H3: 팔로잉 0명 → 휴면률 압도적으로 高 | 카이제곱 검정 |
| H4: 아카이브 행동 → 평균 활성 수명 長 | Kaplan-Meier, Cox |
| H5: 광고 코호트 리텐션 < 자연 유입 | 코호트 비교 |

# 서울 지하철 혼잡도 Cold-start 예측

> 운행 데이터(승하차·열차) 없이 **토지이용 구조 + 시간대 + 방향**만으로 지하철 혼잡도를
> 예측하고, 그 신호가 **다른 노선으로 일반화되는지** 6가지 방법으로 검증한 cold-start 프로젝트.

<br>

## TL;DR

1. 정적 행정동 토지신호는 시간 효과 위에 혼잡도 정보를 사실상 더하지 못한다 (증분 R² ≈ +0.01).
2. cold-start 혼잡도 예측에는 시간가변 데이터가 필수적이다.

분류·회귀 두 태스크, 5가지 타깃 정의, 8개 노선에서 결론이 동일 →
모델·타깃 선택의 우연이 아닌 **데이터 구조에서 비롯된 천장**임을 입증.

<br>

## 결과 요약

### 1) 분류 (4등급 Pooled qcut · nested CV)

| 모델 | Macro F1 | Accuracy | MAE | 비고 |
|---|---|---|---|---|
| Logistic Regression | 0.271 | 0.296 | 1.13 | 선형 baseline |
| **Random Forest** | **0.457** | **0.460** | **0.80** | **메인** (치명오차 0↔3 = 6.3%) |
| LightGBM | 0.446 | 0.449 | 0.82 | RF와 동급 → 데이터 천장의 신호 |

### 2) 회귀 (혼잡도 실제값 · nested CV)

| 모델 | R² | MAE | RMSE |
|---|---|---|---|
| Ridge | 0.020 | 16.7 | 22.2 |
| **Random Forest** | **0.299** | **13.9** | **18.7** |
| LightGBM | 0.230 | 14.6 | 19.6 |

R² 0.30은 토지의 설명력이 아닌, 대부분 **시간대가 만든 설명력** (아래 ablation 참고).

### 3) 토지 설명력 검증 (ablation)

| 검증 방법 | 토지 기여 | 결론 |
|---|---|---|
| 분류 ablation (시간 vs +토지) | +0.022 F1 | 미미 |
| 회귀 ablation (R²) | 0.289 → 0.299 = **+0.010** | ≈0 |
| Permutation importance | 토지 3개 합 ≈ +0.02 (관광 −0.007) | ≈0 |
| Leave-One-Line-Out (8노선) | 평균 **−0.016** (5/8 음수) | transfer 안 됨 |
| Detrend 타깃 (시간효과 제거 잔차) | 토지 only ≈ 0.25 (랜덤) | 신호 없음 |
| 역 집계(고정) 타깃 | 토지 only ≈ 0.25~0.29 | 신호 없음 |

**결론: 토지는 시간 위에 정보를 더하지 못한다.**

### 4) Feature Importance

```
impurity     : 시간대 0.41 | 직장 0.22 | 주거 0.19 | 관광 0.12 | 상하구분 0.06
permutation  : 시간대 0.22 | 상하구분 0.03 | 직장 0.02 | 주거 0.01 | 관광 -0.01
```

![importance](artifacts/feature_importance.png)
![confusion](artifacts/confusion_matrix.png)

<br>

## 방법론

**데이터**
- 토지이용: 서울 상권분석서비스 — 행정동별 주거(상주)·직장·관광(길단위) 인구
- 타깃: 서울교통공사 분기별 역사 혼잡도 (2025 Q1–Q4, 평일), 8개 노선
- 공간: 지하철역 좌표 + 행정동 경계 (2017)

**피처 — catchment 토지 ratio (250m)**
- 역 반경 250m 내 행정동 인구를 **면적가중 합산** → 단일 동 할당의 경계 역 오배정 제거
- 예: 강남역 = 역삼1동 48% + 서초4동 26% + 서초2동 26%
- 전역 min-max 후 합=1 정규화 (노선 간 비교 가능)
- 최종 피처: `상하구분`, `시간대`, `주거_ratio`, `직장_ratio`, `관광_ratio`

**Train/Test Split**
- `StratifiedGroupKFold(group = 노선_역명)` — 같은 역이 train/test 양쪽에 들어가지 않도록 누수 차단
- 식별자(역번호·행정동) 피처 제외, 그룹 키로만 사용
- 호선 인코딩 통일 (`2` ↔ `2호선` 분리 누수 방지)

**튜닝 — Nested Cross-Validation**
- 바깥(성능 추정) / 안쪽(하이퍼파라미터 선택)으로 선택 편향 제거, 안쪽도 그룹-aware
- `RandomizedSearch`로 LR·RF·LightGBM(분류) / Ridge·RF·LightGBM(회귀) 공정 튜닝

<br>

## 레포 구조

```
.
├── README.md
├── requirements.txt
├── pipeline.ipynb                      # 분류 + ablation + 일반화 검증
├── make_catchment_weights.py           # GIS: 좌표 + 경계 → catchment 가중치
└── data/
    ├── station_catchment_weights.csv   # GIS 산출물 (제공)
    └── (원본 데이터 — 아래 참고)
```

<br>

## 실행

```bash
pip install -r requirements.txt
```

이후 `pipeline.ipynb`를 Jupyter 또는 VS Code에서 열고 셀 순서대로 실행.

**Colab의 경우** Drive 마운트 후 첫 번째 셀에서 경로 지정:

```python
from google.colab import drive
drive.mount('/content/drive')

import os
os.environ['DATA_DIR']     = '/content/drive/MyDrive/YOUR_FOLDER/data'
os.environ['WEIGHTS_PATH'] = '/content/drive/MyDrive/YOUR_FOLDER/data/station_catchment_weights.csv'
```

<br>

## 데이터

원본 파일은 용량·라이선스 문제로 repo에 포함하지 않습니다.
아래 경로에서 직접 다운로드 후 `data/` 폴더에 위치시켜 주세요.

| 파일 | 출처 |
|---|---|
| 서울시 상권분석서비스(상주인구-행정동_주거).csv | [서울 열린데이터광장](https://data.seoul.go.kr) |
| 서울시 상권분석서비스(직장인구-행정동_직장).csv | [서울 열린데이터광장](https://data.seoul.go.kr) |
| 서울시 상권분석서비스(길단위인구-행정동_관광).csv | [서울 열린데이터광장](https://data.seoul.go.kr) |
| 서울교통공사_지하철혼잡도정보_분기별.csv (4개) | [서울교통공사 정보공개](https://www.seoulmetro.co.kr) |
| 서울교통공사_1_8호선_역사_좌표.csv | [서울교통공사 정보공개](https://www.seoulmetro.co.kr) |
| 서울_행정동_경계_2017.geojson | [datainworld/administrative_district](https://github.com/datainworld/administrative_district) (원본: 통계지리정보서비스 SGIS / 행정안전부) |

`station_catchment_weights.csv`는 위 좌표·경계 파일로부터 `make_catchment_weights.py`가 생성하며, 이미 `data/`에 포함되어 있습니다.

<br>

## 한계와 향후 방향

정적 행정동 인구는 시간축이 없어 혼잡도의 시간 변동을 설명하지 못함 — 이것이 cold-start의 본질.
천장 돌파에 필요한 데이터 (영향 순):

1. **시간대별 생활인구** (행정동×시간대 추정 체류인구) — cold-start 정신 유지하며 정적성을 깰 수 있는 방향
2. **OD 통근 flow** (시간×방향 수요) — 혼잡도 정의에 가장 근접
3. **역 구조 피처** (환승·출구 수) — 정적이나 역간 변별력은 토지보다 강함

토지 파생 피처(연령·가구유형 등)는 같은 정적 성격의 재표현이라 효과 없음을 ablation으로 확인.

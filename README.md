# 역세권 토지이용 비율 기반 서울 지하철 혼잡도 예측

> 운행 데이터 없이 토지비율(관광, 직장, 거주)만으로 지하철 역사 내 혼잡도를 예측할 수 있을까?

<br>

## TL;DR

*1. **정적인 토지정보는 혼잡도 예측에 기여를 하지 못하며, 시간대가 대부분의 설명력을 차지한다.** <br> 
2. 핵심 방법론 - 토지비율 피처엔지니어링: 서울시 모든 행정동에 대해 3가지 토지비율 계산 -> 역사 250m 반경 내 행정동 면적가중 합산* 


<br>

## 결과 요약

### 1) 분류 모델 (4등급 Pooled qcut)

| 모델 | Macro F1 | Accuracy | MAE | 비고 |
|---|---|---|---|---|
| Logistic Regression | 0.271 | 0.296 | 1.13 | 선형 baseline |
| **Random Forest** | **0.457** | **0.460** | **0.80** | **메인** (치명오차 0↔3 = 6.3%) |
| LightGBM | 0.446 | 0.449 | 0.82 | Boosting 비교군 |
<br>

### 2) 회귀 모델 (혼잡도 실제값)

| 모델 | R² | MAE | RMSE |
|---|---|---|---|
| Ridge | 0.020 | 16.7 | 22.2 |
| **Random Forest** | **0.299** | **13.9** | **18.7** |
| LightGBM | 0.230 | 14.6 | 19.6 |

<br>


### 3) 토지 설명력 검증 (ablation)

| 검증 방법 | 토지 기여 | 결론 |
|---|---|---|
| 분류 ablation (시간 vs +토지) | +0.022 F1 | 미미 |
| 회귀 ablation (R²) | 0.289 → 0.299 = **+0.010** | ≈0 |
| Permutation importance | 토지 3개 합 ≈ +0.02 (관광 −0.007) | ≈0 |
| Leave-One-Line-Out (8노선) | 평균 **−0.016** (5/8 음수) | transfer 안 됨 |
| Detrend 타깃 (시간효과 제거 잔차) | 토지 only ≈ 0.25 (랜덤) | 신호 없음 |
| 역 집계(고정) 타깃 | 토지 only ≈ 0.25~0.29 | 신호 없음 |

**결론: 토지정보는 혼잡도 예측에 기여를 하지 못한다.**

<br>

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


**전처리** 
- 데이터 클리닝: 역명 표기 통일, 결측치 처리 
- 데이터 축소: 역번호·행정동 코드 등 식별자 컬럼 제거 (leakage 방지)
- 데이터 통합: 행정동명 기준으로 인구 데이터와 혼잡도 데이터 병합
- 변환: 시간별 혼잡도 컬럼 → time_slot + congestion_level 두 변수로 재구조화


**피처 엔지니어링**
- 각 토지비율 전역 min-max 정규화 -> 전체 토지비율 합=1 정규화 (역별)
- 역 반경 250m 내 행정동 토지비율 **면적가중 합산**
    - 방법: 역 좌표 기준 반경 250m와 행정동 GeoJSON을 공간 조인 후 겹치는 면적 비율을 가중치로 산출 
- 최종 피처: `상하구분`, `시간대`, `주거_ratio`, `직장_ratio`, `관광_ratio`
    -  `상하구분`, `시간대`는 예측 대상을 특정하기 위해 포함. 토지의 기여분은 두 변수를 통제한 뒤 ablation으로 별도 측정. 

**모델링**
- `StratifiedGroupKFold(group = 노선_역명)` — 같은 역이 train/test 양쪽에 들어가지 않도록 데이터 누수 차단
- 식별자(역번호·행정동) 피처 제외, 그룹 키로만 사용
- 호선 인코딩 통일 (`2` ↔ `2호선` 분리 누수 방지)
- 튜닝: Nested Cross-Validation, RandomizedSearch
  

<br>

## 레포 구조

```
.
├── README.md
├── requirements.txt
├── pipeline.ipynb                      # 전처리 포함 전체 코드 
├── make_catchment_weights.py           # 역사 별 행정동 면적 가중치 산출 코드 
└── data/
    ├── station_catchment_weights.csv   # GIS 산출물 (제공)
    └── (원본 데이터 — 아래 참고)
```

<br>

## 실행
Drive 마운트 후 첫 번째 셀에서 경로 지정:

```python
from google.colab import drive
drive.mount('/content/drive')

import os
os.environ['DATA_DIR']     = '/content/drive/MyDrive/YOUR_FOLDER/data'
os.environ['WEIGHTS_PATH'] = '/content/drive/MyDrive/YOUR_FOLDER/data/station_catchment_weights.csv'
```

<br>

## 데이터
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

## 느낀 점
운영 데이터 없이도 혼잡도를 예측할 수 있냐는 가능성 검증 관점에서 시작된 주제였는데, 역시나 결과가 안 좋았고, 마찬가지로 운영 데이터를 추가하면 성능이 더 오르지 않았겠냐는 교수님의 피드백을 받았다. 
또한 문제정의 단계에서는 구체성에 꽂힌 나머지 물리적 범위를 좁히는 식(전호선->2호선 대상)으로 진행했는데, 물리적인 범위를 좁히는 것보다는 EDA를 더 철저하게 해서 타겟과 피처 간의 인과관계를 더 명확하게 정의하는 것이 구체적인 문제정의의 올바른 방향이었을 것 같다. 

# KAMP Injection Molding Quality Prediction & Inspection Prioritization

> 제6회 K-인공지능 제조데이터 분석 경진대회  
> 사출성형 공정 데이터 기반 **품질불량 예측·식별한계 진단·검사 우선순위 최적화** 프로젝트

이 저장소는 KAMP 사출성형 데이터에 대해 수행한 전체 분석 과정을 정리한 연구/경진대회용 코드와 결과물을 포함합니다.

단순히 “불량/양품을 분류하는 모델”을 만드는 데서 끝내지 않고, 다음 질문을 순차적으로 다룹니다.

1. 현재 제공된 공정변수만으로 불량을 **식별할 수 있는가?**
2. CN7과 RG3의 예측가능성이 왜 다른가?
3. 동일한 공정조건에서 Pass/Fail이 충돌하는 경우 모델 성능의 한계는 무엇인가?
4. 단일 shot 분류보다 **run 단위 검사자원 배분**이 더 중요한가?
5. 가스·미성형·초기허용불량을 분리한 **physics-guided mechanism model**이 도움이 되는가?
6. 불량이 독립적으로 발생하는지, 아니면 **episode/burst** 형태로 발생하는가?
7. 실제 현장에서 제한된 검사량으로 더 많은 불량을 잡으려면 어떤 정책이 필요한가?

---

## 1. 핵심 요약

전체 KAMP 원본 라벨 데이터는 정제 후 다음 구조로 재구성했습니다.

- Raw labeled rows: **7,996**
- Exact-duplicate 제거 후: **5,232**
- Shot 단위 통합: **2,624 shots**
- Defective shots: **52**
- Production runs: **15**
  - CN7: 7 runs
  - RG3: 8 runs

### 주요 결과

#### A. Current KAMP 데이터의 Identifiability 한계

대회 제공 24개 공정변수 기준으로 exact-X 조건을 묶었을 때:

| Product | Labeled rows | Fails | Exact conditions | Conflicting conditions | Fails in conflict |
|---|---:|---:|---:|---:|---:|
| CN7 | 1,211 | 17 | 606 | 11 | 11 / 17 (64.7%) |
| RG3 | 1,182 | 25 | 591 | 25 | 25 / 25 (100%) |

특히 RG3에서는 모든 불량이 **동일 공정조건 X에서 Pass/Fail이 함께 존재하는 conflict group**에 포함됩니다.

이는 “모델이 약하다”는 것과 구분해야 합니다.  
현재 24개 공정변수만으로는 일부 불량의 label을 완전히 구분할 정보가 관측되지 않았다는 의미입니다.

---

#### B. v5 Row-level 예측

Exact-condition leakage를 막은 group-safe nested CV 결과:

| Product | Selected model | PR-AUC | Top 10% Recall | Lift@10% |
|---|---|---:|---:|---:|
| CN7 | Random Forest | **0.326** | **94.1%** | **9.41×** |
| RG3 | Random Forest | **0.034** | **8.0%** | **0.80×** |

Permutation test에서도:

- CN7 PR-AUC: **p = 0.0099**
- RG3 PR-AUC: **p = 0.8317**

따라서 CN7에는 공정변수 기반 predictive signal이 존재하지만, RG3는 현재 X만으로 일반화 가능한 signal이 매우 약합니다.

---

#### C. Legacy/full KAMP context 분석

Legacy raw 데이터에서 `PART_NAME / SIDE` context를 추가하면 동일-X ambiguity가 크게 줄어듭니다.

- CN7: conditional entropy reduction **100%**
- RG3: conditional entropy reduction **100%**

하지만 SIDE를 추가했을 때 새로운 조건에 대한 predictive PR-AUC 증가는 제한적이었습니다.

- CN7 RandomForest: `X only 0.595 → X + SIDE 0.631`
- RG3 RandomForest: `X only 0.084 → X + SIDE 0.128`

즉:

> **Identifiability resolution ≠ Generalization improvement**

SIDE는 동일-X 충돌을 설명하는 중요한 context이지만, 새로운 공정조건에서의 품질 예측을 자동으로 해결하지는 않습니다.

---

#### D. Run-aware inspection

2,624 shots / 52 fails / 15 runs 재구성 후 run-aware LOGO validation을 수행했습니다.

v6 compact implementation의 global allocation 결과:

| Budget | Time-only Recall | Risk-only Recall | Hybrid 60min + Risk |
|---:|---:|---:|---:|
| 10% | 19.2% | 23.1% | **32.7%** |
| 20% | 36.5% | 51.9% | **59.6%** |
| 30% | 80.8% | 71.2% | **82.7%** |
| 40% | 90.4% | 80.8% | **96.2%** |

팀 최종 보고서의 90-feature constrained logistic 정책에서는 20% 검사 시나리오에서 **37 / 52 defects**를 포착했습니다.

중요한 관찰은 모델의 이득이 같은 run 안에서 shot을 완벽히 분리하는 데서보다, **어느 run에 검사자원을 더 배정할지 결정하는 데서 더 크게 발생했다는 점**입니다.

---

#### E. Defect Episode / Burst

불량 발생의 시간적 의존성을 확인했습니다.

- 이전 shot이 양품일 때 다음 shot 불량 확률: **1.29%**
- 이전 shot이 불량일 때 다음 shot도 불량일 확률: **36.54%**

단, 이 효과는 모든 run에 균일하지 않으며 일부 run에서 강한 burst/episode가 나타납니다.

따라서 52개의 defective shots를 모두 독립 사건으로 취급하는 대신,
**shot-level recall + episode-level recall**을 함께 평가할 필요가 있습니다.

---

#### F. Physics-guided mechanism model

불량을 하나의 binary target으로만 보지 않고:

- Gas
- Short shot
- Startup defect

세 mechanism으로 분리한 one-vs-rest logistic model을 구축했습니다.

최종 score:

```text
1 - (1 - p_gas) * (1 - p_shortshot) * (1 - p_startup)
```

LOGO OOF 결과:

| Model | PR-AUC | ROC-AUC | Recall@10% | Recall@20% |
|---|---:|---:|---:|---:|
| Generic phase-aligned | 0.0447 | 0.6858 | 25.0% | 51.9% |
| Physics-guided mechanism | **0.1035** | **0.7299** | **26.9%** | 50.0% |

20% recall 자체는 개선되지 않았지만, PR-AUC와 상위 risk tail ranking에서는 개선이 나타났습니다.

---

## 2. 프로젝트 흐름

```text
KAMP competition files
        │
        ▼
Data audit / label normalization
        │
        ├── duplicate removal
        ├── recording-error correction
        ├── LH/RH → shot aggregation
        └── run segmentation
        │
        ▼
Identifiability audit
        │
        ├── exact-X conflicts
        ├── near-duplicate sensitivity
        ├── empirical ambiguity limits
        └── legacy SIDE/PART_NAME context
        │
        ▼
Predictive modeling
        │
        ├── Logistic Regression
        ├── Random Forest
        ├── XGBoost
        ├── condition-level target
        ├── soft-label target
        ├── anomaly / one-class models
        └── physics-guided mechanism heads
        │
        ▼
Run-aware validation
        │
        ├── LOGO (Leave-One-Run-Out)
        └── Sequential past → future
        │
        ▼
Operational inspection
        │
        ├── Time-only
        ├── Risk-only
        ├── 60min + Risk
        ├── defect episode
        └── dynamic feedback / budget router
```

---

## 3. Repository 구성

저장소에는 탐색 과정의 여러 버전이 포함되어 있습니다.

### 권장 메인 파일

| File | 역할 |
|---|---|
| `final_v5.ipynb` | Current KAMP identifiability + row/condition/soft-target 분석의 핵심 |
| `KAMP01_Final_Analysis_v5.py` | v5 전체 Python export |
| `KAMP01_v7_SCOREMAX_fixed.ipynb` | Run-aware + domain-aware v7 통합 분석 |
| `KAMP01_v7_SCOREMAX.py` | v7 Python export |
| `KAMP01_v8_SCOREMAX.ipynb` | Hard budget router + low-budget bootstrap 추가 실험 |
| `KAMP01_v8_SCOREMAX.py` | v8 Python export |
| `KAMP_52개_불량샷_모델비교_정리.docx` | 팀 최종모델 vs mechanism model 사례 비교 |

### 이전 개발 버전

```text
KAMP_01_Injection_Full_Analysis.ipynb
KAMP_01_Injection_Full_Analysis_v2.ipynb
KAMP01_Injection_Full_Analysis_v3.ipynb
KAMP01_Injection_Full_Analysis_v4_External.ipynb
KAMP01_Final_Analysis_v5.py
final_v5.ipynb
v6_novelty.ipynb
v6_kamp_upload_external_auto_fixed.ipynb
KAMP01_v7_SCOREMAX_fixed.ipynb
KAMP01_v8_SCOREMAX.ipynb
```

v1–v4는 탐색/개발 history 보존용이며, 재현용 메인 진입점은 v5 이후를 권장합니다.

---

## 4. 결과 파일 구조

### v5 — Identifiability / prediction

대표 결과:

```text
E0_*   data / feature audit
E1_*   exact-X entropy / identifiability
E2_*   nested model comparison / OOF predictions
E3_*   calibration
E4_*   error analysis / SHAP / permutation importance
E5_*   Top-k / cost scenarios
E6_*   selective classification
E7_*   labeled-vs-unlabeled domain check
E8_*   anomaly / PCA / AE / one-class
E9_*   condition-level modeling
E10_*  bootstrap confidence intervals
E11_*  soft-target analysis
E12_*  unlabeled anomaly analysis
E14_*  legacy context / SIDE entropy
E15_*  AIRTLab external comparison
E21_*  legacy context predictive ablation
E22_*  current ↔ legacy condition linkage
E23_*  repeated soft-label CV
E24_*  permutation test
E25_*  feature-family ablation
E26_*  AIRTLab robustness audit
```

주요 종합 파일:

```text
FINAL_summary_v5.csv
FINAL_rubric_evidence_map_v5.csv
FINAL_execution_audit.csv
result_manifest.csv
source_hashes.csv
```

---

### v6 / v7 — run-aware / dynamic QC

```text
V6_01_shot_table.csv
V6_03_logo_predictions.csv
V6_03_sequential_predictions.csv
V6_03_run_aware_budget_curves.csv
V6_03_within_run_auc.csv

V6_04_side_oof_predictions.csv
V6_05_conformal_side_predictions.csv
V6_06_hierarchical_inspection_policy.csv
V6_07_value_of_information.csv

V7_11_current_identifiability_summary.csv
V7_12_phase_alignment_summary.csv
V7_13_adaptive_state_policy_summary.csv
V7_14_episode_sensitivity.csv
V7_14_defect_transition_pooled.csv
V7_15_nested_router_summary.csv
V7_16_mechanism_comparison.csv
V7_16_mechanism_oof_predictions.csv
V7_17_early_warning_summary.csv
V7_18_probayes_status.csv
```

종합 결과:

```text
FINAL_v6_novelty_summary.csv
FINAL_v7_evidence_promotion_gate.csv
FINAL_v7_rubric_evidence_map.csv
FINAL_v7_scoremax_headlines.csv
```

---

## 5. 데이터

### 5.1 Competition KAMP files

분석 실행 시 사용자가 직접 업로드하는 공식 대회 파일:

```text
moldset_labeled_cn7.csv
moldset_labeled_rg3.csv
moldset_unlabeled_cn7.csv
moldset_unlabeled_rg3.csv
```

또는 위 파일이 포함된 competition ZIP을 업로드할 수 있습니다.

> **주의:** 저장소에 원본 KAMP 데이터를 포함할 경우 반드시 대회/데이터 이용조건을 먼저 확인하십시오.  
> 공개 권한이 명확하지 않은 원본 데이터는 GitHub에 직접 업로드하지 않는 것을 권장합니다.

---

### 5.2 Public / supplementary sources

일부 notebook은 다음 공개 source를 자동으로 가져오도록 작성되어 있습니다.

#### Legacy/full KAMP

```text
https://github.com/johnwslee/injection_molding_analysis
```

#### AIRTLab injection molding quality dataset

```text
https://github.com/airtlab/machine-learning-for-quality-prediction-in-plastic-injection-molding
```

#### SCATIM

```text
https://github.com/sc4t1m/scatimdata
```

RWTH / ProBayes 데이터는 notebook 내 source URL을 참고하십시오.

외부 데이터는 KAMP 성능 수치를 직접 대체하기 위한 것이 아니라,
**context completeness / sensor richness / external-domain evidence** 확인을 목적으로 사용합니다.

---

## 6. 실행 방법

### Google Colab 권장

가장 간단한 방법:

1. `KAMP01_v7_SCOREMAX_fixed.ipynb` 또는 `KAMP01_v8_SCOREMAX.ipynb`를 Colab에서 엽니다.
2. `Run all`
3. 업로드 창이 나오면 **KAMP 공식 competition file만 업로드**
4. 공개 supplementary source는 notebook이 자동으로 가져옵니다.
5. 결과는 CSV 및 ZIP으로 저장됩니다.

### 분석 목적별 권장 notebook

#### Current KAMP identifiability / 모델 비교

```text
final_v5.ipynb
```

#### Run-aware / mechanism / episode 분석

```text
KAMP01_v7_SCOREMAX_fixed.ipynb
```

#### Budget-controlled dynamic policy 실험

```text
KAMP01_v8_SCOREMAX.ipynb
```

> v8은 v7 이후 추가 실험용 코드입니다. 현재 repository에 v8 실행 결과가 없다면 v8 수치를 확정 결과처럼 인용하지 마십시오.

---

## 7. 전처리

Full/legacy KAMP pipeline의 주요 전처리는 다음과 같습니다.

### Label

```text
1 = Fail
0 = Pass
```

원본/가이드북 간 표기 차이를 확인한 후 실제 불량사유와 일치하도록 통일합니다.

### Duplicate

Raw 7,996 rows에서 exact duplicates 제거:

```text
7,996 → 5,232
```

### Shot aggregation

같은 shot의 LH/RH part가 공정값을 공유하므로 shot 단위로 통합합니다.

```text
5,230 part rows → 2,624 shots
```

한쪽 part라도 불량이면 해당 shot을 defective shot으로 정의합니다.

### Run segmentation

같은 제품에서 timestamp gap이 30분을 초과하면 새로운 production run으로 정의합니다.

```text
15 runs
CN7 = 7
RG3 = 8
```

### Recording-error correction

- `Average_Screw_RPM`이 `Max_Screw_RPM`보다 비현실적으로 약 10배 큰 구간을 보정
- 물리적으로 0이 되기 어려운 sensor temperature 0은 missing으로 처리
- 비라벨 historical data의 sensor-zero 문제를 분석

---

## 8. Feature engineering

### Raw process variables

24개 원시 공정변수:

- Injection / Filling / Plasticizing / Cycle time
- Clamp close/open
- Cushion / Plasticizing position
- Injection speed / pressure
- Screw RPM
- Back pressure
- Barrel temperature 1–6
- Hopper temperature
- Mold temperature 3–4

### Run context

- `elapsed_min`
- `log_elapsed_min`
- `early60`
- `run_shot_index`
- `product_RG3`

### Process-derived features

예:

- barrel mean / std / gradient
- mold mean / side difference
- melt-to-mold temperature difference
- filling-to-injection time ratio
- switch-over-to-injection pressure ratio
- back-pressure stability
- plasticizing position per second
- previous-shot change
- previous-5-shot range
- mold-temperature slope

---

## 9. v7 Physics-guided mechanism model

### 9.1 Targets

`Reason`은 feature로 사용하지 않고 mechanism target 생성에만 사용합니다.

```text
Gas         → _mech_gas
Short shot  → _mech_shortshot
Startup     → _mech_startup
```

### 9.2 Model

각 mechanism은 one-vs-rest Logistic Regression입니다.

```python
Pipeline([
    ("imp", SimpleImputer(strategy="median")),
    ("sc", StandardScaler()),
    ("clf", LogisticRegression(
        class_weight="balanced",
        C=0.5,
        max_iter=4000
    ))
])
```

### 9.3 Combined score

```python
score_mechanism_combined = (
    1
    - (1 - p_gas)
    * (1 - p_shortshot)
    * (1 - p_startup)
)
```

### 9.4 Feature families

#### Gas

주로:

- Barrel temperatures
- Hopper temperature
- Plasticizing time / position
- Screw RPM
- Back pressure
- Cycle time
- lag-1 screw / plasticizing / back-pressure variables

#### Short shot

주로:

- Injection time
- Filling time
- Cushion position
- Injection speed
- Injection pressure
- Switch-over pressure
- Mold temperature
- recent delta / 5-shot range

#### Startup

주로:

- elapsed time
- early60
- run shot index
- mold temperature
- filling / speed
- recent process variation

정확한 목록은:

```text
V7_16_feature_lists.txt
```

참조.

---

## 10. Validation

### 10.1 Current KAMP v5

동일 exact-condition X가 train/test에 섞여 optimistic score가 발생하지 않도록
**exact-condition group-safe nested CV**를 사용합니다.

### 10.2 Full/legacy run-aware

#### LOGO

```text
Leave-One-Run-Out
```

test run 하나 전체를 제거하고 나머지 run으로 학습합니다.

따라서 같은 production run의 일부 shot이 train/test에 동시에 존재하지 않습니다.

#### Sequential validation

지원되는 run-aware / phase-aligned experiment에서는:

```text
train = test run보다 과거에 시작한 runs
test  = future run
```

구조를 사용합니다.

### 중요: v7 mechanism model

`V7_16_mechanism_oof_predictions.csv`의 mechanism model은 **LOGO OOF**입니다.

현재 v7_16 코드에서는 mechanism head 자체에 대해 별도 past→future sequential validation을 수행하지 않았습니다.

따라서:

> “v7 mechanism model도 sequential validation을 완료했다”

라고 주장하면 안 됩니다.

---

## 11. Leakage policy / 시간 정보 사용

### 사용

- 현재 shot 공정값
- 현재까지의 run elapsed time
- 현재 shot index
- previous shot
- previous-five-shot history

### 사용하지 않음

- test run의 label
- 미래 shot 값
- run 종료 후 계산되는 whole-run mean / median을 현재 shot feature로 사용하는 것
- held-out run을 이용한 scaler/imputer fitting

Imputer와 scaler는 각 training fold에서만 fit됩니다.

### 모델의 operational timing

v7 mechanism model은:

> **현재 shot 성형이 끝난 직후 공정 log를 이용해 검사 우선순위를 정하는 모델**

입니다.

즉 “현재 shot을 성형하기 전에 불량을 예방하는 pre-shot prediction model”은 아닙니다.

---

## 12. Identifiability와 predictive performance의 구분

이 프로젝트에서 가장 중요한 개념 중 하나입니다.

### Identifiability

```text
같은 X인데 서로 다른 Y가 존재하는가?
```

관측 feature만으로 개별 label을 결정할 수 있는지에 관한 문제입니다.

### Generalization

```text
새로운 X에서 Y를 얼마나 잘 예측하는가?
```

SIDE context가 identical-X conflict를 완전히 해소할 수 있어도,
새로운 조건에서 predictive score가 크게 오르지 않을 수 있습니다.

따라서:

```text
SIDE resolves ambiguity
```

와

```text
SIDE solves prediction
```

은 같은 주장으로 취급하지 않습니다.

---

## 13. Negative / inconclusive experiments

성공한 실험만 남기지 않고 실패한 가설도 결과로 보존했습니다.

### Side predictor

- static RH-first baseline이 learned side model보다 우수
- learned side prediction은 main method에서 제외

### Conformal side prediction

목표 coverage를 충족하지 못해 main contribution으로 사용하지 않습니다.

### Phase-aligned lag

previous-cycle plasticizing alignment를 실험했으나 주요 held-out 성능 향상은 관찰되지 않았습니다.

### Adaptive stabilization

고정 60분 rule을 일관되게 대체하지 못했습니다.

### Run-level early warning

15 runs의 작은 표본으로 인해 안정적인 결과를 확보하지 못했습니다.

### ProBayes automatic ablation

현재 자동 target detector가 reliable binary target을 확인하지 못한 실행에서는 analysis를 `SKIPPED` 처리했습니다.

실패한 실험을 성능 개선으로 포장하지 않는 것이 본 repository의 원칙입니다.

---

## 14. Reproducibility

분석 과정에서 다음 자료를 저장합니다.

- OOF predictions
- fold-level metrics
- feature dictionary
- model comparison
- bootstrap confidence intervals
- permutation null distribution
- source hashes / manifest
- execution audit
- selected-model summary

주요 파일:

```text
source_hashes.csv
result_manifest.csv
result_manifest_v6.csv
FINAL_execution_audit.csv
```

---

## 15. 현재 권장 해석

현재 결과는 다음과 같이 해석하는 것이 가장 안전합니다.

### 확인된 것

- CN7에는 현재 process X에서 유의미한 predictive signal이 존재
- RG3는 exact-X ambiguity가 매우 강함
- SIDE / cavity context는 identifiability에 중요
- run context는 inspection allocation에 유용
- 일부 defective shots는 burst/episode 형태로 발생
- physics-guided mechanism separation은 high-risk ranking에 추가 정보를 줄 가능성이 있음

### 확인되지 않은 것

- 관측 feature가 실제 불량의 causal cause라는 주장
- RG3가 “예측 불가능”하다는 절대적 주장
- SIDE 하나만 추가하면 품질예측 문제가 해결된다는 주장
- mechanism model의 sequential generalization 완료
- v8 dynamic router의 최종 우월성 (실행 결과 확인 전)

---

## 16. Limitations

### Small sample

Full run-aware dataset:

```text
15 runs
52 defective shots
8 runs containing defects
```

run 하나가 전체 결과를 크게 흔들 수 있습니다.

### Missing local/cavity information

같은 shot의 LH/RH part는 machine-level process values를 공유하므로,
한쪽 cavity만 불량인 원인을 현재 shot-level X만으로 식별하기 어렵습니다.

추가적으로 유용할 수 있는 데이터:

- cavity pressure
- cavity-specific mold temperature
- vent maintenance / cleaning history
- dryer temperature / drying time
- material moisture
- material lot
- melt-flow information
- startup procedure records

### Selection / research iteration

여러 실험을 반복하면서 연구가 발전했기 때문에,
후기 exploratory result를 완전 독립 test result처럼 해석하면 안 됩니다.

핵심 결론은 LOGO / nested CV / permutation / bootstrap 등 가능한 범위에서 robustness check를 함께 제시합니다.

---

## 17. Suggested GitHub structure

현재 작업 파일 전체를 보존하면서도 repository를 읽기 쉽게 하려면 다음 구조를 권장합니다.

```text
.
├── README.md
│
├── notebooks/
│   ├── final_v5.ipynb
│   ├── KAMP01_v7_SCOREMAX_fixed.ipynb
│   ├── KAMP01_v8_SCOREMAX.ipynb
│   └── archive/
│       ├── KAMP_01_Injection_Full_Analysis.ipynb
│       ├── KAMP_01_Injection_Full_Analysis_v2.ipynb
│       ├── KAMP01_Injection_Full_Analysis_v3.ipynb
│       ├── KAMP01_Injection_Full_Analysis_v4_External.ipynb
│       └── v6_*.ipynb
│
├── src/
│   ├── KAMP01_Final_Analysis_v5.py
│   ├── KAMP01_v7_SCOREMAX.py
│   ├── KAMP01_v8_SCOREMAX.py
│   └── archive/
│
├── results/
│   ├── v5/
│   │   ├── FINAL_summary_v5.csv
│   │   ├── E1_*.csv
│   │   ├── E2_*.csv
│   │   ├── ...
│   │   └── E26_*.csv
│   │
│   └── v7/
│       ├── FINAL_v7_scoremax_headlines.csv
│       ├── V6_*.csv
│       └── V7_*.csv
│
├── docs/
│   ├── KAMP_52개_불량샷_모델비교_정리.docx
│   ├── V7_16_feature_lists.txt
│   └── V7_16_methodology_notes.txt
│
└── data/
    └── README.md
```

원본 데이터는 repository에 직접 포함하지 않고 `data/README.md`에 획득 방법만 설명하는 것을 권장합니다.

---

## 18. Version history

### v1–v3
- basic preprocessing
- baseline classification
- early model comparison

### v4
- external / legacy context analysis 확장
- exact-X ambiguity 분석 강화

### v5
- identifiability-focused analysis 정리
- exact-condition group-safe nested CV
- row / condition / soft-target 비교
- bootstrap / permutation
- legacy SIDE context
- AIRTLab robustness experiments

### v6
- full raw KAMP reconstruction
- shot / run aggregation
- run-aware LOGO / sequential
- hierarchical inspection
- side / conformal experiments

### v7
- current identifiability audit 통합
- process phase alignment
- adaptive stabilization
- defect episodes
- nested dynamic policy
- physics-guided mechanism heads
- run-level early warning
- external ProBayes attempt

### v8
- hard-budget dynamic feedback router
- ultra-low-budget mechanism validation
- run-cluster bootstrap

> v8 코드는 후속 검증용입니다. 결과 archive가 없는 경우 결과를 확정적으로 인용하지 마십시오.

---

## 19. Main files for review

프로젝트를 빠르게 리뷰하려면 아래 순서만 보면 됩니다.

```text
1. README.md
2. final_v5.ipynb
3. KAMP01_v7_SCOREMAX_fixed.ipynb
4. results/v5/FINAL_summary_v5.csv
5. results/v7/V7_11_current_identifiability_summary.csv
6. results/v7/V6_03_run_aware_budget_curves.csv
7. results/v7/V7_14_defect_transition_pooled.csv
8. results/v7/V7_16_mechanism_comparison.csv
9. results/v7/V7_16_mechanism_oof_predictions.csv
10. docs/KAMP_52개_불량샷_모델비교_정리.docx
```

---

## 20. Competition framing

이 프로젝트의 최종 메시지는 다음과 같습니다.

> 사출성형 품질문제를 단순한 shot-level binary classification으로만 다루면,
> 관측되지 않은 cavity/side context와 run-state variation 때문에 모델의 한계가 발생한다.
>
> 따라서 품질 AI는 먼저 **현재 데이터로 식별 가능한 불량과 식별하기 어려운 불량을 구분**하고,
> 그다음 **run-aware risk ranking, mechanism-aware interpretation, episode-aware inspection**
> 으로 QC 자원을 배분해야 한다.

---

## 21. Notes for reviewers

- 모든 model score는 해당 experiment의 validation protocol을 확인한 후 해석하십시오.
- `V7_16_mechanism_oof_predictions.csv`는 LOGO OOF 결과입니다.
- current-shot variables를 사용하는 model은 inspection-prioritization model이지 pre-shot prevention model이 아닙니다.
- 동일-X ambiguity bound는 empirical dataset bound이며 formal Bayes error bound가 아닙니다.
- external dataset 결과는 KAMP 결과와 직접 수치 비교하지 않습니다.
- negative experiments도 reproducibility를 위해 repository에 보존했습니다.

---

## 22. Authors / Team

제6회 K-인공지능 제조데이터 분석 경진대회  
일반국민 / 대학(원)생 부문

Team information can be added here before publication.

```text
Team:
Members:
Contact:
```

---

## 23. License

코드 라이선스는 공개 전 팀 정책에 따라 지정하십시오.

예:

```text
MIT License
```

단, **KAMP 원본 데이터 및 외부 데이터의 라이선스는 각 데이터 제공처의 조건을 따릅니다.**
코드 라이선스가 데이터 재배포 권한을 의미하지 않습니다.

---

## 24. Citation

Repository를 외부 발표/문서에서 인용할 경우 다음 형식을 사용할 수 있습니다.

```text
[Team Name], "Identifiability-Aware Inspection Prioritization for
Injection Molding Quality Control," K-AI Manufacturing Data Analysis
Competition, 2026.
```

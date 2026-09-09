# KIM Precipitation Correction

KIM의 강수 예측을 IMERG 관측 강수를 기준으로 딥러닝 보정하는 연구 프로젝트입니다.

이 저장소는 **실험 코드, 설정, 실험별 변경점과 현재 연구 진행상황**을 기록하기 위한 용도로 사용합니다. 대용량 원자료, 학습 산출물, 체크포인트는 Google Drive에 두고 GitHub에는 올리지 않는 것을 원칙으로 합니다.

---

## 1. 연구 목표

현재 핵심 목표는 다음과 같습니다.

1. **KIM h006 강수 예측을 IMERG에 가깝게 보정**한다.
2. 단순히 전체 영역의 RMSE/Bias만 낮추는 것이 아니라,
   - 강수 영역의 위치
   - 강수 코어의 강도
   - 강수 구조와 형태
   - 강수 시스템의 이동
   을 더 잘 복원하는 것을 목표로 한다.
3. 모든 사례를 하나의 분포로 취급하지 않고 **기상장/강수 형태에 따른 regime 분류**를 이용해 보정 성능을 높인다.
4. 최신 실험에서는 KIM의 현재 예측장만 사용하는 것에서 더 나아가, **목표 시각 이전의 IMERG 관측 강수 이력(history)** 과 **강수 이동(motion) 정보**를 입력에 포함하는 방향을 시험한다.

쉽게 말하면, 기존 방식이 현재의 KIM 예보장 한 장만 보고 오답을 고치는 방식이었다면, 현재 방향은 **직전 몇 시간 동안 실제 비가 어디에서 어디로 이동해 왔는지까지 보고 KIM 예보를 보정하는 방식**으로 확장하는 것이다.

---

## 2. 현재 기본 문제 설정

### KIM

- forecast lead: `h006`
- 강수 누적 구간: `3 h`
- 강수 timestamp 해석: `end`
- 따라서 KIM 초기시각을 `t0`라고 하면 보정 대상 강수 구간은 기본적으로

```text
[t0 + 3 h, t0 + 6 h)
```

으로 해석한다.

### IMERG

- 30분 간격 HDF5 강수 자료 사용
- KIM과 동일한 valid window의 IMERG를 누적
- IMERG regular lat-lon grid를 KIM grid로 보간
- 최종적으로 KIM 예측과 IMERG target을 같은 격자/시간 구간에서 비교

### 공간 영역

현재 주 평가/학습 영역:

```text
112–142°E
23–47°N
```

KIM subset 크기는 약 `192 × 241` grid이다.

---

## 3. 기존 보정 파이프라인

현재까지의 주요 baseline 계열은 다음과 같다.

### 모델

- `ResUNetGN`
- single-head regression
- Group Normalization 사용

### KIM 기반 입력

기존 주요 코드에서는 대략 다음 정보를 사용한다.

- pressure-level KIM 변수 7개
- surface/unis KIM 변수 8개
- KIM precipitation
- 500 hPa 수증기 이류(`qadv500`)

즉 기존 대표 입력은 총 17채널 계열이다.

### Regime clustering

기존 실험에서는 학습 사례를 기상/강수 상태에 따라 나누기 위해 다음 계열을 시험했다.

- PCA → weighted KMeans
- PCA → SOM

클러스터링용 대표 정보:

- 지상기압 계열
- KIM 강수
- 500 hPa 상대와도

### 강수 중심 학습/평가 개선

최근 baseline/hybrid 계열에서는 다음 방향을 추가해 왔다.

- rainy crop
- 강한 강수/core 영역에 높은 가중치
- 전체 영역뿐 아니라 강수 core와 주변/외부 영역을 분리한 평가
- RMSE/Bias map 및 composite
- synoptic diagnostic figure

---

## 4. 현재 최신 방향: EXP55

현재 Google Drive에서 확인되는 최신 실험 계열은

```text
exp55_imerg_history_som_motion
```

이며 run name은

```text
out_kim6h_km_vort500_qadv500_exp55_imerg_history_som_motion
```

이다.

### EXP55에서 하려는 것

기존 KIM 기반 보정에 다음 정보를 추가하는 것이 핵심이다.

#### 1) IMERG 과거 강수 이력

현재 로그에서 확인되는 history lag:

```text
12 h, 9 h, 6 h, 3 h, 0 h
```

각 history는 목표 강수 구간보다 앞선 IMERG 관측으로 구성하여, **정답 강수 구간을 입력으로 보는 data leakage가 발생하지 않도록** 설계하는 것이 원칙이다.

#### 2) SOM 기반 regime 정보

KMeans 계열뿐 아니라 SOM을 사용하여 서로 비슷한 기상/강수 상태를 묶고, 사례별 특성에 맞는 보정을 수행하는 방향을 사용한다.

#### 3) precipitation motion 정보

단순한 이전 강수장의 존재 여부만 보는 것이 아니라, 시간에 따른 강수장의 변화로부터 **강수 시스템의 이동 방향/속도/조직화 정보**를 모델에 제공하는 방향이다.

즉 EXP55의 핵심 질문은 다음과 같다.

> **KIM의 대기 상태와 강수 예보 + 직전 IMERG 강수의 시간 변화 + regime 정보를 함께 보면, 강수 위치·코어·구조를 더 정확하게 보정할 수 있는가?**

---

## 5. 현재 EXP55 상태와 발견된 문제

Drive의 최신 training log 기준:

- 설정된 기간에서 KIM pair: `1524`
- 기간 필터 후: `1462`
- IMERG target availability 검사 통과: `1462 / 1462`
- 그러나 IMERG history strict precheck 결과:

```text
valid = 0
skipped = 1462
```

즉 **target 자체는 모두 존재하지만, EXP55가 요구하는 과거 IMERG history가 완전하지 않아 실제 학습까지 진입하지 못한 상태**다.

첫 실패 사례는 다음과 같다.

```text
KIM init       : 2023-01-01 00 UTC
Target window  : 2023-01-01 03–06 UTC
12 h history   : 2022-12-31 12–15 UTC
```

해당 history window에 필요한 일부 30분 IMERG 파일이 없어 strict history check에서 제외되었고, 현재 조건에서는 최종적으로 모든 sample이 제외되어 학습이 중단되었다.

---

## 6. 바로 다음에 해야 할 일

### 우선순위 1 — IMERG history coverage 문제 해결

다음 둘 중 하나를 선택한다.

1. 필요한 2022-12-31 초기 history IMERG 파일을 확보한다.
2. 모든 history lag가 완전히 존재하는 시각 이후부터 학습 sample을 시작하도록 valid period 시작시각을 뒤로 민다.

가능하면 **strict history 조건은 유지**하는 쪽이 안전하다. 결측 history를 임의로 0 또는 다른 시각 자료로 대체하면 학습 입력의 의미가 달라질 수 있기 때문이다.

### 우선순위 2 — 시간 정렬/data leakage 재검증

특히 `lag=0 h` history가 실제 target window와 겹치지 않는지 코드 레벨에서 확인한다.

체크해야 할 원칙:

```text
모든 IMERG history의 종료시각 <= target 시작시각
```

이어야 한다.

### 우선순위 3 — EXP55 재학습

history availability 검사를 통과한 뒤 실제 학습을 수행한다.

### 우선순위 4 — baseline과 정량 비교

EXP55를 이전 best/baseline과 동일한 validation set에서 비교한다.

주요 비교 대상:

- 전체 영역 RMSE / Bias
- 강수 영역 RMSE / Bias
- 강수 core 성능
- 위치 오차
- 강수 구조/형태
- 강수 강도
- 사례별 composite 및 synoptic pattern

---

## 7. 현재 데이터/출력 위치

Colab 기준 데이터 루트:

```text
/content/drive/MyDrive/colab_data/kim_precip_corr
```

```text
KIM   : /content/drive/MyDrive/colab_data/kim_precip_corr/KIM
IMERG : /content/drive/MyDrive/colab_data/kim_precip_corr/IMERG
```

실험 output:

```text
/content/drive/MyDrive/DL/KIM_precip_corr/
```

예:

```text
/content/drive/MyDrive/DL/KIM_precip_corr/exp55_imerg_history_som_motion/
```

---

## 8. 저장소 운영 원칙

이 저장소에는 앞으로 다음을 기록한다.

```text
KIM-Prcp-Corr/
├── README.md
├── experiments/       # EXP별 코드/설정/변경점
├── src/               # 공통 모듈화 코드
├── docs/              # 연구 메모 및 설계 문서
└── notebooks/         # 필요한 Colab notebook
```

반대로 아래 대용량 자료는 GitHub에 올리지 않는다.

- KIM 원자료
- IMERG 원자료
- cache
- model checkpoint
- 대규모 output figure/animation

---

## 9. 현재 코드 동기화 상태

Google Drive의 `exp55_imerg_history_som_motion` output 폴더에서는 최신 training log와 진단 파일은 확인되었지만, **현재 확인 범위에서는 EXP55의 실제 source `.py`/`.ipynb` 파일이 output 폴더 안에 직접 저장된 것은 확인되지 않았다.**

따라서 다음 단계에서는 실제 EXP55 source script/notebook을 찾아 이 저장소의 `experiments/exp55/` 아래로 동기화하는 것이 필요하다.

---

## 현재 한 줄 요약

> **KIM h006 강수 예보를 IMERG로 보정하되, 최신 단계에서는 과거 IMERG 강수 시계열과 강수 이동 정보, SOM 기반 regime 분류를 결합해 강수 위치·코어·구조까지 개선하려고 하며, 현재 EXP55는 과거 IMERG history 파일 coverage 문제 때문에 학습 직전 단계에서 멈춰 있다.**

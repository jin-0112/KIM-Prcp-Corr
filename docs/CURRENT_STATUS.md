# 현재 연구 상태

업데이트: 2026-09-09

## 지금 하려는 일

KIM `h006` 강수 예측을 IMERG 관측 강수에 맞게 보정하는 기존 딥러닝 파이프라인을 확장하여, **강수의 양뿐 아니라 위치·코어·공간 구조와 이동까지 더 잘 복원하는 모델**을 만드는 것이 현재 목표다.

현재 최신 방향은 `EXP55: exp55_imerg_history_som_motion`이다.

핵심 아이디어는 다음 세 가지다.

1. **KIM 대기장/강수 예측**을 기본 입력으로 사용한다.
2. 목표 강수 구간 이전의 **IMERG 과거 강수 이력**을 추가하여 실제 강수 시스템이 최근 어떻게 발달하고 이동했는지를 알려준다.
3. **SOM 기반 regime 분류와 precipitation motion 정보**를 이용하여 서로 다른 강수/종관 상황을 구분하고 보정한다.

즉,

```text
KIM 현재 예측장
       +
IMERG 과거 관측 강수 시계열
       +
SOM regime / 강수 이동 정보
       ↓
강수 보정 모델
       ↓
IMERG에 가까운 corrected precipitation
```

을 목표로 한다.

---

## EXP55에서 확인된 설정

```text
RUN_NAME       = out_kim6h_km_vort500_qadv500_exp55_imerg_history_som_motion
LEAD_HOURS     = 6
PRCP_STEP      = 3 h
TIME_MODE      = end
VALID_PERIOD   = 2023-01 ~ 2024-12
DOMAIN         = 112–142E, 23–47N
KIM SUBSET     = 192 × 241
HISTORY LAGS   = [12, 9, 6, 3, 0] h
```

KIM init이 `t0`일 때 현재 기본 target window는

```text
[t0 + 3 h, t0 + 6 h)
```

이다.

IMERG target은 이 동일한 구간의 30분 자료를 모아 KIM grid에 맞춰 사용한다.

---

## 현재 막힌 지점

최신 Drive training log에서:

```text
period-filtered KIM pairs       = 1462
IMERG target valid              = 1462
IMERG history valid             = 0
IMERG history skipped           = 1462
```

즉 target IMERG는 모두 존재하지만, EXP55가 요구하는 과거 history window의 파일이 완전하지 않아 학습 sample이 하나도 남지 않았다.

첫 확인 실패:

```text
KIM init      : 2023-01-01 00 UTC
Target        : 2023-01-01 03–06 UTC
12 h history  : 2022-12-31 12–15 UTC
```

이 구간에서 필요한 일부 30분 IMERG 파일이 없어 strict history check가 실패한다.

현재 로그의 최종 상태:

```text
[STOP] no selected-period pairs with valid IMERG history inputs.
```

---

## 다음 작업

### 1. IMERG history coverage부터 해결

우선 다음 중 하나를 수행한다.

- 필요한 초기 IMERG history 파일을 추가 확보하거나
- 모든 history lag가 존재하는 시각 이후로 학습 시작시각을 이동한다.

가능하면 strict history check 자체는 유지한다.

### 2. Data leakage 검사

가장 중요한 시간 정렬 조건:

```text
history_end <= target_start
```

모든 lag에 대해 이 조건을 자동 검사하도록 한다.

특히 `lag=0 h`의 의미와 실제 window 계산을 source code에서 재확인한다.

### 3. EXP55 재학습

history precheck를 통과한 뒤 모델 학습을 수행한다.

### 4. 이전 best와 동일 조건 비교

EXP55 개선 여부를 다음 기준으로 평가한다.

- 전체 RMSE / Bias
- 강수 영역 RMSE / Bias
- 강수 core의 강도
- 강수 위치
- 강수 공간 구조
- 강수 시스템 이동/형태
- composite 및 synoptic case

### 5. Source code를 GitHub로 동기화

현재 Drive의 EXP55 output 폴더에서는 training log와 결과/진단 파일은 확인했지만, 실제 EXP55 source `.py` 또는 `.ipynb`가 그 output 폴더 안에 직접 저장된 것은 확인하지 못했다.

실제 source를 찾으면 다음과 같이 관리한다.

```text
experiments/
└── exp55_imerg_history_som_motion/
    ├── train.py 또는 train.ipynb
    ├── README.md
    └── config 설명
```

---

## 한 문장으로 정리

**현재 연구는 KIM 강수 보정에 과거 IMERG 강수의 시간 변화와 SOM/motion 정보를 추가해 강수의 위치·코어·구조까지 개선하려는 단계이며, 지금 가장 먼저 해결해야 할 것은 EXP55의 IMERG history 시간 coverage 문제다.**

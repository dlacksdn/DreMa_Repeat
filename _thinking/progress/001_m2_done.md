# 001. 진행 상황 — M2 환경 구축 완료

작성: 2026-09-19
이 문서는 **"지금 어디쯤인지 한눈에 보는"** 용도다. 근거와 수치는 `Data_Analysis/003` 에 있다.

---

## 한 줄

> **M1(증강) · M2(환경) 완료. 남은 것은 M4(학습)와 M5(평가)다.**

---

## 1. 마일스톤

| 단계 | 내용 | 상태 |
|---|---|---|
| — | 코드 분석 | ✅ `Code_Analysis/001` |
| — | 샘플 데이터 분석 | ✅ `Data_Analysis/001` |
| **M1** | 증강 생성·검증 | ✅ `Data_Analysis/002` |
| **M2** | 확인사항 2가지 + τ 확정 | ✅ `Data_Analysis/003` |
| **M2** | 평가 데이터 31 GB 확보 | ✅ 무결성 검증 완료 |
| **M2** | PerAct + RLBench 환경 | ✅ **이 문서** |
| M4 | PerAct 학습 2회 | ⬜ 다음 |
| M5 | 평가 | ⬜ |

---

## 2. 환경 — 컨테이너 2개

둘 다 `restart=unless-stopped`, `/home/rils/dlacksdn → /workspace` 마운트.

| 컨테이너 | 하는 일 | 핵심 구성 |
|---|---|---|
| **`DreMa`** | 3D 복원 · 증강 생성 | Ubuntu 22.04 · py3.11 · torch 2.8.0+cu128 · PyBullet · 가우시안 CUDA 확장 |
| **`DreMaPerAct`** | 데이터 변환 · 학습 · 평가 | Ubuntu 20.04 · py3.10(miniforge) · torch 2.7.1+cu128 · CoppeliaSim 4.1 · PyRep · RLBench · YARR · PerAct |

**환경을 나눈 이유**: DreMa 저자가 `COPPELIA.md` 에서 분리를 명시했다. 실제로 의존성이 충돌한다.

### 검증된 것 (`run_logs/20260919_2305_m2_finalize.log`)

```
torch 2.7.1+cu128   cuda=True   capability (12,0) = sm_120   ← RTX 5090 정상 인식
✅ PerAct 학습 스택 전부 import 성공   (LOW_DIM_SIZE = 4)
✅ RLBench 시뮬레이터 기동 성공        (xvfb 헤드리스)
✅ RTX 5090 행렬곱 OK
```

### 긴 작업 띄우는 법

```bash
./_setup/run.sh         <이름> "<명령>"   # DreMa 컨테이너
./_setup/peract/run.sh  <이름> "<명령>"   # DreMaPerAct 컨테이너
docker exec DreMa tmux ls ; docker exec DreMaPerAct tmux ls
```
컨테이너 내부 tmux 에서 돌아 **VSCode·SSH·노트북을 꺼도 살아남는다.** 로그는 `run_logs/` 에 자동 저장.

---

## 3. 데이터 위치

| 경로 | 내용 |
|---|---|
| `data/slide_block_to_color_target_episode0_start/` | 원본 샘플 2.1 GB · **불가침** |
| **`data/aug_tau015_paper/`** | ⭐ **M4 학습용.** τ=0.015 · 원본1 + 증강17 |
| `data/aug_tau035_default/` | 비교용. τ=0.035 · 원본1 + 증강33 |
| `data/aug_rejectcheck/` | ⚠️ 기각분 조사용 · **학습 금지** |
| `data/eval_raw/` | 평가 데이터 `val.tar.gz` 13.7 GB · `test.tar.gz` 17.2 GB |

자세한 것은 `data/README.md`.
증강 영상(GIF)은 `_thinking/videos/`, 생성 스크립트는 `_setup/make_gif.py`.

---

## 4. M2 에서 확정된 것 3가지

| | 결론 |
|---|---|
| ① 궤적 스텁 | **무해.** PerAct 는 관절각을 읽지 않는다. 신경망 입력은 `[그리퍼열림, 손가락2, 시간]` **4개뿐** |
| ② 논문 τ | **`generation.threshold` = 0.015.** 코드 기본값 0.035 와 다름 → **0.015 채택** |
| ③ radius_filter | 논문은 썼지만 **33시간·21코어**가 들어 **포기.** Scharr 유지 |

**덤**: `slide_block` 은 **미는 과제(non-prehensile)** 다. 집어 옮기는 게 아니다.
- RLBench 지시문: *"push the block until it is sitting on top of the target"*
- 그리퍼를 닫는 건 쥐려는 게 아니라 **밀대를 만들려는 것** (닫힌 폭 4.0 cm < 블록 5.8 cm)
- 성공 조건도 "블록이 목표 판 위에 있는가" 뿐

---

## 5. 논문과 다른 점 — 의도적 이탈 3개

재현 결과를 해석할 때 반드시 고려해야 한다.

| # | 항목 | 논문 | 우리 | 영향 |
|---|---|---|---|---|
| 1 | **원본 시범 수** | 4개 (변형 green/blue/pink/yellow) | **1개 (green)** | **가장 큼.** 학습 데이터 65개 → 18개 (**28%**) |
| 2 | **깊이 필터** | radius outlier (Open3D) | **Scharr** | PerAct 입력 포인트클라우드가 다름 |
| 3 | `packaging` | 21.3 | 최신 | **없음** (빌드 도구. 코드가 직접 안 씀 — grep 확인) |

→ **논문 수치(48.4 % / 62.0 %)보다 낮게 나오는 것이 정상이다.** 우리가 볼 것은 절대값이 아니라 **"원본만 < 전부" 방향**이다.

---

## 6. 다음 — M4 학습

### 논문 설정 (부록 L)

| 항목 | 값 |
|---|---|
| 방법 | `PERACT_BC` |
| 배치 크기 | **4** |
| 반복 | **100k** (단일과제) |
| 체크포인트 | **5k 마다** (원본 PerAct 는 10k. 저자가 촘촘히 바꿈) |
| 카메라 | **front · left_shoulder · right_shoulder** (3대, wrist 제외) |
| 이미지 | 128×128 |
| 검증 | 40개 / 5k 마다 → 최고 모델 선택 |
| 평가 | 50개 환경 × **5회 반복** |

### 학습 2회 = 대조 실험

```
모델 A (대조군) : 원본 시범만        → 성공률 ?%   [논문 48.4]
모델 B (실험군) : 원본 + 증강 17개   → 성공률 ?%   [논문 62.0]
                        ↓
              두 수치를 비교해야 증강의 효과를 말할 수 있다
```

### 순서

```
1. prepare_data_for_peract.py 로 aug_tau015_paper 를 PerAct 형식으로 변환
     ※ "steps/images 불일치" 경고 34회는 정상 (마지막 1스텝 절단, 영향 없음)
2. val/test tar.gz 압축 해제
3. 모델 A 학습 → 모델 B 학습
4. eval.py 로 검증셋에서 최고 체크포인트 선택 → 테스트셋 평가
```

---

## 7. 재개 방법

```bash
cd /home/rils/dlacksdn/DreMa
docker exec DreMa tmux ls ; docker exec DreMaPerAct tmux ls   # 돌고 있는 작업
ls -t run_logs/ | head                                        # 최근 로그
grep -l "EXIT CODE" run_logs/*.log | tail                     # 끝난 작업

# PerAct 환경 들어가기
docker exec -it DreMaPerAct bash -lc 'source /etc/profile.d/peract_env.sh; bash'
#   python            = /opt/conda/envs/peract/bin/python
#   COPPELIASIM_ROOT  = /opt/coppeliasim
#   PERACT_ROOT       = /opt/third_party/peract
#   소스              = /opt/third_party/{peract,RLBench,YARR_peract,PyRep}
#   GUI 프로그램은 항상 xvfb-run 으로 실행
```

---

## 요약

> **M1·M2 완료.** 증강 데이터(`aug_tau015_paper`, 18개)와 평가 데이터(31 GB)가 준비됐고, PerAct 학습 환경이 RTX 5090 에서 검증까지 끝났다.
> **컨테이너는 2개** — `DreMa`(복원·증강), `DreMaPerAct`(학습·평가). 긴 작업은 각자의 `run.sh` 로 띄우면 노트북을 꺼도 살아남는다.
> **논문과 다른 점 3개**를 알고 있어야 한다. 특히 **원본 시범이 1개(논문은 4개)** 라 학습 데이터가 논문의 28% 다. **수치가 낮게 나오는 것이 정상**이고, 볼 것은 "원본만 < 전부" 방향이다.
> **다음은 M4** — 데이터 변환 → 학습 2회(원본만 / 전부) → M5 평가.

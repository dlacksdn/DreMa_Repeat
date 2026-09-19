# 003. 계획 정정 및 실행 환경 사용법

작성: 2026-09-19
대상: `001_repro_plan.md`, `002_repro_plan_scoped.md`
`_thinking`은 덧붙이기 원칙이므로 앞 문서는 고치지 않고, **이 문서가 최신**이다.

---

## 1. 앞 계획서에서 **틀린 것** — 재조사 금지

### ❌ "Hydra가 작업 폴더를 바꿔 상대경로가 깨진다"

001·002 모두 이것을 **위험 1순위**로 적었으나, **사실이 아니다.**

실측 결과:
```
시작 전 작업폴더: /workspace/DreMa
실행 중 작업폴더: /workspace/DreMa      ← 안 바뀜
assets/ 보이나 : True
data/ 보이나   : True
```

Hydra 1.3.6은 `outputs/날짜/시각/` 폴더를 **만들기만 하고** 작업 위치는 그대로 둔다. `hydra.job.chdir: false` 설정이 **불필요**하다.

→ **위험 목록에서 삭제한다.**

### ❌ "환경이 3개 필요하다"

001은 DreMa / 데이터생성 / PerAct 3개로 잡았으나 **2개면 된다.**

RLBench 포크와 YARR 포크는 원래 PerAct 설치 절차의 일부다. 저자가 분리하라고 한 것은 "DreMa ↔ PerAct"이지 "RLBench ↔ PerAct"가 아니다.

| 환경 | 역할 | 상태 |
|---|---|---|
| `DreMa` 컨테이너 | 3D 복원 · 증강 | ✅ 완료 |
| PerAct 컨테이너 | CoppeliaSim + RLBench + YARR + PerAct (변환·학습·평가) | ❌ M2에서 구축 |

---

## 2. 새로 확정된 사실

| 항목 | 내용 |
|---|---|
| **증강 소요 시간** | 에피소드당 31초, 37개 전체 **19분**. 001의 예상보다 훨씬 빠름 |
| **복원 품질** | 평균 PSNR **32.2 dB** — 증강의 토대는 신뢰할 만함 |
| **RLBench 불필요 구간** | 3D 복원·증강에는 RLBench가 필요 없다. **변환·평가·새 과제 생성에만** 필요 |
| **데이터 출처** | 공식 Google Drive·OneDrive 링크는 **둘 다 죽었다.** HuggingFace `leobarcellona/DreMa_data` 가 유일한 경로 |
| **궤적 스텁 영향** | `gripper_pose`만 변환되고 `joint_positions`·손목카메라는 원본 그대로 — **수치로 확정됨** (Data_Analysis/002) |

---

## 3. M2에서 반드시 확인할 것 2가지

### ① 궤적 스텁이 학습에 영향을 주는가
`prepare_data_for_peract.py`(leobarcellona/RLBench 포크)가 **`joint_positions`를 읽는지** 확인한다.
- 읽는다 → 증강 데이터가 물리적으로 모순된 상태로 학습에 들어간다. 대응 필요.
- 안 읽는다 → 무해. `gripper_pose`만 쓰면 문제없다.

### ② 논문 τ 값
논문은 검증 임계값 **τ = 0.015 m**라 적었는데, 설정에는 두 후보가 있다.
```yaml
simulation.trajectory.keypoint_threshold: 0.015   # 이것?
simulation.generation.threshold: 0.035            # 이것?
```
코드에서 어느 쪽이 증강 채택 판정에 쓰이는지 확인해 논문 조건을 맞춘다.
(현재까지 확인: 증강 채택 판정은 `generation.threshold` = 0.035)

---

## 4. 실행 환경 사용법 — 새 세션 필독

### 컨테이너
```
이름   : DreMa
이미지 : pytorch/pytorch:2.8.0-cuda12.8-cudnn9-devel
정책   : restart=unless-stopped  (호스트 재부팅해도 자동 기동)
마운트 : /home/rils/dlacksdn  ->  /workspace
작업폴더: /workspace/DreMa
```
**모든 실행은 컨테이너 안에서 한다.** 호스트에 직접 설치하지 않는다.

설치 완료 상태: CUDA 확장 4개, 파이썬 의존성 전부, `tmux`, 한글 폰트 `NanumGothic`.

### 긴 작업 띄우기 — `_setup/run.sh`
```bash
./_setup/run.sh <작업이름> "<명령>"
```
컨테이너 **내부 tmux**에서 돌기 때문에 **VSCode·SSH를 꺼도 살아남는다.** 로그는 `run_logs/YYYYMMDD_HHMM_<이름>.log`에 자동 저장된다.

```bash
docker exec DreMa tmux ls                       # 상태 확인
docker exec -it DreMa tmux attach -t <이름>     # 직접 보기 (나올 때 Ctrl+b 후 d)
```

> ⚠️ **Claude 세션 자체는 살아남지 않는다.** 작업은 계속 돌지만 판단해 줄 사람이 없다.
> → 반복 작업(M3·M4·M5)은 **연쇄 스크립트로 묶고**, 판단이 필요한 지점에서는 **작업만 띄워두고 멈춘다.**

### 직접 스크립트를 짤 때
```python
import sys
sys.path.insert(0, "/workspace/DreMa")
import drema.environment.builder        # 순환 참조 회피 — 반드시 먼저
from drema.environment.observer.camera import CameraManager
```
스크립트는 `_setup/`에 두고 실행한다(`.gitignore` 처리됨).

### 설정 변경은 파일 수정 대신 명령줄 덮어쓰기
```bash
python generate_new_data.py \
  data.source_path=data/<폴더> \
  simulation.visualization.visualize=False \
  simulation.generation.generated_data_path=./data/<조건별폴더>
```
저장소를 더럽히지 않고, 로그에 어떤 값으로 돌렸는지 남아 재현에 유리하다.

---

## 5. 현재 데이터 위치

| 경로 | 내용 |
|---|---|
| `data/slide_block_to_color_target_episode0_start/` | 원본 샘플 (2.1 GB) |
| `data/generated_data/.../episode*` | 증강 **34개** (정상 실행 결과) |
| `data/generated_rejected_check/.../episode*` | **17개** — 기각된 3개를 되살리려 `threshold=999`로 재실행한 것 |

조건을 바꿔 재실행할 때는 **`generated_data_path`를 다른 폴더로 지정**한다. 기존 산출물을 덮어쓰지 않는다(CLAUDE.md).

---

## 6. 갱신된 진행 상태

| 단계 | 상태 |
|---|---|
| 환경 구축 (DreMa 컨테이너) | ✅ 완료 |
| 코드 분석 | ✅ 완료 → `Code_Analysis/001` |
| 샘플 데이터 확보·분석 | ✅ 완료 → `Data_Analysis/001` |
| **M1 증강 생성·검증** | ✅ **완료** → `Data_Analysis/002` |
| **M2 PerAct + RLBench 환경** | ⬜ **다음 차례** (GPU 불필요) |
| M4 PerAct 학습 2회 | ⬜ |
| M5 평가 | ⬜ |

**평가 데이터** `val.tar.gz`(13 GB)·`test.tar.gz`(16 GB)는 회선이 10 Mbps라 8시간 걸린다. **M2 시작할 때 미리 받아 두는 것**을 권한다.

---

## 7. M2 위험 — 가장 불확실한 구간

PerAct는 2022년 코드(torch 1.x)인데, RTX 5090은 **torch 2.7 이상**을 요구한다. **저자도 해본 적 없는 조합**이다.

| 항목 | 선택 | 이유 |
|---|---|---|
| 베이스 | Ubuntu **20.04** + CUDA 12.8 | CoppeliaSim 4.1이 20.04용 배포판 |
| 파이썬 | **3.10** (문서의 3.8 아님) | torch 2.7+는 3.9 이상 필요 |
| 디스플레이 | **xvfb** | CoppeliaSim이 GUI 라이브러리를 요구 |

DreMa 컨테이너 때는 한 줄 패치로 끝났지만, PerAct는 더 클 수 있다. **시간을 넉넉히 잡는다.**

---

## 요약

> **틀린 것 2가지를 바로잡는다** — ① Hydra는 작업 폴더를 바꾸지 않는다(위험 아님) ② 환경은 3개가 아니라 **2개**면 된다.
> **실행은 반드시 `./_setup/run.sh`로.** 컨테이너 tmux에서 돌아 VSCode를 꺼도 살아남는다. 단 Claude 세션은 죽으므로 판단 지점에서는 멈춘다.
> **M2에서 확인할 것 2가지** — 궤적 스텁이 PerAct 학습에 영향을 주는지, 논문 τ가 어느 설정값인지.
> **M2가 가장 불확실하다.** PerAct(2022) + torch 2.7 조합은 전례가 없다.

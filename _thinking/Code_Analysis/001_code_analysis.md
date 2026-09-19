# 001. 코드 분석 — 원본 저장소 정밀 조사

작성: 2026-09-19 (조사 자체는 2026-09-12 수행)
대상: `leobarcellona/drema_code` @ `eca6370`
방법: 엔트리포인트 3개를 기준으로 코드 전수 조사 + 주요 주장은 실행으로 재검증

---

## 사전지식

| 용어 | 뜻 | 비유 |
|---|---|---|
| **엔트리포인트** | 사람이 직접 실행하는 파일 | 건물의 정문 |
| **스텁(stub)** | 껍데기만 있고 내용이 빈 함수 | 문패만 걸린 빈 가게 |
| **죽은 코드** | 어디서도 호출되지 않는 코드 | 아무도 안 쓰는 뒷문 |
| **래스터라이저** | 3D 가우시안을 2D 화면에 그리는 CUDA 프로그램 | 인쇄기 |
| **IK (역기구학)** | 손 위치를 주면 관절 각도를 역산하는 계산 | 목적지를 주면 경로를 찾는 내비 |

---

## 0. 전체 구조

```
create_simulation.py    사진 200장 → 3D 복원 (가우시안·메쉬·URDF)
simulate.py             복원된 장면에서 로봇 궤적 재생 (화면 필요)
generate_new_data.py    장면을 변형해 증강 시범 생성   ← 논문의 핵심
visualize_new_data.py   생성 결과 확인
```

**핵심 설계**: PyBullet이 물리를 계산하고, 가우시안은 그 변환을 따라가는 **껍데기**다.

> [비유] PyBullet은 인형의 **뼈대**, 가우시안은 씌운 **옷**이다. 뼈대가 움직이면 옷을 같은 만큼 움직여 준다. 옷 자체가 물리를 갖지는 않는다.

---

## 1. create_simulation.py — 3D 복원

### 실행 순서
```
데이터 로드 → (옵션) 환경 전체 GS 학습 → labels.txt 파싱
→ 테이블 추출 (depth+mask → RANSAC 평면 → convex hull → URDF)
→ 객체별 반복 [마스크 격리 → GS 재학습 → bbox/DBSCAN/색상 필터
              → PLY 저장 → TSDF 메쉬 → VHACD/URDF]
→ (옵션) 로봇 링크별 GS → 배경에서 객체 제거
```

**중요**: GS 학습은 별도 선행 단계가 아니라 **객체 수 + 1회 매번 처음부터** 돈다. 체크포인트 재사용 경로가 없다.

### 필수 입력
| 경로 | 비고 |
|---|---|
| `images/*.png` | |
| `depth_scaled/*.npy` | 미터 단위 |
| `object_mask/*.png` | **파일 목록의 기준** |
| `poses/*.txt` | ⚠️ **파일명이 정확히 8자여야 함** (`0000.txt` OK, `00001.txt` 무시됨) |
| `labels.txt` | `이름;번호`. 번호 < 60 은 버림 |
| `assets/_prototype*.urdf` | CWD 상대경로로 복사됨 |

- COLMAP `sparse/` 는 **불필요**. 포인트클라우드는 depth 역투영으로 생성한다.
- README는 `object_pose/` 라고 안내하지만 **코드는 `poses/`** 를 읽는다.

### 2DGS / 3DGS 선택 — 설정으로 결정
```
create_simulation.py:70-79
  use_original_guassians=True  → 3DGS  (use_depth ? DepthTrainer : BaseTrainer)
  use_original_guassians=False → 2DGS  (use_depth ? SurfDepthTrainer : SurfTrainer)
```
기본값은 **2DGS + depth**. 키 이름에 오타(`guassians`)가 있다.

---

## 2. simulate.py — 시뮬레이션 실행

### 래스터라이저는 설정이 아니라 **PLY가 결정한다**
```python
# drema/environment/assets/gaussians.py:143
if self.gs.get_scaling.shape[1] == 2:  render_surf(...)   # 2DGS
else:                                  render_depth(...)  # 3DGS
```
PLY 헤더의 `scale_*` 채널 수로 자동 분기한다.

> ⚠️ **2DGS 자산과 3DGS 자산을 섞으면 `torch.cat` 차원 불일치로 즉사한다.**
> 학습을 3DGS로 해놓고 로봇 자산을 기본값(2DGS)으로 두면 이 상황이 된다. **두 설정을 반드시 함께 바꿔야 한다.**

### 로봇 자산은 저장소에 동봉돼 있다
`assets/franka_panda/` (URDF + DAE 46개), `assets/robot_surf_gaussians/`(2DGS 11개), `assets/robot_original_gaussians/`(3DGS 11개). **재학습 불필요.**

---

## 3. generate_new_data.py — 증강 (논문 핵심)

### 증강은 정확히 3종뿐
| 종류 | 개수 | 동작 |
|---|---|---|
| 환경 평행이동 | 8 | 궤적 + 물체 + 배경 가우시안을 xy 이동 (z 고정) |
| 환경 회전 | 11 | 고정 중심 기준 z축 회전. **테이블도 회전** |
| 물체 회전 | 17 | 궤적 마지막 위치 기준 회전. 테이블·배경 고정 |

**합계 최대 37개** (원본 1 + 36).

> ❗ **"물체별 개별 변형"이나 "새 물체 배치"는 구현돼 있지 않다.** 세 증강 모두 전체 물체에 **동일한 강체 변환**을 적용한다.

### 검증 장치
증강 후 시뮬레이션 결과와 해석적 예측을 비교해, 오차가 `generation.threshold`(기본 0.035 m) 이내일 때만 채택한다.

### radius filter는 살아 있다
README는 "논문은 radius filter, 여기서는 Scharr"라고 하지만 **둘 다 있고 설정으로 전환 가능**하다.
```yaml
simulation.output.radius_filter: False   # True 로 바꾸면 논문 조건
```
단 `nb_points=16, radius=0.05`가 하드코딩이라 조정 불가이고, 설정의 `threshold`는 **Scharr 전용**이다.

### 속도/헤드리스 플래그
`simulation.visualization.visualize: False` 하나로 세 가지가 꺼진다 — PyBullet GUI, 시각화 카메라 로드, **매 스텝 추가 렌더링**(속도 차이의 주원인). README가 말한 "시각화를 피하면 빨라진다"가 이것이다.

---

## 4. 미구현·고장 — 재현에 영향 있는 것

| # | 항목 | 영향 | 상태 |
|---|---|---|---|
| 1 | `update_trajectory_with_current_data()` **스텁** | 증강 궤적의 `joint_positions`·손목카메라가 **원본 그대로**. `gripper_pose`만 변환 | 002에서 수치 확정 |
| 2 | **손목 카메라 렌더링 고장** | CUDA에 `near_n = 0.2` 하드코딩 → **20cm 이내 가우시안 전부 컬링**. 파이썬 `znear=0.0001`은 무시됨 | 서브모듈 재컴파일 없이 수정 불가 |
| 3 | **`lambda_normal` 미발동** | `iteration > 7000` 조건인데 총 `iterations=7000` → 2DGS normal 정규화가 한 번도 안 켜짐 | 복원 품질 영향 (우리는 복원을 건너뛰므로 현재 무관) |
| 4 | **SH 색상 미회전** | 회전 증강 시 빛 반사 방향이 안 따라 돎 → 광택이 어색 | 실행은 정상 |
| 5 | `simulate.py` **헤드리스 불가** | `pynput`이 import 시점에 X 디스플레이 요구 → 아무것도 시작 전에 죽음 | `generate_new_data.py`는 무관 |

### 2번 상세 — 손목 카메라
```c
// submodules/diff-surfel-rasterization/cuda_rasterizer/auxiliary.h:37
__device__ const float near_n = 0.2;
// forward.cu:360
if (depth < near_n) continue;              // 20cm 이내 스킵
// 3DGS 경로: auxiliary.h:154  if (p_view.z <= 0.2f) ... return false;
```
손목 카메라는 그리퍼에서 **4cm** 거리에 붙어 있고 조작 대상도 20cm 이내다. 그래서 화면이 비어 보인다. **논문이 카메라 3대만 쓴 이유가 이것이다.**

---

## 5. 버그 목록

| 항목 | 증상 |
|---|---|
| `objects_list != ["All"]` | 순회 중 `del` → `RuntimeError: dictionary changed size during iteration`. **객체 부분 선택 기능은 사용 불가** |
| `simulate.py` 도달 실패 미처리 | waypoint 실패(-1)를 `== 1`로만 검사 → 도달 불가 지점에서 **무한 정지**. `generate_new_data.py`는 `!= 0`으로 올바름 |
| `q` 키 | README는 "q로 종료"라지만 `simulate.py`는 **ESC**. `generate_new_data.py`에서 q를 누르면 `None` 반환 → **TypeError 크래시** |
| intrinsics 제자리 변형 | `camera.py:160`이 원본 궤적 배열을 복사 없이 변형 → 조건에 따라 scale이 중복 적용 |
| 평행이동 비대칭 | 테이블은 안 움직이는데(`translate_table=False`) 배경 가우시안은 움직임. 회전은 테이블도 회전 — 의도인지 불명 |
| 자체 데이터 불가 | `save_images`가 `low_dim_obs.pkl` 등 3종을 **무조건 복사** → README대로 만든 자체 데이터는 `FileNotFoundError` |
| 에피소드 번호 | `episode += 1000`을 성공 여부와 무관하게 실행 → "1000번대=환경회전"이 보장되지 않음 |

---

## 6. 죽은 코드 — 신경 쓰지 말 것

엔트리포인트 4개에서 정적 import 그래프를 따라간 결과, **도달 가능한 깨진 import는 없다.**

| 파일 | 문제 | 도달? |
|---|---|---|
| `drema_scene/txt_loader.py` | `gaussin_splatting_utils` (오타), `scene.colmap_loader` | ❌ 죽음 |
| `r2s_builder/data_manager.py` | `frame_manager` | ❌ 죽음 |
| `gaussian_renderer/*/network_gui.py` | `scene.cameras` | ❌ 죽음 |
| `gaussian_splatting_utils/mesh_utils.py:264` | `utils.mcube_utils` — **파일 자체가 없음** | ❌ `extract_mesh_unbounded()` 안, 호출처 없음 |
| `r2s_builder/extractors/gaussians_extractor.py` | `class GaussianBuilder: pass` 완전 스텁 | ❌ |

**죽은 설정 구간**: `coppelia_params.yaml`의 `preparation:`(1-9행)과 `reference_frame:`(26-30행) **전체가 미사용**이다. 유일한 소비자인 `data_manager.py`가 고아 모듈이다. → **설정 파일만 보고 동작을 추정하면 오해한다.**

---

## 7. 순환 참조 — 실제로 터진다

`drema/scene/__init__.py` ↔ `gaussian_splatting_utils/camera_utils.py`가 서로를 import한다. **어느 쪽에서 먼저 들어가느냐**에 따라 성공/실패가 갈린다.

```python
# ❌ 실패
from drema.environment.observer.camera import CameraManager

# ✅ 성공 — 엔트리포인트와 같은 순서로 먼저 진입
import drema.environment.builder
from drema.environment.observer.camera import CameraManager
```

엔트리포인트 실행에는 문제가 없지만, **직접 스크립트를 짤 때 반드시 걸린다.**

---

## 8. 환경 관련

### requirements.txt 누락
코드가 쓰는데 목록에 없다: **`scipy`(9개 파일), `PIL`(7개), `tqdm`(3개)**.
반대로 `trimesh==3.10`은 목록에 있으나 직접 import하는 코드가 없다(`object2urdf`의 의존성).

`setuptools==49.4` 핀은 **CUDA 확장 빌드를 깨뜨리므로** 빌드 후에 설치하거나 제외해야 한다.

### 개인 경로 하드코딩
`configs/config_real.yaml:2`, `visualize_new_data.py:13-14`, `drema/environment/observer/camera.py:340` 에 `/home/leonardo/...` 가 남아 있다.

### 원본 저장소의 결함 (수정 완료)
| 문제 | 조치 |
|---|---|
| `diff-surfel-rasterization/third_party/glm` 비어 있음 | 형제 서브모듈과 동일한 glm 0.9.9.9로 채움 |
| `simple_knn.cu`가 nvcc 12.8에서 `FLT_MAX` 미정의 | `#include <float.h>` 추가 |

커밋 `f679399` 참조.

---

## 9. 오해였던 것 — 재조사하지 말 것

| 의혹 | 실제 |
|---|---|
| **Hydra가 작업 폴더를 바꿔 상대경로가 깨진다** | ❌ **실측 결과 바꾸지 않는다.** `outputs/` 폴더만 만든다 |
| 오브젝트 URDF 출력 경로 불일치 | ❌ `object2urdf`가 `object_folder/<name>.urdf`에 쓰므로 `builder.py:147`의 기대와 **일치한다** |
| 깨진 import가 실행을 막는다 | ❌ 전부 죽은 코드다 |

---

## 요약

> **구조**: PyBullet이 물리, 가우시안은 껍데기. 복원(create_simulation) → 증강(generate_new_data) 2단계가 핵심이다.
> **가장 중요한 함정 3가지**: ① 궤적 갱신 스텁(관절각 미변환) ② 손목 카메라 CUDA 하드코딩 ③ 2DGS/3DGS 혼용 시 즉사.
> **재현에 유리한 점**: 로봇 자산 동봉으로 재학습 불필요, radius filter 선택 가능, COLMAP 불필요.
> **주의**: `poses/` 파일명 8자 규칙, 설정의 `preparation`·`reference_frame` 구간은 죽은 코드, 직접 스크립트 작성 시 순환 참조.
> **Hydra chdir 문제는 존재하지 않는다.** 실측으로 확인했다.

# open-rmf-test

Open-RMF(Robotics Middleware Framework) 연습용 저장소.
오피스 데모(`office.launch.xml`)를 띄우고 로봇에 태스크를 보내보는 공부 기록.

## 환경
- Ubuntu 24.04 (noble)
- ROS 2 **Jazzy**
- Gazebo (gz)

## 이 repo에 뭐가 들어있나
- 이 README (셋업/실행/패키지 구조)
- [`트러블슈팅.md`](./트러블슈팅.md) — 빌드/실행 중 겪은 문제와 해결법
- `.gitignore`
- ⚠️ `rmf_ws/` 는 **git에 안 올라감** (`.gitignore` 처리됨).
  남의 레포(rmf_demos)를 받아온 외부 소스 + colcon 빌드 산출물이라 재생성 가능 → 추적 안 함.
  아래 순서대로 누구나 다시 만들 수 있음.

---

## 1. 셋업 (최초 1회)

### 1-1. 빌드 도구 + 코어 RMF 설치 (apt)
```bash
sudo apt update
sudo apt install -y \
  ros-jazzy-rmf-dev \
  python3-colcon-common-extensions \
  python3-vcstool \
  python3-rosdep \
  git
```
> 오피스 월드 패키지(`rmf_demos`, `rmf_demos_gz`, `rmf_demos_maps`)는 Jazzy용 apt 바이너리가 없어서
> **소스 빌드**가 필요함. 코어/시뮬 플러그인은 apt에 있어서 빌드는 가벼움.

### 1-2. rosdep 초기화
```bash
sudo rosdep init   # 이미 했으면 "already exists" 무시
rosdep update
```

### 1-3. 워크스페이스 만들고 데모 클론 (jazzy 브랜치)
```bash
mkdir -p rmf_ws/src
cd rmf_ws/src
git clone https://github.com/open-rmf/rmf_demos.git -b jazzy
cd ../..
```

### 1-4. 의존성 설치 + 빌드
```bash
source /opt/ros/jazzy/setup.bash
cd rmf_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build
cd ..
```

### 1-5. 로봇 모델 경로 패치 (⚠️ 필수 — 안 하면 Gazebo가 바로 꺼짐)
월드 파일은 로봇을 `model://Open-RMF/TinyRobot`로 찾는데, 로컬 모델엔 `Open-RMF/` 접두사가
없어서 못 찾는다. `~/.gazebo/models/Open-RMF/` 아래로 링크해두면 해결됨 (오피스 런처가 `~/.gazebo/models`를 리소스 경로에 자동 추가하므로).
```bash
mkdir -p ~/.gazebo/models/Open-RMF
for m in ~/open-rmf-test/rmf_ws/src/rmf_demos/rmf_demos_assets/models/*; do
  ln -sfn "$m" ~/.gazebo/models/Open-RMF/"$(basename "$m")"
done
```

---

## 2. 오피스 데모 실행
```bash
source rmf_ws/install/setup.bash   # ⚠️ /opt/ros 가 아니라 빌드한 워크스페이스를 source!
ros2 launch rmf_demos_gz office.launch.xml
```
Gazebo에 오피스 맵 + 로봇, RViz 시각화가 같이 뜸. (첫 실행은 에셋 다운로드로 느릴 수 있음)

> 참고: `server_uri:=ws://localhost:7878` 인자는 **붙이지 말 것.** rmf-web 대시보드용인데
> 대시보드를 안 띄우면 접속 에러만 남긴다 (데모 동작엔 무관한 무해한 에러).

## 3. 로봇에 태스크 보내기 (다른 터미널)
```bash
source rmf_ws/install/setup.bash

# 순찰(patrol)
ros2 run rmf_demos_tasks dispatch_patrol -p pantry hardware_2 -n 3

# 배달(delivery)
ros2 run rmf_demos_tasks dispatch_delivery -p pantry -ph coke_dispenser -d hardware_2 -dh coke_ingestor
```

---

## 4. 직접 만든 맵을 Gazebo로 열기 (traffic_editor 맵 → world → Gazebo)

`traffic_editor`로 그린 맵(`<이름>.building.yaml`)을 데모와 별개로 Gazebo에 직접 띄워보는 흐름.
(아래 예시는 `asd.building.yaml` → `workspace.world` 기준)

### 4-1. yaml → .world 변환
```bash
source /opt/ros/jazzy/setup.bash
cd rmf_ws
ros2 run rmf_building_map_tools building_map_generator gazebo \
  asd.building.yaml \   # 입력: traffic_editor가 저장한 building yaml
  workspace.world \     # 출력: Gazebo world 파일
  .                     # 출력: 층 모델(asd_L1 등)을 생성할 폴더
```
- 인자 순서: `building_map_generator gazebo <입력.building.yaml> <출력.world> <모델출력폴더>`
- 명령어 이름은 `building_map_generator` (gen**e**rator — 오타 주의).
- 세 번째 `.` = 현재 폴더에 **`<건물명>_<층명>`** 모델 폴더(예: `asd_L1/`)를 만든다.
  - 모델 이름은 파일명이 아니라 yaml 안의 `name:`(건물명) + 층 키(`L1`)에서 나온다.
- world는 3D 형상을 직접 담지 않고 `model://asd_L1`로 그 폴더를 **참조**만 한다.

### 4-2. .world → Gazebo 열기
world가 참조하는 층 모델(`asd_L1`)과 가구 모델(`Sofa` 등)의 위치를
`GZ_SIM_RESOURCE_PATH`로 알려줘야 한다 (⚠️ 안 하면 빈 화면 / `model not found` 경고).
```bash
source /opt/ros/jazzy/setup.bash
export GZ_SIM_RESOURCE_PATH=$HOME/open-rmf-test/rmf_ws:$HOME/.gazebo/models
gz sim ~/open-rmf-test/rmf_ws/workspace.world
```
- `$HOME/open-rmf-test/rmf_ws` → `asd_L1`을 찾는 경로 (모델 폴더의 **부모**를 넣는다)
- `$HOME/.gazebo/models` → `Sofa` 같은 공용 가구 모델 경로

> 매번 두 줄 치기 귀찮으면 스크립트로 묶어두면 된다 (예: `rmf_ws/open_world.sh` →
> `bash rmf_ws/open_world.sh` 한 줄로 실행). `rmf_ws/`는 git에 안 올라가니 스크립트도 로컬용.

> Classic 시절 `sudo cp -r 모델 /usr/share/gazebo-11/models` 방식은 **새 Gazebo(gz-sim)엔 안 통한다.**
> 모델을 시스템에 복사하는 대신 위처럼 `GZ_SIM_RESOURCE_PATH`로 경로를 알려주거나,
> `~/.gazebo/models/`(Gazebo가 기본으로 뒤지는 폴더)에 두면 된다.

---

## src/rmf_demos 패키지 구조 (참고)

`rmf_ws/src/rmf_demos`(클론한 데모 소스)는 여러 ROS 2 패키지로 구성됨:

| 패키지 | 역할 |
|---|---|
| `rmf_demos` | 데모 **공통 launch + fleet/태스크 설정** (office.launch.xml 등의 베이스) |
| `rmf_demos_gz` | **새 Gazebo(Harmonic)** 시뮬레이션 launch ← **우리가 쓰는 것** |
| `rmf_demos_gz_classic` | Gazebo Classic용 launch (Jazzy에선 Classic이 EOL이라 **빌드 안 됨**) |
| `rmf_demos_maps` | 데모 맵(`.building.yaml`) → world/nav graph 생성 |
| `rmf_demos_assets` | 3D 모델 (로봇·가구·디스펜서 등) |
| `rmf_demos_tasks` | 태스크 디스패치 스크립트 (`dispatch_patrol`/`delivery`/`clean`) |
| `rmf_demos_fleet_adapter` | 데모 로봇용 fleet adapter + REST API fleet manager |
| `rmf_demos_dashboard_resources` | rmf-web 대시보드 리소스 |
| `rmf_demos_panel` | 웹 기반 태스크 제출 패널 |
| `rmf_demos_bridges` | 통신 스택 간 브리지 노드 |
| `docs/` | 추가 문서 (`faq.md`, `ignition.md`, `secure_office_world.md`) |

> **브랜치 주의**: 클론한 건 **`jazzy` 브랜치**다. rmf_demos README엔
> "Ubuntu 22.04 / ROS 2 Humble / Gazebo Fortress"라고 적혀 있지만, 그건 README 본문이
> 갱신 안 된 것뿐이고 **`jazzy` 브랜치가 Jazzy + Gazebo Harmonic용**이라 정상 동작함
> (실제로 빌드·실행 확인 완료).

---

## 메모
- **환경 source 규칙**: 새 터미널마다 `source rmf_ws/install/setup.bash` 필요.
  (이게 ROS의 환경 격리 방식 — Python venv는 ROS와 충돌하므로 쓰지 않음.)
- 맵 구조가 궁금하면 `traffic-editor`(apt: `ros-jazzy-rmf-traffic-editor`)로 오피스 맵을 열어볼 것.
- 실배포 시엔 데모 대신 **코어(`ros-jazzy-rmf-dev`) + 직접 만든 맵 + fleet adapter**가 필요함.

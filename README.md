# open-rmf-test

Open-RMF(Robotics Middleware Framework) 연습용 저장소.
오피스 데모(`office.launch.xml`)를 띄우고 로봇에 태스크를 보내보는 공부 기록.

## 환경
- Ubuntu 24.04 (noble)
- ROS 2 **Jazzy**
- Gazebo (gz)

## 이 repo에 뭐가 들어있나
- 이 README (셋업/실행 방법)
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
없어서 못 찾는다. `~/.gazebo/models/Open-RMF/` 아래로 링크해두면 해결됨 (자세한 이유는 아래 트러블슈팅).
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
> 대시보드를 안 띄우면 접속 에러만 남긴다 (아래 트러블슈팅 참고).

## 3. 로봇에 태스크 보내기 (다른 터미널)
```bash
source rmf_ws/install/setup.bash

# 순찰(patrol)
ros2 run rmf_demos_tasks dispatch_patrol -p pantry hardware_2 -n 3

# 배달(delivery)
ros2 run rmf_demos_tasks dispatch_delivery -p pantry -ph coke_dispenser -d hardware_2 -dh coke_ingestor
```

---

## 문제 해결 (Troubleshooting)

실제로 겪은 문제들과 해결법. (Ubuntu 24.04 + Jazzy 기준)

### ① Gazebo가 열렸다가 바로 꺼짐 — `Unable to find uri[model://Open-RMF/TinyRobot]`
- **증상**: 런처 로그 끝에
  `[Err] Error Code 14: ... Msg: Unable to find uri[model://Open-RMF/TinyRobot]` 가 뜨고
  `process has finished cleanly` 로 gz가 종료됨. (그래픽 문제 아님 — RViz는 OpenGL 정상)
- **원인**: 월드 파일(apt `rmf_building_map_tools`가 생성)은 로봇을 **`model://Open-RMF/TinyRobot`**
  (Fuel 스타일, `Open-RMF/` 접두사)로 참조하는데, 소스 `rmf_demos_assets`(jazzy)는 모델을
  접두사 없이 `models/TinyRobot`로 제공함 → 경로 한 단계가 안 맞아 못 찾음 (버전 불일치).
- **해결**: 오피스 런처가 `GZ_SIM_RESOURCE_PATH`에 `~/.gazebo/models`를 자동 추가하므로,
  거기에 `Open-RMF/` 네임스페이스를 만들고 로컬 로봇 모델을 링크한다 (= 1-5 단계).
  ```bash
  mkdir -p ~/.gazebo/models/Open-RMF
  for m in ~/open-rmf-test/rmf_ws/src/rmf_demos/rmf_demos_assets/models/*; do
    ln -sfn "$m" ~/.gazebo/models/Open-RMF/"$(basename "$m")"
  done
  ```
  검증: `ls ~/.gazebo/models/Open-RMF/TinyRobot/model.config` 가 나오면 OK.

### ② `Connection to ws://localhost:7878 failed` 에러가 계속 뜸
- **원인**: 실행 시 `server_uri:=ws://localhost:7878` 를 붙이면 rmf-web 대시보드에 접속을
  시도하는데, 대시보드를 안 띄웠으니 실패함.
- **해결**: 대시보드를 안 쓰면 **`server_uri` 인자를 빼고** 실행. (데모 동작엔 무관한 무해한 에러)

### ③ `package 'rmf_demos_gz_classic' not found`
- **원인**: Jazzy에서는 **Gazebo Classic이 단종**돼서 `rmf_demos_gz_classic`을 안 빌드함.
- **해결**: 새 Gazebo용 **`rmf_demos_gz`** 를 사용. (옛 튜토리얼의 `_classic` 명령어 주의)

### ④ (참고) Wayland 세션
- gz Harmonic이 Wayland를 감지하면 **자동으로 `QT_QPA_PLATFORM=xcb`** 를 설정하므로
  Wayland여도 GUI는 정상. ("열렸다 꺼짐"의 원인은 Wayland가 아니라 위 ①번임.)

---

## 메모
- **환경 source 규칙**: 새 터미널마다 `source rmf_ws/install/setup.bash` 필요.
  (이게 ROS의 환경 격리 방식 — Python venv는 ROS와 충돌하므로 쓰지 않음.)
- 맵 구조가 궁금하면 `traffic-editor`(apt: `ros-jazzy-rmf-traffic-editor`)로 오피스 맵을 열어볼 것.
- 실배포 시엔 데모 대신 **코어(`ros-jazzy-rmf-dev`) + 직접 만든 맵 + fleet adapter**가 필요함.

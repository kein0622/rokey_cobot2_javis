# 🤖 JARVIS — 시니어 보조 협동로봇

ROS2 + WebRTC 기반 음성·제스처·비전 통합 협동로봇 제어 시스템

> 본 레포지토리는 팀 프로젝트 **JARVIS** 를 정리한 개인 포트폴리오 버전입니다.
> 본인이 담당한 **상태 제어 / 예외 처리, 작업공간 스캔, 손동작(MediaPipe) 제어** 파트를 중심으로 기술합니다.

---

## 📌 프로젝트 배경

대한민국은 빠르게 고령화 사회로 진입하고 있으며, 요양 인력 부족 문제가 심화되고 있습니다.
JARVIS는 이러한 문제를 해결하기 위해 시니어의 일상을 보조하는 협동로봇 시스템으로 개발되었습니다.

단순한 물건 전달을 넘어, **RAG(Retrieval-Augmented Generation)** 기반으로 개별 사용자의 건강 상태와 현재 상황을 파악하여 맞춤형 음식이나 물건을 추천합니다. 예를 들어 당뇨 환자에게는 혈당에 맞는 음식을, 외출 전에는 날씨에 맞는 준비물을 스스로 판단해 제안합니다.

**핵심 기능 요약**
- 음성 명령 / 브라우저 제스처 / 손동작으로 물건 픽앤플레이스
- RAG 기반 사용자 건강 상태 인식 → 상황에 맞는 물건 추천
- 카메라 + VLM으로 주변 환경 스캔 후 자동 픽
- 로봇 이상 상태 감지 및 자동 복구

---

## 🔗 링크

| 항목 | 링크 |
|---|---|
| 팀 원본 레포지토리 | [https://github.com/junsu122/Rokey_Cobot2](https://github.com/junsu122/Rokey_Cobot2) |
| 전체 시연영상 | [▶ JARVIS 전체 시연영상 보기](https://youtu.be/0AUDXOjfGCY) |

---

## 🛠️ 활용 장비 및 재료
![활용재료](assets/활용재료.png)
<p align="center">&lt;활용재료&gt;</p>

---

## 🔄 전체 흐름
![전체 아키텍쳐](<./assets/architecture.png>)
<p align="center">&lt;전체 아키텍처&gt;</p>

![전체 플로우](<./assets/flow_chart.png>)
<p align="center">&lt;전체 플로우차트&gt;</p>

---

## 🙋 내 주요 업무

### 1. [손동작(MediaPipe) 기반 로봇 제어 — `gesture_robot_pkg`]

웹캠 영상에서 MediaPipe로 손을 추적하여, 손동작만으로 협동로봇을 직접 제어하는 텔레오퍼레이션 기능을 개발했습니다.

> **동작 로직**
> 1. 웹캠에서 MediaPipe Hand Landmark(21개) 추적 → Index-tip / Palm-center 추출
> 2. `WAITING → CALIBRATING → CONTROLLING ↔ PAUSED` 상태 머신으로 제어 흐름 관리
> 3. 캘리브레이션 시점의 TCP를 기준점으로, 손 이동량(delta)을 로봇 목표 좌표로 변환
> 4. 주먹(fist)/검지 포인팅 등 제스처로 제어 시작·일시정지 전환
> 5. `move_line` 서비스를 호출해 손 움직임을 로봇 동작으로 실시간 매핑 (~30 Hz)

![MediaPipe를 이용한 M0609 제어](./assets/손으로움직이기.gif)
<p align="center">&lt;MediaPipe를 이용한 손동작 제어&gt;</p>

---

### 2. [작업공간 ㄹ자 스캔 알고리즘 — `jarvis_voice_pkg / workspace_scan_node`]

사용자가 "배고파" 등 추천을 요청하면, 로봇이 테이블 전체를 훑으며 물체를 탐색하는 스캔 기능을 개발했습니다.

> **동작 로직**
> 1. `/scan_request` 수신 → 스캔 시작
> 2. 작업공간을 격자(nx × ny)로 나눠 **ㄹ자(지그재그) 경로**로 순차 이동
> 3. 각 스캔 포즈에서 RealSense 컬러 이미지를 캡처하여 VLM(GPT-4o Vision)으로 물체 탐지
> 4. 타겟 물체 발견 여부를 `/scan_result` 로 발행, 진행 상태를 `/scan_status` 로 실시간 발행
> 5. 취소 요청 시 즉시 중단

테이블 전 영역을 한 시야에 담기 어려워, **지그재그 경로로 카메라를 분할 이동**시키며 부분 시야를 누적해 전체를 커버하도록 설계했습니다.

---

### 3. [상태 제어 / 외력감지·비상정지·복구 — `robot_state_control` (초기 설계·프로토타입)]

로봇의 이상 상태를 감지하고 동작을 안전하게 멈췄다가 복구하는 상태 제어 파이프라인의 **초기 형태를 설계·구현**했습니다.

> **평상시** : `main_node` 가 픽앤플레이스 전체 흐름(START/PAUSE/RESUME/STOP)을 관리
>
> **이상 감지 시** : `emergency_stop_node` 가 로봇 상태를 주기적으로 감시하다가
> 외력감지(Protective Stop, state_code=5) / 비상정지(Emergency Stop, state_code=6) 발생 시
> `/exception/event` 로 이상 신호를 발행 → `main_node` 가 동작을 일시정지/중단
>
> **복구 시** : 웹 대시보드 버튼 또는 손바닥(MediaPipe) 제스처로 복구 신호 전송 →
> `exception_manager_node` 가 로봇에 복구 명령(`/dsr01/recovery_command`)을 전달하고
> `main_node` 에 정상 복귀 신호를 보내 픽앤플레이스 동작 재개

![예외처리 플로우](./assets/exception_flow.png)
<p align="center">&lt;외력감지 / 비상정지 복구 플로우&gt;</p>

<table>
  <tr>
    <td align="center"><b>외력감지(Protective Stop) - 웹</b></td>
    <td align="center"><b>외력감지 - 카메라</b></td>
  </tr>
  <tr>
    <td><img src="./assets/예외처리.gif"/></td>
    <td><img src="./assets/예외처리camera.gif"/></td>
  </tr>
  <tr>
    <td align="center"><b>비상정지(Emergency Stop) - 웹</b></td>
    <td align="center"><b>비상정지 - 카메라</b></td>
  </tr>
  <tr>
    <td><img src="./assets/이스탑.gif"/></td>
    <td><img src="./assets/이스탑camera.gif"/></td>
  </tr>
</table>

---

## 🔧 Trouble Shooting

### [스캔 시 거리에 따른 YOLO 인식률 저하]

**증상**
> 테이블을 ㄹ자로 스캔할 때, 카메라와 물체 사이 거리가 멀어지면 YOLO 객체 인식이 제대로 되지 않음.

**원인**
> 한 시야에 테이블 전체를 담으려고 카메라를 멀리 두면, 물체가 화면에서 작아져 YOLO가 인식하지 못함.

**해결 방법**
> 테이블을 한 번에 보지 않고, **작업공간을 격자로 분할하여 지그재그(ㄹ자) 경로로 가까이서 순차 스캔**하도록 변경.
> 각 구간을 근거리에서 촬영해 인식률을 확보하고, 구간별 결과를 누적해 전체 작업공간을 커버.

---

## 📡 주요 ROS2 토픽

| 토픽 | 설명 |
|---|---|
| `/gesture_event` | MediaPipe 손 랜드마크(21개) 및 제스처 이벤트 |
| `/teleop/enable` | 손동작 텔레오퍼레이션 활성화 |
| `/scan_request` | 작업공간 스캔 시작/취소 명령 |
| `/scan_result` | 스캔 결과 (found / not_found) |
| `/scan_status` | 스캔 진행 상태 실시간 발행 |
| `/exception/event` | 외력감지·비상정지 등 예외 발생 알림 |
| `/exception/resolved` | 예외 해소(복구 완료) 신호 |
| `/dsr01/robot_state_summary` | 로봇 상태 요약 (모니터링용) |
| `/dsr01/recovery_command` | 복구 명령 |

---

## 📁 주요 파일 경로 (담당 파트)

| 파일 | 경로 |
|---|---|
| 손동작 텔레오퍼레이션 | `src/gesture_robot_pkg/gesture_robot_pkg/webcam_teleop_node.py` |
| 작업공간 스캔 | `src/jarvis_voice_pkg/jarvis_voice_pkg/workspace_scan_node.py` |
| 비상정지 감지 | `src/robot_state_control/robot_state_control/emergency_stop_node.py` |
| 예외 관리 | `src/robot_state_control/robot_state_control/exception_manager_node.py` |
| 손바닥 복구 | `src/robot_state_control/robot_state_control/hand_recovery.py` |
| 메인 오케스트레이션 | `src/robot_state_control/robot_state_control/main_node.py` |

---

## 💻 구동 방법 (로컬 버전)

```bash
# 1. 빌드
cd ~/Project_Rokey_Cobot2
colcon build
source install/setup.bash

# 2. 센서 연동 (RealSense 및 dsr01 연결)

# 3. 메인 컨트롤러 런치
ros2 launch gesture_robot_pkg gesture_robot.launch.py

# 4. 상태 컨트롤러 런치
ros2 launch robot_state_control robot_state_control.launch.py

# 5. 객체 너비 확인 노드 런치
ros2 launch object_width_filter object_width_filter.launch.py
```

> 의존성 설치: `pip install -r requirements.txt` (ROS2 Humble 환경)

---

## 🎬 전체 시연영상

[▶ JARVIS 전체 시연영상 보기](https://youtu.be/0AUDXOjfGCY)

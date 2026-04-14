# 🤖 모바일 매니퓰레이터 — 장애물 감지 및 자율 제거 로봇

> TurtleBot3 Waffle + SO-100 로봇팔을 결합한 자율주행 기반 장애물 제거 시스템

---

## 📌 프로젝트 개요

본 프로젝트는 임베디드 소프트웨어 전공 졸업작품으로, **자율주행 로봇(TurtleBot3 Waffle)** 위에 **6축 로봇팔(SO-100)** 과 **뎁스 카메라(RealSense D435i)** 를 탑재하여 주행 중 전방 장애물을 스스로 감지하고 제거하는 **모바일 매니퓰레이터** 시스템을 구현합니다.

## 🎯 핵심 시나리오

```
Waffle 자율주행 중
→ LiDAR로 전방 장애물 감지 → Waffle 자동 정지
→ RealSense + YOLOv8으로 장애물 3D 위치 파악
→ SO-100이 장애물을 집어 측면으로 치움
→ LiDAR 재스캔으로 경로 클리어 확인
→ Waffle 주행 재개
```

## 🗂 시스템 아키텍처
<img width="827" height="893" alt="image" src="https://github.com/user-attachments/assets/2e57bded-bff2-4ad3-bb19-1e7c9bef117e" />

![시스템 블록도](blockdiagram.png)

## 🛠 기술 스택

| 분류 | 기술 |
|---|---|
| 운영체제 | Ubuntu 24.04 (WSL2) |
| 미들웨어 | ROS2 Jazzy |
| 자율주행 | Nav2, SLAM (Cartographer), AMCL |
| 물체 인식 | YOLOv8 (커스텀 학습), RealSense D435i |
| 로봇팔 제어 | MoveIt2, Feetech SDK |
| 시각화 | RViz2 |

## 🔧 하드웨어 구성

| 부품 | 역할 |
|---|---|
| TurtleBot3 Waffle | 자율주행 모바일 베이스 |
| SO-100 로봇팔 | 장애물 파지 및 제거 |
| Intel RealSense D435i | RGB + Depth 장애물 인식 |
| LiDAR | 전방 경로 차단 감지 |
| Raspberry Pi 4 | Waffle 온보드 컴퓨터 |
| Desktop PC (GPU) | YOLOv8 추론, ROS2 노드 실행 |

## 🚀 개발 진행 순서

- [x] ROS2 Jazzy 개발환경 세팅 (WSL2 Ubuntu 24.04)
- [ ] TurtleBot3 Waffle ROS2 연동 확인
- [ ] SO-100 서보 연결 및 제어
- [ ] RealSense 드라이버 + 3D 역투영 구현
- [ ] YOLOv8 장애물 커스텀 학습
- [ ] MoveIt2 IK + grasp_planner 구현
- [ ] 모드 매니저 자동 전환 구현
- [ ] 전체 통합 테스트

## 👤 개발자

- **park-taemin** — 임베디드 소프트웨어 전공
- GitHub: [https://github.com/park-taemin/capstone-design](https://github.com/park-taemin/capstone-design)

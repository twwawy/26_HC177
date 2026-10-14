# 지형 적응 보행 관련 모듈

## STM32

`SW/STM32/workspace/Hexapod/Core/` 아래에 실시간 제어 코드를 둔다.
헤더는 `Inc/`, 구현은 `Src/`에 같은 분류로 배치한다.

| 모듈 | 역할 |
|---|---|
| `app/hexapod_app` | 센서·통신·보행 제어 실행 주기 연결 |
| `common/robot_config`, `common/robot_types` | 공통 상수와 상태·계획 자료형 |
| `communication/adaptive_spi_protocol` | 128바이트 실행·관측 프레임 코덱 |
| `communication/jetson_spi` | SPI DMA와 수신 처리 |
| `high_control/rl_controller` | 계획 승인, 관측 유효 시간, 명령 타임아웃 |
| `high_control/gait_manager` | 보행 위상과 Tripod·Wave 전환 |
| `high_control/foot_trajectory`, `high_control/swing_trajectory` | 실행 계획을 발끝 궤적으로 변환 |
| `high_control/workspace_limiter` | 발 작업영역과 관절 한계 처리 |

## Jetson

| 파일 | 역할 |
|---|---|
| `SW/Jetson/adaptive_runtime.py` | 점군과 상태 추정값을 받아 실행 계획 계산 |
| `SW/Jetson/adaptive_policy_contract.py` | 정책 입출력 규격 |
| `SW/Jetson/adaptive_execution_plan.py` | 최종 실행 계획 생성 |
| `SW/Jetson/adaptive_spi_protocol.py` | SPI 프레임 코덱 |
| `SW/Jetson/make_offline_fixture.py` | 오프라인 입력 데이터 생성 |

## MJX 시뮬레이션·학습

| 파일·폴더 | 역할 |
|---|---|
| `mjx/adaptive_contract.py` | 제어·정책 버전과 공통 규격 |
| `mjx/adaptive_foothold_estimator.py`, `mjx/foothold_feasibility.py` | 착지 후보와 실행 가능성 계산 |
| `mjx/adaptive_gait_controller.py`, `mjx/wave_gait_scheduler.py` | 발 궤적과 보행 순서 |
| `mjx/hybrid_gait_supervisor.py` | Tripod·짧은 보폭·Wave·정지 판단 |
| `mjx/adaptive_gait_perception.py`, `mjx/adaptive_grid.py` | 지형 관측과 지역 높이 지도 |
| `mjx/adaptive_grid_network.py`, `mjx/adaptive_gait_policy.py` | 정책 네트워크와 체크포인트 |
| `mjx/adaptive_gait_env.py` | 학습 환경, 관측과 보상 |
| `mjx/train_adaptive_curriculum.py`, `mjx/train_adaptive_gait.py` | 단계별 학습과 PPO |
| `mjx/view_foothold_planner.py`, `mjx/view_adaptive_gait.py` | 보행·착지 후보 뷰어 |
| `mjx/policies/` | 보관된 학습 정책과 출처 기록 |
| `mjx/tests/` | 제어 계약과 모듈 시험 코드 |

실행 스크립트는 `scripts/`, 사용법과 규격은 [개발 문서 안내](README.md)에 있다.

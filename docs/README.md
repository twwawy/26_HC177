# 개발 문서

로봇 구성은 루트 [README](../README.md), 실행 옵션과 제어 규격은 아래 문서를 참고한다.

## 하드웨어와 실시간 제어

| 문서 | 내용 |
|---|---|
| [하드웨어 구성](../HW/README.md) · [부품 목록](../HW/parts.md) | 기구 제작 자료와 주요 부품 |
| [STM32 설정](../SW/STM32/STM32F446RE%20설정%20정리본.md) | 핀 배치와 주변장치 설정 |
| [STM32 코드 구조](../SW/STM32/STM32%20코드%20구조%20정리본.md) | 제어·센서·통신 모듈 |
| [Tripod·Wave 조작](../SW/STM32/WAVE_GAIT.md) | S1 보행 선택과 한 발 보행 순서 |
| [지형 적응 제어 구성](ADAPTIVE_INTEGRATION_V4.md) | 24차원 실행 계획과 STM32 처리 경로 |
| [SPI 명세](ADAPTIVE_SPI_V3.md) | 128바이트 프레임, 단위와 CRC |
| [Jetson 소프트웨어](../SW/Jetson/README.md) | 실행 계획 계산과 센서 연결 범위 |
| [매니퓰레이터 통신](../SW/STM32/Manipulator_UART_Protocol.md) | UART 명령과 상태 규격 |

## 시뮬레이션과 학습

| 문서 | 내용 |
|---|---|
| [지형 적응 뷰어](HEXAPOD_MJX_ADAPTIVE_GAIT_USAGE.md) | Stage 0, 지형·센서 모드, 화면 조작 |
| [기본 보행 뷰어](HEXAPOD_FOOTHOLD_PREVIEW_USAGE.md) | 펌웨어 보행과 기존 18차원 정책 재생 |
| [학습 단계](HEXAPOD_MJX_ADAPTIVE_GAIT_LEARNING_PLAN.md) | 지형·보행별 학습 순서 |
| [높이 지도 CNN](ADAPTIVE_ELEVATION_GRID_V5.md) | 현재 관측과 네트워크 구조, 이전 가중치 호환성 |
| [GT 전진 완주 학습](ADAPTIVE_GT_TEACHER.md) | 지형 정보를 사용하는 학습과 기본 보행 비교 |
| [접촉 복구와 재시도](ADAPTIVE_RECONTACT.md) | 위상 경계 재접촉과 완주율 70% 승급 기준 |
| [RC 명령 추종](ADAPTIVE_RC_TRAINING.md) | 전진·후진·회전·정지 명령 학습 |
| [학습 실행 옵션](ADAPTIVE_OBSERVE_CYCLES.md) | 학습 구간, 평가, W&B 기록 |
| [완주 보상](ADAPTIVE_COMPLETION_REWARD.md) | 종료 보상과 체크포인트 재개 |
| [평가 영상](ADAPTIVE_EVAL_VIDEOS.md) | 현재·최고 정책 영상과 저장 파일 |
| [계단 경로 제한](ADAPTIVE_PATH_BOTTLENECK.md) | 경로 여유와 v5→v6 가중치 이전 |
| [계단 발 높이와 영상 대기](ADAPTIVE_VIDEO_WAIT_CLEARANCE.md) | 추가 여유 높이, 동기식 렌더링과 실행 로그 |
| [요철 지형](ADAPTIVE_ROUGH_BOXES.md) | 박스 기반 지형과 가중치 이전 |
| [Tripod 정지 원인](TRIPOD_STOP_FIX.md) | 기본 발 위치와 정지 진단 |

뷰어와 학습 스크립트는 기본적으로 `$HOME/.venvs/hexapod-mjx/bin/python`을 사용한다.
다른 환경을 사용한다면 실행 전에 Python 경로를 지정한다.

```bash
export HEXAPOD_PYTHON="/path/to/venv/bin/python"
bash scripts/view_foothold_planner.sh --controller adaptive --terrain flat --perception oracle
```

학습 환경 의존성은 [`mjx/requirements-train.txt`](../mjx/requirements-train.txt)에 있다.
보관된 18차원 정책과 현재 24차원 정책은 입력·출력 규격이 다르다. 이전 체크포인트는
기록된 소스 버전과 관측·네트워크 계약을 맞춰 재생한다.

## 설계와 실험 기록

- [제어기 구성](../SW/Controller/Controller_Architecture.md) · [상세 설계](../SW/Controller/Controller_detail.md)
- [착지점과 보행 전환 설계](HEXAPOD_HYBRID_GAIT_ANALYSIS.md)
- [지형 적응 제어 설계 결정](ADAPTIVE_INTEGRATION_ANALYSIS.md) · [관련 모듈](ADAPTIVE_INTEGRATION_FILES.md)
- [Simulink 실험 결과](../SW/Controller/Simulink/evaluation/RESULTS.md)
- [가상 자세와 실측 문헌 비교](../SW/Controller/Simulink/evaluation/BODY_STABILITY_COMPARISON.md)
- [종합설계 최종보고서](2026-inha-capstone-final-report-public.pdf)

Simulink의 몸체 고정 모델, MJX 보행 시뮬레이션, 실제 로봇 시험은 시험 조건이 다르다.
수치는 각 문서의 조건과 함께 참고한다.

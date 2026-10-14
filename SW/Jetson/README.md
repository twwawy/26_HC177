# Jetson 소프트웨어

Jetson Orin Nano Super에서 지형 인식과 상위 보행 계획을 처리한다.
실시간 발끝 궤적, 역기구학과 서보 출력은 STM32가 담당한다.

## 현재 코드

| 파일 | 역할 |
|---|---|
| [adaptive_runtime.py](adaptive_runtime.py) | 점군·상태 추정·로봇 관측으로 보행 실행 계획 계산 |
| [adaptive_policy_contract.py](adaptive_policy_contract.py) | 24차원 정책 규격 |
| [adaptive_execution_plan.py](adaptive_execution_plan.py) | 정책과 지형 계산을 최종 실행 계획으로 변환 |
| [adaptive_spi_protocol.py](adaptive_spi_protocol.py) | 실행 계획·관측 SPI 코덱 |
| [make_offline_fixture.py](make_offline_fixture.py) | 오프라인 입력 데이터 생성 |

`adaptive_runtime.py`는 시간과 좌표계가 맞춰진 외부 상태 추정값과 월드 좌표 점군을 받는다.
점군의 센서 왜곡 보정과 로봇 자체 점 제거는 입력 단계에서 처리해야 한다.
학습 가중치 없이 zero-action 계획을 계산하는 경로를 제공한다.
정책을 사용할 때는 관측 규격에 맞는 완전한 입력 벡터가 필요하다.

현재 코드에는 Livox·RealSense 실시간 수집, odometry 생성, 센서 시간 동기화,
SPI Master·DRDY 이벤트 루프가 포함되지 않는다. 이 연결을 완료해야 실제 로봇에서 운용할 수 있다.

## STM32 연결

Jetson은 SPI Master, STM32는 SPI2 Slave다. MOSI·MISO·SCLK·CS·DRDY 배치는
[STM32 설정](../STM32/STM32F446RE%20설정%20정리본.md)을 참고한다.

현재 지형 적응 실행 경로는 **128바이트 프레임**을 사용한다.
최종 발 목표, 몸체 자세·높이와 보행 타이밍을 고정소수점으로 전달하며,
CRC·세션·시퀀스·계획 ID와 관측 유효 시간을 검사한다.
로봇 관측은 같은 시점에 캡처한 상태·상세 두 페이지로 전달한다.

- [지형 적응 SPI 명세](../../docs/ADAPTIVE_SPI_V3.md)
- [실행 계획과 제어 구성](../../docs/ADAPTIVE_INTEGRATION_V4.md)

이전 센서·GPS용 규격은 64바이트 전이중 트랜잭션이다. 같은 v3 표기를 사용하지만
프레임 크기와 명령 배치가 다르므로 현재 지형 적응 경로와 함께 사용할 수 없다.
이전 설계는 [센서·GPS SPI 문서](../STM32/STM32-Jetson%20SPI%2032바이트%20패킷%20프로토콜.md)에 남아 있다.

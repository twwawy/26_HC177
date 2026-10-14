# STM32–MJX adaptive hybrid 통합 v4

## 구성과 소스 기준

main base: `de77791d6386309295be5c8d173a1f51eca99930`.
기준 버전 직전의 `9752c760002ac2dc7cc493ed634a881f87b7fd1e`와 STM32 코드는 같으며, URDF와 메시만 변경됐다.
adaptive source: `f9d80587ff086a1f3df51b81185e742094c109c9`.
STM32 기본 보행 제어에 루트 `mjx/`의 지형 적응 제어와 Jetson 실행 계획 경로를 연결한다.
[설계 결정](ADAPTIVE_INTEGRATION_ANALYSIS.md), [관련 모듈](ADAPTIVE_INTEGRATION_FILES.md).

## 책임 분리

```text
LiDAR + odom/state → geometry/feasible set → PolicyAction24
                   → geometry + residual + safety projection
                   → AdaptiveExecutionPlan → STM32 contact/gait/IK/PWM
```

Geometry: terrain Z, normal/slope, coverage/confidence, edge, support extent,
IK/joint margin, path obstacle/coverage, feasible stride 및 Tripod/Wave feasibility.
RL: landing XY, clearance, roll/pitch/height, stride preference, apex/transfer timing.
Supervisor: normal Tripod → short Tripod → observed failure일 때 Wave → HOLD.
UNKNOWN은 unsafe의 증거가 아니므로 자동 Wave 전환을 허가하지 않는다.
단, 위를 보는 LiDAR가 첫 착지 영역을 못 본 상태에서 mapping과 이동이 서로 기다리는 것을 막기 위해
시작 시 6발 접촉이 확인되고 feasible Tripod landing이 아직 한 번도 만들어지지 않았을 때만 0.5 stride의
classical Tripod를 이어간다. feasible plan이 한 번 확보되면 bootstrap을 영구 종료하며 이후 UNKNOWN은
다시 HOLD한다. 영구 blind walking을 막기 위해 최대 24 phase로 제한한다. 이는 Wave 전환 근거로
쓰지 않는다. 엄격한 UNKNOWN 검사에는 viewer의
`--no-bootstrap-classical`을 사용한다.
STM32: actual transition, raw/confirmed contact, Early/Late, support recovery,
workspace/IK, joint/PWM/relay/Kill/Fault의 최종 권한.

## Policy / observation / checkpoint

`ACTION_CONTRACT=adaptive_hybrid_geometry_residual_24_v4`, ACTION_SIZE=24.
다리 순서 RF RM RB LF LM LB.

| Action | 의미 | 범위 |
|---|---|---|
| 0:12 | 6×landing XY residual | X ±0.06 m, Y ±0.04 m |
| 12:18 | 6×clearance residual | ±0.04 m |
| 18 | terrain baseline 위 roll residual | ±5° |
| 19 | terrain baseline 위 pitch residual | ±10° |
| 20 | body height offset residual | ±0.03 m, config constant |
| 21 | stride preference | 0.5–1.3, zero=1 |
| 22 | apex plateau 중심 residual | ±0.15 |
| 23 | XY transfer 중심 residual | ±0.15 |

Actor **4434-D**: proprio157 + global23 + reference54 + candidate4200.
Critic **4749-D**: actor + GT/error300 + terrain15.
Reference는 per-leg XYZ, nominal 거리, validity, dx_min/dx_max/dy_min/dy_max다.
각 축 feasible extent는 safe 후보의 extrema이며 사각형 내부가 모두 safe라는 뜻은 아니다.
기존 candidate geometry 28-feature 배열 및 robot/global 정보는 유지한다.
OBSERVATION_CONTRACT=`adaptive_hybrid_geometry_local25_extent_24_v4`.
REWARD_CONTRACT=`adaptive_hybrid_efficient_progress_v4`.

v3 및 이전 adaptive checkpoint는 contract/shape mismatch로 명시적으로 거절한다.
Stage31 18-D는 기존 controller/viewer 경로로 보존했다. v4로 자동 변환하지 않는다.
metadata에는 축별 XY limit, local offsets, timing, source SHA256 및 observation 크기를 저장한다.
보관된 Stage31 가중치는 18차원 정책용이며 이 24차원 계약에 적용할 수 없다.

## Wide search와 local residual

Wide: nominal 주위 X/Y `[-8,-4,0,4,8] cm`, 25개.
Reference 선정 뒤 local: X `[-6,-3,0,3,6] cm`, Y `[-4,-2,0,2,4] cm`, 25개.
`reference + decoded XY` 요청과 가장 가까운 **safe local candidate**를 선택한다.
wide와 local은 같은 support/IK/path 검사기를 사용한다. 최종 조합 support margin도 재검사한다.
RL request가 불가능해도 검증한 neutral reference가 있으면 이를 사용한다.

Map은 64×64, 5 cm rolling grid를 유지한다. 여러 local 후보가 동일 cell의 height/confidence를
공유할 수 있지만 XY endpoint, IK, joint margin, swept path는 별도로 검사한다.
`test_real_geometry_projection_and_z_ownership`은 실제 후보 검사기를 사용하여 가능한 다리에서
selected XY가 ±6/±4 cm 끝점까지 도달함을 확인한다. 단순 decoder 상수 검사가 아니다.

Terrain Z는 query 결과다. XY를 바꾸면 다른 cell의 Z가 선택될 수 있지만 Z action은 없다.
Swing geometry는 terrain surface에서 계산하고 IK endpoint는 foot radius를 더해 변환한다.
STM32에는 policy residual을 다시 해석시키지 않고 최종 pre-posture XYZ를 보낸다.

## Clearance / 자세 / body height

```text
required = max(path_max - max(start_surface_z, landing_surface_z), 0)
h = clip(max(required + 0.06 + RL_h, required + 0.02, 0.04), 0.04, 0.18)
```

상한 때문에 `h < required+0.02`가 되면 후보를 거절한다.
Manual의 0.20 m swing 및 0.07 m radial bulge는 그대로다.
Adaptive는 검사한 start→target quintic 경로이며 fixed radial bulge를 추가하지 않는다.

Roll/pitch는 terrain normal baseline(각 ±12° 제한) + residual 후 absolute ±15°로 제한한다.
STM32 기존 posture controller가 이 absolute reference를 받는다. PI/rate/workspace는 유지한다.
Body height는 correction `body_offset_m.z`와 별도 상태다. phase target을 latch하고
0.04 m/s로 변화시키며 6발 workspace 검사를 통과한 값만 적용한다. 종료 시 memory를 재기준화한다.
Swing 중 새 plan은 active body target을 바꾸지 않는다.

## Stride / phase / trajectory

Geometry bank `[1.3,1,.75,.5,.25,.125]`에서 RL preference 이하의 가능한 값을 선택한다.
Tripod와 Wave 모두 stride preference를 적용한다. Oversize Tripod는 normal도 feasible일 때만 허가한다.

```text
Tripod T = clip(1.0 * accepted_stride, 0.5, 1.4) s
Wave   T = clip(1.0 * accepted_stride, 0.6, 1.4) s
speed_scale = accepted_stride * baseline/T * (0.2 if Wave else 1)
```

기준 시간은 둘 다1초. Accepted twist와 T를 함께 전달하므로 phase displacement는
`T*(-vx+wz*y, -vy-wz*x, -vz)`다. Wave stance는5 phase, 첫 순회 이동은 절반이다.
MJX adaptive 전후 속도 상한0.10 m/s, yaw18°/s로 main에 맞췄다. Legacy18-D 상수는 변경하지 않았다.
STM32는 pending duration을 실제 phase 진입/검증된 continuation에만 active로 복사한다.

`Q(u)=10u³−15u⁴+6u⁵`, 모든 u를[0,1]에 clip한다.
진행률p, apex A∈[.3,.7], transfer B∈[.35,.65]:

```text
qxy = Q((p-(B-.25))/.5)
qup = Q(p/(A-.2))
qdn = Q((p-(A+.2))/(.8-A))
top = max(start_z,end_z)+clearance
xy = start_xy + qxy*(end_xy-start_xy)
z = start_z + qup*(top-start_z) + qdn*(end_z-top)
```

A는 단일 최고점 시각이 아니라 폭0.4 plateau 중심이다.
Python planned_swing과 C CalculateAdaptive가 위 수식을 공유한다.
Workspace preflight와 실제 FootTrajectory는 동일 C 함수를 호출한다.
Adaptive에서는 legacy touchdown approach Z 재형성을 추가하지 않는다. 접촉 확인과
Early/Late safety 및 최종 joint/PWM 제한은 계속 적용한다.

## 실행 계획과 gait authority

`RobotAdaptiveExecutionPlan_t`: session, source observation, command sequence, plan ID,
swing mask, requested gait, execute flag, duration, absolute roll/pitch, body-height offset,
applied twist, six final XYZ/clearance/apex/transfer.

제출은 기존 RlController의 session/history/sequence/age/plan 검증을 재사용한다.
execute=false로 HOLD 또는 gait negotiation을 하고 새 광고 계획을 기다린다.
requested gait가 달라도 mid-swing actual gait는 바뀌지 않는다. 모두 접촉한 경계에서 새 패턴의
preview를 발행하고, 해당 실행 plan이 workspace 검사를 통과해야 actual pattern을 바꾸고 이륙한다.
FootTrajectory pending→active 복사는 swing entry 한 번이다. 새 action/map/timing이 active를 덮지 않는다.

Wave order 0,5,1,3,2,4. MJX의 raw/confirmed contact는 simulation input으로 구현한 근사다.
STM32의 1-ms FSR 샘플링을 그대로 재현했다고 보지 않는다. Late .12 m/s, inward .096 m/s,
한계10 cm이며 support loss는 회복/정지로 처리한다. Wave common-Z recovery는 비활성이다.
Adaptive Tripod common-Z도 geometry Z와 별도 누적 이동이 충돌하지 않도록 적용하지 않는다.

## Jetson / SPI

[128-byte v3 명세](ADAPTIVE_SPI_V3.md). C/Python explicit offset·fixed point·CRC codec을 추가했다.
STM32 DMA→main-loop decode→app submit 경로에 연결했다. 실제 SPI 전기적 동작은 미검증이다.
두 observation page를 같은 capture로 묶고 receiver는 session/sequence/plan 일치를 확인한다.

`SW/Jetson/adaptive_runtime.py`는 coherent observation + 외부 state estimate + deskew/self-filter된
world LiDAR returns를 받아 동일 JAX geometry와 supervisor를 사용한다. zero-action 지원.
Odometry 회전은 hardware forward/left/up→world, position/CoM은 동일 world frame이다.
Map timestamp와 now는 같은 monotonic clock을 사용한다.

**Live Livox/odom 수집, 시간 동기화, SPI master/DRDY 이벤트 루프는 배치 환경 연결 작업으로 남아 있다.**
Nonzero policy 사용 시 완전한4434-D actor vector를 caller가 제공해야 한다. 현재 SPI 관측만으로
odom velocity와 slip/history 등은 실제 상태 추정값으로 제공해야 한다.
현재 경로를 실물에서 사용하려면 센서 수집과 SPI 운용을 연결해야 한다.

## 통합 당시 시험 기록과 남은 과제

실행: 기존 MJX hybrid9 + v4 5 = **14 tests PASS**.
새 검사는 실제 local geometry endpoint 사용, Z ownership, Python/C32개 golden trajectory,
C SPI 디코딩/CRC/중복/timeout, baseline timing 및 boundary duration/gait transition을 포함한다.
STM32 native portable11개 **PASS**: calibration conversion, user command, kinematics, controller,
workspace, gait, mode transition, safety, communication, RL input, RL stop.
핵심 C 모듈과 app/SPI의 native 문법 검사, Python 문법 검사 통과 기록이 있다.

Native compiler는 기존 controller/mode-transition test의 array-size 추론 warning을 출력했다.
CPU 환경의 14개 검사 기록이며, HAL I/O stub은 호출 시 중단한다.
이 기록에는 ARM ELF 빌드, 실제 DMA/DRDY, 전체 뷰어 rollout, Stage 0 지표, PPO와 실물 보행이 포함되지 않는다.
ARM 빌드 스크립트는 지형 적응 C 소스를 포함한다.

남은 위험: foot-centre/실제 발 접촉점·servo model 보정, phase-entry frame과 움직이는 몸체의 오차,
posture PI/필터 및 raw-touchdown 취소 후 재개 궤적의 sim/hardware 차이, fixed-sample path 검사 사이 충돌,
발 이외 몸체 충돌, state-estimation/deskew/clock/latency, LiDAR coverage, quantization,
60-ms 관측 lease 안의 planner latency. 긴 계산은 HOLD/timeout으로 나타나야 하며 lease를 임의로 늘리지 않는다.
현재 height map 누적 max/min은 동적 지형 및 odom drift에 대한 별도 평가가 필요하다.
이 결과는 계단 등판 성공·학습 성공·실기 검증 완료를 의미하지 않는다.

## 실행 순서

```bash
cd Hexapod-Robot
source "$HOME/.venvs/hexapod-mjx/bin/activate"

# 1. Stage 0: 반드시 가중치 없이 확인
bash scripts/view_foothold_planner.sh --controller adaptive --terrain flat --perception oracle
bash scripts/view_foothold_planner.sh --controller adaptive --terrain steps --perception oracle
bash scripts/view_foothold_planner.sh --controller adaptive --terrain steps --perception lidar --stage0

# 2. 필요 시 계약 검사 재현
JAX_PLATFORMS=cpu python -m unittest mjx.tests.test_adaptive_hybrid mjx.tests.test_adaptive_v4
python3 scripts/test_stm32_host.py

# 3. Offline synthetic point-cloud → zero-action → 128-byte file (전송 없음)
python SW/Jetson/make_offline_fixture.py --output /tmp/hexapod-flat-input.npz
python SW/Jetson/adaptive_runtime.py --input /tmp/hexapod-flat-input.npz --output /tmp/hexapod-command.bin

# 4. ARM GCC 설치/경로 설정 후, flash 없이 ELF만 생성
python3 scripts/build_stm32.py --compiler arm-none-eabi-gcc

# 5. Stage 0 통과 후에만 새 v4 학습
bash scripts/train_adaptive_gait.sh --stage 1 --terrain-level 0 --perception lidar --num-envs 64
# 다음 curriculum에서 --terrain-level 3, 5 등을 적용; Stage 2 Wave / Stage 3 hybrid
```

Viewer에서는 nominal(연보라), wide reference(흰색), RL request(청록), local candidate 상태색,
projected(주황), latched execution(진빨강)을 비교한다. 방향키 연속 보행/정지 및 G FOV 토글은
[viewer 사용법](HEXAPOD_MJX_ADAPTIVE_GAIT_USAGE.md)을 참고한다.
Flat normal Tripod → 계단 short-step → observed Tripod 실패 시 Wave → 평탄화 복귀 → unknown/HOLD 순으로
확인한다. oracle에서도 landing이 없으면 planner/IK/path, oracle만 되면 LiDAR/map filter부터 확인한다.

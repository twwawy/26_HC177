# 평가 정책 영상과 체크포인트 재개

## 영상과 평가 기록

`--best-video`를 켜면 학습 후 평가마다 현재 정책을 렌더링한다.
W&B의 `cycle/current_video`, `cycle/current_score`, `cycle/current_step`에 기록한다.
최고 정책은 완주율을 우선하고, 완주율이 같으면 보상 점수로 선택한다.

최고 정책이 갱신된 평가에서만 `cycle/best_video`와 best-policy artifact를 추가한다.
그 평가의 현재 영상을 재사용하므로 같은 정책을 두 번 렌더링하지 않는다.
초기 step 0과 cycle 종료 시의 중복 영상은 제외한다.

| 기록 | 저장 위치 |
|---|---|
| 현재 정책 영상 | `videos/eval_<step>_current.gif` |
| 영상·점수·체크포인트 연결 | `monitor/current_video_<step>.json` |
| 각 평가의 정책 | current-policy artifact |
| 갱신된 최고 정책 | best-policy artifact |

같은 seed와 명령 조건으로 영상을 비교한다. 단일 재생 결과와 여러 환경의 평가 성공률은 다를 수 있다.
영상 생성은 동기식이므로 렌더링 중에는 PPO가 기다린다.

## 계단 체크포인트 재개

현재 v6 경로 보정과 보관된 계단 가중치를 함께 사용하려면 저장소 루트에서 실행한다.

```bash
bash scripts/resume_stair5_path_v6.sh
```

기본 체크포인트는 다음 경로다.

```text
mjx/runs/adaptive-curriculum/forward-gt-terrain-mid/02_tripod-stair5_try02/checkpoints/000000204800
```

다른 위치에 보관했다면 `HEXAPOD_CHECKPOINT`를 지정한다.

```bash
HEXAPOD_CHECKPOINT="/path/to/checkpoints/000000204800" \
  bash scripts/resume_stair5_path_v6.sh
```

스크립트는 `--migrate-path-v6`로 기록된 v5 소스의 해시를 검사한 뒤 명시적으로 이전한다.
원본 가중치를 덮어쓰지 않는다. v5와 v6는 입력 차원이 달라 일반 `--restore`만으로는 호환되지 않는다.
관련 조건은 [경로 제한과 가중치 이전](ADAPTIVE_PATH_BOTTLENECK.md)을 참고한다.

재개 설정은 학습량 800000, 평가 설정 9, 평가 환경 16, 영상 최대 20초,
추가 baseline 비교 0이다. Brax 초기 평가를 제외한 학습 후 평가는 보통 8회이며,
실제 평가 간격은 PPO 배치 단위로 결정된다.
완주율 70%를 통과한 뒤 다음 지형으로 진행한다.

## 보관된 v5 기록

`best_video_7_0e1017aebbbc13860373.gif`에 연결된 체크포인트는 위 경로의 `000000204800`이다.
기록된 소스 버전은 `d0370b3f30b0d3c83541c61a1df44bc87ebcf7d7`이며 terrain 작업의 v5 관측을 사용한다.
이전 v5 재개 경로에는 v6 경로 보정이 없으므로 보관된 소스 버전의 재생과 v6 이전을 구분한다.
학습·영상·실물 보행 성능은 각각의 실행 기록으로 평가해야 한다.

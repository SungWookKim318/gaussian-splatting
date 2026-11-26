# 아이디어: Frustum Culling 개선

상태: **진행 중** · 관련 분석: [rasterizer.md](../code-analysis/rasterizer.md#culling)

## 처음 가설

- 기존 3DGS는 가우시안 중심점 기준으로 culling → 중심이 frustum 밖이어도 일부가 화면에 걸칠 수 있음 → 팝핑
- 가우시안 크기(타원체)와 frustum의 실제 겹침을 판정하면 팝핑이 줄 것. 성능 손해와 함께 측정해 리포트 작성

## 코드 확인 후 수정된 문제 정의

- 상하좌우: 이미 크기를 고려함 (2D 3σ 정사각형, 넉넉한 근사) → 가설과 달리 잘못 잘리는 경우는 없음
- **Near: 중심점만 봄** (`z <= 0.2`) → 가설이 정확히 맞는 곳. 카메라가 큰 가우시안에 다가가면 몸통이 앞에 있어도 중심이 넘어가는 순간 사라짐
- 정확한 3D 타원체 판정은 어디에도 없음
- 팝핑의 주원인은 culling이 아니라 중심 깊이 정렬(아이디어 2, StopThePop). 이 아이디어는 그중 **culling에서 오는 팝핑**에 한정

## 방향: 6개 평면에 대한 정확한 타원체 판정

타원체-평면 판정 (평면 법선 n, k = 3):

```
타원체가 평면에서 가장 멀리 튀어나온 거리 = k · sqrt(nᵀ Σ n)

near (view 공간, n = (0,0,1)):
  μ_z + k · sqrt(Σ_zz) <= near  →  완전히 near 뒤 → 제거
```

| 평면 | 예상 효과 (검증 필요) |
|---|---|
| 상하좌우 | 3σ 정사각형보다 엄격 → 처리 가우시안 감소, 화질 동일 → **성능 이득** |
| Near | 중심이 넘어가도 몸통이 앞에 있으면 유지 → **팝핑 감소**, 처리량 증가 |

## 핵심 난제: near 뒤 중심을 가진 가우시안의 투영

중심 z가 0에 가깝거나 음수면 투영(z 나눗셈)과 2D 공분산(중심에서의 Jacobian 근사)이 깨짐 → 화면을 덮는 얼룩이나 NaN 가능.

- (a) 투영 시 중심 z를 near로 clamp: 간단, 모양 왜곡
- (b) near plane에서 잘린 앞부분의 중심/공분산으로 근사: 정확, 수식 많음
- → (a) 먼저 구현하고 아티팩트도 결과로 기록

**주의**: 투영 계산을 바꾸면 backward.cu의 `computeCov2DCUDA`도 같은 방식으로 맞춰야 학습 gradient가 맞음. 판정(살릴지 말지)만 바꾸면 backward 수정 불필요.

## 수정 지점

- [auxiliary.h:166](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/auxiliary.h#L166) `in_frustum`: near 판정
- [forward.cu](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/forward.cu) `preprocessCUDA`: 상하좌우 판정, 투영 처리
- (투영을 바꾸는 경우) [backward.cu](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/backward.cu) `computeCov2DCUDA`

## 실험 계획

- 카메라 경로: test 시점은 멀어서 차이가 안 보임 → 물체(트럭)에 **다가가는 연속 경로**를 만들어 프레임 시퀀스 렌더
- 팝핑 지표: 연속 프레임 픽셀 차이, 프레임마다 culling 결과가 바뀌는 가우시안 수
- 성능 지표: 프레임당 렌더 시간, 처리 가우시안 수
- 학습 영향: 판정이 radii를 바꾸므로 학습 시 가우시안 수와 test PSNR도 측정
- 비교: baseline / 개선 (a) / 가능하면 (b)

## 리포트 구성안

1. 문제 제기: 원 논문의 guard band 설명 vs 구현 (중심점 near culling, 3σ 정사각형 근사)
2. 관련 연구: StopThePop 등
3. 분석 과정: 가설 → 코드 확인 → 상하좌우는 이미 근사 구현됨 발견 → 빠진 부분(near)과 근사 부분(상하좌우)을 개선 대상으로 재정의
4. 개선: 6개 평면 정확 타원체 판정 + near에서 살린 가우시안의 투영 처리
5. 실험 결과
6. 한계: 팝핑 주원인(정렬)은 범위 밖

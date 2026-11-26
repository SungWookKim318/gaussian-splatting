# 코드 분석: 래스터라이저 (culling, 정렬)

대상: [submodules/diff-gaussian-rasterization/cuda_rasterizer/](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/)

## 읽는 순서

`train.py:111 render()`에서 호출을 따라 내려가는 순서.

1. [gaussian_renderer/\_\_init\_\_.py](../../gaussian_renderer/__init__.py) (128줄, 정독): Python에서 CUDA로 넘기는 입구. [L123](../../gaussian_renderer/__init__.py#L123) `visibility_filter = (radii > 0)`
2. [diff_gaussian_rasterization/\_\_init\_\_.py](../../submodules/diff-gaussian-rasterization/diff_gaussian_rasterization/__init__.py) (훑기): [L44](../../submodules/diff-gaussian-rasterization/diff_gaussian_rasterization/__init__.py#L44) `torch.autograd.Function`으로 forward/backward를 직접 정의 → `loss.backward()`가 backward.cu까지 연결됨
3. [rasterizer_impl.cu](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/rasterizer_impl.cu) `forward()` ([L198~](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/rasterizer_impl.cu#L198)): 파이프라인 순서
   1. `FORWARD::preprocess` (L250): 투영 + culling
   2. `duplicateWithKeys`: 타일마다 키 생성
   3. `DeviceRadixSort` (L306): 정렬
   4. `identifyTileRanges` (L317): 타일별 구간
   5. `FORWARD::render` (L325): α-blending
4. [forward.cu](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/forward.cu) `preprocessCUDA` (정독): 가우시안 하나당 스레드 하나. near 판정 → 3D 공분산 → 2D 공분산 → 3σ 반지름 → 타일 판정
5. [backward.cu](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/backward.cu) `computeCov2DCUDA` ([L147](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/backward.cu#L147)): forward `computeCov2D`를 손으로 미분한 코드

참고:
- [scene/cameras.py](../../scene/cameras.py): 카메라 행렬
- [scene/gaussian_model.py](../../scene/gaussian_model.py)의 getter: scale `exp`, opacity `sigmoid`, rotation 정규화

## Culling

`preprocessCUDA`에서 `radii[idx] = 0`으로 초기화한 뒤, 중간에 `return`되면 0으로 남음 = culling.

| Frustum 평면 | 구현 | 위치 |
|---|---|---|
| Near | **중심점만** 봄: `p_view.z <= 0.2f`면 제거 | [auxiliary.h:166](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/auxiliary.h#L166) |
| 상하좌우 | 2D 화면에서 `ceil(3σ_max)` 정사각형이 타일과 겹치는지로 판정 | [forward.cu:237-245](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/forward.cu#L237-L245) |
| Far | 검사 없음 | |

- `in_frustum`의 상하좌우 검사(±1.3)는 **주석 처리**되어 있음
- 3σ: 2D 공분산의 큰 고유값 → `sqrt` = 긴 축 표준편차 σ → ×3. 분포의 약 99% = 논문의 "99% 신뢰구간"
- 긴 축 기준 원을 다시 정사각형으로 감싸므로 실제 타원보다 넉넉함 → **들어오는데 잘리는 경우는 없고, 안 들어오는데 남는 경우는 있음**
- `det == 0`이어도 제거 ([forward.cu:227-228](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/forward.cu#L227-L228))
- Guard band: 화면 밖 가우시안의 2D 공분산은 위치를 화면 1.3배에서 clamp해서 계산 → 가장자리 모양 왜곡 가능 ([forward.cu:82-87](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/forward.cu#L82-L87))
- 투영 행렬의 near는 `znear = 0.01`([scene/cameras.py:81](../../scene/cameras.py#L81)), culling은 0.2 → 차이 이유 미확인
- 판정 결과(radii)는 렌더링, densification(`visibility_filter`), sparse_adam(`visible`)에 모두 전파됨

## 정렬 (팝핑의 주원인)

- 깊이 = 가우시안 **중심**의 view 공간 z ([forward.cu:258](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/forward.cu#L258) `depths[idx] = p_view.z`)
- 정렬 키 = 타일 번호(상위 32bit) + 깊이(하위 32bit) ([rasterizer_impl.cu:102-104](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/rasterizer_impl.cu#L102-L104))
- 이미지 전체를 radix sort 한 번 → 타일 내 256픽셀이 같은 순서로 blending
- 팝핑이 생기는 이유:
  - 중심 깊이 ≠ 픽셀 광선 위의 실제 깊이 (크고 기울어진 가우시안일수록 차이가 큼)
  - 카메라 이동 중 두 중심의 깊이 대소가 바뀌는 순간 blending 순서가 뒤집혀 색이 튐
  - 학습 시점에서는 틀린 순서까지 맞춰 학습되므로, 그 사이 시점에서 주로 튐
- 부차 원인: near-plane 중심점 culling (근접 시점), 3σ 잘림, `alpha < 1/255` 건너뛰기
- 후속 연구: StopThePop (Radl et al., SIGGRAPH 2024): 픽셀별 실제 최대 밀도 깊이로 정렬

## 빠른 이유

- 신경망을 평가하지 않음 (NeRF는 광선 샘플마다 MLP 실행)
- frustum 밖 가우시안 제외
- 16×16 타일 ([config.h](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/config.h)), 전역 정렬 1회, 타일 = 스레드 블록, 픽셀 = 스레드
- 조기 종료 ([forward.cu:361-364](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/forward.cu#L361-L364)): `alpha < 1/255`면 건너뛰고, `T < 0.0001`이면 중단
- backward는 forward의 정렬 결과를 재사용하고 뒤에서 앞으로 순회하며 중간값을 재계산

## 논문 vs 코드

논문에 있음 (장 번호는 기억 기준 → **원문 확인 필요**):
- 99% 신뢰구간 기반 frustum culling, guard band (near에 가깝거나 frustum에서 멀리 벗어난 가우시안은 2D 공분산이 불안정해서 제거) (6장)
- 타일 단위 전역 정렬로 인한 근사 α-blending (6장)
- blending 순서가 갑자기 바뀔 수 있다는 한계 = 팝핑 (7장 Limitations)
- `α < 1/255` 건너뛰기, α 상한 0.99, `T < 0.0001` 중단 (부록)

코드에만 있음:
- near 0.2, guard band 비율 1.3, 2D 공분산에 더하는 `h_var = 0.3`
- 상하좌우 검사를 주석 처리하고 3σ 정사각형으로 대체

2024년 10월 추가 ([README.md:42](../../README.md#L42)):
- sparse_adam 등 학습 가속 (Taming 3DGS 코드), depth regularization (Hierarchical 3DGS), anti-aliasing, exposure compensation

## 확인할 것

- [ ] 논문 원문에서 guard band, 팝핑 한계 문장 위치
- [ ] znear 0.01 vs culling 0.2 차이 이유

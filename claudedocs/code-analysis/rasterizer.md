# 읽기 체크리스트: 래스터라이저

`train.py:111 render()`에서 호출을 따라 내려가는 순서. 경로 기준: `submodules/diff-gaussian-rasterization/`

## 1. Python 입구

- [ ] [gaussian_renderer/\_\_init\_\_.py](../../gaussian_renderer/__init__.py) `render()` 전체 (128줄)
- [ ] [diff_gaussian_rasterization/\_\_init\_\_.py](../../submodules/diff-gaussian-rasterization/diff_gaussian_rasterization/__init__.py) L44 `_RasterizeGaussians` forward/backward 구조

## 2. CUDA 파이프라인

- [ ] [rasterizer_impl.cu](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/rasterizer_impl.cu#L198) L198 `Rasterizer::forward()` 호출 순서
  - [ ] L250 `preprocess`
  - [ ] L70 `duplicateWithKeys` (키 생성 L102-104)
  - [ ] L306 `SortPairs`
  - [ ] L116 `identifyTileRanges`
  - [ ] L325 `render`

## 3. Preprocess (culling)

- [ ] [auxiliary.h](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/auxiliary.h#L151) L151 `in_frustum`
- [ ] [forward.cu](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/forward.cu) `preprocessCUDA`
  - [ ] L183-191 radii 초기화, near 판정
  - [ ] L114 `computeCov3D`
  - [ ] L74 `computeCov2D` (L82-87 clamp)
  - [ ] L227-245 고유값, 3σ 반지름, `getRect`
  - [ ] L20 `computeColorFromSH` 호출 위치
  - [ ] L258 이후 depth, radii 저장
- [ ] [auxiliary.h](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/auxiliary.h#L45) L45 `getRect`

## 4. Render (blending)

- [ ] [forward.cu](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/forward.cu#L276) L276 `renderCUDA`
  - [ ] L310-312 shared memory
  - [ ] L361-364 early termination

## 5. Backward

- [ ] [backward.cu](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/backward.cu#L454) L454 `renderCUDA`
- [ ] [backward.cu](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/backward.cu#L147) L147 `computeCov2DCUDA`

## 참고

- [ ] [scene/cameras.py](../../scene/cameras.py) L80-87 카메라 행렬 (znear 0.01)
- [ ] [scene/gaussian_model.py](../../scene/gaussian_model.py) getter 활성화 함수 (`exp`, `sigmoid`, 정규화)
- [ ] [config.h](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/config.h) `BLOCK_X`, `BLOCK_Y`
- [ ] guard band 이력: `git log -S"p_proj.x < -1.3" -- cuda_rasterizer/auxiliary.h` (서브모듈에서)

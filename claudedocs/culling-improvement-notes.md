# 3DGS 개선 과제 노트: Frustum Culling

작성일: 2026-11-25 · 브랜치: `study/frustum-culling`

## 1. 과제 요건

- 3DGS 또는 후속 논문의 코드를 기반으로 개선
- 아이디어는 후속 논문에서 가져와도 되지만, **개선 내용은 직접 구현**
- AI 도움은 허용되지만 내용은 모두 이해해야 함
- 개선 아이디어, 구현 내용, 결과를 report로 작성
- report 검증은 지정된 skill로 수행 → **어떤 skill인지 아직 확인 안 됨**

## 2. 실행 환경

- 코드 읽기: macOS (CUDA 없음 → 빌드/실행 불가)
- 실행: Windows PC, RTX 3070 8GB / Ryzen 3900X / RAM 32GB
- 현재 실행 중인 설정:

```bat
python train.py -s ./datasets/tandt/truck -m ./output/truck_7k ^
    --iterations 7000 --resolution 2 --data_device cpu --eval
```

- 학습 후: `python render.py -m <출력>` → `python metrics.py -m <출력>`
- 참고: truck은 `--data_device cpu`를 빼도 될 가능성이 큼 (VRAM은 이미지보다 가우시안이 주로 차지)

## 3. 코드 이해 요점 (train.py)

- train.py는 논문 Algorithm 1(Optimization and Densification)을 코드로 옮긴 것
- 루프: 랜덤 카메라 → `render()` → loss = 0.8·L1 + 0.2·(1−SSIM) → `loss.backward()` → densification → `optimizer.step()`
- **loss는 iteration당 1개(스칼라)**, gradient는 파라미터마다 존재 (가우시안 1개 = 파라미터 59개: xyz 3, scale 3, rotation 4, opacity 1, SH 48)
- densification은 backward 다음이어야 함: `viewspace_point_tensor.grad`(view-space positional gradient)를 입력으로 쓰기 때문 ([scene/gaussian_model.py:471](../scene/gaussian_model.py#L471))
- `radii`: CUDA 래스터라이저가 계산한 투영 반지름(px). 0이면 culling된 것
  - `visibility_filter = radii > 0` → densification 통계
  - `visible = radii > 0` → sparse_adam (보이는 것만 업데이트)
- sparse_adam은 판정을 하지 않음. 래스터라이저의 판정 결과(radii)를 받아 업데이트 대상만 좁힘
  - 기본 Adam은 gradient가 0이어도 모멘텀 때문에 안 보이는 가우시안도 조금 움직임
- `alpha_mask`: RGBA 입력의 알파 채널. JPG(truck)는 전부 1이라 효과 없음
- 기본값 기준 타임라인: densify 500~15000 (100마다), SH 차수 1000마다 +1, opacity 리셋 3000마다

## 4. 아이디어 목록과 현재 판정

| # | 아이디어 | 판정 |
|---|---|---|
| 1 | Frustum culling 개선 (가우시안 크기 고려) | **선택**. 단, 범위를 재정의함 (5장) |
| 2 | 가우시안 형태를 고려한 depth sorting | 팝핑의 주원인이지만 구현 난이도 높음 → 관련 연구/한계로 언급 (StopThePop) |
| 3 | 학습 파라미터 / VRAM 감소 | 문제 발견 단계 |
| 4-1 | SH 차수 감소 | 옵션(`--sh_degree`)만 바꾸면 됨 → 직접 구현 조건 미달. 보조 실험용 |
| 4-2 | Adaptive SH | 가능하지만 SH 수학 부담 |
| 4-3 | SH 대체 | 보류 |
| 5 | Densification 휴리스틱 개선 | Python만으로 가능, 정량 측정 쉬움 → 대안 후보 |

## 5. Culling 분석: 현재 코드가 실제로 하는 일

| Frustum 평면 | 현재 구현 | 위치 |
|---|---|---|
| Near | **중심점만** 봄: `p_view.z <= 0.2f`면 제거 | [auxiliary.h:166](../submodules/diff-gaussian-rasterization/cuda_rasterizer/auxiliary.h#L166) |
| 상하좌우 | 2D 화면에서 `ceil(3σ_max)` 정사각형이 타일과 겹치는지로 판정 (넉넉한 근사) | [forward.cu:237-245](../submodules/diff-gaussian-rasterization/cuda_rasterizer/forward.cu#L237-L245) |
| Far | 검사 없음 | |

- `in_frustum`의 상하좌우 검사(±1.3)는 **주석 처리**되어 있음
- 화면 밖 가우시안의 2D 공분산은 위치를 화면 1.3배에서 clamp해서 계산 (guard band) → 가장자리 모양 왜곡 가능 ([forward.cu:82-87](../submodules/diff-gaussian-rasterization/cuda_rasterizer/forward.cu#L82-L87))
- 투영 행렬의 near는 `znear = 0.01`([scene/cameras.py:81](../scene/cameras.py#L81))인데 culling은 0.2 → 값이 다른 이유 확인 필요
- 3σ = 2D 공분산 큰 고유값의 sqrt × 3. 분포의 약 99% = 논문의 "99% 신뢰구간"

**결론**: "크기를 고려한 culling"은 상하좌우에서 근사적으로 이미 구현되어 있음. **near는 중심점 기준이고, 정확한 3D 판정은 어디에도 없음.**

## 6. 선택한 방향: 6개 평면에 대한 정확한 타원체 판정

- 타원체-평면 판정 (평면 법선 n, k = 3):
  - 타원체가 평면에서 가장 멀리 튀어나온 거리 = `k · sqrt(nᵀ Σ n)`
  - near (n = (0,0,1), view 공간): `μ_z + k·sqrt(Σ_zz) <= near` 이면 완전히 뒤 → 제거
- 예상 효과 (검증 필요):
  - 상하좌우: 3σ 정사각형보다 엄격 → 처리 가우시안 감소, 화질 동일 → **성능 이득**
  - near: 중심이 넘어가도 몸통이 앞에 있으면 유지 → **팝핑 감소**, 처리량 증가
- 핵심 난제: 중심이 near 뒤(z ≈ 0 또는 음수)인 가우시안은 투영(z 나눗셈, Jacobian 근사)이 깨짐
  - (a) 투영 시 중심 z를 near로 clamp: 간단, 모양 왜곡
  - (b) near plane에서 잘린 앞부분의 중심/공분산으로 근사: 정확, 수식 많음
  - → (a) 먼저 구현하고 아티팩트를 결과로 기록
- **주의**: 투영 계산을 바꾸면 [backward.cu](../submodules/diff-gaussian-rasterization/cuda_rasterizer/backward.cu)의 `computeCov2DCUDA`도 맞춰야 학습 gradient가 맞음. 판정만 바꾸면 backward 수정 불필요
- 판정이 바뀌면 radii가 바뀌고 → 렌더링, densification, sparse_adam에 모두 전파됨 → 학습 결과(가우시안 수)도 같이 측정할 것

## 7. 팝핑 원인 정리

- 주원인: 정렬 키 = 타일 번호(상위 32bit) + **중심점 깊이**(하위 32bit) ([rasterizer_impl.cu:102-104](../submodules/diff-gaussian-rasterization/cuda_rasterizer/rasterizer_impl.cu#L102-L104))
  - 중심 깊이 ≠ 픽셀 광선상의 실제 깊이
  - 카메라 이동 시 두 중심의 깊이 대소가 바뀌는 순간 blending 순서가 뒤집혀 색이 튐
  - 타일 내 256픽셀이 같은 순서 사용
- 부차 원인: near-plane 중심점 culling (근접 시점), 3σ 잘림, `alpha < 1/255` 건너뛰기
- 후속 연구: StopThePop (Radl et al., SIGGRAPH 2024): 픽셀별 실제 깊이로 정렬

## 8. 논문 vs 코드 구분 (리포트 인용 시)

- 논문에 있음 (위치는 기억 기준 → **원문 확인 필요**):
  - 99% 신뢰구간 기반 frustum culling, guard band (6장)
  - 타일 단위 전역 정렬로 인한 근사 α-blending (6장)
  - blending 순서가 갑자기 바뀌는 한계 = 팝핑 (7장 Limitations)
  - `α < 1/255`, α 상한 0.99, T < 0.0001 중단 (부록)
- 코드에만 있음: near 0.2, guard band 1.3, `h_var = 0.3`, 상하좌우 검사 주석 처리
- 2024년 10월 추가 ([README.md:42](../README.md#L42)): sparse_adam/가속 (Taming 3DGS 코드), depth regularization, anti-aliasing, exposure compensation

## 9. 리포트 방침

- **Baseline** = 이 저장소 기본 설정 (추가 기능 전부 기본값 꺼짐 → 원 논문 방식). fused-ssim 사용 여부만 명시
- 기존 기능을 되돌렸다가 "직접 구현"이라고 하지 않기 (README/git 기록에 출처가 남아 있음)
- 구성안:
  1. 문제 제기: 원 논문 guard band 설명 vs 구현(중심점 near culling, 3σ 정사각형 근사)
  2. 관련 연구: StopThePop 등
  3. 개선: 6개 평면 정확 타원체 판정 + near에서 살린 가우시안의 투영 처리
  4. 실험: 물체에 다가가는 연속 카메라 경로 렌더
     - 팝핑 지표: 연속 프레임 픽셀 차이, 프레임마다 culling 결과가 바뀌는 가우시안 수
     - 성능 지표: 프레임당 렌더 시간, 처리 가우시안 수
     - 비교: baseline / 개선 (a) / 가능하면 (b)
  5. 한계: 팝핑 주원인(정렬)은 범위 밖
- 가설 → 코드 확인 → 이미 근사 구현됨 발견 → 빠진 부분(near)과 근사 부분(상하좌우)을 개선, 이 과정 자체를 분석으로 기술

## 10. 다음 읽을 순서

1. [gaussian_renderer/\_\_init\_\_.py](../gaussian_renderer/__init__.py) (128줄, 정독): Python에서 CUDA로 넘기는 입구
2. [diff_gaussian_rasterization/\_\_init\_\_.py](../submodules/diff-gaussian-rasterization/diff_gaussian_rasterization/__init__.py) (훑기): `autograd.Function`으로 forward/backward 연결
3. [rasterizer_impl.cu](../submodules/diff-gaussian-rasterization/cuda_rasterizer/rasterizer_impl.cu) `forward()` (L198~): preprocess → duplicateWithKeys → sort → identifyTileRanges → render
4. [forward.cu](../submodules/diff-gaussian-rasterization/cuda_rasterizer/forward.cu) `preprocessCUDA` (**정독, 수정 지점**)
5. [backward.cu](../submodules/diff-gaussian-rasterization/cuda_rasterizer/backward.cu) `computeCov2DCUDA` (투영을 바꿀 경우)
- 참고: [scene/cameras.py](../scene/cameras.py) (카메라 행렬), [scene/gaussian_model.py](../scene/gaussian_model.py)의 getter (scale `exp`, opacity `sigmoid`, rotation 정규화)

## 11. 미뤄둔 실험

- 실험 A: 250 iter마다 가우시안 수와 보이는 수 출력 → densification, opacity 리셋 효과 확인
- 실험 B: `--densify_until_iter 0`으로 densification 끄고 test PSNR 비교

## 12. 남은 확인 사항

- [ ] 리포트 검증용 skill이 무엇인지
- [ ] 논문 원문에서 guard band / 팝핑 한계 문장 위치 확인
- [ ] znear 0.01 vs culling 0.2 차이 이유
- [ ] 수정한 CUDA 코드를 Windows PC에서 재빌드하는 절차

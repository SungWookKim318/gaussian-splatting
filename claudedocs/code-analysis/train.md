# 읽기 체크리스트: train.py

## 1. 시작점

- [ ] [L254-282](../../train.py#L254-L282) `__main__`: 인자 파싱 → `training()` 호출
- [ ] [arguments/\_\_init\_\_.py](../../arguments/__init__.py) 기본 하이퍼파라미터

## 2. 준비

- [ ] [L48-71](../../train.py#L48-L71) `GaussianModel`, `Scene`, `training_setup`, 배경색, 카메라 목록

## 3. 학습 루프 (L73~)

- [ ] [L91-95](../../train.py#L91-L95) learning rate 갱신, SH 차수 증가
- [ ] [L97-103](../../train.py#L97-L103) 랜덤 카메라 선택
- [ ] [L111-112](../../train.py#L111-L112) `render()`와 반환값 (`image`, `viewspace_points`, `visibility_filter`, `radii`)
- [ ] [L114-116](../../train.py#L114-L116) `alpha_mask` (생성: [scene/cameras.py:42-48](../../scene/cameras.py#L42-L48))
- [ ] [L118-126](../../train.py#L118-L126) loss (L1 + SSIM)
- [ ] [L142](../../train.py#L142) `loss.backward()`
- [ ] [L146](../../train.py#L146) `torch.no_grad()`
- [ ] [L163-174](../../train.py#L163-L174) densification, opacity 리셋
  - [ ] [scene/gaussian_model.py:471-473](../../scene/gaussian_model.py#L471-L473) `add_densification_stats`
- [ ] [L176-186](../../train.py#L176-L186) optimizer step (default / sparse_adam)
- [ ] [L188-190](../../train.py#L188-L190) checkpoint 저장

## 4. 건너뛰어도 되는 부분

- [L74-87](../../train.py#L74-L87) `network_gui`
- [L128-140](../../train.py#L128-L140) depth regularization
- `prepare_output_and_logger`, `training_report`

## 5. 미뤄둔 확인 실험

- [ ] 실험 A: L146 아래에서 250 iter마다 `gaussians.get_xyz.shape[0]`, `visibility_filter.sum()` 출력
- [ ] 실험 B: `--densify_until_iter 0`으로 densification을 끄고 7000 iter test PSNR 비교

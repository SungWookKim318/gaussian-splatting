# 코드 분석: train.py

대상: [train.py](../../train.py)

## 전체 구조

train.py는 논문 부록 Algorithm 1(Optimization and Densification)을 코드로 옮긴 것.

| 논문 | train.py |
|---|---|
| SfM 점으로 가우시안 초기화 | [L50-51](../../train.py#L50-L51) `GaussianModel`, `Scene` |
| 랜덤 카메라 뷰 선택 | [L97-103](../../train.py#L97-L103) |
| Rasterize | [L111](../../train.py#L111) `render(...)` |
| Loss = 0.8·L1 + 0.2·(1−SSIM) | [L118-126](../../train.py#L118-L126) |
| Densify (clone/split) & prune | [L163-171](../../train.py#L163-L171) |
| opacity 리셋 | [L173-174](../../train.py#L173-L174) |
| Adam 업데이트 | [L176-186](../../train.py#L176-L186) |
| SH 차수 점진 증가 | [L94-95](../../train.py#L94-L95) |

처음 읽을 때 건너뛰어도 되는 부분:
- [L74-87](../../train.py#L74-L87) `network_gui`: 학습 중 SIBR 뷰어 실시간 미리보기
- [L128-140](../../train.py#L128-L140) depth regularization: 2024년 추가, 기본값 꺼짐
- `training_report`, `prepare_output_and_logger`: 로그, 출력 폴더

## 학습 루프의 PyTorch 패턴

```
render(forward) → loss(스칼라 1개) → loss.backward() → [densification] → optimizer.step() → zero_grad()
```

- **loss는 iteration당 1개.** `l1_loss`가 전 픽셀 평균, [L148](../../train.py#L148)의 `.item()`은 원소 1개짜리 텐서에만 쓸 수 있음
- **gradient는 파라미터마다.** 가우시안 1개 = 파라미터 59개 (xyz 3, scale 3, rotation 4, opacity 1, SH 48)
- 신경망은 없음. 가우시안 속성이 곧 학습 파라미터이고, 래스터라이저는 미분 가능한 고정 함수
- N개 가우시안의 속성은 `(N, 3)` 같은 큰 텐서 하나에 저장 ([scene/gaussian_model.py:166-171](../../scene/gaussian_model.py#L166-L171), `nn.Parameter`)
- `with torch.no_grad():` ([L146](../../train.py#L146)): 이후는 미분 기록 불필요
- **densification이 backward 다음인 이유**: `viewspace_point_tensor.grad`(view-space positional gradient)를 입력으로 쓰기 때문. backward 전에는 `.grad`가 비어 있음 ([scene/gaussian_model.py:471-473](../../scene/gaussian_model.py#L471-L473))
- densification이 backward와 step 사이에 있어서, 가우시안 수가 바뀌면 Adam 상태도 같이 늘리거나 줄여야 함 (gaussian_model.py에서 처리)

## render()가 돌려주는 값

[L111-112](../../train.py#L111-L112)
- `image`: 렌더 이미지
- `viewspace_points`: 화면 공간 위치. densification 판단용 gradient를 받음
- `visibility_filter`: `radii > 0`
- `radii`: 투영 반지름(px). 0이면 culling됨 → [rasterizer.md](rasterizer.md#culling)

## radii, visibility_filter, sparse_adam

- 판정은 CUDA 래스터라이저가 하고, Python은 결과(`radii`)를 마스크로 씀
  - `visibility_filter = radii > 0` → densification 통계
  - [L181](../../train.py#L181) `visible = radii > 0` → sparse_adam
- 화면 밖 가우시안은 렌더링에 참여하지 않으므로 gradient는 원래 0
- 기본 Adam([L185](../../train.py#L185))은 gradient가 0이어도 **모멘텀 때문에 움직임**. sparse_adam은 `visible`인 것만 업데이트 (Taming 3DGS에서 가져온 2024년 추가 기능)
- `radii > 0`은 "화면 범위 안"이지 "보임"이 아님. 가려진 가우시안도 포함되며, 그 gradient는 0에 가까움

## alpha_mask

[L114-116](../../train.py#L114-L116), 생성은 [scene/cameras.py:42-48](../../scene/cameras.py#L42-L48)
- RGBA 입력이면 알파 채널을 마스크로, 3채널(JPG)이면 전부 1
- 렌더 이미지에 곱해 마스크 0인 영역의 loss 기여를 없앰 (정답 이미지의 해당 영역도 검정이어야 함)
- truck은 JPG라 효과 없음. L48 때문에 `None`이 될 수 없어 L114의 `if`도 항상 참
- 문법: `x[3:4, ...]`는 `(1, H, W)`로 차원 유지 → `(3, H, W)`와 곱할 때 브로드캐스팅

## 기본 하이퍼파라미터 타임라인

[arguments/\_\_init\_\_.py](../../arguments/__init__.py) 기준

```
iter    1  SH 0차, 가우시안 수 = SfM 점 수
iter  500  densification 시작 (100마다)
iter 1000  SH 1차 (이후 1000마다 +1, 최대 3)
iter 3000  opacity 리셋 (3000마다)
iter 15000 densification 종료
```

- 200 iter 실행은 densification, SH 증가, opacity 리셋이 모두 시작 전이라 흐릿했음
- 7000 iter 실행: densify 진행 중에 종료, SH 3차 도달, 리셋 2회 (3000, 6000 직후 loss 상승은 정상)

## 미뤄둔 확인 실험

- 실험 A: [L146](../../train.py#L146) 아래에 250 iter마다 `gaussians.get_xyz.shape[0]`과 `visibility_filter.sum()` 출력 → densification과 opacity 리셋 효과 확인
- 실험 B: `--densify_until_iter 0`으로 densification을 끄고 7000 iter test PSNR 비교

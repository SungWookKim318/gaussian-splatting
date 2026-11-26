# 아이디어: K-Buffer 기반 Gaussian Sorting (초기 구상)

## 문제

- 기존 3DGS의 정렬은 Gaussian 중심점의 Depth만 기준으로 함
- Gaussian의 크기, 회전은 반영되지 않아 겹치는 Gaussian의 실제 깊이 순서와 Blending 순서가 달라질 수 있음 → Popping

## 출발점

- 전통적인 그래픽스에서도 Alpha(반투명) 때문에 정렬이 필요하고, 같은 문제가 있음
- 거기서 쓰는 기법을 차용: A-Buffer, K-Buffer
- A-Buffer는 모든 Fragment를 저장해야 해서 부담이 큼 → K-Buffer 선택

## 아이디어

- 3DGS는 원래 16×16 Tile 기반 렌더링
- Tile마다, 또는 Tile 안의 각 픽셀마다 크기 K의 큐(K-Buffer)를 두고 깊이 순서를 보정
- Tile 단위로 처리하니 큐를 둬도 메모리 사용량이 폭증하지는 않을 것으로 기대

## 확인하고 싶은 것

- Popping이 줄어드는지
- 메모리 사용량, 속도
- Tile 단위 큐와 픽셀 단위 큐의 비교

## 미정

- Gaussian의 "실제 Depth": 계산 과정에서 Gaussian의 가장 가까운 Depth를 구해 쓰고 싶음. 수식으로 어떻게 접근할지는 아직 모름
  - 후보 ① Gaussian당 하나 (Tile 단위): 3σ 타원체의 카메라 기준 최소 깊이 `z_near = μ_z − 3·sqrt(Σ_zz)` (Σ: view 공간 공분산). 기존 정렬 키를 교체하는 방식
  - 후보 ② 픽셀당 하나 (픽셀 단위): 픽셀 광선 `o + t·d`가 3σ 타원체에 처음 들어가는 t. `(o + t·d − μ)ᵀ Σ⁻¹ (o + t·d − μ) = 9`의 작은 근 (변형: 밀도 최대 지점 `t* = dᵀΣ⁻¹(μ − o) / (dᵀΣ⁻¹d)`)

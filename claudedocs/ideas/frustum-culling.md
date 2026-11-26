# 아이디어: Frustum Culling 개선 (초기 구상)

## 문제

- 기존 3DGS는 View Frustum을 크게 만든 Guard Band 안에 Gaussian 중심점이 있는지로 Culling
- Gaussian의 실제 영역은 고려하지 않음 → 화면에 걸치는 Gaussian이 잘려 Popping이 생길 수 있음

## 아이디어

- 중심점 대신 Gaussian의 실제 영역이 View Frustum에 들어오는지 테스트
  - Bounding Sphere
  - Ellipsoid (Covariance) - Plane Intersection
- Projection, 2D Covariance 계산 전에 3D에서 먼저 Culling

## 확인하고 싶은 것

- Popping이 줄어드는지
- 성능이 얼마나 떨어지는지 (또는 연산이 줄어드는지)
- 기존 2D 투영 기반 판정과 3D Frustum 판정의 비교

## 메모: 코드 확인

- 상하좌우 Guard Band 검사는 주석 처리됨 (서브모듈 커밋 `3a07ac2`, 2023-07-02). 지금은 투영 후 화면에서 3σ 범위로 판정
- Near는 아직 중심점 기준 (`z <= 0.2`, [auxiliary.h:166](../../submodules/diff-gaussian-rasterization/cuda_rasterizer/auxiliary.h#L166))

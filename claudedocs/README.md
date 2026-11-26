# 3DGS 개선 과제 노트

## 문서 구성

| 폴더 | 내용 | 추가 규칙 |
|---|---|---|
| [code-analysis/](code-analysis/) | 코드가 **실제로 무엇을 하는지** (사실, 근거 줄 번호) | 분석한 파일/주제마다 하나 |
| [ideas/](ideas/) | 개선 아이디어별 문제 제기, 방향, 실험 계획 | 아이디어마다 하나 |

- 코드 분석: [train.py](code-analysis/train.md), [래스터라이저 읽기 체크리스트](code-analysis/rasterizer.md)
- 아이디어: [Frustum Culling](ideas/frustum-culling.md), [K-Buffer Sorting](ideas/k-buffer-sorting.md)

## 아이디어 현황

| # | 아이디어 | 상태 | 문서 |
|---|---|---|---|
| 1 | Frustum culling 개선 (중심점 대신 Gaussian 실제 영역으로 판정) | 초기 구상 | [frustum-culling.md](ideas/frustum-culling.md) |
| 2 | K-Buffer 기반 Gaussian Sorting (Tile/픽셀마다 K개 큐로 깊이 순서 보정) | 초기 구상 | [k-buffer-sorting.md](ideas/k-buffer-sorting.md) |
| 3 | 학습 파라미터 / VRAM 감소 | 문제 발견 단계 | |
| 4-1 | SH 차수 감소 | `--sh_degree` 옵션뿐이라 직접 구현 조건 미달 → 보조 실험용 | |
| 4-2 | Adaptive SH | 가능하지만 SH 수학 부담 | |
| 4-3 | SH 대체 | 보류 | |
| 5 | Densification 휴리스틱 개선 | 대안 후보: Python만으로 가능, 정량 측정 쉬움 | |

## 과제 요건

- **마감: 2026-11-30, report를 email로 제출**
- 3DGS 또는 후속 논문의 코드를 기반으로 개선
- 아이디어는 후속 논문에서 가져와도 되지만, **개선 내용은 직접 구현**
- AI 도움은 허용되지만 내용은 모두 이해해야 함
- 개선 아이디어, 구현 내용, 결과를 report로 작성
- report 검증은 지정된 skill로 수행
- 아이디어의 신규성(선행 연구 중복 여부)은 요구되지 않음
- 첫 리포트 (이전 리포트, 피드백 없음)
- 구현과 결과 이미지/동영상은 cloud download link로 전달 (연구실 서버 경로 없음)

## 리포트 공통 방침

- **Baseline** = 이 저장소의 기본 설정. 2024년 추가 기능(sparse_adam, depth regularization, anti-aliasing, exposure)은 모두 기본값이 꺼져 있으므로 원 논문 방식과 같음. fused-ssim 사용 여부만 명시
- 저장소에 이미 있는 기능을 되돌렸다가 "직접 구현"이라고 하지 않기 (README와 git 기록에 출처가 남아 있음). 쓰려면 출처를 밝히고 baseline 설정으로 기술
- 논문 내용과 코드에만 있는 내용을 구분해서 인용 → 예: [frustum-culling.md 2장 배경](ideas/frustum-culling.md)

## 실행 환경

- 코드 읽기: macOS (CUDA 없음 → 빌드/실행 불가)
- 실행: Windows PC, RTX 3070 8GB / Ryzen 3900X / RAM 32GB
- 현재 실행 중인 설정:

```bat
python train.py -s ./datasets/tandt/truck -m ./output/truck_7k ^
    --iterations 7000 --resolution 2 --data_device cpu --eval
```

- 학습 후: `python render.py -m <출력>` → `python metrics.py -m <출력>`
- truck은 `--data_device cpu`를 빼도 될 가능성이 큼 (VRAM은 이미지보다 가우시안이 주로 차지)

## 작업 방식 (Git)

- 원격: origin = 본인 fork, upstream = graphdeco-inria
- `base/project_setup`: **공통 기준 브랜치**. 이 노트(아이디어 목록, 코드 분석, 작업 방식)를 여기서 관리
- 아이디어별 구현은 `base/project_setup`에서 브랜치를 따서 분리 작업
  - 공통 문서가 바뀌면 base에 커밋하고, 각 아이디어 브랜치로 merge해서 반영
  - 아이디어 문서(`ideas/*.md`)의 실험 결과 등은 해당 아이디어 브랜치에서 갱신해도 됨
- 기존 `study/frustum-culling`(`8f05d34`)은 base보다 이전 상태. Frustum culling 구현을 시작할 때 base에서 새로 따거나 base를 merge해서 사용
- CUDA 수정 대상은 서브모듈 `submodules/diff-gaussian-rasterization` 안에 있음. 서브모듈의 원격은 원 저자 저장소라 push 방법을 따로 정해야 함

## 남은 확인 사항

- [x] 리포트 검증용 skill 확인 (과제 요건에 주소 기록)
- [ ] 서브모듈 수정분 관리 방법 (fork 등)
- [ ] 수정한 CUDA 코드를 Windows PC에서 재빌드하는 절차

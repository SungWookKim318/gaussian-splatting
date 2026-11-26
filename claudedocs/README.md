# 3DGS 개선 과제 노트

## 문서 구성

| 폴더 | 내용 | 추가 규칙 |
|---|---|---|
| [code-analysis/](code-analysis/) | 코드가 **실제로 무엇을 하는지** (사실, 근거 줄 번호) | 분석한 파일/주제마다 하나 |
| [ideas/](ideas/) | 개선 아이디어별 문제 제기, 방향, 실험 계획 | 아이디어마다 하나 |

- 코드 분석: [train.py](code-analysis/train.md), [래스터라이저 (culling, 정렬)](code-analysis/rasterizer.md)
- 아이디어: [Frustum Culling](ideas/frustum-culling.md)

## 아이디어 현황

| # | 아이디어 | 상태 | 문서 |
|---|---|---|---|
| 1 | Frustum culling 개선 (타원체 정확 판정) | **진행 중** | [frustum-culling.md](ideas/frustum-culling.md) |
| 2 | 가우시안 형태를 고려한 depth sorting | 보류: 팝핑 주원인이지만 난이도 높음 (StopThePop) | |
| 3 | 학습 파라미터 / VRAM 감소 | 문제 발견 단계 | |
| 4-1 | SH 차수 감소 | `--sh_degree` 옵션뿐이라 직접 구현 조건 미달 → 보조 실험용 | |
| 4-2 | Adaptive SH | 가능하지만 SH 수학 부담 | |
| 4-3 | SH 대체 | 보류 | |
| 5 | Densification 휴리스틱 개선 | 대안 후보: Python만으로 가능, 정량 측정 쉬움 | |

## 과제 요건

- 3DGS 또는 후속 논문의 코드를 기반으로 개선
- 아이디어는 후속 논문에서 가져와도 되지만, **개선 내용은 직접 구현**
- AI 도움은 허용되지만 내용은 모두 이해해야 함
- 개선 아이디어, 구현 내용, 결과를 report로 작성
- report 검증은 지정된 skill로 수행

## 리포트 공통 방침

- **Baseline** = 이 저장소의 기본 설정. 2024년 추가 기능(sparse_adam, depth regularization, anti-aliasing, exposure)은 모두 기본값이 꺼져 있으므로 원 논문 방식과 같음. fused-ssim 사용 여부만 명시
- 저장소에 이미 있는 기능을 되돌렸다가 "직접 구현"이라고 하지 않기 (README와 git 기록에 출처가 남아 있음). 쓰려면 출처를 밝히고 baseline 설정으로 기술
- 논문 내용과 코드에만 있는 내용을 구분해서 인용 → [rasterizer.md의 논문 vs 코드](code-analysis/rasterizer.md#논문-vs-코드)

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

## Git

- 브랜치: `study/frustum-culling` (origin = 본인 fork, upstream = graphdeco-inria)
- CUDA 수정 대상은 서브모듈 `submodules/diff-gaussian-rasterization` 안에 있음. 서브모듈의 원격은 원 저자 저장소라 push 방법을 따로 정해야 함

## 남은 확인 사항

- [ ] 리포트 검증용 skill이 무엇인지
- [ ] 서브모듈 수정분 관리 방법 (fork 등)
- [ ] 수정한 CUDA 코드를 Windows PC에서 재빌드하는 절차

# :loud_sound: AudioSeal: 선제적 로컬라이즈드 워터마킹

<a href="https://www.python.org/"><img alt="Python" src="https://img.shields.io/badge/-Python 3.8+-blue?style=for-the-badge&logo=python&logoColor=white"></a>
<a href="https://black.readthedocs.io/en/stable/"><img alt="코드 스타일: 검은색" src="https://img.shields.io/badge/code%20style-black-black.svg?style=for-the-badge&labelColor=gray"></a>

음성 로컬라이즈드 워터마킹 방법인 오디오씰을 소개합니다.
워터마킹의 견고성을 손상시키지 않으면서도 최첨단 검출기 속도를 자랑하는 오디오씰을 소개합니다. 오디오에 워터마크를 삽입하는 제너레이터와 편집이 있는 경우에도 긴 오디오에서 워터마킹된 조각을 감지하는 검출기를 함께 훈련합니다.
오디오씰은 샘플 수준(1/16k 해상도)에서 자연음과 합성음 모두에 대해 최첨단 탐지 성능을 달성하며, 신호 품질의 변경을 제한하고 다양한 유형의 오디오 편집에 견고하게 대응합니다.
오디오씰은 기존 모델을 훨씬 능가하는 고속 싱글 패스 디텍터로 설계되어 최대 2배 빠른 감지 속도를 구현하므로 대규모 실시간 애플리케이션에 이상적입니다.

자세한 내용은 [논문](https://arxiv.org/pdf/2401.17264.pdf)에서 확인할 수 있습니다.


![fig](https://github.com/facebookresearch/audioseal/assets/1453243/5d8cd96f-47b5-4c34-a3fa-7af386ed59f2)


# :메이트: 설치

오디오씰에는 파이썬 >=3.8, 파이토치 >= 1.13.0, [omegaconf](https://omegaconf.readthedocs.io/), [julius](https://pypi.org/project/julius/), numpy가 필요합니다. PyPI에서 설치하려면:

```
pip 설치 오디오실
```

소스에서 설치하려면: 이 리포지토리를 복제하고 편집 가능한 모드로 설치합니다:

```
git clone https://github.com/facebookresearch/audioseal
cd audioseal
pip 설치 -e .
```

# :기어: 모델

다음 모델에 대한 체크포인트를 제공합니다:

- 오디오씰 생성기](src/cards/audioseal_wm_16bits.yaml).
  오디오 신호(파형)를 입력으로 받고, 입력과 동일한 크기의 워터마크를 출력하며, 입력에 추가하여 워터마킹할 수 있습니다.
  선택적으로 워터마크에 인코딩할 16비트 비밀 메시지를 입력으로 받을 수도 있습니다.
- 오디오씰 검출기](src/cards/audioseal_detector_16bits.yaml).
  오디오 신호(파형)를 입력으로 받고, 오디오의 각 샘플마다(1/16k 초마다) 입력에 워터마크가 포함될 확률을 출력합니다.
  선택적으로 워터마크에 인코딩된 비밀 메시지를 출력할 수도 있습니다.

이 메시지는 선택 사항이며 탐지 출력에 영향을 미치지 않습니다. 예를 들어 모델 버전을 식별하는 데 사용될 수 있습니다(최대 $2**16=65536$까지 선택 가능).

**참고**: 자신만의 워터마커를 만들고자 하는 분들을 위해 교육 코드를 공개하기 위해 노력하고 있습니다. 계속 지켜봐 주세요!

# :주판: 사용법

오디오씰은 오디오 샘플에서 워터마킹하고 워터마크를 감지할 수 있는 간단한 API를 제공합니다. 사용 예시:

```python

에서 오디오씰을 가져옵니다.

# 모델 이름은 audioseal/cards에 있는 YAML 카드 파일 이름에 해당합니다.
모델 = 오디오씰.로드_제너레이터("audioseal_wm_16bits")

# 다른 방법은 체크포인트에서 직접 로드하는 것입니다.
# 모델 = 워터마커.부터_사전학습(체크포인트_경로, 디바이스 = wav.디바이스)

워터마크 = model.get_watermark(wav)

# 선택 사항: 워터마크에 삽입할 16비트 메시지를 추가할 수 있습니다.
# msg = torch.randint(0, 2, (wav.shape(0), model.msg_processor.nbits), device=wav.device)
# watermark = model.get_watermark(wav, message = msg)

워터마크_오디오 = wav + 워터마크

detector = AudioSeal.load_detector("audioseal_detector_16bits")

# 하이레벨에서 메시지를 탐지합니다.
결과, 메시지 = detector.detect_watermark(watermarked_audio)

print(result) # 결과는 오디오에 워터마크가 있을 확률을 나타내는 실수입니다,
print(message) # 메시지는 16비트의 바이너리 벡터입니다.


# 로우 레벨에서 메시지를 감지합니다.
결과, 메시지 = 검출기(워터마크_오디오)

# 결과는 각 프레임에 대한 워터마킹의 확률(양수 및 음수)을 나타내는 배치 크기 x 2 x 프레임의 텐서입니다.
# 워터마킹된 오디오는 결과[:, 1, :] > 0.5가 되어야 합니다.
print(result[:, 1 , :])

# 메시지는 각 비트가 1이 될 확률을 나타내는 배치 x 16 크기의 텐서입니다.
# 탐지기가 오디오에서 워터마킹을 감지하지 못하면 메시지는 임의의 텐서가 됩니다.
print(message)
```

<!-- # 기여하고 싶으신가요?

 개선 사항이나 제안이 있는 [풀 리퀘스트](https://github.com/fairinternal/fair-getting-started-recipe/pulls)를 환영합니다.
 이슈에 플래그를 지정하거나 개선 사항을 제안하고 싶지만 어떻게 실현해야 할지 모르겠다면 [GitHub 이슈](https://github.com/fairinternal/fair-getting-started-recipe/issues)를 생성하세요.


# 고마운 분들:
* 기여와 피드백을 제공해 주신 Jack Urbaneck, Matthew Muckley, Pierre Gleize, Ashutosh Kumar, Megan Richards, Haider Al-Tahan, Vivien Cabannes에게 감사드립니다.
* CIFAR10 [PyTorch 튜토리얼](https://pytorch.org/tutorials/beginner/blitz/cifar10_tutorial.html
)의 기반이 되는 교육
* 코드 구성에 대한 영감을 주는 [Hydra Lightning 템플릿](https://github.com/ashleve/lightning-hydra-template) -->

# 라이선스

- 이 리포지토리의 코드는 [라이선스 파일](LICENSE)에 있는 MIT 라이선스에 따라 릴리스됩니다.
- 이 리포지토리의 모델 가중치는 [LICENSE_weights 파일](LICENSE_weights)에 있는 CC-BY-NC 4.0 라이선스에 따라 릴리스됩니다.

# 메인테이너:
- [Tuan Tran](https://github.com/antoine-tran)
- 하디 엘사하르](https://github.com/hadyelsahar)
- [피에르 페르난데스](https://github.com/pierrefdz)
- 로빈 산 로만](https://github.com/Sparker17)

# 인용

이 리포지토리가 유용하다고 생각되면 별 :별: 표시를 하고 다음과 같이 인용해 주세요:

```
@article{sanroman2024proactive,
  title={지역화된 워터마킹으로 음성 복제를 사전에 탐지},
  저자={San Roman, Robin, Fernandez, Pierre, Elsahar, Hady, D´efossez, Alexandre, Furon, Teddy, Tran, Tuan},
  저널={arXiv 사전 인쇄},
  year={2024}
}
```

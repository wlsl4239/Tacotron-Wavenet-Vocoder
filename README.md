# Tocotron + Korean TTS
Tacotron모델을 구현하여 한국어 TTS구현하는 project입니다.

Based on 
- https://github.com/keithito/tacotron
- https://github.com/carpedm20/multi-speaker-tacotron-tensorflow
- https://github.com/ibab/tensorflow-wavenet
- https://github.com/r9y9/wavenet_vocoder
- https://github.com/Rayhane-mamah/Tacotron-2
- https://github.com/hccho2/Tacotron-Wavenet-Vocoder


## Tacotron History
- [keithito](https://github.com/keithito/tacotron)가 Tocotron을 처음 구현하여 공개하였고, 이를 기반으로 한국어를 적용한 [carpedm20](https://github.com/carpedm20/multi-speaker-tacotron-tensorflow)의 구현이 있다.
- carpedm20의 구현은 deep voice2에서 제안하고 있는 multi-speaker도 같이 구현했다.
- Tacotron모델에서는 vocoder로 Griffin Lim 알고리즘을 사용하고 있다.


## Tacotron 2
- [Tacotron2](https://arxiv.org/abs/1712.05884)에서는 모델 구조도 바뀌었고, Location Sensitive Attention, Stop Token, Vocoder로 Wavenet을 제안하고 있다.
- Tacotron2의 구현은 [Rayhane-mamah](https://github.com/Rayhane-mamah/Tacotron-2)의 것이 있는데, 이 역시, keithito, r9y9의 코드를 기반으로 발전된 것이다.
- Tacotron2에서는 Locatin Sensitive Attention과 Stop Token이 결합하여 Tacotron1보다 빠른게 수렴됨.


## This Project
* Tacotron 모델에 Wavenet Vocoder를 적용하는 것이 1차 목표이다.
* Tacotron과 Wavenet Vocoder를 같이 구현하기 위해서는 mel spectrogram을 만들때 부터, 두 모델 모두에 적용할 수 있도록 만들어 주어야 한다(audio의 길이가 hop_size의 배수가 될 수 있도록). 이렇게 해야, wavenet training할 때, upsampling이 원할하다.
* Tacotron2의 stop token이나 Location Sensitive Attention을 Tacotron1에 적용하는  그렇게 효과적이지 못했다(제 경험상).
* carpedm20의 구현과 다른 점
    * Tensorflow 1.3에서만 실행되는 carpedm20의 구현을 tensorflow 1.8이상에서도 작동할 수 있게 수정. Tensorflow가 버전이 업되면서, AttentionWrapperState에서 attention_state가 추가되었는데, 이 부분을 맞게 수정해 줌.
    * dropout bug 수정 
	* DecoderPrenetWrapper, AttentionWrapper 순서를 바로 잡음. 이렇게 해야 keithito의 구현과 같아지고 논문에서의 취지와도 일치함. AttentionWrapper를 DecoderPrenetWrapper가 감싸야, Prenet의 결과가 AttentionWrapper의 입력으로 들어간다.
	* mel spectrogram 생성 방식을 keithito의 구현 방법으로 환원(이것도 keithito가 추후에 수정한 것임). 이렇게 mel spectrogram 생성방식을 바꾸면 train 속도가 많이 향상됨. 20k step 이상 train해야 소리가 들리기 시작했는데, 이렇게 하면 8k step부터 소리가 들린다.
	* padding이 된 곳에 Attention이 가지 않도록 보완.
	* Attention 모델 추가: LocationSensitiveAttention, GmmAttention 등
* ibab의 wavenet 구현과 다른 점
	* [fast generation](https://github.com/tomlepaine/fast-wavenet)을 위해서 tf.Variable을 이용해서 구현했다. 이 project에서는 Tensorflow middle level api tf.layers.conv1d를 이용하여, 코드를 이해하기 쉽게 만들었다.
* 참고 코드 등에서의 복잡한 option을 많이 줄였습니다.


## 단계별 실행

### 실행 순서
- data 만들기
- tacotron training 후, synthesize.py로 test.
- wavenet training 후, generate.py로 test(tactron이 만들지 않은 mel spectrogram으로 test할 수도 있고, tacotron이 만든 mel spectrogram을 사용할 수도 있다.)
- 2개 모델 모두 train 후, tacotron에서 생성한 mel spectrogram을 wavent에 local condition으로 넣어 test하면 된다.


### Data 만들기
- audio data(e.g. wave 파일)을 다운받고,  1~3초(최대 12초)길이로 잘라주는 작업을 해야 한다. 그리고 잘라진 audio와 text(script)의 sync를 맞추는 것은 고단한 작업이다. Google Speech API를 이용하는 것도 하나의 방법이 될 수 있다.
- Google Speech API로 생성한 text의 Quality가 좋지 못하기 때문에, 수작업으로 (아주) 많이 보정해 주어야 한다.
- 특별히 data를 확보할 방법이 없으면, [carpedm20](https://github.com/carpedm20/multi-speaker-tacotron-tensorflow)에서 설명하고 있는대로 하면 된다. 여기서는 data를 다운받은 후, 침묵(silence)구간을 기준으로 자른 후, Google Speech API를 이용하여 text와 sync를 맞추고 있다.
- 한글 data는 [KSS Dataset](https://www.kaggle.com/bryanpark/korean-single-speaker-speech-dataset)가 있고, 영어 data는 [LJ Speech Dataset](https://keithito.com/LJ-Speech-Dataset/), [VCTK corpus](http://homepages.inf.ed.ac.uk/jyamagis/page3/page58/page58.html) 등이 있다.
- KSS Dataset이나 LJ Speech Dataset는 이미 적당한 길이로 나누어져 있기 때문에, data의 Quality는 우수하다.
- 각 speaker별로 wav 파일을 특정 directory에 모은 후, text와 wav파일의 관계를 설정하는 파일을 만든 후, preprocess.py를 실행하면 된다. 다음의 예는 son.py에서 확인 할 수 있듯이 'son-recognition-All.json'에 필요한 정보를 모아 놓았다.
- 각자의 상황에 맞게 preprocessing하는 코드를 작성해야 한다. 이 project에서는 son, moon 2개의 example이 포함되어 있다.
> python preprocess.py --num_workers 8 --name son --in_dir .\datasets\son --out_dir .\data\son
- 위의 과정을 거치든 또는 다른 방법을 사용하든 speaker별 data 디렉토리에 npz파일이 생성되면 train할수 있는 준비가 끝난다. npz파일에는 dict형의 data가 들어가게 되는데, key는 ['audio', 'mel', 'linear', 'time_steps', 'mel_frames', 'text', 'tokens', 'loss_coeff']로 되어 있다. 중요한 것은 audio의 길이가 mel, linear의 hop_size 배로 되어야 된다는 것이다.


### Tacotron Training
- train_tacotron.py 내에서 '--data_paths'를 지정한 후, train할 수 있다.
```
parser.add_argument('--data_paths', default='.\\data\\moon,.\\data\\son')  
parser.add_argument('--data_paths', default='.\\data\\moon,.\\data\\son,\\data\\kss')
```
- train을 이어서 계속하는 경우에는 '--load_path'를 지정해 주면 된다. (checkpoint파일이 있어야 에러가 안뜸.)
```
parser.add_argument('--load_path', default='logdir-tacotron/moon+son_2018-12-25_19-03-21')
```
  
- train_tacotron.py 내에서 '--test_interval'의 default값을 지정하는 것으로 지정 step마다 test를 수행함.

- train_tacotron.py 내에서 '--checkpoint_interval'의 default값을 지정하는 것으로 지정 step마다 checkpoint 파일을 작성함. 이 checkpoint 파일이 있어야 train을 이어서 하거나, 음성을 생성할 수 있다.

- speaker가 1명 일 때는, hparams의 model_type = 'single'로 하고 train_tacotron.py 내에서 '--data_paths'를 1개만 넣어주면 된다. (굳이 싱글로 할 필요는 없다.)
```
parser.add_argument('--data_paths', default='D:\\Tacotron-Wavenet-Vocoder\\data\\moon')
```

- 하이퍼파라메터를 hparmas.py에서 argument를 train_tacotron.py에서 다 설정했기 때문에, train 실행은 다음과 같이 단순합니다.
```
> python train_tacotron.py
```
- train 후, 음성을 생성하려면 다음과 같이 하면 된다. '--num_speaker', '--speaker_id'는 잘 지정되어야 한다.
```
> python synthesizer.py --load_path logdir-tacotron/moon+son_2018-12-25_19-03-21 --num_speakers 2 --speaker_id 0 --text "오스트랄로피테쿠스 아파렌시스는 멸종된 사람족 종으로, 현재에는 뼈 화석이 발견되어 있다." 
```

### 한국어 폰트 깨짐문제
- utils/plot.py의 10, 11번째 줄을 주석해제 한 후 plot.py를 실행.


### Result
- tacotron모델에서는 griffin lim vocoder를 통해서 audio sample을 만들어 내는데, 음질이 나쁘지 않다.
- [sample-son](https://www.dropbox.com/s/7bvlwjy09do5yxb/son-%EC%98%A4%EC%8A%A4%ED%8A%B8.wav?dl=0): tacotron(griffin-lim, step 106K)
- [sample-moon](https://www.dropbox.com/s/y1kgmzka0cxp81d/moon-%EC%98%A4%EC%8A%A4%ED%8A%B8.wav?dl=0): tacotron(griffin-lim, step 106K)
- [sample-son](https://www.dropbox.com/s/feptz8bfx7vsxlj/son-wavenet.wav?dl=0): tacotron + wavenet vocoder(step 245K)
- [sample-moon](https://www.dropbox.com/s/rcz29g64v6pyzhv/moon-wavenet.wav?dl=0): tacotron + wavenet vocoder(step 245K)


### 음성을 처음 공부하는 분들께
* Tensorflow의 [Simple Audio Recognition](https://www.tensorflow.org/tutorials/sequences/audio_recognition)은 음성관련 공부를 처음 시작하는 사람들에게 좋은 시작점이 될 수 있다.
* 이를 통해, wav로 된 음성을 stft으로 변환하고 다시 mel spectrogram으로 변환하는 과정을 공부할 수 있다. 
* Simple Audio Recognition을 공부한 후에는 Tacotron을 공부할 수 있수도 있지만, 딥러닝에서의 기본인 RNN, Attention에 관한 공부를 미리해 두며 더욱 좋다.
* 이 [자료](https://github.com/hccho2/hccho2.github.io/blob/master/DeepLearning.pdf)는 음성인식 기초, Tacotron, Wavenet 등에 관한 내용을 정리한 것입니다(page 133).
* 또한 Tensorflow에서 Attention Mechanism이 어떻게 작동되는지에 관한 자료도 정리되어 있습니다(page 69).
* Facebook TFKR의 [글](https://www.facebook.com/groups/TensorFlowKR/permalink/813421485665578/)도 참고하세요.
* 




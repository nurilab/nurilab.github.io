---
layout: post
title: 'KicomAV 가이드 (악성코드 패턴 추가 방법 포함)'
author: Seok-Eun Park
date: 2020-04-01 09:37
tags: [kicomav,opensource]
---
## 키콤백신이란?

키콤백신은 하우리 창업자(바이로봇 개발 및 악성코드 분석 총괄)이자 현재 누리랩 대표이신 최원혁씨가 만든 안티 바이러스이다. 또한 키콤백신은 현재 오픈소스로 공개되어 있으며, 꾸준히 업데이트 진행중이다.

![키콤백신 홈페이지 메인 화면](/files/kicomav_guide_1.png)

## 키콤백신 설치 방법

키콤백신은 현재 0.33버전까지 업데이트되어 공개되어 있다. 키콤백신 설치 방법은 다음과 같다.
키콤백신(http://www.kicomav.com/)사이트에 들어가 아래로 조금 내리면 그림과 같이 다운로드 링크가 보인다. 해당 링크를 클릭하면 kicomav-master.zip이라는 압축파일을 다운로드 받게 된다.

![키콤백신 다운로드](/files/kicomav_guide_2.png)

압축을 풀고 들어가게 되면, 아래 그림과 같은 파일들이 보인다. 키콤백신 설치 후 압축을 풀었다면, 이제 키콤백신을 동작시키 위한 몇가지 모듈을 설치해야 한다.

![키콤백신 압축 해제](/files/kicomav_guide_3.png)

키콤백신은 파이썬2.7로 만들어졌으며, Pylzma와 yara 모듈을 사용한다. 키콤백신을 구동시키기 위해서는 아래 3가지를 설치해야 한다. 파이썬은 아래 링크를 클릭하여 설치하면 되며, Pylzma 및 yara는 파이썬의 pip 명령어를 통해 설치하면 된다.

- 파이썬 2.7 : <https://www.python.org/ftp/python/2.7.14/python-2.7.14.msi>
- Pylzma : <http://www.lfd.uci.edu/~gohlke/pythonlibs/#pylzma>
- yara : <https://pypi.python.org/pypi/yara-python>

```
pip install pylzma-0.4.9-cp27-cp27m-win32.whl
```

위 과정을 모두 거쳤다면 윈도우상에서 KICOM AV를 사용하기위한 마지막 절차로 빌드과정을 거쳐야 한다. 빌드 과정을 거치게 되면 다음과 같은 화면이 출력된다. 해당 과정까지 모두 거쳤다면 키콤백신을 사용할 준비를 모두 마친 것이다.

![키콤백신 빌드](/files/kicomav_guide_4.png)

## 키콤백신 검사

키콤백신은 현재 커맨드라인 버전으로만 개발되어 있다. 그렇기에 명령 프롬프트를 이용해 동작시켜야 한다. 바로 이전에 빌드 과정을 거친 후 다시 키콤백신 폴더로 가보면 Release라는 폴더가 있는 것을 확인할 수 있다.

![빌드 후 키콤백신 폴더](/files/kicomav_guide_5.png)

Release폴더에 들어가면 k2.py보게 되는데 이 파일이 백신을 구동시키는 파일이다. 백신을 실행시키는 방법은 아래 명령어를 통해 실행할 수 있다.

```
python k2.py [path] [options]
```

간단한 악성코드 검사를 위해 아래 그림과 같이 ```python k2.py .``` 를 실행하면, 현재 폴더를 검사하게 된다.

![k2.py 실행](/files/kicomav_guide_6.png)

다음으로 키콤백신에서 사용 가능한 옵션들은 아래 그림과 같다.

![k2.py 옵션 목록](/files/kicomav_guide_7.png)

키콤백신의 옵션은 악성코드 발견 시에만 검사 결과를 출력한다. 필자는 임의의 악성파일을 Release에 옮겨놓았다. 이제 해당 파일에 대한 검사를 실시 해보았다. 사용한 옵션은 다음과 같다.

- 모든 결과를 보고 싶을 경우 : -I 옵션
- 압축 파일 내부 검사를 하고 싶을 경우 : -r 옵션

아래 그림은 압축 파일에 숨은 악성코드 검사를 실시한 것이다. 검사 결과 virus.zip 이라는 폴더가 있고 그안에 00e80…~ 라는 파일이 있고, eb3a6..~ 라는 파일이 Attached 되있는 것을 알 수 있다. 이렇게 키콤백신을 이용해 압축파일 내부까지 확인 할 수 있다. 또한 키콤 백신의 또 다른 기능은 한글 파일 같은 문서에 숨겨져 있는 파일까지 검사가 가능하다.

![압축 파일에 숨은 악성코드 검사](/files/kicomav_guide_8.png)

아래 그림은 바로 위에서 설명한 한글 파일내에 virus.zip이라는 파일이 숨겨져 있을 경우의 검사 결과이다. 위에서 검사한 virus.zip 파일을 그림와 같이 Word 문서에 숨긴 후 검사를 실시한 결과이다. 검사 결과를 보면 특이점을 발견할 수 있는데, 타 백신과 달리 키콤 백신은 문서파일 또한 압축파일로 생각하기 때문에 상세하게 검사를 하는 것을 확인할 수 있다.

![Word 파일내 숨겨진 악성파일](/files/kicomav_guide_9.png)

![Word 파일내 악성파일 검사](/files/kicomav_guide_10.png)

## 키콤백신 패턴 추가

다음은 키콤백신에 악성코드 패턴을 추가하는 방법이다. 가장 먼저 악성코드의 이름을 하나로 통일하기 위해 모든 파일명을 SHA256 형태의 이름으로 변경하는 작업을 실시한다. 해당 작업을 위해 사용하는 파일은 ```xsha.py```라는 파일이다. ```xsha.py``` 사용법은 다음과 같다.

```
xsha.py [src 폴더명] [target 폴더명] 
```

위 명령어에 대한 설명을 하자면, src폴더에 존재하는 악성코드를 target 폴더명으로 이름을 변경하여 복사한다. 아래 그림은 ```malware_md5``` 폴더를 ```1``` 폴더로 이름을 변경해서 복사하는 모습이다.

![xsha.py](/files/kicomav_guide_11.png)

![1 폴더로 복사된 모습](/files/kicomav_guide_12.png)

다음은 ```fileformat.exe```를 이용해 ```1``` 폴더에 있는 파일들을 파일 포맷별로 분류하여 각 포맷에 맞는 폴더로 복사한다.

![fileformat.exe](/files/kicomav_guide_13.png)

다음은 분류된 파일 중 pe파일을 분석하여 악성파일 패턴을 등록해야 한다. 키콤백신의 옵션 중 ```–verbose```를 이용해 주요 정보를 확인해야 한다. 아래 그림을 보면 pe파일의 주요 정보가 보인다. 분석 결과 악성파일로 판단될 경우 검사할 섹션의 크기와, 그 섹션의 md5를 추가해줘야 한다. 

![pe파일 분석](/files/kicomav_guide_14.png)

추가하는 방법은 다음과 같다. ```emalware.mdb``` 파일에 패턴을 작성하면 되는데, 아래 그림처럼 ```섹션의 크기:md5:악성코드 이름 #주석문```을 작성한 후 저장하면 된다. 만약 계속해서 패턴을 추가하고자 할때는 sort를 해줘야 한다. 이유는 악성코드 패턴의 파편화를 막기 위해서이다. (키콤백신의 속도가 빨라진다.) 

![emalware.mdb에 패턴 추가](/files/kicomav_guide_15.png)


마지막으로 패턴을 빌드하는 방법이다. ```Sigtool_md5.py```를 이용해 emalware.mdb파일을 빌드하면 된다. 패턴 빌드를 하면  그림과 같이 4개의 파일이 생긴다.(emalware.c01, i01, s01, n01)

![패턴 빌드](/files/kicomav_guide_16.png)

생성된 파일을 키콤백신 ```plugins``` 폴더로 이동하면 키콤백신이 인식을 한다. 검사 결과는 아래 그림을 통해 확인 가능하다.

![악성코드 검사 결과](/files/kicomav_guide_17.png)


## 참고 자료

- <http://www.kicomav.com/>
- 키콤백신 DB 과정.pdf
- <https://github.com/hanul93/kicomav>
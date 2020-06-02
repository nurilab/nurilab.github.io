---
layout: post
title: '초간단 실시간 감시기 만들기 (2)'
author: Bo-Seung Ko
date: 2020-05-25 07:00
tags: [kicomav,opensource,driver]
---

<fieldset style="margin:0px 0px 20px 0px;padding:5px;"><legend><span><strong style="font-weight:bold;">연재 순서</strong></span></legend><!--Creative Commons License--><div style="float: left; width: 88px; margin-top: 3px;"><img alt="Creative Commons License" style="border-width: 0" src="/files/images/exclamationmark.png"/></div><div style="margin-left: 92px; margin-top: 3px; text-align: justify;">
<p style="margin: 0;"><a href="/2020/05/18/kicomav_driver_1/">첫번째 글: 실시간 감시기란</a></p>
<p style="margin: 0; background:#ddd;">두번째 글: 미니필터 드라이버를 사용하여 실시간 파일 I/O 확인하기</p>
<p style="margin: 0;"><a href="/2020/06/01/kicomav_driver_3/">세번째 글: KicomAV 엔진을 사용하여 유해한 파일인지 확인하기</a></p>
</div></fieldset>


## WDK 설치하기

WDK는 Windows Driver Kit의 약어이다. 과거 DDK에서부터 최신 WDK의 변천사는 아래 링크에서 확인할 수 있다.

- <https://en.wikipedia.org/wiki/Windows_Driver_Kit>

윈도우 운영체제에서 동작하는 드라이버를 개발하려면 WDK를 설치해야 한다. 우리는 Windows Driver Kit 7.1.0 버전을 사용할 것이다. 이 버전을 설치하면 다양한 드라이버 샘플 소스코드를 제공받을 수 있고 Visual Studio 빌드 환경을 빌리지 않아도 Command 창을 사용하여 간단하게 컴파일을 할 수 있다. 다양한 환경에서 테스트해보니 필자가 보기에는 Windows Driver Kit 7.1.0 버전으로 하는 것이 가장 쉬울 것 같다.

반면에 Windows Driver Kit 8.0 부터는 설치를 해도 드라이버 샘플 소스코드가 포함되어 있지 않다. 드라이버 샘플 소스코드를 받으려면 Github에서 다운로드 받아야 한다. 그리고 샘플 코드들이 모두 Visual Studio 프로젝트로 구성되어 있어서 WDK 버전에 해당하는 적절한 Visual Studio을 설치해야 한다. 이에 대한 자세한 설명은 아래의 링크에서 확인할 수 있다. (이 과정이 간단해 보이지만 그리 순탄하지만은 않다.)

- <https://github.com/microsoft/Windows-driver-samples>
- <https://docs.microsoft.com/ko-kr/windows-hardware/drivers/download-the-wdk>

아무튼 우리는 Windows Driver Kit 7.1.0 버전을 사용할 것이다. Windows Driver Kit 7.1.0 버전은 사실 Windows 7을 위한 드라이버 개발 키트이다. 하지만 Windows XP용 드라이버 개발을 지원하고 Windows 10 에서 동작하는 드라이버를 개발하는 데에도 문제가 되지 않는다. 실제로 우리는 최신 윈도우 10 환경에서 실시간 감시기의 동작을 확인할 것이다. 

Windows Driver Kit 7.1.0 버전은 아래 과정에 따라서 설치하면 된다. 검색 키워드는 “WDK 7.1 download” 이다. 검색 결과인 아래 링크에서 WDK ISO 파일을 다운로드 받을 수 있다.

- <https://www.microsoft.com/en-us/download/details.aspx?id=11800>

![WDK 7.1 다운로드 페이지](/files/driver2_1.png)

다운로드가 완료되면 임의의 볼륨으로 마운트하거나 압축을 풀어서 “KitSetup.exe” 파일을 실행한다.

![WDK 7.1 설치 파일](/files/driver2_2.png)

설치 과정에서 아래와 같은 화면이 나오면 그림과 같이 체크해준다. 이것은 최소한의 설치 항목이다. 원한다면 모두 체크해도 상관없다.

![WDK 7.1 설치 구성 설정](/files/driver2_3.png)

OK 버튼을 클릭하면 아래의 그림과 같이 설치 경로 입력 창이 뜬다. 기본 값으로 입력하고 OK 버튼을 클릭한다.

![WDK 7.1 설치 경로 입력](/files/driver2_4.png)

설치 도중에 .NET 프레임워크 설치가 필요하다는 창이 뜨면 먼저 .NET 프레임워크를 설치한 후 WDK 설치를 진행한다.

![WDK 7.1 설치 완료](/files/driver2_5.png)

WDK 설치가 완료되면 Finish 버튼을 클릭한다. 이제 WDK 설치가 완료되었다. “C:\WinDDK\7600.16385.1\src” 경로에 다양한 드라이버 샘플 소스코드가 보일 것이다.

![WDK 7.1 샘플 소스코드 경로](/files/driver2_6.png)

우리는 “src\filesys\miniFilter\scanner” 라는 미니필터 드라이버 샘플 소스코드를 기반으로 진행할 것이다. 설치와 동시에 소스코드를 확인할 수 있다는 장점 외에도 Windows Driver Kit 7.1.0 버전의 좋은 점은 오프라인 WDK 도움말 문서를 제공해주는 마지막 버전이라는 것이다. 인터넷이 되지 않는 환경에서도 드라이버 개발에 필요한 함수의 도움말을 확인할 수 있고 드라이버 개발에 필요한 디자인 가이드를 찾아볼 수 있다. 그리고 Visual Studio 빌드 환경이 없어도 Command 창을 사용하여 빌드할 수 있다.

![WDK 7.1 설치 후 프로그램 메뉴 (Windows 10 환경)](/files/driver2_7.png)

## Scanner 샘플 소스코드

Scanner 샘플은 아주 간단한 미니필터 드라이버(filter)와 이와 통신하는 어플리케이션(user)으로 디렉터리가 구성되어 있다. inc 폴더에는 두 모듈간 공유하는 자료구조를 정의하는 헤더 파일이 존재한다. scanner.inf 파일은 미니필터 드라이버를 설치할 때 사용하는 설치 파일이다.

![scanner 샘플 프로그램 경로](/files/driver2_8.png)

우리는 32비트 환경을 타겟으로 하여 진행할 것이다. 명령창을 실행하고 아래의 그림과 같이 입력한다.

![scanner 샘플 프로그램 빌드](/files/driver2_9.png)

```
setenv.bat c:\WinDDK\7600.16385.1 chk Win7
```

우선 setenv.bat 배치 파일을 사용하여 위의 명령을 실행해서 디버그용 32비트 빌드 환경을 설정하였다. 그런 다음 scanner 샘플 경로로 이동하여 “build -cZ” 명령을 실행하여 “Rebuild All” 빌드를 수행하였다. 그 결과 각각 filter, user 디렉터리 안에 타겟 빌드에 해당하는 디렉터리가 생성되었고 각각 scanner.sys, scanuser.exe 파일이 생성되었다. 아래의 그림은 이 두개의 빌드 결과물을 scanner.inf 파일과 동일한 위치에 복사한 것을 보여준다.

![scanner 빌드 결과물](/files/driver2_10.png)

우리는 Windows 10 32비트 환경에서 scanner 샘플을 사용할 것이다. 드라이버를 개발하다 보면 블루스크린이 발생하는 경우가 종종 발생할 수도 있기 때문에 되도록이면 VMware나 VirtualBox 와 같은 가상환경에서 테스트하기를 권장한다.

Windows 10 32비트 환경이 준비되었다면 scanner.inf, scanner.sys, scanuser.exe 파일을 테스트 가상환경으로 복사한다. 어느 경로든 상관없다. 

![scanner 샘플 설치 준비](/files/driver2_11.png)

이제 scanner.inf 파일에서 오른쪽 마우스를 클릭하여 “설치” 메뉴를 클릭하면 scanner.sys 파일은 “System32\drivers” 디렉터리에 복사되고 아래의 그림과 같이 서비스 키가 생성된다. 말 그대로 설치만 된 것이다.

![scanner.sys 미니필터 드라이버에 대한 서비스 정보](/files/driver2_12.png)

Windows Vista 64비트 부터 드라이버를 로드하기 위해서는 드라이버 파일에 디지털 서명을 적용해야 한다. 이 부분은 다소 다른 주제이기도 하고 우리가 사용하는 32비트 테스트 환경은 디지털 서명이 드라이버를 로드하기 위한 필수조건이 아니기 때문에 관련 내용은 그냥 넘어 가도록 한다. 만약 64비트 테스트 환경에서 진행하는 독자가 있다면 드라이버 파일에 디지털 서명을 적용하거나 일단은 이를 우회하기 위해서 “disable driver signature” 정도의 키워드로 검색해보고 해당 내용을 적용해야 할 것이다.

scanner 샘플을 설치한 후에 해야 할 일은 드라이버를 로드하는 것이다. 드라이버를 로드하기 위해서는 관리자 권한이 필요하다. 일단 관리자 권한을 가지는 command 창을 실행하고 드라이버가 설치될 때 사용된 서비스 키 이름으로 아래와 같이 sc start servicename (혹은 net start servicename) 명령을 수행하여 드라이버를 로드한다.

![scanner.sys 드라이버 로드 명령](/files/driver2_13.png)

드라이버를 언로드하려면 sc stop servicename (혹은 net stop servicename)을 수행하면 된다.

![scanner.sys 드라이버 언로드 명령](/files/driver2_14.png)

scanner.sys 드라이버가 로드되었다면 이제 응용 프로그램인 scanuser.exe를 실행한다.

![scanuser.exe 실행 초기 화면](/files/driver2_15.png)

위의 그림과 같이 “scanner /?” 를 입력하면 사용법을 확인할 수 있다. scanuser.exe는 입력 매개변수가 없는 경우에는 기본 설정으로 동작하도록 구현되어 있다. 일단은 그냥 실행한다. 위의 그림과 같은 실행화면이 나오면 정상적으로 동작하는 것이다.

이제 scanner 샘플의 동작 방식을 간단히 설명하겠다. 맡은 바 역할을 요약하면 scanner.sys 미니필터 드라이버는 임의의 프로세스가 모니터링 대상 확장자를 가지는 파일을 열거나 쓰거나 닫을 때 파일의 데이터를 읽어서 scanuser.exe 프로그램에게 전달하고 대답이 올 때까지 무기한 기다린다. scanuser.exe 프로그램은 전달받은 파일 데이터에 “foul” 이라는 아스키 문자열이 존재하는지 확인한다. 만약 존재한다면 안전하지 않다고 판단하여 차단을 결정한다. 이러한 결과를 scanner.sys 미니필터 드라이버에게 전달한다. 이 결과를 전달받으면 scanner.sys 미니필터 드라이버는 해당 파일 연산에 대한 무한대기를 풀고 그 연산을 차단으로 처리한다.

이러한 처리 과정을 대략적으로 이해하고 그림을 보면서 보다 자세하게 들여다보자.

![미니필터 등록 연산](/files/driver2_16.png)

우선 위의 그림은 FltXxx 형식의 이름을 가지는 필터 매니저(FltMgr.sys)의 익스포트 함수를 사용하여 scanner.sys 를 미니필터 드라이버로 등록하는 연산이다. 드라이버가 로드될 때 호출되는 DriverEntry 함수에서 이러한 미니필터 등록 연산을 수행하고 그 결과로 미니필터 핸들을 얻는다. 이 때 다양한 콜백 함수들이 등록된다. 필터 매니저가 scanner.sys 미니필터 드라이버에게 실시간 파일 I/O가 발생했다고 알려주는 방법이 이 콜백 함수를 호출하는 것이다. scanner.sys 미니필터 드라이버는 자신이 관심있는 파일 연산에 대해서만 콜백 함수를 등록하여 실시간 I/O를 모니터링 한다.

![미니필터 등록 해제 연산](/files/driver2_17.png)

드라이버가 언로드 될 때는 미니필터가 등록될 때 얻은 미니필터 핸들을 사용하여 미니필터 등록을 해제할 수 있다. 이후로 미니필터 드라이버는 실시간 파일 I/O를 모니터링 할 수 없다.

![통신 포트 생성 연산](/files/driver2_18.png)

그 다음으로 중요한 것이 scanuser.exe 프로그램과 통신하는 것이다. Scanner 샘플 프로그램은 필터 매니저가 지원하는 통신 방법을 사용하고 있다. scanner.sys 드라이버에서는 서버 포트를 생성하고 이 때 서버 포트에 대한 연결, 연결 종료, 메시지 수신(NULL)을 위한 콜백 함수를 등록한다.

서버 포트 생성이 성공적으로 수행되면 scanuser.exe 프로그램에서는 이 서버 포트로 연결을 수행할 수 있다. 유저레벨에서는 FltLib.dll 라이브러리를 사용하여 필터 매니저와 연계할 수 있다. FltLib.dll에서 제공하는 익스포트 함수는 FilterXxx 와 같은 이름을 갖는다.

아래의 그림은 scanuser.exe 프로그램에서 scanner.sys 미니필터 드라이버의 서버 포트로 연결, 메시지 전송(기능 부재) 및 연결 해제에 대한 내용을 보여준다. scanner.sys 미니필터 드라이버는 서버 포트 연결 콜백 함수(ScannerPortConnect)가 호출되는 시점에 현재 프로세스 정보(scanuser.exe)를 저장하는데 이 프로세스 정보는 실시간 모니터링에서 scanuser.exe 프로그램이 수행하는 파일 연산은 예외하기 위한 용도로 사용된다.

![서버 포트에 대한 콜백 함수 연관 관계](/files/driver2_19.png)

서버 포트의 해제는 미니필터 핸들의 해제 시점과 같이 일반적으로 드라이버 언로드 시점에 행해진다.

![서버 포트 해제 연산](/files/driver2_20.png)

이제 scanner.sys 미니필터 드라이버는 임의의 프로세스가 발생시키는 파일 연산 중에서 열기, 쓰기, 닫기에 대한 실시간 I/O를 필터 매니저로부터 전달받는다. 아래의 그림은 파일 열기에 대한 내용을 정리한 것이다. 주의 깊게 봐야할 것은 Pre/Post에 대한 개념인데 이것은 Dispatch/Complete 이라고도 한다. 파일이라는 대상으로 파일 연산이 수행되는데 수행되기 전 상황을 Pre라고 한다. 같은 맥락으로 Dispatch는 사전적 의미로 “급파하다”, “발송하다”의 의미를 갖는다. 실제 WDM 구조를 가지는 계층화된 드라이버들은 드라이버간 호출 관계를 연결해주는 함수 테이블을 가지고 있는데 이를 Dispatch 함수 테이블이라고 한다. Post/Complete는 연산이 끝날 때 그 결과를 포함하는 호출과정이다.

![CreateFile 함수의 필터링 예](/files/driver2_21.png)

차단해야하는 파일 I/O라고 가정하면 그 시점은 Pre이어야 한다. 파일 쓰기 시점이라면 쓰기 전에 차단 여부를 검사해서 차단을 해야한다. Post 시점에 차단은 이미 쓰여진 파일 데이터를 차단으로 처리하기 위해서 다루기가 쉽지 않을 것이다. 단, 파일 열기인 경우에는 Post 시점에 차단을 적용할 수 있는 방법이 있다. FltCancelFileOpen 함수는 이름에서도 알 수 있듯이 파일 열기를 취소해주는 함수이다. scanner.sys 미니필터 드라이버는 파일 열기 연산에 대한 차단을 Post에서 수행하고 FltCancelFileOpen 함수를 사용하여 차단한다. 파일 열기에 대한 Pre 함수에서는 Post 함수의 호출 여부를 결정하는 용도로만 사용한다. Pre 함수에서 반환하는 값에 따라 Post 함수를 호출할지를 결정할 수 있는 구조를 가지고 있다. 이러한 Post 함수 호출 여부 결정 방식으로 앞에서 잠깐 언급했듯이 scanuser.exe 프로그램이 수행하는 파일 연산을 예외처리 할 수 있다. 서버 포트 연결을 수행한 프로세스와 현재 프로세스가 동일한지 확인하여 동일한 경우에는 예외하는 것이다. 예외 프로세스가 아니라면 Post 함수가 호출되고 scanner.sys 미니필터 드라이버에서 지정한 확장자를 가지는 파일인 경우에만 파일의 내용을 읽어서 scanuser.exe로 보내게 된다.

scanner.c 소스파일에 아래와 같이 모니터링 대상 확장자를 지정하고 있다.

```
const UNICODE_STRING ScannerExtensionsToScan[] =
    { RTL_CONSTANT_STRING( L"doc"),
      RTL_CONSTANT_STRING( L"txt"),
      RTL_CONSTANT_STRING( L"bat"),
      RTL_CONSTANT_STRING( L"cmd"),
      RTL_CONSTANT_STRING( L"inf"),
      /*RTL_CONSTANT_STRING( L"ini"),   Removed, to much usage*/
      {0, 0, NULL}
    }
```

아래의 그림은 앞서 설명한 미니필터 드라이버과 어플리케이션 간의 통신 방식을 기반으로 필터 매니저(FltMgr.sys)와 FltLib.dll 라이브러리를 사용해서 scanner 샘플에서 처리하는 실시간 파일 I/O를 감시하는 방법을 요약해서 보여준다.

![scanner 샘플 차단 동작 요약](/files/driver2_22.png)

실제로 scanner.sys 미니필터 드라이버를 로드하고 scanuser.exe 프로그램을 실행한 다음 txt 확장자를 가지는 임의의 파일을 생성하여 “foul” 문자열을 입력한 후 저장하려고 하면 차단될 것이다. 또는 이미 “foul” 문자열을 포함하고 있는 txt 파일이라면 파일 열기를 시도할 때 차단될 것이다. 아래의 그림은 scanuser.exe 프로그램에서 실시간 차단 여부를 보여주는 내용이다.

![“foul” 문자열이 발견되어 차단했다는 로그 내용](/files/driver2_23.png)

## 파일 데이터 대신 파일 경로 사용하기

지금까지 scanner 샘플에 대한 있는 그대로의 내용을 설명하였다. 우리가 만들 실시간 감시기는 scanner 샘플과는 조금 다르게 파일 데이터 대신 파일 경로를 수집하여 scanuser.exe 프로그램으로 전달할 것이다. 그리고 파일 쓰기 및 닫기 연산에 대해서는 처리하지 않고 오로지 파일 열기 연산만 감시하는 방법을 사용할 것이다. 아래 링크는 필자가 수정한 scanner 샘플 소스코드이다. 편의상 scanner_filepath 소스코드로 부르겠다. 방금 설명한 정도의 수정만 한 것으로 최대한 원본 소스코드를 그대로 사용하였다. 두 개의 소스코드를 비교해보면 수정한 내용을 쉽게 확인할 수 있을 것이다. 우리는 앞으로 scanner_filepath 소스코드를 기반으로 다음 글에서는 KicomAV 엔진을 붙이는 작업을 진행할 것이다.

- 소스코드 : <https://github.com/nurilab/scanner_filepath/>

scanner_filepath 소스코드를 빌드하여 테스트할 때 주의할 점은 기존에 scanner 샘플을 사용 중이었다면 새로운 scanner.sys 미니필터 드라이버로 교체해주어야 한다는 것이다. 이를 위해서 우선 기존에 사용 중이던 scanner.sys 미니필터 드라이버를 언로드해야 한다. 언로드 명령은 앞서 설명한 것과 같이 sc stop servicename (혹은 net stop servicename) 명령을 사용하면 된다. 그런 다음 scanner.inf 파일로 재설치를 하던지 아니면 수동으로 scanner.sys 파일을 “C:\Windows\System32\drivers\” 경로에 복사해주어야 한다.

![scanner.sys 파일의 위치](/files/driver2_24.png)

scanner_filepath 소스코드 빌드 결과물로 테스트해보면 아래의 그림과 같이 “foul” 문자열을 포함하는 임의의 txt 파일을 열 때 해당 파일의 경로가 출력되면서 차단되는 것을 확인할 수 있을 것이다.

![scanner_filepath 소스코드에 대한 테스트 실행 결과](/files/driver2_25.png)


## 마무리

WDK 변천 과정에서부터 시작해서 WDK 7.1을 설치해보고 scanner 샘플을 사용하여 미니필터 드라이버 연산에 대한 기본적인 내용을 알아보았다. 드라이버를 처음 접하는 독자를 고려하여 다소 설명이 길어진 부분도 있고 대략적인 내용만 언급한 부분도 있다. 글에서 언급한 함수들에 대해서 보다 자세한 내용을 알고 싶은 독자는 WDK 도움말을 찾아보면 많은 도움이 될 것이다. 아무튼 scanner_filepath 소스코드를 준비함으로써 우리는 초간단 실시간 감시기에서 실시간 파일 I/O 처리 부분을 마쳤다. 최종 글인 다음 글에서는 KicomAV 엔진을 적용해보도록 하겠다.

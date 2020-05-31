---
layout: post
title: '초간단 실시간 감시기 만들기 (3)'
author: Bo-Seung Ko
date: 2020-06-01 06:30
tags: [kicomav,opensource,driver]
---

<fieldset style="margin:0px 0px 20px 0px;padding:5px;"><legend><span><strong style="font-weight:bold;">연재 순서</strong></span></legend><!--Creative Commons License--><div style="float: left; width: 88px; margin-top: 3px;"><img alt="Creative Commons License" style="border-width: 0" src="/files/images/exclamationmark.png"/></div><div style="margin-left: 92px; margin-top: 3px; text-align: justify;">
<p style="margin: 0;"><a href="/2020/05/18/kicomav_driver_1/">첫번째 글: 실시간 감시기란</a></p>
<p style="margin: 0;"><a href="/2020/05/25/kicomav_driver_2/">두번째 글: 미니필터 드라이버를 사용하여 실시간 파일 I/O 확인하기</a></p>
<p style="margin: 0; background:#ddd;">세번째 글: KicomAV 엔진을 사용하여 유해한 파일인지 확인하기</p>
</div></fieldset>


## KicomAV 엔진을 C 코드에서 사용하기

KicomAV 는 (주) 누리랩의 최원혁 대표가 개발한 안티 바이러스 백신 프로그램이다. KicomAV 는 파이썬으로 개발되었으며 오픈소스여서 관심있는 개발자라면 누구나 소스코드를 확인할 수 있다. 더불어 “파이썬으로 배우는 Anti-Virus 구조와 원리” 라는 책에서는 KicomAV의 구조 및 악성코드 진단 방법을 자세하게 설명하고 있다. 이에 대한 내용은 아래의 링크에서 확인할 수 있다.

- <https://github.com/hanul93/kicomav>
- <http://www.hanul93.com/>

우리가 만들고 있는 초간단 실시간 감시기는 C 언어 기반이다. scanner.sys 미니필터 드라이버와 scanuser.exe 어플리케이션은 C 로 구현되어 있다. KicomAV 엔진이 C/C++ 코드에서 사용되기 위해서는 Python Embedding 방식을 사용해야 한다. 이는 C/C++ 코드에서 파이썬 스크립트를 호출하고 실행할 수 있는 기능이다. Python Embedding 에 대한 보다 자세한 내용은 따로 설명하지 않고 독자에게 맡기도록 하겠다.

필자는 KicomAV 엔진을 scanuser.exe 어플리케이션에서 사용하기 위해서 인터페이스 역할을 수행하는 k2py.dll 을 Python Embedding 방식으로 만들었다. k2py.dll은 네 개의 함수를 익스포트한다. 이 함수들에 대한 프로토타입은 아래와 같고 k2py.h 파일에 정의되어 있다.

```
typedef struct _K2RESULT
{
	int nResult; // 0이면 무해한 파일, 1이면 유해한 파일
	WCHAR wResult[2048]; // 검사정보
} K2RESULT, *PK2RESULT; 

// KicomAV 엔진을 로드한다.
BOOLEAN
K2Load(
    PWCHAR pwEnginePath, // 엔진경로(kavcore 디렉터리 경로)
    ULONG InstanceCount // 인스턴스 개수(max:10)
);

// KicomAV 엔진을 언로드한다.
VOID
K2Unload();

// KicomAV 엔진의 버전을 구한다.
BOOLEAN
K2Version(
    PWCHAR pwVersion, // 엔진버전 정보가 저장될 버퍼의 포인터
    ULONG cchVersion // 엔진버전 정보가 저장될 버퍼의 문자 단위 크기
);

// 파일의 유해성 여부를 검사한다.
// bCure 가 TRUE이고 파일이 유해하다면 해당 파일을 치료(삭제 등)한다
Int
K2Scan(
    ULONG Instance, // 엔진 인스턴스 인덱스(zero based index)
    PWCHAR pwPath, // 파일 경로
    PK2RESULT pK2Result, // 검사결과가 담겨질 버퍼의 포인터
    BOOLEAN bCure // FALSE이면 검사만 하고 TRUE이면 치료를 동반한다.
);
```

k2py.dll은 현재 베타 버전 정도의 수준이며 비공식 버전이다. 위의 프로토타입에서 언급한 인스턴스 개념은 보다 안정화된 버전에서 사용하기 위한 것으로 현재는 엔진 인스턴스가 1개일 때 안정적으로 동작한다.


## 파일 열기 연산을 실시간으로 검사하기

우리는 지난 글에서 파일 열기 연산이 발생할 때 어플리케이션에서 실시간으로 파일 경로를 출력할 수 있도록 scanner 샘플을 수정하였다. 이제 어플리케이션으로 전달된 파일 경로를 K2Scan 함수의 pwPath 매개변수로 입력하여 유해성 여부를 검사할 것이다. 아래의 그림은 초간단 실시간 감시기의 전체적인 동작 개요를 보여준다.

![실시간 감시기 동작 개요](/files/driver3_1.png)

이제 남은 작업은 scanuser.exe 코드에서 k2py.dll의 익스포트 함수를 적절하게 호출하여 실시간 감시기로 동작할 수 있는 기반을 만드는 것이다. 그 작업은 크게 세 단계로 나눌 수 있는데 엔진 로드 단계, 실시간 검사 단계, 엔진 언로드 단계가 그것이다. 아래의 코드는 scanuser.c 소스파일에서 엔진 로드 단계의 코드이다.

```
    if(!GetCurDirW(wDllPath, 512)) // 현재 경로를 구한다.
    {
        hr = GetLastError();
        goto main_cleanup;
    }

#ifdef _WIN64
    wcscat_s(wDllPath, 512, L"\\k2py64.dll");
#else
    wcscat_s(wDllPath, 512, L"\\k2py.dll");
#endif

    hModule = LoadLibrary(wDllPath); // k2py.dll 모듈을 로드한다.
    if(!hModule)
    {
        hr = GetLastError();
        goto main_cleanup;
    }

// 익스포트 함수 주소를 구한다.
pfnK2Load = (K2LOAD) GetProcAddress(hModule, "K2Load");
pfnK2Unload = (K2UNLOAD) GetProcAddress(hModule, "K2Unload");
pfnK2Version = (K2VERSION) GetProcAddress(hModule, "K2Version");
pfnK2Scan = (K2SCAN) GetProcAddress(hModule, "K2Scan");

    if(!pfnK2Load ||
        !pfnK2Unload ||
        !pfnK2Version ||
        !pfnK2Scan)
    {
        hr = 1;
        goto main_cleanup;
    }

    bLoad = pfnK2Load(L".\\kavcore", threadCount); // KicomAV 엔진을 로드한다.
    if(bLoad)
    {
        if(!pfnK2Version(wVersion, 512)) // KicomAV 엔진 버전을 구한다.
        {
            hr = 1;
            goto main_cleanup;
        }

        printf("\nVersion: %ws\n", wVersion);
    }
```

원래 scanuser.exe는 실행할 때 scanner.sys 미니필터 드라이버와 통신하는 스레드 개수와 스레드 당 I/O 요청 개수를 입력받을 수 있다. 아무런 입력 값이 없다면 기본 설정으로 실행된다. 여기서 스레드 개수는 실시간 감시기의 성능과 관련된 것이다. 이 부분에 대해서 잠시 짚고 넘어가보자. 커널 영역은 모든 프로세스가 공유하는 자원이다. 파일시스템 드라이버의 예를 들면 여러 개의 프로세스가 자신이 사용하는 파일에 접근할 때 파일시스템 드라이버 연산을 통과하게 된다. 각각의 프로세스는 내부적으로 스레드 단위로 커널 자원을 호출하게 된다. 이 스레드 들은 시분할에 의해서 자신에게 할당된 시간만큼 커널로 진입하여 파일시스템 드라이버 연산을 수행하려고 한다. 요즘처럼 멀티 CPU가 일반화된 경우에는 각각의 프로세스가 수행하는 파일 연산은 파일시스템 드라이버 입장에서는 동시에 처리해야 하는 함수의 호출인 것이다. 프로세스에서 호출한 CreateFile 함수는 계층화된 드라이버로 전달되어 파일시스템 드라이버의 해당 함수 호출로 연결된다. 멀티 CPU 환경에서는 이러함 함수 호출이 동시에 발생할 수 있는 충분조건을 만족한다. 미니필터 드라이버와 같은 필터 드라이버도 예외는 아니다. 미니필터 드라이버는 파일시스템 드라이버의 필터 드라이버이기 때문에 연산 함수의 동시 호출 또는 재진입 현상이 아주 빈번하게 발생한다. 실시간 감시기는 파일 연산의 동시 발생 순간을 잘 해결하기 위해서 진단 엔진을 멀티 인스턴스로 생성해야 한다. 파일 연산에 대한 Race Condition이 발생한다면 멀티 인스턴스 엔진은 단일 인스턴스일 때와 비교해서 보다 우수한 성능을 발휘할 것이다.

현재 KicomAV 엔진은 구조적으로 멀티 인스턴스를 지원한다. 다만 필자가 제공하는 k2py.dll 인터페이스 모듈과의 연계 상에서 안정적인 동작을 위해 단일 인스턴스로 동작하도록 구현하였다. 이에 scanuser.exe의 기본 설정값에서도 스레드 개수를 2에서 1로 수정하였다. 아무런 입력없이 scanuser.exe를 실행하면 단일 인스턴스 형태로 실시간 감시 기능이 동작하도록 하였다. 파일 연산에 대한 Race Condition이 발생한다면 성능은 다소 떨어질 수 있지만 필자가 의도하는 실시간 감시기의 동작 방식을 이해하기에는 충분하다.

![다중 프로세스의 파일 연산 발생시 단일 인스턴스 엔진 처리 개요](/files/driver3_2.png)

아래의 코드는 scanuser.c 코드에서 실시간 검사 단계의 코드를 적용한 모습이다. 기존 ScanBuffer 함수 대신 k2py.dll의 K2Scan 함수를 사용하여 파일의 유해성 여부를 결정한다.

```
    K2RESULT K2Result = {0,};
    WCHAR wDosFilePath[512] = {0,};

    PWCHAR pwPtr = GetCharPointerW((PWCHAR)notification->Contents, L'\\', 3);
    if(!pwPtr)
    {
        hr = -1;
        break;
    }

    *pwPtr = L'\0';

    // DeviceName에 대한 DosName을 구한다.
    hr = FilterGetDosName((PWCHAR)notification->Contents, wDosFilePath, 512);
    if(FAILED(hr))
        break;

    // FullDosName을 만든다.
    wcscat(wDosFilePath, L"\\");
    wcscat(wDosFilePath, ++pwPtr);

    //printf( "FilePath: %ws\n", wDosFilePath );

    // 기존 “foul” 문자열 존재여부 함수는 주석처리한다.
    //result = ScanBuffer( wDosFilePath );

    // 파일 유해성 검사
    if(!pfnK2Scan(
        Context->InstanceIndex,
        wDosFilePath,
        &K2Result,
        FALSE
    ))
    {
        printf(
            "[InstanceIndex=%d] <FAILED> Pid=%d F=%ws\n",
            Context->InstanceIndex,
            notification->ProcessId,
            wDosFilePath
        );
        result = FALSE; // Failed
    }
    else
    {
        printf(
            "[InstanceIndex=%d] %s Pid=%d F=%ws\n",
            Context->InstanceIndex,
            K2Result.nResult ? "<INFECTED>" : "<OK>",
            notification->ProcessId,
            wDosFilePath
        );
        result = K2Result.nResult ? TRUE : FALSE;
    }
```

마지막으로 아래의 코드는 scanuser.c 코드에서 엔진 언로드 단계의 코드를 적용한 모습이다. scanuser.exe 프로그램이 종료되기 직전에 수행하는 코드이다.

```
    if(bLoad)
        pfnK2Unload(); // KicomAV 엔진을 언로드한다

    if(hModule)
        FreeLibrary(hModule); // k2py.dll 모듈을 언로드한다.
```


지금까지 scanner 샘플에서 수정해야할 주요 포인트에 대해서 설명하였다. 이제 가상환경에서 실제로 동작시켜서 간단한 테스트를 해보도록 하자.

## Eicar 파일 차단하기

Eicar 파일은 백신 프로그램을 테스트할 때 사용하는 파일이라고 생각하면 된다. 실제로 eicar 파일은 PE 파일 포맷을 따르지도 않기 때문에 실행 가능한 파일도 아니다. 다만 command 창에 “EICAR-STANDARD-ANTIVIRUS-TEST-FILE!” 출력해주는 명령을 담고 있는 텍스트 파일이다.

- <https://www.eicar.org/?page_id=3950>

위의 링크에서 eicar.com 파일을 다운로드 받거나 아래의 텍스트를 .com 확장자를 가지는 파일로 저장하면 eicar 파일을 준비할 수 있다. 만약 윈도우 디펜더 등의 백신 프로그램이 동작하고 있다면 테스트를 위해서 잠시 꺼두기 바란다.

```
X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*
```

- <https://github.com/nurilab/scanner_filepath>
- <https://github.com/nurilab/scanner_filepath/tree/master/bin>

위의 링크는 scanner 샘플에 KicomAV 엔진을 반영한 최종 소스코드와 테스트에 사용된 파일 세트이다. 여기에 적용된 KicomAV 엔진에는 eicar 파일만 진단하는 최소한의 엔진 모듈만 포함하고 있다. 그리고 scanner.sys 미니필터 드라이버는 모니터링 대상 확장자를 .com, .exe 로 제한하도록 수정하였다. 그러나 자신만의 백신 엔진을 테스트해보고 싶거나 다양한 파일 포맷을 실시간으로 검사해보고 싶은 독자는 소스코드를 수정해서 테스트해보기 바란다. KicomAV 엔진에 신규 패턴을 적용하는 방법은 아래 링크의 “KicomAV 가이드 (악성코드 패턴 추가 방법 포함)” 에서 확인할 수 있다.

- <https://nurilab.github.io/2020/04/01/kicomav_guide/>

우리는 Windows 10 32 비트의 가상환경에서 테스트할 것이다. Eicar.com 파일은 64비트 환경에서는 동작하지 않기 때문에 32비트 테스트 결과와는 다를 수 있다. 아래의 그림은 scanuser.exe 프로그램을 실행한 경우와 그렇지 않은 경우에 eicar.com 파일을 command 창에서 실행한 결과이다.

![Eicar.com 실시간 차단 테스트 화면](/files/driver3_3.png)


## 마무리

우리는 지금까지 세 개의 글을 통해서 실시간 감시기가 무엇인지 알아보았고 WDK7.1의 scanner 와 KicomAV 라는 공개된 소스코드를 수정하여 백신 테스트 파일인 eicar.com 파일을 실행하는 시점에 실시간으로 차단해 보았다. 비록 제한적이고 부족한 기능이지만 간단하게나마 실시간 감시기의 동작을 이해하는 데는 좋은 예라고 생각한다. 대부분의 상용 백신 제품은 고도화된 실시간 감시기 기능을 제공하고 있다. 한번 검사한 파일에 대해서는 캐시화하여 이후의 파일 연산에서는 진단 엔진을 태우지 않고 캐시 내용을 반영하여 성능향상을 꾀한다. 그리고 파일 모니터링과 더불어 레지스트리 연산, 프로세스 및 스레드 생성 연산 등 다양한 자원 접근 시점을 모니터링하여 악성코드의 행위를 빈틈없이 검사하기도 한다.

초간단 실시간 감시기 만들기 연재를 통해서 백신 프로그램을 처음 만들어 보거나 드라이버 빌드를 처음 해본 독자도 있을 것이다. 뜻이 있는 곳에는 길이 있듯이 혹여나 본 연재를 통해 백신 개발이나 드라이버 개발에 관심있는 독자가 있다면 주저하지 말고 도전해보기 바란다.

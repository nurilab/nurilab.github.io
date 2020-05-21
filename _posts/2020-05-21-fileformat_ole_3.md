---
layout: post
title: 'OLE 파일 구조 분석 (3)'
author: Jin-Hyeong Choe
date: 2020-05-21 18:00
tags: [fileformat, ole]
---

<fieldset style="margin:0px 0px 20px 0px;padding:5px;"><legend><span><strong style="font-weight:bold;">연재 순서</strong></span></legend><!--Creative Commons License--><div style="float: left; width: 88px; margin-top: 3px;"><img alt="Creative Commons License" style="border-width: 0" src="/files/images/exclamationmark.png"/></div><div style="margin-left: 92px; margin-top: 3px; text-align: justify;"><p style="margin: 0"><a href="/2020/05/04/fileformat_ole_1/">1. OLE 파일 구조 분석 (1) - Header, BBAT, SBAT</a></p>
<p style="margin: 0;"><a href="/2020/05/15/fileformat_ole_2/">2. OLE 파일 구조 분석 (2) - Root Storage</a></p>
<p style="margin: 0; background:#ddd;">3. OLE 파일 구조 분석 (3) - Stream Object</p>
</div></fieldset>

## Stream Object

스트림은 OLE 파일 중에서 가장 큰 비중을 차지하고 있는 데이터이다. 스트림의 데이터(오브젝트)를 읽기 위해서는 Root Storage(Root Entry)를 통해 모든 디렉터리들을 먼저 읽어와야 한다. 

스트림은 디렉터리의 타입이 Stream (0x02)인 경우를 말하며 헤더에 정의된 Max Small Stream Size보다 크기가 작은 경우 Small Stream, 같거나 클 경우 Big Stream로 분류한다. Big Stream은 BBAT를, Small Stream은 SBAT를 참조한다. 

Small Stream의 데이터는 Small Sector에 저장되어 있다. Small Sector는 Root Entry의 오브젝트이다. 따라서 Small Stream을 읽기 위해서는 Root Entry의 오브젝트를 먼저 읽어야 한다. 

참고로 Root Entry의 경우에만 오브젝트의 크기(스트림의 크기)가 Max Small Stream Size보다 작더라도 무조건 BBAT에 저장된다. 왜냐하면 Small Stream이 저장되는 공간이 바로 Root Entry의 오브젝트 영역이기 때문이다.

다음 예제를 보면서 직접 스트림의 데이터(오브젝트) 읽기를 따라해보자.

먼저 Big Stream의 오브젝트를 구하는 방법부터 설명하도록 하겠다.
이전에 읽은 디렉터리 중에서 1Table이라는 스트림의 데이터(오브젝트)를 구해보겠다.

- Directory[1]
<table class="tb_ole">
    <tr>
        <td>Name</td>
        <td>1Table</td>
    </tr>
    <tr>
        <td>Type</td>
        <td>Stream (0x02)</td>
    </tr>
    <tr>
        <td>Color</td>
        <td>Black (0x01)</td>
    </tr>
    <tr>
        <td>Left Node</td>
        <td></td>
    </tr>
    <tr>
        <td>Right Node</td>
        <td>0x05</td>
    </tr>
    <tr>
        <td>Child Node</td>
        <td></td>
    </tr>  
    <tr>
        <td>Start Sector Id</td>
        <td>0x08</td>
    </tr> 
   <tr>
        <td>Size</td>
        <td>0x1A8B</td>
    </tr>  
</table>


현재 1Table의 크기는 1A8B이고 Max Small Stream Size보다 크기 때문에 Big Stream으로 분류한다. Big Stream의 경우 BBAT를 참조하여 Start Sector Id와 연결된 모든 섹터를 읽으면 된다.

![](/files/ff_ole3_1.png)

이전에 BBAT를 참조하여 Root Storage를 구한 것과 똑같이 섹터 연결의 끝을 나
타내는 값인 0xFFFFFFFE가 보일 때까지 계속해서 next 값을 계산해서 쫓으면 된
다. 읽은 섹터들을 하나로 연결하면 스트림 데이터(오브젝트)가 완성된다.

<p></p>

<table style="border: 2px solid #dcdcdc;">
    <tr style="background: #fafafa">
        <td>값</td>
        <td>의미</td>
    </tr>
    <tr>
        <td>0xFFFFFFFD</td>
        <td>특수 블록을 의미</td>
    </tr>
    <tr>
        <td>0xFFFFFFFE</td>
        <td>End of Chain</td>
    </tr>
    <tr>
        <td>0xFFFFFFFF</td>
        <td>비어 있음을 의미</td>
    </tr>
    <tr>
        <td>0 ~ </td>
        <td>Next 값 (연결된 다음 섹터 ID)</td>
    </tr>
</table>

현재 BBAT를 참조해서 확인한 Start Sector Id와 연결된 섹터들은 다음과 같다.

![](/files/ff_ole3_2.png)

이제 섹터들의 데이터를 하나씩 전부 읽어온다.

```
SECTOR[8] = (8 + 1) * 512 = 0x1200
```

![](/files/ff_ole3_3.png)

```
SECTOR[9] = (9 + 1) * 512 = 0x1400
```

![](/files/ff_ole3_4.png)

같은 방식으로 마지막 섹터를 제외한 모든 섹터를 읽는다. (총 13번)

```
SECTOR[21] = (21 + 1) * 512 = 0x2C00
```

![](/files/ff_ole3_5.png)

Tabl1의 스트림 오브젝트 크기는 6795바이트이다. 그렇기에 1섹터(512바이트)씩 13번 읽고 마지막 섹터는 512바이트가 아닌 139바이트만큼 읽는다. 읽고 남은 373바이트는 버린다. 

이렇게 섹터 단위로 나누어진 스트림들을 전부 BBAT를 참조하여 읽고 하나로 합친다면 하나의 스트림 오브젝트가 완성된다. 읽은 1Table 스트림 오브젝트의 크기는 6795바이트다.

![1Table Stream Object](/files/ff_ole3_6.png)

다음으로 Small Stream을 읽는 방법을 설명하겠다.

Small Stream은 앞서 설명했듯이 Max Small Stream Size보다 크기가 작은 스트림을 말한다. 이전에 읽은 디렉터리 중에서 CompObj이라는 Small Stream의 오브젝트(데이터)를 읽어보도록 하겠다.


- Directory[5]
<table class="tb_ole">
    <tr>
        <td>Name</td>
        <td>\001CompObj</td>
    </tr>
    <tr>
        <td>Type</td>
        <td>Stream (0x02)</td>
    </tr>
    <tr>
        <td>Color</td>
        <td>Red (0x00)</td>
    </tr>
    <tr>
        <td>Left Node</td>
        <td></td>
    </tr>
    <tr>
        <td>Right Node</td>
        <td></td>
    </tr>
    <tr>
        <td>Child Node</td>
        <td></td>
    </tr>  
    <tr>
        <td>Start Sector Id</td>
        <td>0x00</td>
    </tr> 
   <tr>
        <td>Size</td>
        <td>0x6E</td>
    </tr>  
</table>

Small Stream은 512바이트의 섹터로 이루어진 스트림과 다르게 64바이트 크기의 Small Sector로 구성되어 있다.

- Directory[0]
<table class="tb_ole">
    <tr>
        <td>Name</td>
        <td>Root Entry</td>
    </tr>
    <tr>
        <td>Type</td>
        <td>Root Storage (0x05)</td>
    </tr>
    <tr>
        <td>Color</td>
        <td>Black (0x01)</td>
    </tr>
    <tr>
        <td>Left Node</td>
        <td></td>
    </tr>
    <tr>
        <td>Right Node</td>
        <td></td>
    </tr>
    <tr>
        <td>Child Node</td>
        <td>0x03</td>
    </tr>  
    <tr>
        <td>Start Sector Id</td>
        <td>0x2A</td>
    </tr> 
   <tr>
        <td>Size</td>
        <td>0x80</td>
    </tr>  
</table>

Small Stream은 1섹터의 크기가 64바이트인 Small Sector로 구성되어 있다. 그리고 Small Sector는 Root Entry의 오브젝트에 구성 되어 있다. 그렇기 때문에 Small Stream을 읽으려면 Root Entry의 오브젝트를 먼저 읽어야 한다.

Root Entry 오브젝트의 크기는 512크기보다 작기 때문에 굳이 BBAT를 참조할 필요가 없다. 0x2A번 섹터 하나만 읽으면 된다.

```
SECTOR[42] = (42 + 1) * 512 = 0x5600
```

![](/files/ff_ole3_7.png)

읽어온 Root Entry 오브젝트를 Small Sector의 크기만큼 나눈다. 그러면 1섹터에 8개의 Small Sector가 저장 되었음을 알 수 있다. 읽어온 섹터의 위에서부터 순서대로 0, 1, 2, 3, 4, 5, 6, 7의 Small Sector Array Index를 가지게 된다.

![](/files/ff_ole3_8.png)

- Small Sector[0]

<table class="tb_ole">
    <tr>
        <td>위치</td>
        <td>0x5600 ~ 0x563F</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>64바이트</td>
    </tr>
</table>

- Small Sector[1]

<table class="tb_ole">
    <tr>
        <td>위치</td>
        <td>0x5640 ~ 0x567F</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>64바이트</td>
    </tr>
</table>

- Small Sector[2]

<table class="tb_ole">
    <tr>
        <td>위치</td>
        <td>0x5680 ~ 0x56BF</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>64바이트</td>
    </tr>
</table>

- Small Sector[3]

<table class="tb_ole">
    <tr>
        <td>위치</td>
        <td>0x56C0 ~ 0x56FF</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>64바이트</td>
    </tr>
</table>

- Small Sector[4]

<table class="tb_ole">
    <tr>
        <td>위치</td>
        <td>0x5700 ~ 0x573F</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>64바이트</td>
    </tr>
</table>

- Small Sector[5]

<table class="tb_ole">
    <tr>
        <td>위치</td>
        <td>0x5740 ~ 0x577F</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>64바이트</td>
    </tr>
</table>

- Small Sector[6]

<table class="tb_ole">
    <tr>
        <td>위치</td>
        <td>0x5780 ~ 0x57BF</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>64바이트</td>
    </tr>
</table>

- Small Sector[7]

<table class="tb_ole">
    <tr>
        <td>위치</td>
        <td>0x57C0 ~ 0x57FF</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>64바이트</td>
    </tr>
</table>

- Directory[5]

<table class="tb_ole">
    <tr>
        <td>Name</td>
        <td>\001CompObj</td>
    </tr>
    <tr>
        <td>Type</td>
        <td>Stream (0x02)</td>
    </tr>
    <tr>
        <td>Color</td>
        <td>Red (0x00)</td>
    </tr>
    <tr>
        <td>Left Node</td>
        <td></td>
    </tr>
    <tr>
        <td>Right Node</td>
        <td></td>
    </tr>
    <tr>
        <td>Child Node</td>
        <td></td>
    </tr>  
    <tr>
        <td>Start Sector Id</td>
        <td>0x00</td>
    </tr> 
   <tr>
        <td>Size</td>
        <td>0x6E</td>
    </tr>  
</table>

CompObj의 크기는 110이다. 그러면 2개의 Small Sector를 읽으면 된다. Small Sector는 Sector와 마찬가지로 Address Table을 참조하여 섹터를 연결해야한다. Small Sector는 BBAT가 아닌 SBAT를 참조한다. 

![](/files/ff_ole3_9.png)

현재 파일에서 SBAT의 시작 섹터 번호는 0x29이다.

```
SBAT = (0x29 + 0x01) * 0x200 = 0x5400
```

![](/files/ff_ole3_10.png)

SBAT를 통해 현재 CompObj가 0, 1번의 Small Sector가 연결되어 있음을 확인할 수 있다. SBAT도 BBAT와 마찬가지로 섹터의 끝을 나타내는 값인 0xFFFFFFFE가 나타날 때 까지 계속해서 next값을 쫓아가면 된다. 만약 참조할 Small Sector Id가 128 이상 인 경우에 확장된 SBAT를 참조해야한다. 확장된 SBAT는 현재 SBAT의 가장 끝(127)에 위치하고 있다. 확장 영역에 대해서는 다음에 자세히 설명하겠다.

<p></p>

<table style="border: 2px solid #dcdcdc;">
    <tr style="background: #fafafa">
        <td>값</td>
        <td>의미</td>
    </tr>
    <tr>
        <td>0xFFFFFFFD</td>
        <td>특수 블록을 의미</td>
    </tr>
    <tr>
        <td>0xFFFFFFFE</td>
        <td>End of Chain</td>
    </tr>
    <tr>
        <td>0xFFFFFFFF</td>
        <td>비어 있음을 의미</td>
    </tr>
    <tr>
        <td>0 ~ </td>
        <td>Next 값 (연결된 다음 섹터 ID)</td>
    </tr>
</table>

CompObj를 이루고 있는 Small Sector의 구조는 다음과 같다.

![](/files/ff_ole3_11.png)

이제 CompObj의 실제 데이터 오브젝트를 읽어오도록 하겠다. 0, 1의 Small Sector를 읽어서 합쳐주면 된다. 

![](/files/ff_ole3_12.png)

CompObj의 크기는 110바이트이다. 먼저 1개의 Small Sector를 읽고 나머지 110 – 64 = 46바이트 크기만큼 다음 Small Sector에서 읽어오면 된다. 남은 18바이트는 버린다. 이렇게 실제로 읽은 데이터는 다음과 같다.

![CompObj Stream Object](/files/ff_ole3_13.png)


## 결론

지금까지 3편에 걸쳐 OLE 파일 구조에 대해서 살펴보았다. 설명은 장황하게 길어보이지만 실제 원리는 간단하기 때문에 내용을 천천히 읽어볼 것을 권장한다.

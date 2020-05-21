---
layout: post
title: 'OLE 파일 구조 분석 (2)'
author: Jin-Hyeong Choe
date: 2020-05-15 09:00
tags: [fileformat, ole]
---

<fieldset style="margin:0px 0px 20px 0px;padding:5px;"><legend><span><strong style="font-weight:bold;">연재 순서</strong></span></legend><!--Creative Commons License--><div style="float: left; width: 88px; margin-top: 3px;"><img alt="Creative Commons License" style="border-width: 0" src="/files/images/exclamationmark.png"/></div><div style="margin-left: 92px; margin-top: 3px; text-align: justify;"><p style="margin: 0"><a href="/2020/05/04/fileformat_ole_1/">1. OLE 파일 구조 분석 (1) - Header, BBAT, SBAT</a></p>
<p style="margin: 0; background:#ddd;">2. OLE 파일 구조 분석 (2) - Root Storage</p>
<p style="margin: 0;"><a href="/2020/05/21/fileformat_ole_3/">3. OLE 파일 구조 분석 (3) - Stream Object</a></p>
</div></fieldset>

## Root Storage Sector ID

Root Storage는 최상위 폴더로서 Small Sector와 다른 스트림, 스토리지들을 포함하고 있기 때문에 헤더 다음으로 가장 먼저 읽어야 되는 항목이다. 이를 읽기 위해서는 헤더에서 Root Storage Sector ID(0x30 ~ 0x33)을 읽어서 BBAT를 참조해야 된다. 

Root Storage를 읽으면 OLE 파일의 전체적인 구조를 파악할 수 있다


Root Storage를 읽는 방법은 다음과 같다. 다음 예제를 보고 직접 따라해보자.

1.	헤더에서 Root Storage Sector ID를 읽는다. 

2.	BBAT를 참조하여 모든 Sector를 연결시킨다.

3.	읽어온 Root Storage와 연결된 모든 디렉토리(Stream/Storage)를 분석한다.

4.	OLE 파일의 전체적인 구조 파악한다.

가장 먼저 헤더에서 0x30 ~ 0x33에 위치한 Root Storage Sector ID를 읽는다.

![OLE 헤더 중 Root Storage Sector ID](/files/ff_ole2_1.png)

현재 이 파일에서 Root Storage Sector ID는 39(0x27)이다. BBAT를 참조하여 Root Storage의 시작인 39번 섹터와 연결된 다른 모든 섹터(스토리지, 스트림)들을 전
부 읽어야한다. 그렇기에 다음으로는 BBAT를 읽는다.

BBAT를 읽기 위해서는 BBAT Depot를 전부 읽고 합쳐야 한다.

![OLE 헤더 중 BBAT Depot](/files/ff_ole2_2.png)

현재 BBAT Depot에 저장된 Big Block Allocation Table의 개수는 0x4C에 위치한 0x26 한 개뿐이다. 그렇기 때문에 이 파일에서 BBAT는 0x26번에 위치한 한 개의 섹터만 읽으면 된다. 

만약 BBAT가 1개가 아닌 여러 개일 경우 모두 순서대로 읽고 연결하여 하나의 BBAT를 만들면 된다. 만약 BBAT의 개수가 BBAT Depot(0x4C~0x1FF)에 저장 가능한 최대 개수인 109개를 초과할 경우 헤더에 Start Block of Extra Big Block Allocation Table를 참조해서 확장된 BBAT Depot를 읽어야 한다. 확장된 BBAT에 대해서는 나중에 다시 상세히 설명하도록 하겠다.

현재 BBAT의 주소는 ```(0x26 + 0x01) * 0x200 = 0x4E00```에 위치한다.

0x26은 BBAT의 시작 주소이고 1을 더하는 이유는 처음에 앞서 설명했듯이 데이터 영역에서 Sector ID로 접근할 때에는 헤더를 제외해야 하기 때문에 더하는 것이다. (헤더의 오프셋을 건너 뛰기 위해서 헤더 크기만큼 더한다.) 그리고 마지막으로 섹터 크기(512)만큼 곱해주면 BBAT의 주소를 구할 수 있다.

![BBAT 영역](/files/ff_ole2_3.png)

0x4E00 ~ 0X4FFF 까지가 BBAT 영역이다. (1섹터의 크기)

BBAT에는 섹터에 연결된 다음 섹터를 나타내는 DWORD(4바이트) 값들이 저장되
어 있다. 즉, 1개의 BBAT(1섹터)에는 128개의 next 값들이 저장되어 있다.

이 파일에서 Root Storage Sector ID는 0x27(39)이다. 그리고 BBAT의 시작 주소는 0x4E00이다. 그렇다면 다음과 같이 BBAT에 처음으로 접근할 수 있다.

```
0x4E00 + (0x27 * 0x04) = 0x4E9C 
```

0x4E00은 BBAT의 시작 주소, 0x27은 Root Storage Sector ID이고 4를 곱하는 이유는 BBAT에 저장된 next값 하나의 크기가 4바이트이기 때문이다.

![BBAT 영역 링크 추적 방법](/files/ff_ole2_4.png)

0x4E9C에는 0x28이라는 값이 저장되어 있다. 섹터 연결의 끝을 나타내는 값인
0xFFFFFFFE가 보일 때까지 계속해서 같은 방식으로 next 값을 계산해서 쫓아가서 
읽은 섹터들을 전부 하나로 연결하면 Root Storage를 완성할 수 있다.

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
<center><figcaption style="font-size:14px;">BBAT에 저장된 DWORD 값의 의미</figcaption></center>

```
0x4E00 + (0x28 * 0x04) = 0x4EA0
```

![](/files/ff_ole2_5.png)

0x4EA0에는 0xFFFFFFFE (FE FF FF FF)가 저장되어 있다. 이걸로 Root Storage는 0x27, 0x28 섹터가 연결되어 있음을 알 수 있다. BBAT를 참조하여 파악한 Root Storage의 구조는 다음과 같다.

![](/files/ff_ole2_6.png)

SECTOR[39]는 Root Storage Sector ID에서 확인한 값이고, SECTOR[40]은 BBAT를 통해 연결된 것을 확인하였다. 이제 Root Storage의 구조를 확인하였으니 실질적인 내용을 읽고 분석해보도록 한다. 먼저 각 섹터들의 메모리를 읽어온다.

```
SECTOR[39] : (0x27 + 0x01) * 0x200 = 0x5000 ~ 0x51FF (512Byte)
```

![39번째 섹터 내용](/files/ff_ole2_7.png)

```
SECTOR[40] : (0x28 + 0x01) * 0x200 = 0x5200 ~ 0x53FF (512Byte)
```

![40번째 섹터 내용](/files/ff_ole2_8.png)

Root Storage 영역(39 ~ 40섹터)에는 OLE 파일을 구성하고 있는 디렉토리(스트림 또는 스토리지)들이 저장되어 있다. 디렉토리 1개의 크기는 0x80(128) 바이트이다. 각각의 디렉토리에는 이름, 크기, 타입, 자식 노드 등 디렉토리에 대한 정보들이 저장되어 있다. 저장된 디렉토리의 데이터 구조는 다음과 같다.

## 디렉토리 데이터 구조

### (1) Name

![](/files/ff_ole2_9.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Name</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x00 ~ 0x3F</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>64바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td></td>
    </tr>
    <tr>
        <td>설명</td>
        <td>Stream 또는 Storage의 이름이다.</td>
    </tr>
</table>

### (2) Name Length

![](/files/ff_ole2_10.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Name Length</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x40 ~ 0x41</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>2바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td></td>
    </tr>
    <tr>
        <td>설명</td>
        <td>앞에서 설명한 Name에 저장된 문자열의 크기. (문자의 끝을 나타내는 NULL Character 포함)</td>
    </tr>
</table>

### (3) Type

![](/files/ff_ole2_11.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Type</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x42</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>1바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td></td>
    </tr>
    <tr>
        <td>설명</td>
        <td>현재 읽고있는 디렉토리의 타입을 나타내는 값<br>
0 : Empty<br>  
1 : Storage<br>
2 : Stream<br>
3 : Lock bytes<br>
4 : Property<br>
5 : Root Storage<br></td>
    </tr>
</table>

### (4) Node Color

![](/files/ff_ole2_12.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Node Color</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x43</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>1바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td></td>
    </tr>
    <tr>
        <td>설명</td>
        <td>현재 읽고있는 디렉토리의 Node Color 값. 
디렉토리들은 Red-Black Tree 구조로 구성되어 있다.
0x00 : Red<br>
0x01 : Black</td>
    </tr>
</table>

### (5) Left Node

![](/files/ff_ole2_13.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Left Node</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x44 ~ 0x47</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td></td>
    </tr>
    <tr>
        <td>설명</td>
        <td>현재 노드(디렉토리)의 좌측 자식 노드의 인덱스 번호. 존재하지 않는 경우 0xFFFFFFFF 값이 기록된다.</td>
    </tr>
</table>

### (6) Right Node

![](/files/ff_ole2_14.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Right Node</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x48 ~ 0x4B</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td></td>
    </tr>
    <tr>
        <td>설명</td>
        <td>현재 노드(디렉토리)의 우측 자식 노드의 인덱스 번호. 존재하지 않는 경우 0xFFFFFFFF 값이 기록된다.</td>
    </tr>
</table>

### (7) Child Node

![](/files/ff_ole2_15.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Child Node</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x4C ~ 0x4F</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td></td>
    </tr>
    <tr>
        <td>설명</td>
        <td>현재 노드(디렉토리)의 자식 노드의 인덱스 번호.
존재하지 않는 경우 0xFFFFFFFF 값이 기록된다.</td>
    </tr>
</table>


### (8) CLSID

![](/files/ff_ole2_16.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>CLSID</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x50 ~ 0x5F</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>16바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td></td>
    </tr>
    <tr>
        <td>설명</td>
        <td>현재 노드의 CLSID</td>
    </tr>
</table>

### (9) User Flag

![](/files/ff_ole2_17.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>User Flag</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x60 ~ 0x63</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td>00 00 00 00</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>User Flag (사용하지 않음)</td>
    </tr>
</table>


### (10) Creation Time

![](/files/ff_ole2_18.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Creation Time</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x64 ~ 0x6B</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>8바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td>00 00 00 00 00 00 00 00</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>생성 시간 (일반적으로 0)</td>
    </tr>
</table>


### (11) Modify Time

![](/files/ff_ole2_19.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Modify Time</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x6C ~ 0x73</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>8바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td>00 00 00 00 00 00 00 00</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>마지막으로 수정한 시간</td>
    </tr>
</table>


### (12) Start Sector ID

![](/files/ff_ole2_20.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Start Sector ID</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x74 ~ 0x77</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td></td>
    </tr>
    <tr>
        <td>설명</td>
        <td>현재 오브젝트의 시작 섹터 ID.<br>
디렉토리의 타입이 Stream 또는 Root Storage인 경우 이 값을 참조해야한다. 헤더에 존재하는 Root Storage Sector ID와 같은 메커니즘이다.</td>
    </tr>
</table>


### (13) Size of Low

![](/files/ff_ole2_21.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Size of Low</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x78 ~ 0x7B</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td></td>
    </tr>
    <tr>
        <td>설명</td>
        <td>현재 오브젝트의 크기 (Low)</td>
    </tr>
</table>


### (14) Size of High

![](/files/ff_ole2_22.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Size of High</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x7C ~ 0x7F</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td>00 00 00 00</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>현재 오브젝트의 크기 (High)<br>
Size of Low 크기의 확장 개념이다. </td>
    </tr>
</table>

## 디렉토리 예

각각의 디렉토리를 분석해보면 다음과 같다.

Left Node, Right Node, Child Node의 값이 0xFFFFFFFF이고 다른 값이 전부 0인 경우 비어있는 공간임을 뜻하며 읽지 않아도 된다. 만약 파일에 디렉토리를 새로 추가하게 될 경우 빈 공간을 먼저 활용하고 공간이 부족할 경우 새로 할당한다.

![](/files/ff_ole2_23.png)

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

- Directory[2]
<table class="tb_ole">
    <tr>
        <td>Name</td>
        <td>WordDocument</td>
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
        <td>0x01</td>
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
        <td>0x1000</td>
    </tr>  
</table>

- Directory[3]
<table class="tb_ole">
    <tr>
        <td>Name</td>
        <td>\005SummaryInformation</td>
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
        <td>0x02</td>
    </tr>
    <tr>
        <td>Right Node</td>
        <td>0x04</td>
    </tr>
    <tr>
        <td>Child Node</td>
        <td></td>
    </tr>  
    <tr>
        <td>Start Sector Id</td>
        <td>0x16</td>
    </tr> 
   <tr>
        <td>Size</td>
        <td>0x1000</td>
    </tr>  
</table>

![](/files/ff_ole2_24.png)

- Directory[4]
<table class="tb_ole">
    <tr>
        <td>Name</td>
        <td>\005DocumentSummaryInformation</td>
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
        <td></td>
    </tr>
    <tr>
        <td>Child Node</td>
        <td></td>
    </tr>  
    <tr>
        <td>Start Sector Id</td>
        <td>0x1E</td>
    </tr> 
   <tr>
        <td>Size</td>
        <td>0x1000</td>
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

분석한 각각의 디렉토리 정보들을 통해 저장된 메모리 구조와 OLE 파일에 저장된 디렉토리(스토리지 또는 스트림)의 구조를 다음과 같이 파악할 수 있다.

![](/files/ff_ole2_25.png)

![](/files/ff_ole2_26.png)

![](/files/ff_ole2_27.png)


## 결론

이번 글에서는 Root Storage를 수집하여 해당 디렉토리에 어떤 스트림과 스토리지가 존재하는지 확인하는 방법을 살펴보았다. 다음에는 스트림을 읽는 방법에 대해서 알아보기로 한다.

---
layout: post
title: 'OLE 파일 구조 분석 (1)'
author: Jin-Hyeong Choe
date: 2020-05-04 18:00
tags: [fileformat, ole]
---

## OLE File Format

OLE란 Object Linking and Embedding 약자로서 객체 연결 및 삽입을 뜻하며
복합 이진 파일 형식(Compound File Binary Format)이라고도 한다.

Microsoft에서 제작한 OLE File Format은 마치 하나의 작은 파일 시스템과 같은 구조를 지니고 있어 상/하위 파일 버전에 대한 뛰어난 호환성을 가지고 있다. 그렇기에 MS Office 외에 다른 워드 프로세서들도 OLE 파일 형식을 사용하고 있다.

![OLE 파일 구조](/files/ff_ole1_1.png)


## Storage and Stream

OLE 파일 구조는 마치 하나의 작은 파일 시스템과 같은 구조를 가지고 있다. 스트림은 문서 내용을 담고 있는 파일, 스토리지는 파일을 담는 폴더와 같다. 스토리지와 스트림의 내용은 문서 종류(doc, hwp, xls 등)에 따라 다를 수 있다.

![파일 종류에 따른 OLE 내부 구조](/files/ff_ole1_2.png)

## Header and Data Section

OLE 파일 구조는 크게 헤더 영역과 데이터 영역으로 나뉜다.

![OLE 파일 구조 (헤더와 데이터)](/files/ff_ole1_3.png)

헤더 영역에는 File Signature, Sector Count, Sector Size, Start Sector ID 등 OLE 파일을 구성하고 있는 주요 정보들이 저장되어 있고, 데이터 영역에는 앞서 설명한 스트림과 스토리지 그리고 그것들을 구성하고 있는 테이블 정보(BBAT, SBAT)가 저장된다. 데이터 영역 대부분은 스트림이 차지한다

## File Sector

OLE 파일은 섹터 단위로 구성된다. 1섹터 크기는 512바이트이다. 예를 들어 만약 파일 크기가 2560바이트일 경우 헤더 영역은 고정 512바이트, 데이터 영역은 파일 크기에서 헤더 크기만큼 뺀 2560-512=2048바이트로 구성된 것이다. 여기서 주의해야할 점은 나중에 데이터 영역에 Sector ID로 접근을 할 때 헤더 크기는 빼고 계산해야 한다는 것이다. 이 부분은 나중에 다시 설명하도록 하겠다. 

![OLE 파일 구조 (섹터)](/files/ff_ole1_4.png)
![OLE 파일 구조 (헤더와 데이터 섹터 맵핑)](/files/ff_ole1_5.png)

## Header Information

OLE 파일에서 가장 먼저 읽어야 할 것은 헤더이다. 헤더에는 OLE 파일에 대한 다양한 정보를 저장하고 있다. 헤더 영역은 파일 시작 위치부터 1섹터(512바이트) 만큼이다. Hex Viewer 또는 Microsoft에서 제공하는 OffVis를 통해 직접 확인해 볼 수 있다.

![OLE 파일 구조 (헤더: Magic ID)](/files/ff_ole1_6.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Magic ID</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x00 ~ 0x07</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>8바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td>D0 CF 11 E0 A1 B1 1A E1</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>OLE 파일임을 나타내는 Signature. <br>정상적인 OLE 파일의 경우 이 값은 반드시 0xE11AB1A1E011CFD0 이어야 한다.</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: CLSID)](/files/ff_ole1_7.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>CLSID</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x08 ~ 0x17</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>16바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td>0x00 …</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>OLE CLSID 값</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: Minor Version)](/files/ff_ole1_8.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Minor Version</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x18 ~ 0x19</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>2바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td> </td>
    </tr>
    <tr>
        <td>설명</td>
        <td>파일의 Minor Version 값</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: Major Version)](/files/ff_ole1_9.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Major Version</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x1A ~ 0x1B</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>2바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td> </td>
    </tr>
    <tr>
        <td>설명</td>
        <td>파일의 Major Version 값</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: Byte Order)](/files/ff_ole1_10.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Byte Order</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x1C ~ 0x1D</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>2바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td>FE FF(Little Endian), FF FE(Big Endian)</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>값 저장 방식</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: Sector Shift (BBAT))](/files/ff_ole1_11.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Sector Shift (BBAT)</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x1E ~ 0x1F</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>2바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td>09 00</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>1 섹터 크기 (2 승수로 나타내며 9인 경우 2^9 = 512)</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: Small Sector Shift (SBAT))](/files/ff_ole1_12.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Small Sector Shift (SBAT)</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x20 ~ 0x21</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>2바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td>06 00</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>SBAT 섹터 1개 크기 (2 승수로 나타내며 6인 경우 2^6 = 64)</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: Reserved1)](/files/ff_ole1_13.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Reserved1</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x22 ~ 0x23</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>2바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td>0x00 …</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>사용하고 있지 않음</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: Reserved2)](/files/ff_ole1_14.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Reserved2</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x24 ~ 0x27</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td>0x00 …</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>사용하고 있지 않음</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: Reserved3)](/files/ff_ole1_15.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Reserved3</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x28 ~ 0x2B</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td>0x00 …</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>사용하고 있지 않음</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: Number of Big Block Allocation Table Depot)](/files/ff_ole1_16.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Number of Big Block Allocation Table Depot</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x2C ~ 0x2F</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td> </td>
    </tr>
    <tr>
        <td>설명</td>
        <td>BBAT Depot 개수를 나타낸다. 이 개수가 109개 이상일 경우엔 Extra BBAT를 참조해야 한다.</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: Root Storage Sector ID)](/files/ff_ole1_17.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Root Storage Sector ID</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x30 ~ 0x33</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td> </td>
    </tr>
    <tr>
        <td>설명</td>
        <td>Root Entry(Root Storage) 시작 섹터 번호. 헤더 다음으로 가장 먼저 읽어야할 부분이다. 이 위치가 0x27인 경우 다음과 같이 계산할 수 있다. 이전에 앞서 설명한대로 헤더 다음 섹터부터 0번이기 때문에 헤더 크기만큼 추가로 더해야한다.<br>
0x27 * 0x200(Sector Size) + 0x200(Header Size) = 0x5000</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: Reserved4)](/files/ff_ole1_18.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Reserved4</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x34 ~ 0x37</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td>0x00 …</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>사용하고 있지 않음</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: Max Small Stream Size)](/files/ff_ole1_19.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Max Small Stream Size</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x38 ~ 0x3B</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td>00 10 00 00</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>이 사이즈 미만인 스트림을 읽을 때는 SBAT를 참조해야 한다. 이 사이즈 미만인 스트림은 Small Stream이다.</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: SBAT Depot Start Sector ID)](/files/ff_ole1_20.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>SBAT Depot Start Sector ID</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x3C ~ 0x3F</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td> </td>
    </tr>
    <tr>
        <td>설명</td>
        <td>Small Block Allocation Table Depot 시작 섹터 번호이다. 이 위치를 따라가면 SBAT Depot를 만날 수 있다.</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: SBAT Sector Count)](/files/ff_ole1_21.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>SBAT Sector Count</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x40 ~ 0x43</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>4바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td> </td>
    </tr>
    <tr>
        <td>설명</td>
        <td>Small Block Allocation Table 크기 값이며, 몇 개의 섹터를 차지하는지에 대한 값이다. 1이면 SBAT 크기는 다음과 같이 결정된다.<br>
1 * 512(Sector Size) = 512</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: Start Block of Extra Big Block Allocation Table)](/files/ff_ole1_22.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Start Block of Extra Big Block Allocation Table</td>
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
        <td>FE FF FF FF (BBAT 개수가 109개 이하일 경우)</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>확장된 BBAT 시작 섹터 번호. Number of Big Block Allocation Table 값이 109개가 넘어서면 여기를 참조해야 한다. 이 값을 따라가면 확장된 BBAT가 나오게 된다.</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: Number of Extra Big Block Allocation Table)](/files/ff_ole1_23.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>Number of Extra Big Block Allocation Table</td>
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
        <td>00 00 00 00 (BBAT 개수가 109개 이하일 경우)</td>
    </tr>
    <tr>
        <td>설명</td>
        <td>앞서 나온 확장된 BBAT 개수. 예를 들어 만약 BBAT가 120개여서 확장된 경우 이 값은 확장된 크기인 11이다.</td>
    </tr>
</table>

![OLE 파일 구조 (헤더: BBAT Depot)](/files/ff_ole1_24.png)

<table class="tb_ole">
    <tr>
        <td>이름</td>
        <td>BBAT Depot</td>
    </tr>
    <tr>
        <td>위치</td>
        <td>0x4C ~ 0x01FF</td>
    </tr>
    <tr>
        <td>크기</td>
        <td>436바이트</td>
    </tr>
    <tr>
        <td>일반적인 값</td>
        <td> </td>
    </tr>
    <tr>
        <td>설명</td>
        <td>여기서부터 헤더 끝인 0x01FF까지 109개의 BBAT 엔트리들이 저장된다. 4바이트씩 Sector ID를 가지고 있으며, BBAT 개수가 109개 이상일 경우 Start Block of Extra Big Block Allocation Table를 참조해야 한다.</td>
    </tr>
</table>

## BBAT (Big Block Allocation Table)

OLE 파일은 섹터 단위로 이루어져 있다. 마찬가지로 OLE 파일을 구성하고 있는 데이터들도 전부 섹터 단위로 구성되어 있다. 1섹터 크기는 512바이트이다. 512바이트를 초과하면 한 개 섹터에 데이터를 모두 저장할 수가 없기 때문에 나눠서 저장을 해야한다. 예를 들어 4480바이트 크기 데이터는 9섹터에 각각 512바이트씩 나눠서 저장한다.

![BBAT Depot 참조 과정](/files/ff_ole1_25.png)

![BBAT Depot 데이터](/files/ff_ole1_26.png)

BBAT는 나눠서 저장한 데이터들을 연결하는 연결고리 저장소이다. 데이터들은 항상 순차적으로 섹터에 저장하지 않기 때문에 반드시 BBAT를 참조하여 직접 연결을 해줘야만 한다. BBAT에는 참조하는 섹터에 연결된 다음 섹터 번호가 저장되어 있다. 데이터를 이루는 섹터는 단일 연결 구조로 구성되어 있다.

## Small Block Allocation Table (SBAT) 

파일 헤더에 나와있는 Max Small Stream Size보다 크기가 작은 스트림을 읽을 땐 BBAT가 아닌 SBAT를 참조해야 한다. SBAT는 BBAT와 똑같은 메커니즘이다. Small Sector 한 개 크기는 64바이트이다. 만약 256바이트 스트림을 저장할 경우 4개의 Small Sector에 각각 64바이트씩 저장해야 한다. Small Stream (Small Sector)는 Root Storage에 저장되어 있다. 따라서 Small Stream을 읽기 위해서는 Root Storage를 BBAT를 통해 먼저 읽어야 한다. 

![SBAT Depot 참조 과정](/files/ff_ole1_27.png)

![SBAT Depot 데이터](/files/ff_ole1_28.png)

## Root Storage (Root Entry)

헤더 다음으로 가장 먼저 읽어야 할 항목이다. 파일 내에 존재하는 모든 스트림과 스토리지를 포함하고 있는 최상위 폴더이고 Small Stream (Small Sector)을 저장하고 있다. BBAT를 참조하여 헤더의 Root Storage Sector ID와 연결된 섹터들을 전부 합친다면 파일 전체적인 구조를 파악할 수 있다.

![Root Storage 구조 출력](/files/ff_ole1_29.png)


## 결론

이번 글에서는 OLE 파일 구조 전체적인 정보에 대해 기술하였다. 다음에는 Root Storage 구조에 대한 자세히 알아보기로 한다.

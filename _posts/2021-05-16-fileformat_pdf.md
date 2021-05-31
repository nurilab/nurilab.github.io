---
layout: post
title: 'PDF 파일 구조 분석'
author: Ji-Hun Kim
date: 2021-05-16 09:00
tags: [fileformat, pdf]
---

<fieldset style="margin:0px 0px 20px 0px;padding:5px;"><legend><span><strong style="font-weight:bold;">연재 순서</strong></span></legend><!--Creative Commons License--><div style="float: left; width: 88px; margin-top: 3px;"><img alt="Creative Commons License" style="border-width: 0" src="/files/images/exclamationmark.png"/></div><div style="margin-left: 92px; margin-top: 3px; text-align: justify;">
<p style="margin: 0; background:#ddd;">첫번째 글: PDF 파일 구조 분석</p>
<p style="margin: 0;"><a href="/2021/05/18/pdf_exploit/">두번째 글: PDF 취약점과 진단 방안</a></p>
<p style="margin: 0;"><a href="/2021/05/31/pdf_exploit_scan/">세번째 글: 악성 PDF 파일 진단</a></p>
</div></fieldset>

## 1. 분석 목적

PDF 파일의 구조를 파악하여 취약점을 확인하고 파일의 악성 여부 진단이 가능하도록 한다.

## 2. PDF의 구조

![<그림 1-1> PDF 파일의 전체 구조](/files/ff_pdf_1.png)

PDF 파일은 <그림 1-1> 과 같이 크게 Header, Body, Cross Reference Table, Trailer 부분으로 이루어 진다.

### (1) PDF Header

8 Byte의 크기를 가지며, PDF 파일의 Signature 값과(%PDF) 버전정보를 담는다.

### (2) PDF Body

PDF 의 Body 영역은 PDF 문서가 가지는 내용에 대한 Object 들이 기록되어 있다.
해당 Object는 다양한 압축, 암호화 형태를 가질 수 있으며, 일반적으로 ```obj``` ~ ```endobj``` 문자열 사이에 데이터가 존재한다.

### (3) Cross Reference Table

PDF 의 Cross Reference Table(이하 xref table) 는 Body 에 존재하는 Object 들에 대한 위치정보, 오브젝트가 유효한지 등의 정보를 가진다. 해당 테이블은 일반적으로 문자열로 존재하는 Table 형태를 가지지만, 최신 문서의 경우 Table 이 아닌 Binary Stream 형태를 가질 수도 있으며, 두 가지가 결합된 경우도 있을 수 있다. 또한 1개의 PDF 에서 여러 개의 xref table 이 존재 할 수 있다.

### (4) Trailer

PDF 의 Trailer 은 PDF 분석 시 Header 의 유효성 체크 후 가장먼저 분석 되어야 할 위치이다. Trailer 은 xref table 의 위치, 특수 Object (암호화 Object, Root Object, ID 값 등)의 위치 정보를 가진다. 이 Trailer 는 1개의 PDF 에서 여러 번 존재 할 수 있다.

### (5) PDF 파일이 수정된 경우의 변화

![<그림 1-2> PDF 파일이 수정된 경우 구조](/files/ff_pdf_2.png)

PDF 파일이 수정되는 경우 위 <그림 1-2>와 같이 기존 구조에 덧붙어 기록이 된다. 따라서 PDF 를 분석하기 위해서는 수정된 모든 Trailer 와 xref table, Body 를 분석 하여야 한다.

## 3. Object Data type

PDF문서는 여러 작은 Object들로 구성된 모임이다. Object는 다음과 같은 8가지 기본적인 유형을 갖는다.

### (1) Boolean

오브젝트 내에서 참(```True```)과 거짓(```False```)를 나타낼 때 사용되며 배열의 요소나 Dictionary 의 Entry 값으로서 사용된다.

![<그림 1-3> Boolean Data type 예시](/files/ff_pdf_3.png)

### (2) Numeric

숫자를 표현할 때 사용되며 크게 Integer object와 Real object로 구분된다. 

![<그림 1-4> Numeric Data type 예시](/files/ff_pdf_4.png)

### (3) String

문자열을 나타낼 때 사용되며 Literal String과 Hexadecimal String가 있다.
Literal String은 ```(```로 문자열의 시작을, ```)```로 문자열의 끝을 알린다.
또한, Literal String 내에서는 ```(```, ```)``` 또는 ```\```를 허용하지 않는다.
```(```, ```)``` 쌍으로 이뤄져 있을 때에는 특별한 처리 없이 문자열로 인식되며, ```(```, ```)```또는 특수 문자들을 문자 상수로 표현하기 위해 ```\```가 사용되기 때문이다.

![<그림 1-5> String Data type (Literal String) 예시](/files/ff_pdf_5.png)

Hexadecimal String은 ```<```와 ```>``` 안의 0-9, A-F로 이루어진 2바이트의 데이터가 하나의 Hex data로 인식되는 문자열이다. 2바이트 짝이 맞지 않을 경우 마지막에 0이 생략된 것으로 간주한다.

![<그림 1-6> String Data type (Hexadecimal String) 예시](/files/ff_pdf_6.png)

### (4) Name

Name은 하나의 symbol로써 같은 문자열의 Name을 갖는 다른 Object가 있을 수 없다. Name은 ```/```이후의 문자열로, ```#```이후 등장하는 2바이트의 문자는 하나의 Hex값으로 취급한다.

![<그림 1-7 > Name Data type 예시](/files/ff_pdf_7.png)

### (5) Array

Array는 ```[```로 시작하여 ```]```로 끝나는 배열을 의미한다.
배열 내 각 요소들은 Array 포함한 PDF의 모든 Data type을 가질 수 있다. 

![<그림 1-8> Array Data type 예시](/files/ff_pdf_8.png)

### (6) Dictionary

Dictionary는 ```<<```와 ```>>``` 내에서 Key-Value의 쌍으로 구성된다. 
Key는 반드시 Data type이 Name이여야 하고, Value는 Dictionary를 포함한 모든 Data type을 가질 수 있다. 

![<그림 1-9> Dictionary Data type 예시](/files/ff_pdf_9.png)

### (7) Stream

Stream은 길이에 제한이 없는 연속적인 바이트의 집단이다.
상대적으로 크기가 큰 이미지나 파일 내용은 Stream을 통해 표현한다.

![<그림 1-10> Stream Data type 예시](/files/ff_pdf_10.png)

Stream은 해당 Object내에 선행되는 Dictionary에서 몇 가지 속성을 지시한다.

![<그림 1-11> Stream에 대한 Dictionary의 Entries](/files/ff_pdf_11.png)

### (8) Indirect

Indirect는 ```n1 n2 R```과 같이 표현되며 참조하는 오브젝트를 의미한다.

![<그림 1-12> Indirect Data type 예시](/files/ff_pdf_12.png)

### (9) 계층 구조

Body 섹션 안의 Object들은 트리 형태로 계층적 구조를 이루고 있다.
계층의 최상위인 Root Object는 항상 catalog dictionary Data type을 가지며, Indirect를 통해 다른 모든 개체에 도달 할 수 있다.

![<그림 1-13> Object의 기본적인 계층 구조](/files/ff_pdf_13.png)


## 4. 마무리

PDF Exploit을 분석하는 가장 기본은 역시 PDF 파일의 구조를 아는 것이다. 이후부터는 PDF Exploit이란 무엇이며 PDF Exploit 진단 방법 및 악성코드 진단 예시를 살펴볼 예정이다.

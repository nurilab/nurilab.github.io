---
layout: post
title: '파이썬을 이용하여 파일 업로드/다운로드 자동화하기'
author: Seok-Eun Park
image: /files/covers/python.jpg
date: 2020-04-13 18:00
tags: [python]
---
## 개요

누리랩에서는 악성코드를 수집하고 이를 분석한 뒤 유사도를 이용한 악성코드 탐지 정보를 아마존 EC2에 업로드한다. 누리랩의 주요 제품들은 이 정보를 다운로드 하여 악성코드 차단을 수행하게 된다.

악성코드는 아래 그림에서 보듯이 고객 또는 누리랩의 제품 서비스 그리고 허니팟등을 이용해 수집 서버에 파일을 모아두게 되는데 이는 FTP 접속을 통해 악성코드를 내려 받을 수 있다. 하지만 아마존 EC2는 SFTP를 통해 악성코드 유사도 정보를 업로드해야 한다.

 ![누리랩 제품 서비스 구성도](/files/python_ftp_1.png)

여기에서는 파이썬을 이용해 FTP 서버에서 파일을 업로드/다운로드 방법과 아마존 EC2 인스턴스에 SFTP 접속하여 파일을 업로드/다운로드 하는 방법을 설명한다.

## FTP에 파일 업로드/다운로드하기

파이썬은 ```ftplib``` 모듈을 기본적으로 지원한다. 따라서 복잡한 설정없이 간단하게 FTP를 통해 파일을 다운로드 할 수 있다.

```
from ftplib import FTP

ftp = FTP('주소')
ftp.login('아이디', '비번')

ftp.cwd('test') # "test"디렉터리로 이동
ftp.retrlines('LIST') # 디렉터리의 내용을 목록화
ftp.retrbinary('RETR README', open('README', 'wb').write) # README 파일 저장
ftp.quit()
```

업로드 하는 방법은 다음과 같다.

```
ftp.cwd('upload')  # 업로드할 FTP 폴더로 이동
myfile = open(filename,'rb')  # 로컬 파일 열기
ftp.storbinary('STOR ' +filename, myfile )  # 파일을 FTP로 업로드
myfile.close()  # 파일 닫기
```

위 예제에서 볼 수 있듯이 파이썬의 ```ftplib``` 모듈은 사용법이 그렇게 어렵지 않다. 


## 아마존 EC2에 파일 업로드/다운로드하기

아마존 EC2는 FTP를 지원하지 않는다. 대신 SFTP를 지원하는데 파이썬의 ```fptlib``` 모듈은 SFTP를 지원하지 않는다. 따라서 ```paramiko``` 모듈을 별도로 설치해야 한다.

```
pip install paramiko
```

```paramiko```의 자세한 정보는 <http://www.paramiko.org/>에서 참고하기 바란다.

SFTP에 접속하여 파일을 다운로드 간단한 소스 코드는 아래와 같다.

```
import paramiko

host = '주소'
username = '아이디'
ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect(host, username=username, key_filename='key파일명.pem')
sftp = ssh.open_sftp()

sftp.get(filepath, localpath)  # 파일 다운로드

sftp.close()
ssh.close()
```

파일을 업로드하기 위해서는 다음과 같이 하면 된다.

```
sftp.put(localpath, filepath)  # 파일 업로드
```

여기에서 눈여겨 봐야 할 부분은 파일 ```key파일명.pem```이다. 일반적으로 아마존 EC2에서 제공해주는 Key 파일은 확장자가 ```.ppk```이다. 이 파일명을 ```ssh.connect()``` 함수의 인자 값으로 입력하면 접속 실패한다.

 
## .ppk를 .pem으로 변경하기

아마존 EC2 인스턴스에 SFTP로 접속하기 위해서는 ```파일명.pem```파일이 필요하다. 만약 제공받은 key 파일이 ```.ppk```인 경우, 아래와 같은 방법으로 ```.pem``` 파일로 변경 가능하다.

(1) PuTTY Key Generator 실행하여, ```File -> Load Private Key``` 선택하여 ```*.ppk``` 파일을 로드한다.

 ![ppk 파일 로드하기](/files/python_ftp_2.png)

(2) ```Conversions``` 탭을 클릭하여 ```Export OpenSSH Key```를 선택하게 되면 아래와 같이 경고창이 하나 뜨는데, ```예(Y)```를 선택한다.

 ![Key(*.pem) 내보내기](/files/python_ftp_3.png)

(3) 2번에서 ```예(Y)```를 선택하면 Key 파일을 어디에 저장할 것인지 묻는다. 저장 위치를 선택하면 ```파일명.pem```파일이 생성된다.

 
## 참고 자료

- <https://python.flowdas.com/library/ftplib.html>
- <http://docs.paramiko.org/en/stable/api/sftp.html>

---
layout: post
title: '우리도 스웩 (Swagger를 이용한 REST API 설계 및 검증)'
author: Kei Choi
image: /files/covers/python.jpg
date: 2020-04-19 12:00
tags: [python]
---

## REST API 개발로 전환하게 된 배경

누리랩은 주로 보안 관련 라이브러리 개발해 판매해왔다. 그러다보니 윈도우용 dll, 리눅스용 so 뿐만 아니라 업체 요구사항에 따라 자바 연동을 위해 JNI를 개발해야 하는 경우도 많았다. 심지어 업체 제품 운영환경까지 고려하다보면 솔라리스, AIX로 보안 라이브러리를 포팅하기도 했다. 그러다 보니 라이브러리 개발에 있어 업체 운영환경에 영향을 최소화할 방법이 필요했다.

이후 누리랩의 대다수 보안 라이브러리는 REST API 방식을 채택하게 되었고 필요에 따라 도커를 통해 업체 장비에 임베딩하는 형태로 개발되기도 한다.

## REST API

REST API[^1]는 기본적으로 http 또는 https를 이용하여 URI를 통해 서비스를 요청하고 주로 JSON 또는 XML로 응답받는 구조를 취한다. 따라서 호출하는 쪽은 서비스 호출 규약만 알고 있다면 라이브러리 호출만큼이나 간단하다. 과거 dll과 같은 라이브러리를 제공할 경우 dll 내부에서 오류가 발생하면 실행 파일 전체가 죽어버리는 문제가 존재하지만 REST API로 구성된 서비스는 별도의 프로그램이기 때문에 서비스가 죽더라도 원래 프로그램에는 크게 영향을 미치지 않는다. 

## REST API 매뉴얼과 예제 코드

누리랩 대다수의 라이브러리를 REST API로 만든 후 이제 이 서비스를 호출하는 방법을 담은 매뉴얼과 예제 코드를 업체 개발자에게 전달하는 일이 필요하다.

초창기에는 진짜 이 매뉴얼을 메모장으로 작성해서 전달했다. 예제 소스코드와 함께... (지금 생각해보니 충격 그 자체다!)

![초창기 서비스 텍스트 매뉴얼](/files/swagger_1.png)

![초창기 서비스 텍스트 예제 소스 코드](/files/swagger_2.png)

아무래도 서비스가 아무리 좋아도 보여지는게 허술하면 제 값을 받기가 힘들다. 그래서 다음으로 채택한 것이 스핑크스(Sphinx)[^2]를 이용한 매뉴얼 작업이었다.

![스핑크스를 이용한 매뉴얼](/files/swagger_3.png)

확실히 스핑크스를 이용하여 작성된 매뉴얼은 텍스트 매뉴얼보다는 보기에도 좋고 가독성이 뛰어나다. 하지만 예제 소스 코드는 여전히 별도로 제공해야 한다.

실제로 매뉴얼 및 예제 소스 코드도 개발자가 별도로 관리하여 항상 최신으로 유지해야하는 번거로움이 있다. 그렇지 않으면 실제 동작하는 서비스와 매뉴얼은 전혀 다른 결과 값을 제공하기도 한다.

이를 해결하려고 도입한 것이 Swagger이다.

# Swagger

Swagger는 Open Api Specification(OAS)를 위한 프레임워크로 REST API 스펙 명세 및 관리를 위한 프로젝트이다. 공식 홈페이지는 다음과 같다.

- <https://swagger.io/>

대부분 파이썬에서 REST API 서비스를 만드는 가장 간단한 방법으로 Flask를 많이 사용한다. Flask용 Swagger 툴이 존재하는데 ```flask_restplus```이다. ```pip``` 명령어를 통해 간단하게 설치할 수 있다.

```
pip install flask_restplus
```

```flask_restplus```의 자세한 설명은 아래 홈페이지를 참조하기 바란다.

- <https://flask-restplus.readthedocs.io/en/stable/>

## flask_restplus를 이용한 계산기 예제

그럼 간단하게 ```flask_restplus```를 어떻게 사용하는지 간단한 계산기를 예로 들어보겠다.

```
# -*- coding:utf-8 -*-
# Kei Choi (whchoi@nurilab.com)

# REST API로 구현한 계산기 예제

import werkzeug
werkzeug.cached_property = werkzeug.utils.cached_property

from flask import Flask
from flask_restplus import Resource, Api, reqparse


# -----------------------------------------------------
# api
# -----------------------------------------------------
app = Flask(__name__)
api = Api(app, version='1.0', title='Calc API',
          description='계산기 REST API 문서',)

ns = api.namespace('calc', description='계산기 API 목록')
app.config.SWAGGER_UI_DOC_EXPANSION = 'list'  # None, list, full


# -----------------------------------------------------
# 덧셈을 위한 API 정의
# -----------------------------------------------------
sum_parser = ns.parser()
sum_parser.add_argument('value1', required=True, help='연산자1')
sum_parser.add_argument('value2', required=True, help='연산자2')


@ns.route('/sum')
@ns.expect(sum_parser)
class FileReport(Resource):
    def get(self):
        """
                Calculate addition
        """
        args = sum_parser.parse_args()

        try:
            val1 = args['value1']
            val2 = args['value2']
        except KeyError:
            return {'result': 'ERROR_PARAMETER'}, 500

	result = {'result': 'ERROR_SUCCESS', 'value': int(val1) + int(val2)}
        return result, 200


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)  # , debug=True)
```

위 예제에서 볼 수 있듯이 간단한 덧셈 계산기를 REST API로 만들었다. 이제 이 파이썬 소스코드를 실행해보자 그러면 5000번 포트로 웹 서버를 구동하게 된다. 웹브라우저를 실행하여 아래 주소로 접근해보자.

```
http://127.0.0.1:5000
```

그럼 다음과 같은 화면을 볼 수 있다.

![Swagger를 적용한 계산기 API 문서 겸 사용 예제](/files/swagger_4.png)

현재 계산기 API 목록은 하나만 존재하면 ```GET``` 방식으로 ```/calc/sum``` URL을 호출하면 덧셈을 할 수 있다고 간략한 설명이 나온다. 여기에서 ```GET```을 클릭하면 구체적인 사용 방법을 볼 수 있다.

![덧셈 API 사용 방법에 대한 설명](/files/swagger_5.png)

위 그림에서 볼 수 있듯이 파라미터로 ```value1```과 ```value2```를 통해 값을 전달하면 결과가 리턴된다. 하지만 이것만으로는 설명이 부족한 듯 하여 실제 사용을 통해 결과를 보길 원할 것이다. 우측 상단에 있는 ```Try it out```을 사용하여 실제로 덧셈 API를 실행을 해 볼 수 있다.  

![덧셈 API 실행 결과](/files/swagger_6.png)

```value1```과 ```value2```에 차례로 숫자 값을 입력 한 후 ```Execute``` 버튼을 우르면 다음과 같은 결과가 나온 것을 확인할 수 있다.

```
{
  "result": "ERROR_SUCCESS",
  "value": 8
}
```

실제 ```3+5```의 결과가 JSON으로 리턴되었다. 덧셈 결과 그림에서 볼 수 있듯이 ```curl``` 명령어를 통해서 실행하는 방법에 대해서도 보여주고 있다. 자세히 보면 ```curl``` 명령어를 통해 아래 URL에 접속하는 것을 확인할 수 있다.

```
http://127.0.0.1:5000/calc/sum?value2=3&value1=5
```

실제 웹브라우저를 통해 해당 URL에 접속하면 다음과 같은 결과를 볼 수 있다.

![웹브라우저를 통해 덧셈 API 실행 결과](/files/swagger_7.png)

인터넷을 통해 다양한 Swagger 예제도 참고하기 바란다[^3].

## Swagger를 이용한 예

계산기 예제에서 볼 수 있듯이 REST API를 실제 개발만 하였을 뿐인데 간단한 문서(매뉴얼)와 예제 소스 코드(위 예제에서는 curl만)가 생성되었으며 간단히 실행도 할 수 있었다.

따라서 개발자 입장에서는 별도 문서화 작업 및 예제 소스코드를 만들기 위한 노력이 상당히 줄어 든다. 

실제 Swagger를 이용한 예는 많다. 그 중에서도 대표적인 곳이 VirusTotal API 문서이다.

- <https://developers.virustotal.com/reference>

![VirusTotal의 Swagger 사용 예](/files/swagger_8.png)


## 결론

확실히 Swagger은 REST API 개발자 입장에서는 필요하다. 초창기 텍스트 매뉴얼에서 이제는 실제 문서 겸 실행되는 예제를 바로 웹사이트에서 볼 수 있으니 "매뉴얼데로 했는데 안되요", "예제 소스 코드에 오타가 있어서 이상해요"라는 문의는 없어질 것이다.

Swagger 화면은 웹으로 되어 있으니 각 회사에 맞는 테마를 입혀 준다면 한층더 간지나는(?) 서비스를 제공할 수 있을 것이다. 그래서 스웨거(Swagger)인가?

![출처 : JTBC 아는 형님](/files/swagger_9.jpg)

그렇다면 "우리도 스웩" 해야지~

## 참조

[^1]: RESTful API란?: <https://brainbackdoor.tistory.com/53>
[^2]:  파이썬 소스코드 문서화하기: <http://www.hanul93.com/python-sphinx/>
[^3]: Flask + REST API + Swagger: <https://blog.naver.com/PostView.nhn?blogId=wideeyed&logNo=221571623994>
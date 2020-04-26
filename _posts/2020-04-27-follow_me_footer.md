---
layout: post
title: '꼬리야.. 날 살려라 (HTML footer 강제 위치 변경)'
author: Kei Choi
date: 2020-04-27 00:00
tags: [html, css, javascript]
---

## 개요

최근 루페(LUPE) 홈페이지 개편을 준비하고 있다. 누리랩 홈페이지와 루페 홈페이지가 너무 동떨어진 느낌을 주는 것 같아 가급적 누리랩 홈페이지와 동일한 모습을 갖추는 것을 목표로 하였다. 물론 개편할 때 이전에 부족했던 부분을 최대한 개선하는 것도 중요하다.

## 깨진 유리창 법칙

깨진 유리창 이론이라는 것이 있다. 미국 범죄학자인 제임스 윌슨과 조지 켈링이 1982년 3월에 공동 발표한 "깨진 유리창(Fixing Broken Windows: Restoring Order and Reducing Crime in Our Communities)"이라는 글에 처음으로 소개된 사회 무질서에 관한 이론으로 깨진 유리창 하나를 방치해 두면, 그 지점을 중심으로 범죄가 확산되기 시작한다는 이론이다. 사소한 무질서를 방치하면 큰 문제로 이어질 가능성이 높다는 의미를 담고 있다. "깨진 유리창 법칙(마이클 레빈 저)"에서는 깨진 유리창을 통해 기업 비즈니스에 큰 영향을 미치는 것에 주의를 기울이라고 조언한다[^1].

<center><p /><img src="/files/book_001.jpg"><br><font size="2">깨진 유리창 법칙(마이클 레빈 저)</font></center>

## 눈에 거슬렸던 깨진 유리창 - footer

루페 홈페이지에서 깨진 유리창은 다름 아닌 footer였다. 아래는 정상적인 루페 홈페이지 화면이다.

![루페 홈페이지](/files/js_footer_1.png)

정상적으로 보이는 이 홈페이지의 비밀은 바로 홈페이지 높이에 변화를 주면 아래와 같은 모습이 된다. 즉, body 내용과 footer과 겹쳐지는 모습을 보이게 된다.

![루페 홈페이지 (body와 footer의 겹칩 현상)](/files/js_footer_2.png)

이번 홈페이지 개편에서는 이 문제를 해결하고 싶었다.

## CSS를 사용한 위치 지정 : position

이전 홈페이지에서 footer는 css의 위치는 ```position: relative;```로 기준점을 잡고 ```position: absolute;```를 사용하여 상대 거리를 지정하였다.

즉, 홈페이지 좌측 상단을 기준점으로 하였고 눈에 보이는 홈페이지 높이(```window.innerHeight```)에서 footer의 높이만큼을 빼서 footer를 위치시켰다. 어렵게 이야기했지만 항상 눈에 보이는 홈페이지 하단에 항상 footer를 위치시켰다. 

물론 루페 홈페이지는 내용이 많지 않기 때문에 문제 없는 것처럼 보였다. 하지만 화면 높이가 작아지면 내용이 겹쳐지는 문제가 발생하게 된다.

## 기본 레이아웃 개발

우선 기본 레이아웃을 아래와 같이 개발하였다. header 영역과 content 영역 그리고 footer 영역으로 나누어 HTML로 코딩하였고 CSS를 사용하여 화면 색깔과 글꼴 및 각 요소 위치를 지정하였다.

![기본 레이아웃](/files/js_footer_3.png)

content 영역 다음에 footer가 위치하기 때문에 content 내용이 많으면 스크롤 바가 생기면서 화면 하단에 위치하겠지만 content 내용이 적으면 위 그림처럼 footer가 위로 올라가는 문제가 발생하는 상황이다.

## JavaScript를 이용한 문제 해결

우선 이 문제를 JavaScript를 통해 이 문제를 해결할 수 있을 것으로 판단하였다. 그러기 위해서는 모든 영역에 대한 높이를 구할 수 있어야 했다. 

브라우저 영역 중 홈페이지가 출력되는 윈도우 높이는 쉽게 구할 수 있다.

```
window.innerHeight
```

header 영역과 content 영역 그리고 footer 영역 크기를 다음과 같이 구할 수 있다(여러분이 작성하는 HTML 각 Tag 및 class, id에 따라 접근하는 방법이 다를 수 있다)[^2].

```
header_height = document.getElementsByTagName('header')[0].clientHeight + 2;
content_height = document.getElementsByClassName('content')[0].clientHeight;
footer_height = document.getElementsByTagName('footer')[0].clientHeight;
```

위 세 영역을 모두 더한 값이 ```window.innerHeight``` 보다 작다면 강제로 content 높이를 변경하여 footer를 하단에 위치하도록 함수를 만들었다[^3].

```
function do_reposition_footer() {
    header_height = document.getElementsByTagName('header')[0].clientHeight + 2;
    content_height = document.getElementsByClassName('content')[0].clientHeight;
    footer_height = document.getElementsByTagName('footer')[0].clientHeight;

    body_height = header_height + content_height + footer_height;

    if (window.innerHeight > body_height) {
        t = window.innerHeight - header_height - footer_height - 2;
        document.getElementsByClassName('content')[0].style.height = t + 'px';
    }
}
```

이제 홈페이지가 최초로 로딩될 때 ```do_reposition_footer()``` 함수를 호출하도록 하였다. 참고로 ```DOMContentLoaded``` 이벤트는 IE8 이상에서만 사용가능하다[^4].

```
window.addEventListener('DOMContentLoaded', function(event) {
    do_reposition_footer();
});
```

홈페이지가 로딩된 시점 이외에도 브라우저 크기 변할때에도 footer 위치는 변화가 필요하다[^5]. 홈페이지가 로딩되는 시점과  차이가 있다면 한번 셋팅된 content 영역은 브라우저 크기가 변해도 content 크기는 바뀌지 않기 때문에 content 최적의 크기를 가지게 한 다음 새롭게 footer 위치를 계산해야 한다.

```
window.addEventListener('resize', function(event) {
    document.getElementsByClassName('content')[0].style.height = '';
    do_reposition_footer();
});
```

이제 홈페이지를 로딩하면 브라우저 하단에 footer가 배치됨을 확인할 수 있다.

![홈페이지가 로딩된 시점의 footer 위치](/files/js_footer_4.png)

브라우저 크기를 점점 줄이면 footer는 부라우저 하단에 계속 위치하기 위해 위치 계산을 한다. 그러다가 content와 footer가 겹쳐질 것 같으면 스크롤 바가 생기면서 겹쳐지지 않는다.

![브라우저를 줄여도 footer는 content와 겹쳐지지 않음](/files/js_footer_5.png)

## 결론

웹페이지 개발자들이 footer 때문에 많이 고민하는 듯하여 우리가 사용했던 방법을 공개하였다. 물론 더 좋은 해법을 가진 고급 개발자분도 계시겠지만 이 방법을 어떻게 해결해야 할지 몰라 고민하는 개발자분들에게 조금이라도 도움이 되었으면 하는 마음으로 글을 적어 보았다.

풀지 못하는 문제는 없다고 생각한다. 다만 우리는 깊이 고민하지 않고 풀기를 포기하고 검색에만 의존하기 때문에 검색이 되지 않으면 포기해 버리는 경향이 있는듯 하다.

조금만 더 고민하고 깨진 유리창을 고칠 수만 있다면 기업 이미지는 더욱 좋아질거라 믿으며 이번 글을 마무리 한다.

## 참조

[^1]: 깨진 유리창 법칙: <http://www.yes24.com/Product/Goods/69724939?Acode=101>
[^2]: Get the size of the screen, current web page and browser window : <https://stackoverflow.com/questions/3437786/get-the-size-of-the-screen-current-web-page-and-browser-window>
[^3]: Setting DIV width and height in JavaScript: <https://stackoverflow.com/questions/10118172/setting-div-width-and-height-in-javascript>
[^4]: 페이지 로드 후 불러오기, onload, ready, DOMContentLoaded: <https://itun.tistory.com/510>
[^5]: JavaScript window resize event: <https://stackoverflow.com/questions/641857/javascript-window-resize-event>





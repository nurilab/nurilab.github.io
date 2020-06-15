---
layout: post
title: '안드로이드 실행 파일 포맷 - dex (3)'
author: Kei Choi
date: 2020-06-15 18:30
tags: [fileformat, python, dex]
---

<fieldset style="margin:0px 0px 20px 0px;padding:5px;"><legend><span><strong style="font-weight:bold;">연재 순서</strong></span></legend><!--Creative Commons License--><div style="float: left; width: 88px; margin-top: 3px;"><img alt="Creative Commons License" style="border-width: 0" src="/files/images/exclamationmark.png"/></div><div style="margin-left: 92px; margin-top: 3px; text-align: justify;">
<p style="margin: 0;"><a href="/2020/05/12/fileformat_dex_1/">첫번째 글: classes.dex 파일 포맷 (Header, String IDs)</a></p>
<p style="margin: 0;"><a href="/2020/06/08/fileformat_dex_2/">두번째 글: classes.dex 파일 포맷 (Type IDs, Proto IDs)</a></p>
<p style="margin: 0; background:#ddd;">세번째 글: classes.dex 파일 포맷 (Field IDs, Method IDs)</p>
<p style="margin: 0;">네번째 글: classes.dex 파일 포맷 (Class Defs, Map List)</p>
</div></fieldset>


## 1. classes.dex 추가 파일 포맷

이전 글에 이어 계속 설명을 이어갑니다.

### (5) Field IDs

Field 정보 역시 헤더에 시작 위치와 개수가 저장되어 있다.

```
print hdr['field_ids_size']      # 전체 Field 정보 개수
print hex(hdr['field_ids_off'])  # 전체 Field 정보 시작 위치
```

[실행결과]
```
326
0x2634
```

0x2634 위치에 가면 8Byte씩으로 구성된 Field 정보가 나타난다.

![Field 정보 시작 위치](/files/ff_dex3_1.png)

8Byte 정보는 다음과 같은 형태로 구성되어 있다.

![Field 정보 구성 요소](/files/ff_dex3_2.png)

```
field_size = hdr['field_ids_size']  # 전체 Field 정보 개수
field_off  = hdr['field_ids_off']   # 전체 Field 정보 시작 위치

class_idx = struct.unpack('<H', mm[field_off  :field_off+2])[0] # class_idx
type_idx  = struct.unpack('<H', mm[field_off+2:field_off+4])[0] # type_idx
name_idx  = struct.unpack('<L', mm[field_off+4:field_off+8])[0] # name_idx

print hex(class_idx)
print hex(type_idx)
print hex(name_idx)

print '----------------------------------'

print string_ids[type_ids[class_idx]]
print string_ids[type_ids[type_idx]]
print string_ids[name_idx]
```

[실행결과]
```
0x20
0x3
0x27f
----------------------------------
Landroid/graphics/Rect;
I
bottom
```

위 결과를 통해 얻은 Filed 정보 내용은 ```int android.graphics.Rect.bottom```이 된다.

![Field 정보 리스트](/files/ff_dex3_3.png)

classes.dex 파일에는 위 그림처럼 다양한 Field 정보가 존재하며 이들을 모두 구하기 위한 방법은 아래와 같다. (위 그림은 그림 크기를 고려하여 일부 정보만이 갭쳐되었다.)

```
#-----------------------------------------------------------------
# field_id_list : dex 파일의 field 리스트를 추출한다.
#-----------------------------------------------------------------
def field_id_list(mm, hdr) :
    field_list = []
    field_ids_size = hdr['field_ids_size'  ]
    field_ids_off  = hdr['field_ids_off'   ]
    for i in range(field_ids_size) :
        class_idx = struct.unpack('<H', mm[field_ids_off+(i*8)  :field_ids_off+(i*8)+2])[0]
        type_idx  = struct.unpack('<H', mm[field_ids_off+(i*8)+2:field_ids_off+(i*8)+4])[0]
        name_idx  = struct.unpack('<L', mm[field_ids_off+(i*8)+4:field_ids_off+(i*8)+8])[0]
        field_list.append([class_idx, type_idx, name_idx])
    return field_list
    
#---------------------------------------------------------------------
# TEST
#---------------------------------------------------------------------
field_ids = field_id_list(mm, hdr)# 전체 문자열 출력하기
for i in range(len(field_ids)) :
    (class_idx, type_idx, name_idx) = field_ids[i]
    class_str = string_ids[type_ids[class_idx]]
    type_str  = string_ids[type_ids[type_idx]]
    name_str  = string_ids[name_idx]
    mag = '%s %s.%s' % (type_str, class_str, name_str)
    print '[%4d] %s' % (i, mag)
```

[실행결과]
```
[   0] I Landroid/graphics/Rect;.bottom
[   1] I Landroid/graphics/Rect;.left
[   2] I Landroid/graphics/Rect;.right
[   3] I Landroid/graphics/Rect;.top
[   4] I Landroid/os/Build$VERSION;.SDK_INT
         (중간 생략)
[ 323] Landroid/content/Context; Lcom/example/com/android/receiver/d;.c
[ 324] Lcom/example/com/android/receiver/NetChangeReceiver; Lcom/example/com/android/receiver/e;.a
[ 325] Landroid/content/Context; Lcom/example/com/android/receiver/e;.b
```

## (6) Method IDs

모든 Method도 classes.dex 내부에서는 별도로 정의된 공간에 저장되어 있다. 헤더에는 Method 개수와 위치가 저장되어 있다.

```
print hdr['method_ids_size']      # 전체 Method 정보 개수
print hex(hdr['method_ids_off'])  # 전체 Method 정보 시작 위치
```

[실행결과]
```
1078
0x3064
```

0x3064에 가면 8Byte로 구성된 Method 정보를 볼 수 있다.

![Method 정보 시작 위치](/files/ff_dex3_4.png)

8Byte 정보는 아래와 같이 구성이 되어 있다.

![Method 정보 구성 요소](/files/ff_dex3_5.png)

```
method_size = hdr['method_ids_size']  # 전체 Method 정보 개수
method_off  = hdr['method_ids_off']   # 전체 Method 정보 시작 위치

class_idx = struct.unpack('<H', mm[method_off  :method_off+2])[0] # class_idx
proto_idx = struct.unpack('<H', mm[method_off+2:method_off+4])[0] # proto_idx
name_idx  = struct.unpack('<L', mm[method_off+4:method_off+8])[0] # name_idx

print hex(class_idx)
print hex(proto_idx)
print hex(name_idx)

print '----------------------------------'

print string_ids[type_ids[class_idx]]
print string_ids[proto_ids[proto_idx][0]]
print string_ids[name_idx]
```

[실행결과]
```
0x5
0x1b
0x2df
----------------------------------
Landroid/animation/ValueAnimator;
J
getFrameDelay
```

위 결과를 통해 함수 원형은 J, 즉 리턴값이 long이며 인자값은 없다. 따라서 이 Method는 ```long android.animation.ValueAnimator.getFrameDelay()```가 된다.

![Method 정보 리스트](/files/ff_dex3_6.png)

classes.dex 파일에는 위 그림처럼 다양한  Method 정보가 존재하며 이들을 모두 구하기 위한 방법은 아래와 같다. (위 그림은 그림 크기를 고려하여 일부 정보만이 갭쳐되었다.)

```
#-----------------------------------------------------------------
# method_id_list : dex 파일의 method 리스트를 추출한다.
#-----------------------------------------------------------------
def method_id_list(mm, hdr) :
    method_list = []
    method_ids_size = hdr['method_ids_size'  ]
    method_ids_off  = hdr['method_ids_off'   ]
    for i in range(method_ids_size) :
        class_idx = struct.unpack('<H', mm[method_ids_off+(i*8)  :method_ids_off+(i*8)+2])[0]
        proto_idx = struct.unpack('<H', mm[method_ids_off+(i*8)+2:method_ids_off+(i*8)+4])[0]
        name_idx  = struct.unpack('<L', mm[method_ids_off+(i*8)+4:method_ids_off+(i*8)+8])[0]
        method_list.append([class_idx, proto_idx, name_idx])
    return method_list
    
#---------------------------------------------------------------------
# TEST
#---------------------------------------------------------------------
method_ids = method_id_list(mm, hdr)# 전체 문자열 출력하기
for i in range(len(method_ids)) :
    (class_idx, proto_idx, name_idx) = method_ids[i]
    class_str = string_ids[type_ids[class_idx]]
    name_str  = string_ids[name_idx]
    print '[%04d] %s.%s()' % (i, class_str, name_str)
```

[실행결과]

```
[0000] Landroid/animation/ValueAnimator;.getFrameDelay()
[0001] Landroid/app/Activity;.<init>()
[0002] Landroid/app/Activity;.invalidateOptionsMenu()
[0003] Landroid/app/Activity;.onActivityResult()
[0004] Landroid/app/Activity;.onConfigurationChanged()
         (중간 생략)
[1074] Lorg/apache/http/client/HttpClient;.execute()
[1075] Lorg/apache/http/client/methods/HttpGet;.<init>()
[1076] Lorg/apache/http/impl/client/DefaultHttpClient;.<init>()
[1077] Lorg/apache/http/util/EntityUtils;.toString()
```


## 2. 결론

이번 글에서는 Field와 Method에 대해서 알아보았다. 다음편이 classes.dex의 마지막인 Class Defs와 Map List를 살펴볼 것이다. 실제 classes.dex 파일 포맷을 보면서 이게 무슨 의미가 있나? 생각할지도 모르겠다.

이는 앞으로 다룰 classes.dex 디컴파일러를 위해 필요한 정보이다. 그때 이 정보가 엄청난 도움이 된다는 것을 알게 될 것이다.


## 3. 참조

1. Dalvik Executable Format: <https://source.android.com/devices/tech/dalvik/dex-format>

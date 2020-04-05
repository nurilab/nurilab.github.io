---
layout: post
title: '디스크 이미지 파일 포맷 - ISO'
author: Kei Choi
# image: /files/covers/nurilab.png
date: 2020-04-04 09:14
tags: [fileformat]
---

## 개요

ISO 이미지(ISO image)는 국제 표준화 기구(ISO)가 제정한 광학 디스크의 압축 파일(디스크 이미지)이다[^1]. 최근 이메일을 이용한 첨부 파일 형태의 공격 중 iso를 이용한 형태가 급증하고 있다.

![ISO 첨부 파일을 이용한 이메일 공격](/files/ff_iso_1.png)

결국 ISO 파일 내부에 존재하는 별도의 파일을 추출해야지만 악성코드를 탐지할 수 있다.

## ISO 구조  

ISO 파일은 디스크 이미지인 만큼 섹터 단위로 구성되어 있다. 하나의 섹터의 크기는 2048(0x800) Byte이다[^2]. ISO 구조를 해석하기 위해서는 볼륨 디스크립터부터 해석을 해야 하며 16 섹터에 위치하고 있다.

<center><p /><img src="/files/ff_iso_2.png"><br><font size="2">볼륨 디스크립터 구조</font></center>

볼륨 디스크립터에서 볼 수 있듯이 오프셋 0에는 해당 볼륨의 타입이 정의 되어 있으며 아래 그림에서 확인할 수 있다.

<center><p /><img src="/files/ff_iso_3.png"><br><font size="2">볼륨 타입</font></center>

위 볼륨 타입 중 파일 목록을 구하기 위해서는 1(Primary Volume Descriptor)과 2(Supplementary Volume Descriptor)를 살펴보게 된다. 두 영역 모두 파일 목록 정보가 존재하는 이유는 Primary Volume Descriptor 영역에는 8.3 포맷의 짧은 파일명으로 파일 목록이 저장된 반면 Supplementary Volume Descriptor 영역에는 긴 파일명으로 파일 목록이 저장되어 있다.

따라서 Supplementary Volume Descriptor 영역을 우선적으로 읽고 이 영역이 없다면 Primary Volume Descriptor 영역에서 파일 목록을 읽도록 한다. 반대로 이야기하면 Primary Volume Descriptor 영역은 반드시 존재하는 영역이다.

## Primary Volume Descriptor와 Supplementary Volume Descriptor

이 영역은 ISO를 읽는데 있어 중요한 정보를 담고 있으며 두 정보가 동일하다. 앞에서도 언급했듯이 Root 디렉토리의 정보를 짧은 파일명으로 가지는지 긴 파일명으로 가지는지에 대한 차이만 존재할 뿐이다.

<center><p /><img src="/files/ff_iso_4.png"><br><font size="2">Primary Volume Descriptor 구조</font></center>

위 그림에서 빨간색으로 표시한 영역은 Root 디렉토리의 정보를 담고 있는 곳이다. 이 정보의 구조는 아래 그림과 같다.

<center><p /><img src="/files/ff_iso_5.png"><br><font size="2">Root 디렉토리 정보의 구조</font></center>

ISO 파일 포맷은 특이하게도 특정 정보에 대해 리틀-엔디안 방식과 빅-엔디안 방식 두 가지를 모두 저장하고 있다는 점이다. Root 디렉토리 정보 구조에서 표시한 실제 Root 디렉토리의 위치와 해당 영역의 크기는 모두 리틀-엔디안 방식과 빅-엔디안 방식을 저장하고 있다.

```
ISO_SECTOR_START = 16  # Primary Volume Descriptor 시작 섹터
ISO_SECTOR_SIZE = 2048  # 섹터 크기

# ---------------------------------------------------------------------
# IsoFile 클래스
# ---------------------------------------------------------------------
class IsoFile:

    ... (생략) ...

    def get_infos_directorys_entry(self):
        off_dir_entry = 0
        size_dir_entry = 0
        
        i = ISO_SECTOR_START
        while True:
            self.fp.seek(i * ISO_SECTOR_SIZE)
            mm = self.fp.read(0x100)
            
            if mm[1:6] == 'CD001':
                # 01: Primary Volume Descriptor
                # 02: Supplementary Volume Descriptor
                if mm[0] in ['\x01', '\x02']:
                    off_dir_entry = kavutil.get_uint32(mm, 0x9e) * ISO_SECTOR_SIZE
                    size_dir_entry = kavutil.get_uint32(mm, 0xa6)
                    self.scan_type = ord(mm[0])  # 파일명 해석 방법이 달라짐
                elif mm[0] == '\xff':  # Volume Descriptor Set Terminator
                    break
            else:
                break
                    
            i += 1  # Next Sector
            
        return off_dir_entry, size_dir_entry
```

위 소스코드는 16섹터를 시작으로 다음 섹터로 진행하면서 볼륨 디스크립터의 타입이 1 또는 2라면 Root 디렉토리 정보 구조에 접근하여 Root 디렉토리의 시작 위치와 크기를 구한다. 만약 볼륨 디스크립터 타입이 0xff라면 해석을 종료한다.

## Directories Entry

이 여역은 Root 디렉토리 정보에서 가리키는 영역에 접근하면 존재한다. 구조는 위에서 설명한 "Root 디렉토리 정보의 구조"를 참조하면 된다. 이 정보 중 제일 중요한 정보를 추출하는 함수의 소스코드는 아래와 같다.

```
    def get_info_entry(self, mm):
        info = {}

        size_mm = kavutil.get_uint16(mm, 0)
        if size_mm == 0:
            return {}

        buf = mm[:size_mm]

        start_sector = kavutil.get_uint32(buf, 2)
        length_data = kavutil.get_uint32(buf, 10)
        attribute_data = ord(buf[25])
        size_filename = ord(buf[32])
        if size_filename:
            filename = buf[33:33+size_filename]
            if self.scan_type == 2:  # Supplementary Volume Descriptor
                filename = filename[1::2]
        else:
            filename = ''

        info['size_entry'] = size_mm
        info['start_sector'] = start_sector
        info['length_data'] = length_data
        info['attribute_data'] = attribute_data
        info['filename'] = filename

        return info
```

위 소스 코드에서는 하나의 디렉토리 Entry의 정보를 추출하는데 파일의 데이터 시작 섹터 위치, 파일 크기, 파일 속성 및 파일명을 추출할 수 있다. 다음 Entry 정보는 바로 연이어 등장한다. 따라서 해당 Entry의 크기가 중요하며 이를 통해 다음 Entry를 시작 위치를 알게 된다. 

파일 속성 정보는 다음과 같다.

<center><p /><img src="/files/ff_iso_6.png"><br><font size="2">파일 속성 정보</font></center>

디렉토리 Entry를 해석하는 중 파일 속성 정보를 통해 디렉토리라고 나오면 해당 파일 데이터의 시작 섹터는 다시 하위 디렉토리 Entry 정보를 가지는 구조가 된다.

참고로 디렉토리 Entry를 해석하다보면 0번째와 1번째 Entry 정보는 우리가 흔히 도스 커맨드에서 볼 수 있는 ```.``` 디렉토리와 ```..``` 디렉토리이다. 흔히 현재 디렉토리와 상위 디렉토리의 정보를 가지고 있다. 

또한 하나의 섹터에 여러개의 디렉토리 Entry가 올 수 있다. 이때 섹터에 걸쳐지는 디렉토리 Entry가 존재할 수 있을 것이다. 하지만 ISO 파일 포맷에서는 하나의 섹터 마지막 영역에 디렉토리 Entry 정보를 넣을 수 있을 정도의 크기가 존재하지 않으면 다음 섹터로 정보를 넘겨버린다. 따라서 디렉토리 Entry를 해석할 때에는 디렉토리 Entry의 시작 섹터와 디렉토리 Entry 정보의 크기가 중요하다. 이 크기를 2048 Byte로 나누면 실제 디렉토리 Entry가 차지하는 섹터의 수가 되므로 이를 이용하여 모든 디렉토리 Entry 정보를 읽어야 한다.

```
    def parse_directorys_entry(self, off_sector, size_data, dir_name=''):
        self.fp.seek(off_sector)
        data = self.fp.read(size_data)

        for i in range(size_data / ISO_SECTOR_SIZE):  # 디렉토리 Entry의 모든 섹터를 분석
            buf = data[i * ISO_SECTOR_SIZE:(i+1) * ISO_SECTOR_SIZE]

            while True:
                info = self.get_info_entry(buf)
                if info.get('size_entry', 0) == 0:  # 데이터가 없으면 해당 섹터 분석 종료하고 다음 섹터로...
                    break

                size_entry = info.get('size_entry', 0)
                attr = info.get('attribute_data', 0)
                fname = info.get('filename', '')
                start_sector = info.get('start_sector', 0)
                fsize = info.get('length_data', 0)

                if attr & 2 == 2 and fname != '':  # dir
                    # print '[%d] %s' % (self.idx, fname + '/')
                    # self.idx += 1
                    self.parse_directorys_entry(start_sector * ISO_SECTOR_SIZE, fsize, dir_name + fname + '/')
                elif attr & 2 != 2:  # NOT dir
                    # print '[%d] %s' % (self.idx, fname)
                    # self.idx += 1
                    self.files.append([dir_name + fname, start_sector, fsize])

                buf = buf[size_entry:]
```

## File

파일은 디렉토리 Entry에서 읽은 정보를 토대로 파일 데이터의 시작 섹터로 이동하여 파일 크기 만큼 추출하면 해당 파일이 되므로 간단하다.

```
if __name__ == '__main__':
    z = IsoFile('Confirm Invoice 41202219-2020.iso.vir')

    for name in z.namelist():
        print name
        print '-' * 50

        buf = z.read(name)

        kavutil.HexDump().Buffer(buf, 0)

    z.close()
```

해당 소스 코드를 실행하여 얻은 결과는 다음과 같다.

<center><p /><img src="/files/ff_iso_7.png"><br><font size="2">ISO 악성코드 내부에 숨겨진 파일과 파일 내용</font></center>

## 결론

최근 급증하고 있는 이메일 첨부 파일 중 ISO 파일 포맷에 대해 알아보았다. 사실 복잡한 파일 포맷이 아니기 때문에 해당 구조에 대한 것만 알 수 있다면 파일 정보를 추출하는 것은 어려운 일이 아니다. 또한 FAT 파일 시스템과 같은 구조에 대해 안다면 ISO 파일 포맷에 대한 정보를 이해하는 것도 어렵지 않을 것이다.

## 참조

[^1]:  ISO 이미지: <https://ko.wikipedia.org/wiki/ISO_%EC%9D%B4%EB%AF%B8%EC%A7%80>
[^2]: ISO 9660: <https://wiki.osdev.org/ISO_9660>

---
title: server-network-perf
date: 2017-09-21 09:30:48
tags:
---

# 서버 네트워크 속도(대역폭) 측정하기

{% asset_img 'highway-393492_1920.jpg' '우리는 항상 빠른 속도를 원한다.' %}

AWS는 [EC2 인스턴스 타입](https://aws.amazon.com/ko/ec2/instance-types/)별 네트워크 속도(대역폭)에 대한 정확한 수치를 공개하고 있지 않습니다.
아마도 자원을 공유하는 클라우드의 특성상 정확한 네트워크 속도를 보장하기 어렵기 때문이 아닌가 추측됩니다.
__***[2017-09-23] 정정합니다. [AWS ec2 인스턴스 구성](http://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/ebs-ec2-config.html)을 보면 대역폭이 나와있습니다. 하지만 T2 인스턴스는 해당 표에 없습니다.***__

어쨌든 우리도 서버의 대략적인 네트워크 대역폭을 알아야 실수 없이 서버를 구성할 수 있으니 한 번 측정을 해보도록 하겠습니다.
측정 도구로는 [iperf](https://iperf.fr/)를 사용하도록 하겠습니다.

## 준비물
1. 클라이언트 PC 1대(간단하게 그냥 Macbook Pro와 일반 가정용 인터넷을 사용했습니다.)
1. 서버(AWS EC2 t2.medium) 1대

## iperf 설치하기
1. 클라이언트
  1. ``$ brew install iperf``
1. 서버
  1. ``$ sudo yum install iperf``

## iperf 테스트하기
1. 서버
  1. ``$ iperf -s``
  1. 옵션설명
    * -s: iperf가 서버모드로 실행됩니다.
1. 클라이언트
  1. ``$ iperf -c ${SERVER_IP} -i 1 -t 10``
  1. 옵션설명
    * -c: 클라이언트 모드로 실행
    * ${SERVER_IP}: 테스트할 서버의 IP
    * -i 1: 반복 시간 간격(초단위)
    * -t: 테스트할 시간(초단위)

### 서버로그
```
$ iperf -s
------------------------------------------------------------
Server listening on TCP port 5001
TCP window size: 85.3 KByte (default)
------------------------------------------------------------
[  4] local 172.31.25.62 port 5001 connected with 115.140.62.214 port 56015
[ ID] Interval       Transfer     Bandwidth
[  4]  0.0-10.1 sec  29.3 MBytes  24.2 Mbits/sec
```

### 클라이언트 로그
```
$ iperf -c ###.###.###.### -i 1 -t 10
------------------------------------------------------------
Client connecting to ###.###.###.###, TCP port 5001
TCP window size:  129 KByte (default)
------------------------------------------------------------
[  4] local 192.168.219.121 port 61977 connected with ###.###.###.### port 5001
[ ID] Interval       Transfer     Bandwidth
[  4]  0.0- 1.0 sec  4.00 MBytes  33.6 Mbits/sec
[  4]  1.0- 2.0 sec  3.38 MBytes  28.3 Mbits/sec
[  4]  2.0- 3.0 sec  3.38 MBytes  28.3 Mbits/sec
[  4]  3.0- 4.0 sec  3.50 MBytes  29.4 Mbits/sec
[  4]  4.0- 5.0 sec  2.38 MBytes  19.9 Mbits/sec
[  4]  5.0- 6.0 sec  4.12 MBytes  34.6 Mbits/sec
[  4]  6.0- 7.0 sec  3.88 MBytes  32.5 Mbits/sec
[  4]  7.0- 8.0 sec  4.12 MBytes  34.6 Mbits/sec
[  4]  8.0- 9.0 sec  3.75 MBytes  31.5 Mbits/sec
[  4]  9.0-10.0 sec  4.50 MBytes  37.7 Mbits/sec
[  4]  0.0-10.0 sec  37.1 MBytes  31.1 Mbits/sec
```

## 테스트 결과 읽기
1. ``Interval``: 호출한 시간 간격
1. ``Transfer``: 전송한 데이터 사이즈
1. ``Bandwidth``: 측정된 대역폭(주의: 단위가 Mbits 임. Mbits / 8 === MB)

## 관련자료
[iperf](https://iperf.fr/)
[AWS ec2 인스턴스 타입](https://aws.amazon.com/ko/ec2/instance-types/)
[AWS ec2 인스턴스 구성](http://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/ebs-ec2-config.html)
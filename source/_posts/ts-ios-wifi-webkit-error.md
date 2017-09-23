---
title: ts-ios-wifi-webkit-error
date: 2017-08-24 19:17:22
tags:
---

# [Trouble Shooting] iOS WiFi parse error

## 증상
1. iOS 사파리에서 [가자고](https://www.thegajago.com) 사이트를 열면 ``오류: WebKit에 내부 오류 발생``이라고 보여진다.
  - {% asset_img 'webkit-error.png' '우리는 항상 빠른 속도를 원한다.' %}
2. 사이트의 반응이 매우 느리며 불특정 페이지에서 발생한다.
3. WiFi 연결일때만 발생하며 LTE 연결인 경우 오류가 발생하지 않는다.
4. iOS 10.2 버전 업데이트 이후부터 발생된다.


## 원인
1. iOS WiFi 연결상태에서 gzip decoding에 뭔가 문제가 있는것 같다.
  - 사실 정확한 원인 및 문서를 찾지 못했다.


## 해결책
1. gzip 옵션을 off 한다.
```conf
...
    gzip on; // -> off
    gzip_types text/plain application/xml text/css text/js text/xml application/x-javascript text/javascript application/json appplication/xml+rss;
    gzip_http_version 1.1;
    gzip_disable "MSIE [1-6] \.";
    gzip_vary on;
...
```


## 남아있는 과제
1. nginx gzip 모듈 버전을 업데이트하면 괜찮지 않을까?
2. gzip을 iOS에서만 off 하도록 gzip_disable 값을 셋팅하자.

## 관련자료
- 없음
---
title: how-cookie-works
date: 2017-09-23 09:25:33
tags:
---

# How Cookie Works

## Cookie란?
HTTP는 기본적으로 [무상태 프로토콜](https://ko.wikipedia.org/wiki/무상태_프로토콜)입니다. 
HTTP 서버가 클라이언트에 대한 상태를 가지고 있지 않기 때문에 각각의 요청을 이전 요청과 무관한 독립적인 트랜잭션으로 취급하게 됩니다.
하지만 그렇게 되면 세션과 개인화(회원별 서비스)등을 관리할 수 없게 되기 때문에 Cookie(이하 쿠키)를 사용하게 됩니다.

HTTP의 쿠키는 마치 헨젤과 그레텔의 쿠키(~~돌맹이나 빵 버전도 있습니다.~~)와 같습니다.
헨젤과 그레텔은 자신들이 돌아갈 길을 잊지 않기 위해서 사용하지만 HTTP의 쿠키는 HTTP 요청을 보내는 클라이언트를 식별하기 위해서 사용합니다.
{% asset_img 'breadcrumbs.jpg' '헨젤과 그레텔의 쿠키' %}

## HTTP 메시지
__***[주의] 아래 설명은 웹 브라우저를 기반으로 설명한 내용입니다. 서버나 앱, 크롤러 등 다른 클라이언트에서 HTTP를 호출하는 경우 설정에 따라 다르게 동작합니다.***__
{% asset_img 'http-req-res.png' '쿠키 생성 과정' %}
1. 최초 HTTP 요청
  * 웹브라우저에서 처음 방문하는 웹사이트에 요청을 보냅니다.
1. 세션 ID 발급
  * ``Set-Cookie:s_u_k=2ecf65a9-3678-497c-a620-640a88cabb44; Expires=Fri, 20-Oct-2017 15:24:31 GMT; Path=/``
  * ``Set-Cookie:SESSION=b5e3d1d6-bdcf-4d13-9502-9a9a654d35c1; Path=/; Secure; HttpOnly``
  * ``Set-Cookie`` 필드에 대한 상세 정보는 [MDN 문서](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Set-Cookie)를 참고해주세요.
  * __***[주의] 모든 사이트가 항상 최초의 요청에 세션 ID를 발급하는 것은 아닙니다. 하지만 일반적으로 거의 모든 사이트가 사용자 추적을 위해서 최초 방문시 세션 ID를 발급하며 세션 ID 명이나 값 등의 상세 정책은 사이트마다 다를 수 있습니다.***__
1. 모든 요청 헤더에 Cookie 포함
  * ``Cookie:s_u_k=2ecf65a9-3678-497c-a620-640a88cabb44; SESSION=b5e3d1d6-bdcf-4d13-9502-9a9a654d35c1``
  * 이때 동일한 도메인으로 보내는 모든 요청에 쿠키가 포함되어 요청이 날아갑니다. HTML, CSS, JS, 이미지 등 모든 요청 헤더에 쿠키가 달라붙습니다. 아래 [상세 내용](#주의하세요)을 참고해 주세요.
1. 응답 헤더에는 Cookie 미포함

## 주의하세요~
예전에는 쿠키를 브라우저의 클라이언트 데이터 저장용으로 사용하는 경우도 있었지만 ``localStorage``, ``sessionStorage``(IE8 이상 지원)가 있는 이상 더 이상 쿠키를 데이터 베이스처럼 사용해서는 안됩니다.
쿠키는 크기 제한(브라우저 마다 다릅니다.)이 있으며 HTTP 요청을 주고 받을 때 마다 헤더를 통해서 전송이 되므로 성능상 낭비가 일어나게 됩니다.
IE8 이전의 시대에는 대안이 없었지만 현재는 IE8 미만의 브라우저는 거의 없으므로 클라이언트 데이터 저장은 용도에 맞게 ``localStorage``, ``sessionStorage``에 저장해야 합니다.

쿠키의 크기가 1KB, 화면 하나에 HTTP 요청이 10개, 서버에 사용자가 동시에 100명이 동시에 접속한다고 가정하면 서버에 1MB의 불필요한 트래픽이 발생하게 됩니다.
동시접속 사용자가 적을 경우는 괜찮지만 서비스가 성장하게 되면 반드시 성능상 문제가 발생하게 됩니다.

## 쿠키 상세 정보
{% asset_img 'dev-tools-cookie.png' '크롬 개발자 도구 Application 탭' %}
1. Name
  * 쿠키 이름(키))
1. Value
  * 쿠키 값
1. Domain
  * 쿠키가 적용될 도메인 정보
  * 브라우저는 도메인 정보가 같은 쿠키만 요청 헤더에 ``Cookie`` 필드를 자동으로 추가 합니다.
1. Path
  * 쿠키를 경로를 지정해서 마치 폴더와 같이 관리할 수 있습니다.
1. Expires
  * ``Expires``에 명시된 날짜에 쿠키가 만료(삭제)됩니다.
  * ``Expires``, ``Max-Age``가 명시되지 않은 경우 세션 쿠키입니다. 브라우저가 종료되면 해당 쿠키는 삭제됩니다. 하지만 요즘 브라우저들은 종료되었다가 다시 시작되어도 [이전 세션을 복원하는 기능]((https://support.mozilla.org/ko/kb/restore-previous-session#w_eieo-hiuayi-ykiiaooe-hlou)이 있어서 세션이 계속 유지되는 경우가 많습니다.
1. HttpOnly
  * ``HttpOnly`` 필드가 선언되면 스크립트에서는 해당 쿠키에 접근할 수 없습니다. 하지만 브라우저가 아닌 다른 클라이언트(ex. 크롤러)에서 접근하면 해당 정보를 취득할 수 있습니다.
1. Secure
  * ``Secure`` 필드가 선언되면 해당 쿠키는 HTTPS(SSL) 프로토콜을 사용한 경우에만 전송이 됩니다. 하지만 그렇다고 하더라도 절대로 민감한 정보를 담아서는 안됩니다.
좀 더 상세한 내용은 MDN에서 한글로 번역된 [상세한 문서](https://developer.mozilla.org/ko/docs/Web/HTTP/Cookies)를 참고해 주세요.

## 쿠키를 이용한 세션관리
웹에서는 쿠키를 이용해서 세션을 관리하게 됩니다.
쿠키를 이용한 세션관리는 다음 편에서 별도로 설명하겠습니다.

## 관련자료
[MDN HTTP 쿠키](https://developer.mozilla.org/ko/docs/Web/HTTP/Cookies)
[MDN Set-Cookie](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Set-Cookie)
[RFC-6265](https://tools.ietf.org/html/rfc6265#section-4.1)
[브라우저 세션 복원](https://support.mozilla.org/ko/kb/restore-previous-session#w_eieo-hiuayi-ykiiaooe-hlou)

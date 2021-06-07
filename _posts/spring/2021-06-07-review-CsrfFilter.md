---
title: "Spring Security의 CsrfFilter 알아보기"
date: 2021-06-07T01:01:01-04:00
categories:
  - spring-security
tags:
  - spring
  - security
# classes: wide
toc: true
toc_icon: "list-ul"
toc_sticky: true

---


<img src="{{ '/assets/images/spring/spring-logo.png'}}" alt="" class="align-center">

spring security의 [CSRF](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#csrf)에 대한 정리!



---

<br>

# CSRF?

**Cross Site Request Forgery**의 약자로 사이트 간의 요청을 위조하는 공격을 말합니다. [Wiki](https://ko.wikipedia.org/wiki/%EC%82%AC%EC%9D%B4%ED%8A%B8_%EA%B0%84_%EC%9A%94%EC%B2%AD_%EC%9C%84%EC%A1%B0)  
CSRF는 XSS 공격과 비슷하지만 다른 공격 방법을 이용하는 방식으로, 로그인 상태의 사용자를 이용하여 공격을 시도하는 방법인데요.  
아주아주 간단하게 CSRF 공격 방법에 대해 정리해 보자면  
먼저, 로그인 유저(쿠키나 토큰 정보가 발급된)에게 공격을 위해 구현된 페이지로 접속을 유도합니다.  
해당 패이지로 이동하면 내부에 삽입된 코드나 태그가 동작하여 공격 대상인 서버로 데이터 조작 시도를 합니다.  

CSRF 공격을 막는 가장 기본적인 방법은 **[CORS](https://developer.mozilla.org/ko/docs/Web/HTTP/CORS)를 허용하지 않는** 것입니다.  
하지만 일부 다른 Origin을 허용해야 하는 경우가 있는데 이럴 때 CSRF 공격에 노출되게 됩니다.


---
<br>

# CsrfFilter

CsrfFilter는 Security에서 제공하는 기본 filter 중 1개로, Spring Security는 이 CsrfFilter을 이용하여 CSRF 공격에 대한 방어 기능을 제공합니다.   
이때, CSRF 공격에 대한 방어 기능을 token을 이용해 간단하게 제공해 주는데요.  

먼저 CsrfFilter가 FilterChain에 등록된 모습을 살펴보겠습니다.
아래 사진에서 이전 포스트에서 정리했던 FilterChainProxy의 filter 목록에 등록된 모습을 확인할 수 있습니다.  
<img src="{{'/assets/images/spring/security/csrffilter-filterslist.png'}}" alt="" class="align-center">


그럼 CsrfFilter가 동작하는 부분을 한번 살펴보겠습니다.  
CsrfFilter#doFilterInternal에서는 아래 사진에 표시된 것처럼 3가지 과정으로 분리하여 동작합니다.
<img src="{{'/assets/images/spring/security/csrffilter-dofilterinternal.png'}}" alt="" class="align-center">

## HttpServletRequest에서 token 정보 가져오기

CsrfFilter는 **HttpServletRequest**의 session에서 token 정보를 읽어옵니다.  
이때 최초에 접근할 경우 token 정보가 없으므로 `missingToken == true` 상태가 되고  
`this.tokenRepository.generateToken(request);` 로직을 수행합니다.  
generateToken()를 실행하면서 새로운 토큰을 만들겠네요! :)
<img src="{{'/assets/images/spring/security/csrffilter-dofilterinternal-1.png'}}" alt="" class="align-center">


## LazyCsrfTokenRepository

generateToken()는 `CsrfTokenRepository` interface의 구현체인 `LazyCsrfTokenRepository`를 이용하여 csrf token을 생성합니다.  
이때 LazyCsrfTokenRepository에 구현된 `generateToken()`, `saveToken()`, `loadToken()` 모두 대리 repo를 이용하여 동작하는데요.  
**session을 이용**한 방식을 사용할 경우엔 `HttpSessionCsrfTokenRepository`를 이용합니다.  
설정에 따라 delegate 필드에 다른 구현체가 할당되어 동작하는 것으로 보입니다.  
<img src="{{'/assets/images/spring/security/LazyCsrfTokenRepository.png'}}" alt="" class="align-center">

<br>

`HttpSessionCsrfTokenRepository`에서는 form 인증 방식에 사용되는 csrf token의 key 이름이 왜 **_csrf** 인지 확인할 수 있는 부분과, session에 어떤 key 값으로 토큰을 가져오는지 확인할 수 있는 정보가 있습니다.  
그리고 `saveToken()`, `loadToken()`, `generateToken()`가 어떻게 구현되어 동작하는지도 확인 가능하네요! :)  

<img src="{{'/assets/images/spring/security/HttpSessionCsrfTokenRepository.png'}}" alt="" class="align-center">


---
<br>


# 다시 돌아가서 CsrfFilter

앞서 csrf token 정보를 조회, 생성하는 로직을 살펴보았습니다.  
이제 csrf의 token 정보를 어디서 어떻게 가져오는지, 그리고 없다면 어떻게 생성하는지 정리가 되었는데요.

그럼 다시 처음으로 돌아가서 HttpServletRequest의 session에서 얻은 token 정보를 어떻게 비교할까요?

<img src="{{'/assets/images/spring/security/csrffilter-dofilterinternal.png'}}" alt="" class="align-center">

위 사진에서 볼 수 있듯이 요청의 header에 포함된 client가 보낸 token과 앞서 찾은 token을 비교하는 로직을 수행합니다.  
그리고 비교 결과 동일하지 않다면 `AccessDeniedException` 예외를 던집니다.


---
<br>

> 참고  
> docs : [spring-security](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication)  
> docs : [cors](https://developer.mozilla.org/ko/docs/Web/HTTP/CORS)  

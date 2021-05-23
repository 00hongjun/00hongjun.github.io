---
title: "Spring Security의 Filter 알아보기"
date: 2021-05-23T01:01:01-04:00
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

spring security의 [Filter](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-architecture)에 대한 정리!



---

<br>

# 들어가기 전에..

<img src="{{'/assets/images/spring/security/filterchain.png'}}" alt="" class="align-center">

servlet에서 동작하는 스프링 시큐리티는 filter를 기반으로 합니다.  
위의 그림은 단일 request에 대한 일반적인 filter 동작을 보여줍니다.  
URI를 기반으로 DispatcherServlet에서 HttpServletRequest를 처리할 filter chain과 servlet을 만들고 HttpServletResponse로 응답하는 구조입니다.

스프링 시큐리티는 servlet filter를 기반으로 인증 기능을 지원합니다.  
그리고 filter를 사용하기 때문에 servlet container 안에 있는 다른 application 들과 맞물려 동작할 수 있습니다.  
spring boot의 기본 설정을 사용한다면 `springSecurityFilterChain` filter를 자동으로 등록해 주고 이 filter를 이용하여 스프링 시큐리티의 인증 과정의 전체적인 동작을 관장하게 됩니다. 

<br>

# DelegatingFilterProxy

<img src="{{'/assets/images/spring/security/filterchainproxy.png'}}" alt="" class="align-center">

시큐리티가 filter를 기반으로 동작하는 과정에서 `DelegatingFilterProxy`라는 이름의 filter 구현체를 등록합니다.  
DelegatingFilterProxy는 `FilterChainProxy`라는 filter를 래핑하고 있으며, `SecurityFilterChain`를 통해 다양한 filter들에게 역할을 위임합니다.  
DelegatingFilterProxy는 **Filter bean 인스턴스를 찾고 등록하는 것을 지연**시킬 수 있습니다.  
컨테이너가 시작되기 전에 컨테이너가 필터 인스턴스를 등록해야 하기 때문에 DelegatingFilterProxy가 Filter bean의 등록을 지연시켜 주는 점은 매우 중요합니다. (이 부분은 아래에서 다시 한번 설명합니다.)
<img src="{{'/assets/images/spring/security/delegatingfilterproxy_capture.png'}}" alt="" class="align-center">

---
<br>

# FilterChainProxy

스프링 시큐리티의 기능은 `FilterChainProxy`를 통해 지원됩니다.  
FilterChainProxy는 시큐리티가 필요한 **filter의 list**를 가지고 있습니다.  
FilterChainProxy는 `SecurityFilterChain`를 통해 다른 filter들에게 기능 역할을 위임합니다.  

<img src="{{'/assets/images/spring/security/filterChainProxy_capture.png'}}" alt="" class="align-center">


<img src="{{'/assets/images/spring/security/securityfilterchain_capture.png'}}" alt="" class="align-center">

---
<br>

# SecurityFilterChain

`SecurityFilterChain`은 interface로, FilterChainProxy의 요청에 대해 호출되어야 하는 **filter를 결정**하는 데 사용됩니다.  
지금까지 위에서 살펴 본 DelegatingFilterProxy, FilterChainProxy, SecurityFilterChain의 관계를 그림으로 나타내면 아래와 같습니다.  
<img src="{{'/assets/images/spring/security/securityfilterchain.png'}}" alt="" class="align-center">

SecurityFilterChain의 구현체 filter들은 대부분 bean으로 등록되며, 이 bean들은 DelegatingFilterProxy가 아닌 FilterChainProxy에 등록됩니다.  
FilterChainProxy에 등록하면서 몇 가지 이점을 얻습니다.  
1. FilterChainProxy을 이용하여 filter를 선택하기 때문에 문제가 생겼을 경우 FilterChainProxy를 이용하여 debug를 하기 쉽습니다.
2. FilterChainProxy도 filter이기 때문에 선후 처리가 가능합니다. -> client의 요청에 대한 응답을 한 뒤 인증 관련 context를 자동으로 clear 해줍니다. 
3. [HttpFirewall](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-httpfirewall) 같이 특정 공격에 대한 기본적인 방어가 가능합니다.
4. SecurityFilterChain의 호출 시점에 대한 유연성을 제공합니다. servlet container의 filter는 url만을 기반으로 호출됩니다.  
FilterChainProxy를 사용함으로써 RequestMatcher 인터페이스를 활용하여 HttpServletRequest을 기반으로 호출할 filter를 결정할 수 있습니다.

아래는 HttpServletRequest를 이용하여 filter list를 반환하는 `FilterChainProxy#getFilters` 메서드입니다.  
request의 url를 비교하여 match 되는 chain을 가져옵니다.
<img src="{{'/assets/images/spring/security/filterchainproxy-getfilters-debug.png'}}" alt="" class="align-center">

<br>

<img src="{{'/assets/images/spring/security/multi-securityfilterchain.png'}}" alt="" class="align-center">
FilterChainProxy를 이용하여 위 사진과 같이 url을 기반으로 filter chain을 선택적으로 호출 가능합니다.  
우선 패턴이 일치하는 filter chain을 우선순위로 결정합니다.  

SecurityFilterChain에 등록되는 filter들은 [링크](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-security-filters)를 통해 확인해 주세요.


---
<br>

> 참고  
> docs : [spring-security](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication)  
> docs : [spring-security Filters](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-filters-review)  

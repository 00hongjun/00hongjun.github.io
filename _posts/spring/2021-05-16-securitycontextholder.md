---
title: "Spring Security의 SecurityContextHolder 알아보기"
date: 2021-05-16T01:01:01-04:00
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

spring security의 [Authentication](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication)(인증)의 `SecurityContextHolder`에 대한 겉핥기!  
Servlet Applications에서 동작하는 인증 방식에 대해 간단하게 정리 하며 [Reactive Applications](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#reactive-applications)에 관한 정보는 링크를 확인해 주세요.




---
# 들어가기 전에..
먼저 아래에서 다루지는 않지만 간단한 지식을 정리한 후 SecurityContextHolder에 대해 알아보겠습니다.  

스프링 시큐리티는 servlet filter를 기반으로 인증 기능을 지원합니다.  
그리고 filter를 사용하기 때문에 servlet container안에 있는 다른 application들과 맞물려 동작할 수 있습니다.  
spring boot의 기본 설정을 사용 한다면 `springSecurityFilterChain` filter를 자동으로 등록해 주고 이 filter를 이용하여 스프링 시큐리티의 인증과정의 전채적인 동작을 관장하게 됩니다. 

이 과정에서 `DelegatingFilterProxy`라는 이름의 filter 구현체를 등록하게 되고 DelegatingFilterProxy는 다른 bean filter들에게 적업을 넘겨줍니다.  
이때 DelegatingFilterProxy로 래핑된 `FilterChainProxy`라는 특수 filter를 제공하게 되고 `SecurityFilterChain`를 통해 다양한 filter들에게 위임 합니다.

<img src="{{'/assets/images/spring/security/securityfilterchain.png'}}" alt="" class="align-center">

<img src="{{'/assets/images/spring/security/delegatingfilterproxy_capture.png'}}" alt="" class="align-center">

<img src="{{'/assets/images/spring/security/filterChainProxy_capture.png'}}" alt="" class="align-center">




스프링 시큐리티의 Authentication은 여러 Component로 구성되어 있으며, 이 Component 등이 서로 엮여 인증과 인가를 검증하게 됩니다.  
주요 Component는 아래와 같습니다.

* SecurityContextHolder - 누가 인증했는지에 대한 정보들을 저장하고 있습니다.
* SecurityContext - 현재 인증한 user에 대한 정보를 가지고 있습니다.
* Authentication - SecurityContext의 user, 인증 정보를 가지고 있으며, AuthenticationManager에 의해 제공됩니다.
* GrantedAuthority - 인증 주체에게 부여된 권한 (roles, scopes, etc.)
* AuthenticationManager - Spring Security의 필터에서 인증을 수행하는 방법을 정의하는 API입니다.
* ProviderManager - AuthenticationManager의 구현체
* AuthenticationProvider - 인증 수행을 위해 ProviderManager에 의해 사용 됩니다.
* AbstractAuthenticationProcessingFilter - 인증에 사용되는 기본 Filter입니다. 인증의 흐름이 어떻게 이루어 지는지 잘 보여줍니다.

---

<br>

# SecurityContextHolder

<img src="{{'/assets/images/spring/security/securitycontextholder.png'}}" alt="" class="align-center">

시큐리티의 메인이라고 할 수 있는 SecurityContextHolder입니다.  
`SecurityContextHolder`는 시큐리티가 인증한 내용들을 가지고 있으며, `SecurityContext`를 포함하고 있고 SecurityContext를 현재 스레드와 연결해 주는 역할을 합니다.  
`ThreadLocal`의 전략을 SecurityContextHolder에서 설정 할 수 있습니다.

docs에서는 아래와 같이 SecurityContext 생성과 Authentication 생성을 보여주지만 설명은 일단 패스!
```java
SecurityContext context = SecurityContextHolder.createEmptyContext(); 
Authentication authentication =
    new TestingAuthenticationToken("username", "password", "ROLE_USER"); 
context.setAuthentication(authentication);

SecurityContextHolder.setContext(context); 
```

<br>

시큐리티를 사용하는 이유중 하나는 인증된 사용자 정보를 확인하는것 일것이고, 인증된 사용자의 정보는 아래와 같이 SecurityContextHolder를 통해 확인이 가능 합니다.

```java
SecurityContext context = SecurityContextHolder.getContext();
Authentication authentication = context.getAuthentication();
String username = authentication.getName();
Object principal = authentication.getPrincipal();
Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
Object credentials = authentication.getCredentials();
boolean authenticated = authentication.isAuthenticated();
```

시큐리티는 같은 thread의 application 내에서 어디서든 SecurityContextHolder의 인증 정보를 확인 가능하도록 구현 되어 있습니다.  
어디서든 인증 정보를 확인 가능하도록 도와주는 개념이 `ThreadLocal`입니다.


## ThreadLocal

SecurityContextHolder은 `ThreadLocal`을 이용하여 인증 관련된 정보를 저장합니다.  
ThreadLocal에 정보를 저장하여 관리하기 때문에 동일 스레드에서는 항상 같은 인증 정보로 접근 가능합니다.  
그리고 서비스의 메서드에 SecurityContext를 parameter로 주지 않더라고 어디서나 SecurityContext에 접근 가능하게 해줍니다.  

application에 따라 ThreadLocal의 동작 방식을 다르게 설정할 수도 있는데요. SecurityContextHolder의 설정을 이용하여 변경 가능하며 설정값으로는 아래와 같은 값들이 있습니다. [링크](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-securitycontextholder)   
* SecurityContextHolder.MODE_THREADLOCAL (default)
* SecurityContextHolder.MODE_INHERITABLETHREADLOCAL
* SecurityContextHolder.MODE_GLOBAL

아래는 `SecurityContextHolder.initialize()`의 구현 부분을 캡처한 사진입니다.  

<img src="{{'/assets/images/spring/security/securityContextHolder_strategy_init.png'}}" alt="" class="align-center">

---  

<br>

# SecurityContext

<img src="{{'/assets/images/spring/security/securitycontextholder.png'}}" alt="" class="align-center">

위의 사진을 다시 한번 참고하여 `SecurityContext`에 대해 알아보겠습니다.  
SecurityContext는 SecurityContextHolder를 통해 얻을 수 있으며 Authentication 객체를 가지고 있습니다.

```java
SecurityContext context = SecurityContextHolder.getContext();
```

---

<br>

# Authentication

자 그럼 SecurityContext가 포함하고 있다는 [Authentication](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/core/Authentication.html)라는 것은 무엇일까요?   
<br>
첫 번째, 현재 인증된 user를 나타냅니다. 그리고 SecurityContext를 통해 해당 인증 정보를 얻을 수 있습니다.  
두 번째, user의 자격 증명을 제공하기 위한 AuthenticationManager에 대한 입력입니다.

위에 설명하였듯이 Authentication은 인증 정보를 가지고 있습니다. 그럼 인증과 권한, user id 등의 더 자세한 정보는 어떻게 확인할 수 있을까요?  
SecurityContextHolder의 설명에 나와있던 사진에 있듯이 Authentication은 `Principal`, `credentials`, `authorities`을 가지고 있으며 이 3가지를 통해 확인이 가능합니다.  
* **Principal**  
   user를 식별 하며 '누구?'에 대한 정보  
   UserDetailsService에 의해 반환된 [UserDetails](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-userdetails)의 instance
* **authorities**  
   user에게 부여된 권한 (GrantedAuthority 참고)
   ex) ROLE_ADMINISTRATOR, ROLE_HR_SUPERVISOR와, ROLE_USER 등등..
* **credentials**  
   주체가 올바르다는 것을 증명하는 자격 증명  
   일반적으로 암호이지만 인증 관리자와 관련된 암호일 수 있다

그럼 Principal, credentials, authorities에 대한 정보를 코드를 통해 확인해 보겠습니다.
```java
SecurityContext context = SecurityContextHolder.getContext();
Authentication authentication = context.getAuthentication();
String username = authentication.getName();
Object principal = authentication.getPrincipal();
Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
```

이 코드를 실행하고 디버깅하면 아래와 같은 정보를 확인할 수 있습니다.  
Principal, authorities, credentials 3가지가 Authentication에 포함되어 있습니다. 그리고 Authentication은 `UsernamePasswordAuthenticationToken`으로 구현되었네요!  

<img src="{{'/assets/images/spring/security/authentication_debug.png'}}" alt="" class="align-center">

## GrantedAuthority

권한을 나타내며 `Authentication#getAuthorities()`를 통해 얻을 수 있습니다.  
위의 코드화 실행 결과를 보면 타입이 `Collection<? extends GrantedAuthority>`인 것을 확인할 수 있습니다.  GrantedAuthority의 구현체를 값으로 가지는 Collection을 반환하고 있습니다.  
ROLE_ADMINISTRATOR, ROLE_HR_SUPERVISOR와 같이 'ROLE_' 접두사를 자동으로 붙여 역할을 나타냅니다. 

```java
SecurityContext context = SecurityContextHolder.getContext();
Authentication authentication = context.getAuthentication();
Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
```

---

<br>

# AuthenticationManager

`AuthenticationManager`는 Spring Security의 필터에서 인증을 수행하는 방법을 정의하는 API입니다.
일반적으로 ProviderManager를 이용하여 구현하며 `Authentication authenticate(Authentication authentication) throws AuthenticationException` 메서드 1개만 정의하고 있습니다. 이 메서드는 인자로 받은 Authentication이 유효한지 확인하고 Authentication 객체를 리턴합니다.  

SecurityContextHolder 의해 반환된 Authentication은 AuthenticationManager에 의해 검증됩니다.  
일반적으로 ProviderManager 구현체를 이용합니다.  

<img src="{{'/assets/images/spring/security/authenticationManager.png'}}" alt="" class="align-center">

## ProviderManager

`ProviderManager`는 AuthenticationManager의 구현체입니다.  
특정 인증 유형을 확인할 수 있는 `AuthenticationProvider`의 List를 가지고 있으며 이 providers를 이용해 인증을 진행합니다.

스프링 시큐리티는 AuthenticationProvider interface를 통해 여러 유형의 인증을 지원하고 단일 인증 관리자만(ProviderManager) 노출하면서 매우 구체적인 유형의 인증을 수행할 수 있습니다.

<img src="{{'/assets/images/spring/security/providerManager.png'}}" alt="" class="align-center">

ProviderManager는 인증 확인을 수행할 수 있는 AuthenticationProvider가 없을 경우 아래 사진과 같이 상위 AuthenticationProvider를 참조할 수도 있습니다.

<img src="{{'/assets/images/spring/security/providermanagers-parent.png'}}" alt="" class="align-center">

ProviderManager는 Authentication에서 반환된 인증에 성공한 객체에서 인증 정보를 지우려고 합니다. 이 동작으로 인해 암호와 같은 정보가 HttpSession에서 필요한 시간보다 오래 보존되지 않는 장점이 있습니다.


---

> 참고  
> docs : [spring-security](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication)  
> docs : [spring-security SecurityContextHolder](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-securitycontextholder)  
> docs : [spring-security SecurityContext](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-securitycontext)  
> docs : [spring-security Authentication](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-authentication)  
> docs : [spring-security AuthenticationManager](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-authenticationmanager)  

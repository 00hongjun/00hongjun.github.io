---
title: "Spring Security의 AccessDecisionManager 알아보기"
date: 2021-05-24T01:01:01-04:00
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

spring security의 [AccessDecisionManager](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#authz-pre-invocation)에 대한 정리!

SecurityContextHolder가 관리하는 Authentication을 통한 인증을,  
DelegatingFilterProxy를 통해 스프링 시큐리티가 어떻게 SecurityContextHolder를 활용하는지를 정리해 보았습니다.  
이번에는 인가를 처리하는 AccessDecisionManager에 대해 정리해 보겠습니다.

---
<br>

# AccessDecisionManager

`AccessDecisionManager`는 request에 대한 **access/인가를 결정하는 역할**을 합니다.  

Spring Security는 method 호출이나 웹 request 등에 대한 access를 제어하는 interceptor 제공하고 있으며  
AccessDecisionManager는 `AbstractSecurityInterceptor`에 의해 호출됩니다.

<img src="{{'/assets/images/spring/security/AccessDecisionManager.png'}}" alt="" class="align-center">

AccessDecisionManager는 decide 1개와 supports 메서드 2개를 제공하고 있습니다.  
`decide()`는 Authentication, ConfigAttribute 등을 전달받으며, 전달받은 parameter에 대한 access 제어를 결졍하는 역할을 합니다. 이때 만약 거부된다면 AccessDeniedException을 던집니다.  
`supports()`는 각각 전달받은 parameter를 처리 가능한지 판단합니다. 
전달받고 있는 ConfigAttribute는 권한을 나타내는 문자열을 말합니다(ex. ROLE_ADMIN)



---
<br>

# Voter

Spring Security는 **투표를 기반**으로 request에 대한 access에 대한 승인 여부를 결정합니다. 투표에 대한 AccessDecisionManager의 여러 구현체도 있습니다.  
아래 사진은 AccessDecisionManager의 구현체 들과 구현체들이 결정하는데 필요한 Voter, SecurityConfig에 의해 전달되는 ConfigAttribute의 관계를 보여줍니다.

<img src="{{'/assets/images/spring/security/access-decision-voting.png'}}" alt="" class="align-center">

투표를 기반으로 하는 AccessDecisionManager는 `AccessDecisionVoter`를 투표에 이용합니다.  

```java
public interface AccessDecisionVoter<S> {

   int ACCESS_GRANTED = 1;
   int ACCESS_ABSTAIN = 0;
   int ACCESS_DENIED = -1;

   int vote(Authentication authentication, S object, Collection<ConfigAttribute> attributes);
   boolean supports(ConfigAttribute attribute);
   boolean supports(Class<?> clazz);
}
```
`vote()`는 소스에 정의된 **1, 0, -1 값을 반환**하며, 각각은 승인, 거부, 보류를 나타냅니다. 
승인 결정에 대한 의견이 없는 경우 ACCESS_ABSTAIN, 승인인 경우는 ACCESS_DENIED를 거부일 경우는 ACCESS_GRANTED를 반환합니다.


---

## RoleVoter

`RoleVoter`는 AccessDecisionVoter 구현체입니다.  
권한에 이름을 부여하고 이 이름을 이용하여 access 권한을 부여하는 역할을 합니다.  

ConfigAttribute가 접두사 **ROLE_로 시작하면 투표**합니다.  
ROLE_로 시작하는 하나 이상의 ConfigAttributes와 정확히 동일한 String 표현(getAuthority()를 통해)을 반환하는 GrantedAuthority가 있는 경우 액세스 권한을 부여하기 위해 1을 반환합니다.  
ROLE_로 시작하는 ConfigAttribute와 정확히 일치하는 항목이 없는 경우 RoleVoter는 액세스를 거부하도록 -1을 투표합니다.  
ROLE_로 시작하는 ConfigAttribute가 없으면 투표자는 기권합니다.  

<img src="{{'/assets/images/spring/security/RoleVoter.png'}}" alt="" class="align-center">

---

## Hierarchical Roles

보통 ADMIN은 USER의 권한을 모두 포함합니다. 이런 권한 간의 계층 관계가 있는 경우 어떻게 처리해야 할까요?
권한끼리 계층 관계를 가질 경우 RoleVoter만으로는 표현하기 복잡해집니다.  

Security는 이런 권한 간의 관계 처리를 위해 커스텀 기능을 제공하고 있습니다. 바로 '>'를 이용하여 표현 가능합니다.
이러한 기능은 RoleVoter의 확장 버전인 RoleHierarchyVoter를 이용해 제공하고 있으며, 우리는 `RoleHierarchyImpl`을 이용해 아래와 같은 방식으로 설정 가능합니다.
```java
public AccessDecisionManager accessDecisionManager() {
        RoleHierarchyImpl roleHierarchy = new RoleHierarchyImpl();
        roleHierarchy.setHierarchy("ROLE_ADMIN > ROLE_USER");

        DefaultWebSecurityExpressionHandler handler = new DefaultWebSecurityExpressionHandler();
        handler.setRoleHierarchy(roleHierarchy);

        WebExpressionVoter webExpressionVoter = new WebExpressionVoter();
        webExpressionVoter.setExpressionHandler(handler);

        List<AccessDecisionVoter<? extends Object>> voters = Arrays.asList(webExpressionVoter);
        return new AffirmativeBased(voters);
    }

    public SecurityExpressionHandler securityExpressionHandler() {
        RoleHierarchyImpl roleHierarchy = new RoleHierarchyImpl();
        roleHierarchy.setHierarchy("ROLE_ADMIN > ROLE_USER");

        DefaultWebSecurityExpressionHandler handler = new DefaultWebSecurityExpressionHandler();
        handler.setRoleHierarchy(roleHierarchy);

        return handler;
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .mvcMatchers("/", "/info", "/account/**").permitAll()
            .mvcMatchers("/admin").hasRole("ADMIN")
            .mvcMatchers("/user").hasRole("USER")
            .anyRequest().authenticated()
//            .accessDecisionManager(accessDecisionManager()); // AccessDecisionManager을 커스텀 하는 방법
            .expressionHandler(securityExpressionHandler()); // SecurityExpressionHandler을 커스텀 하는 방법

        http.formLogin();
        http.httpBasic();
    }
```



> 참고  
> docs : [spring-security](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication)  

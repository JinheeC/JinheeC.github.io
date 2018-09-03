---
title: "Spring Security Remember Me 가 동작하지 않는 경우"
categories: 
- Spring Security
excerpt: |
  Bold 된 부분에서 보듯 Internal secret key 를 넣어주고 있다 그리고 rememberMeService 에도 internal secret key로 생성한 TokenBasedRememberMeServices 를 만들어주는 것을 볼 수 있다.
  이 의미는 key와 edaToolsUserDetailsService(맞는 사용자인지 확인하는 로직)를 가지고 토큰기반의 RememberMeService 를 만들어서 사용하겠다는 뜻이고 그 Key 와 같은 Key 로 RememberMeConfigurer 에서 init() 과정을 통해 
feature_text: |
  Spring Security Remember Me 가 동작하지 않는 경우
---

# Spring Security - Cookie 기반 Remember Me 동작을 안할 경우


* table of contents
{:toc .toc}

## AuthenticationProvider로 넘겨주는 Key 와 userDetailsService의 Key 가 같은지 확인하자
RememberMe 설정 시 아래와 같이 Key 를 설정해 주는데 
``` java
Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
private static final String **INTERNAL_SECRET_KEY** = "INTERNAL_SECRET_KEY";
...
	@Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf()
            .disable();

        http.authorizeRequests()
            .antMatchers("/**")
            .hasRole("ADMIN")
            .and()
            .formLogin()
            .successForwardUrl("/")
            .loginPage("/login")
            .successHandler(edaToolAuthenticationSuccessHandler)
            .permitAll()
            .and()
            .rememberMe()
            .key(**INTERNAL_SECRET_KEY**)
            .rememberMeServices(**rememberMeServices()**);

        http.logout()
            .logoutSuccessUrl("/login");
    }
...
	private RememberMeServices **rememberMeServices()** {
        return new TokenBasedRememberMeServices(SecurityConfig.INTERNAL_SECRET_KEY, edaToolsUserDetailsService);
    }
}
```
Bold 된 부분에서 보듯 Internal secret key 를 넣어주고 있다 그리고 rememberMeService 에도 internal secret key로 생성한 TokenBasedRememberMeServices 를 만들어주는 것을 볼 수 있다.
이 의미는 key와 edaToolsUserDetailsService(맞는 사용자인지 확인하는 로직)를 가지고 토큰기반의 RememberMeService 를 만들어서 사용하겠다는 뜻이고 그 Key 와 같은 Key 로 RememberMeConfigurer 에서 init() 과정을 통해 RememberMe에 사용할 RememberMeAuthenticationProvider 를 만들게 된다.  그 provider 가 사용자가 접속을 하면 쿠키의 값과 비교하게 되는데 그 값을 비교한 직후 provider 와 TokenBasedRememberMeService (아래의 코드에서는 (RememberMeAuthenticationToken) authentication 부분 - TokenBasedRememberMeService의 상위클래스인 AbstractRememberMeServices 클래스가 TokenBasedRememberMeService 를 만들때 넣어준 internal secret key 로 RememberMeAuthenticationToken 을 만들게 된다. 그래서 TokenBasedRememberMeService 를 만들때 들고 있는 Key 가 아래의 코드의 RememberMeAuthenticationToken authentication이 들고 있게 됨.) 그래서 아래와 같이 Key 가 같지 않으면 BadCredentialsException 를 내고 쿠키를 삭제하기 때문에 같아야 한다.

``` java
RememberMeAuthenticationProvider.java

public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
		if (!supports(authentication.getClass())) {
			return null;
		}

		if (this.key.hashCode() != ((RememberMeAuthenticationToken) authentication)
				.getKeyHash()) {                         ...........(1)
			throw new BadCredentialsException(
					messages.getMessage("RememberMeAuthenticationProvider.incorrectKey",
							"The presented RememberMeAuthenticationToken does not contain the expected key"));
		}

		return authentication;
}
```

아래의 캡쳐는 위 코드의 (1) 부분에서 디버깅을 걸어본 내용이다. 첫번째 캡쳐는 SecurityConfig 에서 .(key) 와 TokenBasedRememberMeServices(key) 를 같게 넣어줬을때의 결과이고 두번째 캡쳐는 SecurityConfig 에서 .(key) 를 아얘 주지 않았을 때의 결과이다. 랜덤으로 값이 들어가면서 this.key.hashCode() 값과 authentication.getKeyHash() 값이 다른것을 확인할 수 있다.

![Alt text](https://monosnap.com/image/fgblibpKJgdrIl79rTfFsG9OEBgygq)


## ROLE_ 로 시작되는지 확인하자

Spring Security 에서 모든 Authentication 객체는 **GrantedAuthority** 라는 객체의 List 를 가지고 있다. 이 객체는 Principal 에 부여된 "권한"을 뜻하는데 AuthenticationManager에 의해 Authentication 객체 안에 저장되어지고 이 값은 최종 AccessDecisionManager 가 "인증"에 관련된 결정을 내릴때 사용되어진다.
가장 마지막 단계인 `AccessDecisionManager` 는 투표 결과에 기반해서 `AccessDeniedException` 을 던질 것인지 아닌지를 결정하는 인터페이스이다. 3개의 메소드가 있는데 아래와 같다.
``` java
int vote(Authentication authentication, Object object, Collection<ConfigAttribute> attrs);

boolean supports(ConfigAttribute attribute);

boolean supports(Class clazz);
```
투표의 결과로 int 가 반환되는데 각 값은 `ACCESS_ABSTAIN`(기권),  `ACCESS_DENIED`(거부),  `ACCESS_GRANTED`(허용) 을 뜻한다. 

![Alt text](https://monosnap.com/image/PkaWbpM8Q66CNiwowYXhHKpyo2exDa)

투표의 결과를 결정하는 AccessDecisionManager 의 구현체는 3개가 존재한다.  `ConsensusBased` 구현은 기권이 아닌 투표결과의 컨센서스에 기반하여 Grant또는 Deny 를 하게된다. `AffirmativeBased` 는 하나 이상의 ACCESS_GRANTED 를 받게되면 무조건 허용된다. 이 경우 ACCESS_DENIED 표는 무시되어지는것을 명심해야 한다. `UnanimousBased` 는 익명의 ACCESS_GRANTED 를 받아야 허용을 하고 기권은 무시된다. 그리고 익명의 ACCESS_DENIED 가 하나라도 있게되면 엑세스가 거부된다. 이 세가지 구현체는 모두 유권자가 기권할 경우 행동을 통제하는 파라미터가 존재해서 기권에 대한 처리가 가능하다.

그렇다면 그 투표를 하는 주체는 누구일까. 
AuthenticatedVoter 가 그 일을 하게된다. 그런데 스프링 시큐리티는 AuthenticatedVoter의 구현체로 간단한 `RoleVoter` 를 제공한다. RoleVoter 는 사용자한테 해당 ROLE이 할당 된 경우 투표를 하는 주체이고 간단한 ROLE 이름을 설정할수도 있다. 여기서  GrantedAuthority 를  ConfigAttribute.getAttribute() 로 가져와서 이 Role(권한)이 **ROLE_** 이라는 접두어로 시작하면 투표를 하게 된다. 만약 ROLE_ 로 이름이 시작되지 않은경우 **기권을 하게 되므로 주의해야 한다**. 
``` java
public class RoleVoter implements AccessDecisionVoter<Object> {
	private String rolePrefix = "ROLE_";

	public String getRolePrefix() {
		return rolePrefix;
	}

	/**
	 * Allows the default role prefix of <code>ROLE_</code> to be overridden. May be set
	 * to an empty value, although this is usually not desirable.
	 *
	 * @param rolePrefix the new prefix
	 */
	public void setRolePrefix(String rolePrefix) {
		this.rolePrefix = rolePrefix;
	}
...
}
```

그러므로 Spring Security 를 사용하는데 **Remember-Me 등 의 엑세스 권한 로직이 예상한 대로 작동하지 않은 경우 ROLE_ 프리픽스가 틀리지는 않았는지 의심해봐야 한다.** ROLE_ 은 디폴트 값일 뿐이고 원한다면 다른 prefix 로 수정이 가능하다.

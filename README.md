# Session Cookie Study
로그인 처리를 위한 공부, 구현 과정 기록

1) cookie를 활용한 구현
2) session쿠키를 활용한 구현

## 1. Cookie를 활용한 구현
서버에서 쿠키를 클라이언트에게 전달한다면, 이후부터는 클라이언트가 요청할떄 마다 서버에 기존의 쿠키를 전달한다.

### - 쿠키에는 영속 쿠키와 세션 쿠키가 있다.
- 영속 쿠키: 만료 날짜를 입력하면 해당 날짜까지 유지
- 세션 쿠키: 만료 날짜를 생략하면 브라우저 종료시 까지만 유지

### - 쿠키 생성 로직
``` java
Cookie idCookie = new Cookie("memberId", String.valueOf(loginMember.getId()));
response.addCookie(idCookie);
```
로그인에 성공하면 cookie를 생성하고, HttpServletResponse에 담는다.
해당 쿠키를 클라이언트에게 전달하면, 웹 브라우저 종료 전까지는 쿠키id를 서버에 계속 보낸다.

이때 RequestHeader에 Cookie가 설정되어 전달된다.   

### - 컨트롤러에서 쿠키 받기
```java
public String homeLogin(@CookieValue(name = "memberId", required = false) Long memberId, Model model)
        // 생략...
}
```

Spring이 지원해주는 @CookieValue를 통해 클라이언트에서 오는 쿠키를 편하게 조회할 수 있다.

로그인 하지 않는 사용자도 홈에 접근은 가능해야하기 때문에 required = false 를 사용한다.

### - 로그아웃 구현

세션 쿠키이므로 웹 브라우저 종료시 쿠키가 삭제되어 로그아웃 처리된다.
서버에서 해당 쿠키의 종료 날짜를 0으로 지정하여 로그아웃 처리할수 있다.
```java
@PostMapping("/logout")
public String logout(HttpServletResponse response){
    expireCookie(response, "memberId");
    return "redirect:/";
}

private void expireCookie(HttpServletResponse response, String cookieName) {
    Cookie cookie = new Cookie(cookieName, null);
    cookie.setMaxAge(0);
    response.addCookie(cookie);
}
```
___

## 2. Cookie의 단점
### - 쿠키의 값은 클라이언트 측에서 임의로 변경할수가 있다.
### - 쿠키에 보관된 정보는 훔쳐갈수가 있다.

___

## 3. Session처리 방식
앞서 언급했던 문제들을 해결하려면 중요한 정보를 모두 서버에 저장해야 한다. 그리고 클라이언트와 서버는 추정 불가능한 임의의 식별자 값으로 연결해야 한다.
이를 **세션** 이라고 부른다

1) 클라이언트가 loginId, password 정보를 전달하면, 서버에서 해당 사용자가 있는지 확인한다.
2) 해당 User가 있다면 임의의 세션 ID 를 생성하여 세션 저장소에 Key : Value 매칭으로 저장해 둔다.

이때 UUID를 사용하여 세션 ID를 만든다. UUID는 추정이 불가능 하다. 예를 들면 다음과 같다.
```
Cookie: mySessionId=zz0101xx-bab9-4b92-9b32-dadb280f4b61
```
3) 서버에서 생성된 세션 ID를 클라이언트에게 전달한다. 중요한 점은 민감한 사용자의 정보 없이 세션 ID만 전달했다는 의미이다.
4) 클라이언트가 요청할 때 마다 세션ID 쿠키를 서버에 전달한다.

● 해결된 문제점들

1) 쿠키 값 변경이 불가능 해졌다. 중간에 몇글자 바꿔 보내봤자 서버에는 해당 데이터가 없어 접근할수가 없다.

2) 세션 ID는 털려도 중요한 정보가 없다. 중요한 정보는 다 서버에서 보관중이다.

3) 세션의 만료시간을 짧게 유지하여, 해커가 토큰을 털어가도 무용지물이 되도록 할수있다.

___
## 4.Session 구현해보기
1) 세션 생성 : sessionId생성, 세션 저장소에 sessionId와 값을 저장, response에 쿠키를 생성하여 클라이언트에게 전달.

2) 세션 조회 : 클라이언트가 요청한 sessionId 쿠키의 값으로 세션 저장소에서 보관하고 있던 값을 찾아온다.

3) 세션 만료 : 클라이언트가 요청한 sessionId로 세션 저장소에 보관된 값을 찾아 제거한다.

[구현소스로 이동](https://github.com/zbqmgldjfh/SessionCookieStudy/blob/master/src/main/java/hello/login/web/session/SessionManager.java)
___  

## 5.Session 정보와 타임아웃 설정
대부분의 사용자는 로그아웃을 선택하지 않고, 그냥 웹 브라우저를 종료한다.  
문제는 HTTP가 비 연결성(ConnectionLess)이므로 서버 입장에서는 해당 사용자가 웹 브라우저를 종료한 것인지 아닌지를 인식할 수 없다.  
따라서 서버에서 세션 데이터를 언제 삭제해야 하는지 판단하기가 어렵다.

세션 생성 시점이 아니라 사용자가 서버에 최근에 요청한 시간을 기준으로 30분 정도를 유지해주는 것이다.  
이렇게 하면 사용자가 서비스를 사용하고 있으면, 세션의 생존 시간이 30분으로 계속 늘어나게 된다.   
따라서 30분 마다 로그인해야 하는 번거로움이 사라진다. HttpSession 은 이 방식을 사용한다.  

session.getLastAccessedTime() : 최근 세션 접근 시간  
LastAccessedTime 이후로 timeout 시간이 지나면, WAS가 내부에서 해당 세션을 제거한다.
___

## 6. 서블릿 필터
애플리케이션 여러 로직에서 공통으로 관심이 있는 있는 것을 공통 관심사(cross-cutting concern)라고 한다.
이러한 공통 관심사는 AOP로도 처리가 가능하지만, 일반적으로 서블릿 필터나, 스프링 인터셉터를 사용하는것이 좋다.

- 서블릿 필터의 흐름
```
 HTTP 요청 -> WAS-> 필터 -> 서블릿 -> 컨트롤러
```
필터를 적용하면 필터가 호출 된 다음에 서블릿이 호출된다.  

그래서 모든 고객의 요청 로그를 남기는 요구사항이 있다면 필터를 사용하면 된다.
참고로 필터는 특정 URL 패턴에 적용할 수 있다, /* 이라고 하면 모든 요청에 필터가 적용된다.  

참고로 스프링을 사용하는 경우 여기서 말하는 서블릿은 스프링의 디스패처 서블릿으로 생각하면 된다.  
    
     
- 필터 제한
```
HTTP 요청 -> WAS -> 필터 -> 서블릿 -> 컨트롤러 //로그인 사용자
HTTP 요청 -> WAS -> 필터(적절하지 않은 요청이라 판단, 서블릿 호출X) //비 로그인 사용자
```
필터에서 적절하지 않은 요청이라고 판단하면 거기에서 끝을 낼 수도 있다. 그래서 로그인 여부를 체크하기에 딱 좋다.


- 필터 인터페이스
```java
public interface Filter {
    public default void init(FilterConfig filterConfig) throws ServletException {}

    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException;

    public default void destroy() {}
}
```
필터 인터페이스를 구현하고 등록하면 서블릿 컨테이너가 필터를 싱글톤 객체로 생성하고, 관리한다.

- init(): 필터 초기화 메서드, 서블릿 컨테이너가 생성될 때 호출된다.
- doFilter(): 고객의 요청이 올 때 마다 해당 메서드가 호출된다. 필터의 로직을 구현하면 된다. 
- destroy(): 필터 종료 메서드, 서블릿 컨테이너가 종료될 때 호출된다.

___
## 7. 스프링 인터셉터
서블릿 필터가 서블릿이 제공하는 기술이라면, 스프링 인터셉터는 스프링 MVC가 제공하는 기술이다.   
둘다 웹과 관련된 공통 관심 사항을 처리하지만, 적용되는 순서와 범위, 그리고 사용방법이 다르다.  

- 스프링 인터셉터의 흐름
```
HTTP 요청 ->WAS-> 필터 -> 서블릿 -> 스프링 인터셉터 -> 컨트롤러
```
스프링 인터셉터는 디스패처 서블릿과 컨트롤러 사이에서 컨트롤러 호출 직전에 호출 된다.  
스프링 인터셉터에도 URL 패턴을 적용할 수 있는데, 서블릿 URL 패턴과는 다르고, 매우 정밀하게 설정할 수 있다.  

- 스프링 인터셉터 인터페이스
```java
public interface HandlerInterceptor {

    default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {}

    default void postHandle(HttpServletRequest request, HttpServletResponse response, 
        Object handler, @Nullable ModelAndView modelAndView) throws Exception {}

    default void afterCompletion(HttpServletRequest request, HttpServletResponse response, 
        Object handler, @Nullable Exception ex) throws Exception {}
}
```
인터셉터는 컨트롤러 호출 전( preHandle ), 호출 후( postHandle ), 요청 완료 이후( afterCompletion )와 같이 단계적으로 잘 세분화 되어 있다. 

서블릿 필터의 경우 단순히 request , response 만 제공했지만, 인터셉터는 어떤 컨트롤러( handler )가 호출되는지 호출 정보도 받을 수 있다.  
그리고 어떤 modelAndView 가 반환되는지 응답 정보도 받을 수 있다.


● 정상 흐름

- preHandle : 컨트롤러 호출 전에 호출된다. (더 정확히는 핸들러 어댑터 호출 전에 호출된다.)

preHandle 의 응답값이 true 이면 다음으로 진행하고, false 이면 더는 진행하지 않는다.

false 인 경우 나머지 인터셉터는 물론이고, 핸들러 어댑터도 호출되지 않는다. 위 그림에서 1번에서 끝이 나버린다.


- postHandle : 컨트롤러 호출 후에 호출된다. (더 정확히는 핸들러 어댑터 호출 후에 호출된다.)

컨트롤러에서 예외가 발생하면 호출되지 않는다.


- afterCompletion : 뷰가 렌더링 된 이후에 호출된다. (예외가 발생해도 호출된다.)

예외가 발생하면 postHandle() 는 호출되지 않으므로 예외와 무관하게 공통 처리를 하려면 afterCompletion() 을 사용해야 한다.

예외가 발생하면 afterCompletion() 에 예외 정보( ex )를 포함해서 호출된다.

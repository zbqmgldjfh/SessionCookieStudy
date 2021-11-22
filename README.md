# Session Cookie Study
로그인 처리를 위한 공부, 구현 과정 기록

1) cookie를 활용한 구현
2) session쿠키를 활용한 구현

## 1. Cookie를 활용한 구현
서버에서 쿠키를 클라이언트에게 전달한다면, 이후부터는 클라이언트가 요청할떄 마다 서버에 기존의 쿠키를 전달한다.

### - 쿠키에는 영속 쿠키와 세션 쿠키가 있다.
- 영속 쿠키: 만료 날짜를 입력하면 해당 날짜까지 유지
- 세션 쿠키: 만료 날짜를 생략하면 브라우저 종료시 까지만 유지

###- 쿠키 생성 로직
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
##4.Session 구현해보기
1) 세션 생성 : sessionId생성, 세션 저장소에 sessionId와 값을 저장, response에 쿠키를 생성하여 클라이언트에게 전달.

2) 세션 조회 : 클라이언트가 요청한 sessionId 쿠키의 값으로 세션 저장소에서 보관하고 있던 값을 찾아온다.

3) 세션 만료 : 클라이언트가 요청한 sessionId로 세션 저장소에 보관된 값을 찾아 제거한다.

[구현소스로 이동](https://github.com/zbqmgldjfh/SessionCookieStudy/blob/master/src/main/java/hello/login/web/session/SessionManager.java)
___

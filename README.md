* ## 서블릿 예외 처리 시작

서블릿은 다음 2가지 방식으로 예외처리를 지원함
* `Exception` (예외) 
* `response.sendError`(HTTP 상태코드, 오류 메시지)

### Exception(예외)

* #### 자바 직접 실행
자바의 메인 메서드를 직접 실행하는 경우 main 이라는 이름의 쓰레드가 실행된다.

실행 도중에 예외를 잡지 못하고 처음 실행한 main() 메서드를 넘어서 예외가 던져지면, 

예외 정보를 남기고 해당 쓰레드는 종료된다.


* ####  웹 애플리케이션
웹 애플리케이션은 사용자 요청별로 별도의 쓰레드가 할당되고, 서블릿 컨테이너 안에서 실행된다.

애플리케이션에서 예외가 발생했는데, 어디선가 try ~ catch로 예외를 잡아서 처리하면 아무런 문제가
없다. 

그런데 만약에 애플리케이션에서 예외를 잡지 못하고, 서블릿 밖으로 까지 예외가 전달되면 어떻게
동작할까?

> WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
> 


* ### 서블릿 예외 처리 - 오류 페이지 작동원리
서블릿은 `Exception`이 발생해서 서블릿 밖으로 전달되거나 또는  `response.sendError()`가 호출 되었을때 설정된 오류 페이지를 찾는다.

* 예외 발생흐름
> was(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)

* sendError 흐름
> was(sendError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(response.sendError())

WAS는 해당 예외를 처리하는 오류 페이지 정보를 확인한다.
```java
new ErrorPage(RuntimeException.class, "/error-page/500");
```
예를 들어서 `RuntimeException`예외가 WAS까지 전달되면, WAS는 오류 페이지 정보를 확인
`RuntimeException`의 오류페이지로 `/error-page/500`이 지정되어있다. WAS는 오류페이지를 출력하기 위해 `/error-page/500`을 다시 요청한다.

* 오류 페이지 요청 흐름
> WAS `/error-page/500` 다시요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> view


* 예외 발생과 오류 페이지 요청 흐름
> 1. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
> 2. WAS `/error-page/500` 다시요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> view

* #### 중요한 점은 웹 브라우저(클라이언트)는 서버 내부에서 이런 일이 일어나는지 전혀 모른다. 오직 서버 내부에서 오류 페이지를 찾기 위해 추가적인 호출을 한다.



* ### 정리하면 다음과 같다.
1. 예외가 발생해서 WAS까지 전파된다.
2. WAS는 오류 페이지 경로를 찾아서 내부에서 오류 페이지를 호출한다. 이때 오류 페이지 경로로 필터,
   서블릿, 인터셉터, 컨트롤러가 모두 다시 호출된다.


* ### DispatcherType
  * 필터의 경우 `dispatcherType` 이라는 옵션을 제공한다.
  * 오류 페이지에선 `dispatcherType = ERROR`
  * 고객이 처음 요청하면 `dispatcherType = REQUEST`
  

서블릿 스팩은 실제 고객이 요청한 것인지, 서버가 내부에서 오류페이지를 요청하는 것인지 `dispatcherType`으로 구분가능

```java

public enum DispatcherType {
 FORWARD, // 서블릿에서 다른 서블릿이나 jsp를 호출할때  RequestDispatcher.forward(request, response);
 INCLUDE, // 서블릿에서 다른 서블릿이나 jsp의 결과를 포함할때 RequestDispatcher.include(request, response);
 REQUEST, // 클라이언트 요청
 ASYNC, // 서블릿 비동기 호출
 ERROR // 오류 요청
}

```

* ### 전체 흐름 정리
* /hello 정상 요청
> WAS(/hello, dispatchType=REQUEST) -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러 -> View


* /error-ex 오류 요청
> 필터는 DispatchType 으로 중복 호출 제거 ( dispatchType=REQUEST )
인터셉터는 경로 정보로 중복 호출 제거( excludePathPatterns("/error-page/**") )
> 
> 1. WAS(/error-ex, dispatchType=REQUEST) -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러
> 2. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
> 3. WAS 오류 페이지 확인
> 4. WAS(/error-page/500, dispatchType=ERROR) -> 필터(x) -> 서블릿 -> 인터셉터(x) ->
   컨트롤러(/error-page/500) -> View


* ## 스프링 부트 - 오류 페이지

* ### 스프링 부트는 이런 과정을 모두 기본으로 제공한다.
  * `ErrorPage` 를 자동으로 등록한다. 이때 `/error` 라는 경로로 기본 오류 페이지를 설정한다.
    * `new ErrorPage("/error")` , 상태코드와 예외를 설정하지 않으면 기본 오류 페이지로 사용된다.
    * 서블릿 밖으로 예외가 발생하거나, `response.sendError(...)` 가 호출되면 모든 오류는 `/error` 를
    호출하게 된다.
    * `BasicErrorController` 라는 스프링 컨트롤러를 자동으로 등록한다.
    * `ErrorPage` 에서 등록한 `/error` 를 매핑해서 처리하는 컨트롤러다.

> 참고
> ErrorMvcAutoConfiguration 이라는 클래스가 오류 페이지를 자동으로 등록하는 역할을 한다.
>



* ### 뷰 선택 우선순위
`BasicErrorController` 의 처리 순서
1. 뷰 템플릿
   * `resources/templates/error/500.html`
   * `resources/templates/error/5xx.html`
2. 정적 리소스( static , public )
   * `resources/static/error/400.html`
   * `resources/static/error/404.html`
   * `resources/static/error/4xx.html`
3. 적용 대상이 없을 때 뷰 이름( error )
  * `resources/templates/error.html`
 

* ## API 예외 처리
* #### HandlerExceptionResolver
  * 스프링 MVC는 컨트롤러(핸들러) 밖으로 예외가 던져진 경우 예외를 해결하고, 동작을 새로 정의할 수 있는 방법을 제공한다. 
  * 컨트롤러 밖으로 던져진 예외를 해겨ㅑㄹ하고, 동작 방식을 변경하고 싶으면 `HandlerExceptionResolver`를 사용하면 된다. 
  * 줄여서 `ExceptionResolver`

![image](https://user-images.githubusercontent.com/60100532/177977796-5d5a1fe2-a3e8-4573-99b0-494ed437481d.png)

![image](https://user-images.githubusercontent.com/60100532/177977854-9a1045dd-c202-4f9b-9db5-c23ef75512b6.png)





* #### API 예외 처리 - 스프링이 제공하는 ExceptionResolver1
* 스프링 부트가 기본으로 제공하는 `ExceptionResolver` 는 다음과 같다.
  `HandlerExceptionResolverComposite` 에 다음 순서로 등록
1. ExceptionHandlerExceptionResolver
2. ResponseStatusExceptionResolver
3. DefaultHandlerExceptionResolver 우선 순위가 가장 낮다

&nbsp;
&nbsp;

#### ExceptionHandlerExceptionResolver
`@ExceptionHandler` 을 처리한다. API 예외 처리는 대부분 이 기능으로 해결한다.

&nbsp;
&nbsp;

#### ResponseStatusExceptionResolver 
HTTP 상태 코드를 지정해준다.
예) `@ResponseStatus(value = HttpStatus.NOT_FOUND)`



&nbsp;
&nbsp;
#### DefaultHandlerExceptionResolver
스프링 내부 기본 예외를 처리한다.

&nbsp;
&nbsp;

* ### ResponseStatusExceptionResolver
![image](https://user-images.githubusercontent.com/60100532/178018892-9b69ba1c-25b6-42d4-89cd-e4c9568ae564.png)

```java

public class ResponseStatusExceptionResolver extends AbstractHandlerExceptionResolver implements MessageSourceAware {

    @Nullable
    private MessageSource messageSource;


    @Override
    public void setMessageSource(MessageSource messageSource) {
        this.messageSource = messageSource;
    }


    @Override
    @Nullable
    protected ModelAndView doResolveException(
            HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

        try {
            if (ex instanceof ResponseStatusException) {
                return resolveResponseStatusException((ResponseStatusException) ex, request, response, handler);
            }

            ResponseStatus status = AnnotatedElementUtils.findMergedAnnotation(ex.getClass(), ResponseStatus.class);
            if (status != null) {
                return resolveResponseStatus(status, request, response, handler, ex);
            }





    protected ModelAndView applyStatusAndReason(int statusCode, @Nullable String reason, HttpServletResponse response)
    throws IOException {

        if (!StringUtils.hasLength(reason)) {
            response.sendError(statusCode);
        }
        else {
            String resolvedReason = (this.messageSource != null ?
                    this.messageSource.getMessage(reason, null, reason, LocaleContextHolder.getLocale()) :
                    reason);
            response.sendError(statusCode, resolvedReason);
        }
        return new ModelAndView();
    }
```


`ResponseStatusExceptionResolver` 코드를 확인해보면 결국 `response.sendError(statusCode, resolvedReason)`를 호출한다. 

reason을 메시지소스에서 찾는기능도 제공

```java

String resolvedReason = (this.messageSource != null ?
					this.messageSource.getMessage(reason, null, reason, LocaleContextHolder.getLocale()) :
					reason);
			response.sendError(statusCode, resolvedReason);
```


* ### `DefaultHandlerExceptionResolver`
* 
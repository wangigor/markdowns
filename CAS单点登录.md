# CAS单点登录

> 市面上单点登录的产品有很多。
>
> 因为跟外系统对接的关系，认证的服务方想要使用CAS统一认证。
>
> 这才有机会研究下CAS的原理。这里只说client。

> CAS是耶鲁大学Technology and Planning实验室的Shawn Bayern 在2002年出的一个开源系统。
>
> [CAS gitHub](https://github.com/apereo/cas)
>
> [CAS官网](https://apereo.github.io/cas/6.0.x/index.html)
>
> [client端使用jar](https://github.com/apereo/java-cas-client)

## 流程

> 注意。CAS与jwt不同。
>
> - CAS是基于同一个浏览器的cookie做的多域签发。不能跨浏览器。
> - jwt是基于同一套token解析做的token跨域传播。可以跨浏览器。

![image-20201109223606920](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/我再试试.png)

### aaa域登录

- 「首次」用户访问aaa.com，发现未登录。
- 重定向「带上自己的重定向接收地址」到cas认证中心，**cas在登录页创建cas域的浏览器cookie「CASTGC」**
- 用户在cas的登录页完成登录后，登录信息会以密文的形式写入cookie「CASTGC」，用于二次登录。
- cas发起重定向回调「调用方aaa接收」，签发ticket「用于获取用户信息」
- 调用方aaa，接收到ticket，并使用域名aaa.com，发送http请求从cas认证中心获取用户信息「principal」。

### bbb域登录

> 前提：同浏览器

- 「首次」用户访问bbb.com，发现未登录。
- 重定向「带上自己的重定向接收地址」到cas认证中心，**cas在登录页使用自己域内的cookie「CASTGC」完成登录。**
- cas发起重定向回调「调用方bbb接收」，签发ticket「专属于bbb域」。
- 调用方bbb，接收到ticket，并使用域名aaa.com，发送http请求从cas认证中心获取用户信息「principal」。

## 源码解析

> 根据前面的流程，client端可以提炼出来的模块是**「登录状态检查」和「ticket接收，获取用户信息」**
>
> 因为我一开始的认知是token，想把组件拆开来融入系统中，所以先看的源码，走了一点弯路。

### 登录状态检查

```java
//AuthenticationFilter
public final void doFilter(final ServletRequest servletRequest, final ServletResponse servletResponse, final FilterChain filterChain) throws IOException, ServletException {
    final HttpServletRequest request = (HttpServletRequest) servletRequest;
    final HttpServletResponse response = (HttpServletResponse) servletResponse;
    final HttpSession session = request.getSession(false);
  	//本地请求url
    final String serviceUrl = constructServiceUrl(request, response);
  	//从session看有没有cas—ticket解析得到的用户数据
    final Assertion assertion = session != null ? (Assertion) session.getAttribute(CONST_CAS_ASSERTION) : null;    

  	//已登录，跳过本地登录
    if (assertion != null) {
        filterChain.doFilter(request, response);
        return;
    }
		//本次接收到ticket，使用后续filter解析
    final String ticket = CommonUtils.safeGetParameter(request,getArtifactParameterName());

    if (CommonUtils.isNotBlank(ticket) || wasGatewayed) {
        filterChain.doFilter(request, response);
        return;
    }

		//重定向到cas登录页「带本次请求uri和签名域」
    final String urlToRedirectTo = CommonUtils.constructRedirectUrl(this.casServerLoginUrl, getServiceParameterName(), modifiedServiceUrl, this.renew, this.gateway);

    response.sendRedirect(urlToRedirectTo);
}
```

### ticket接收，获取用户信息

```java
//Cas20ServiceTicketValidator extends AbstractCasProtocolUrlBasedTicketValidator
// extends AbstractUrlBasedTicketValidator
public Assertion validate(final String ticket, final String service) throws TicketValidationException {
				
  			//service 可以理解为本地域签名 比如“http://bbb.com”
        final String validationUrl = constructValidationUrl(ticket, service);

        try {
          	//发送至cas-server获取用户信息
            final String serverResponse = retrieveResponseFromServer(new URL(validationUrl), ticket);

          	//『ticket，域名』鉴权失败「由其他组件进行重试」
            if (serverResponse == null) {
                throw new TicketValidationException("The CAS server returned no response.");
            }
						//鉴权成功「后续放入session中用于登录判断」
            return parseResponseFromServer(serverResponse);
        } catch (final MalformedURLException e) {
            throw new TicketValidationException(e);
        }
    }
```

先到这里，server端没看。
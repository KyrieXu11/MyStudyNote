# spring security过滤器的执行顺序

spring security整合springboot不说了，在另一篇文章中，首先来看一下spring security的过滤器链的执行顺序

其实在平时SpringBoot整合SpringSecurity的时候，启动项目在控制台就能看到过滤器链的执行顺序了。

![security_filterchain.png](https://i.loli.net/2020/03/02/anTwlsGh52fPXrC.png)

简单说明一下过滤器的执行顺序吧：

```
WebAsyncManagerIntegrationFilter -> SecurityContextPersistenceFilter -> HeaderWriterFilter -> LogoutFilter -> UsernamePasswordAuthenticationFilter -> DefaultLoginPageGeneratingFilter -> DefaultLogoutPageGeneratingFilter -> RequestCacheAwareFilter -> SecurityContextHolderAwareRequestFilter -> AnonymousAuthenticationFilter -> SessionManagementFilter -> ExceptionTranslationFilter ->自己定义的一个过滤器仿造FilterSecurityInterceptor继承自了(AbstractSecurityInterceptor) -> FilterSecurityInterceptor
```



和下面要说的过滤器执行顺序是大同小异的。

在`FilterComparator.java`中，我们就可以看到过滤器的执行顺序

```java
private static final int INITIAL_ORDER = 100;
	private static final int ORDER_STEP = 100;
	private final Map<String, Integer> filterToOrder = new HashMap<>();

	FilterComparator() {
		Step order = new Step(INITIAL_ORDER, ORDER_STEP);
		put(ChannelProcessingFilter.class, order.next());
		put(ConcurrentSessionFilter.class, order.next());
		put(WebAsyncManagerIntegrationFilter.class, order.next());
        // 这个就是将Authentication放入session中的一个类
		put(SecurityContextPersistenceFilter.class, order.next());
		put(HeaderWriterFilter.class, order.next());
		put(CorsFilter.class, order.next());
		put(CsrfFilter.class, order.next());
		put(LogoutFilter.class, order.next());
		filterToOrder.put(
			"org.springframework.security.oauth2.client.web.OAuth2AuthorizationRequestRedirectFilter",
				order.next());
		filterToOrder.put(
				"org.springframework.security.saml2.provider.service.servlet.filter.Saml2WebSsoAuthenticationRequestFilter",
				order.next());
		put(X509AuthenticationFilter.class, order.next());
		put(AbstractPreAuthenticatedProcessingFilter.class, order.next());
		filterToOrder.put("org.springframework.security.cas.web.CasAuthenticationFilter",
				order.next());
		filterToOrder.put(
			"org.springframework.security.oauth2.client.web.OAuth2LoginAuthenticationFilter",
				order.next());
		filterToOrder.put(
				"org.springframework.security.saml2.provider.service.servlet.filter.Saml2WebSsoAuthenticationFilter",
				order.next());
		put(UsernamePasswordAuthenticationFilter.class, order.next());
		put(ConcurrentSessionFilter.class, order.next());
		filterToOrder.put(
				"org.springframework.security.openid.OpenIDAuthenticationFilter", order.next());
		put(DefaultLoginPageGeneratingFilter.class, order.next());
		put(DefaultLogoutPageGeneratingFilter.class, order.next());
		put(ConcurrentSessionFilter.class, order.next());
		put(DigestAuthenticationFilter.class, order.next());
		filterToOrder.put(
				"org.springframework.security.oauth2.server.resource.web.BearerTokenAuthenticationFilter", order.next());
		put(BasicAuthenticationFilter.class, order.next());
		put(RequestCacheAwareFilter.class, order.next());
		put(SecurityContextHolderAwareRequestFilter.class, order.next());
		put(JaasApiIntegrationFilter.class, order.next());
		put(RememberMeAuthenticationFilter.class, order.next());
		put(AnonymousAuthenticationFilter.class, order.next());
		filterToOrder.put(
			"org.springframework.security.oauth2.client.web.OAuth2AuthorizationCodeGrantFilter",
				order.next());
		put(SessionManagementFilter.class, order.next());
		put(ExceptionTranslationFilter.class, order.next());
		put(FilterSecurityInterceptor.class, order.next());
		put(SwitchUserFilter.class, order.next());
	}
```

来记录一下各个过滤器源码的功能吧(~~怎么又是源码~~？)

## ChannelProcessingFilter

> Ensures a web request is delivered over the required channel.

这个话是官网文档上写的，翻译过来应该是这个意思： **确保请求能够通过通道传输过来**

> ```
> Internally uses a `FilterInvocation` to represent the request, allowing a
> `FilterInvocationSecurityMetadataSource` to be used to lookup the attributes
> which apply.
> ```

内部使用`FilterInvocation`来代表Request，允许`FilterInvocationSecurityMetadataSource`被用去查找被应用的属性。


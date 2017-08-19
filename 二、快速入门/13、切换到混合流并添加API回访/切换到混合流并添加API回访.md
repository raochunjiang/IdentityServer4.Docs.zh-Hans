# IdentityServer4 中文文档 -13- （快速入门）切换到混合流并添加 API 访问

----------------------------------------------------------------------------------------------

在之前的快速入门中我们探讨了 API 访问和用户认证。现在我们想要把这两部分结合起来。

OpenID Connect 和 OAuth 2.0 结合的美妙之处在于，你既可以使用单一的协议，也可以向令牌服务做一次往返交互。

之前我们使用的是 OpenID Connect 隐式流。在隐式流中所有令牌都通过浏览器来传输，这对于 **身份令牌** 来说是完全没有问题的。现在我们还想要请求一个 **访问令牌**。

与身份令牌相比，访问令牌更加敏感，如果没有必要，我们是不会想将他们暴露给“外部世界”的。OpenID Connect 包含了一个叫做 “混合流（Hybrid flowe）” 的流，它为我们提供了两方面优点 —— 身份令牌通过浏览器频道来传输，这样客户端就能够在做任何工作前验证它；如果验证成功了，客户端就会打开一个后端通道来连接令牌服务以检索访问令牌。

## 修改客户端配置

需要修改的东西不是很多。首先我们想要允许客户端使用混合流（Hybrid Flow），另外我们还想要客户端允许服务于服务之间的 API 调用，并且这种调用不会与用户上下文混杂在一起（这与我们的客户端凭证快速入门非常相似）。这是使用 `AllowedGrantTypes` 属性来表示的。

然后我们要添加一个客户端密码，这将被用于在后端通道上检索访问令牌。

最后我们还要允许客户端访问 `offline_access` scope - 这允许为长期使用的 API 访问请求刷新令牌：

```CSharp
new Client
{
    ClientId = "mvc",
    ClientName = "MVC 客户端",
    AllowedGrantTypes = GrantTypes.HybridAndClientCredentials,

    ClientSecrets =
    {
        new Secret("secret".Sha256())
    },

    RedirectUris           = { "http://localhost:5002/signin-oidc" },
    PostLogoutRedirectUris = { "http://localhost:5002/signout-callback-oidc" },

    AllowedScopes =
    {
        IdentityServerConstants.StandardScopes.OpenId,
        IdentityServerConstants.StandardScopes.Profile,
        "api1"
    },
    AllowOfflineAccess = true
};
```

## 修改 MVC 客户端

对 MVC 客户端的修改同样也很少 - ASP.NET Core OpenID Connect 中间件是内置支持混合流的，所以我们只需要更改一些配置值。

我们配置 `ClientSecret` 以让它跟 IdentityServer 上的信息相匹配。添加 `offline_access` scopes，然后设置 `ResponseType` 为 `code id_token`（基本的意思就是“使用混合流”）

```CSharp
app.UseOpenIdConnectAuthentication(new OpenIdConnectOptions
{
    AuthenticationScheme = "oidc",
    SignInScheme = "Cookies",

    Authority = "http://localhost:5000",
    RequireHttpsMetadata = false,

    ClientId = "mvc",
    ClientSecret = "secret",

    ResponseType = "code id_token",
    Scope = { "api1", "offline_access" },

    GetClaimsFromUserInfoEndpoint = true,
    SaveTokens = true
});
```

当你运行 MVC 客户端的时候，不会有太大的区别。除此之外，授权确认页现在还会向你请求访问 额外的 API 和 离线访问（offline access） scope。

## 使用访问令牌

OpenID Connect 中间件会自动为你保存令牌（身份令牌，访问令牌和现在我们例子中的刷新令牌）。这就是 `SaveTokens` 设置的效果。

技术上，令牌是存储在 cookie 的属性片段之内的，访问它们最简单的方式是使用 `Microsoft.AspNetCore.Authentication` 名称空间下的扩展方法。

比如在你的身份信息视图上：

```CSharp
<dt>access token</dt>
<dd>@await ViewContext.HttpContext.Authentication.GetTokenAsync("access_token")</dd>

<dt>refresh token</dt>
<dd>@await ViewContext.HttpContext.Authentication.GetTokenAsync("refresh_token")</dd>
```

为了使用访问令牌访问 API，你所需要做的只是检索令牌，然后将其设置到你的 _`HttpClient`_ 中：

```CSharp
public async Task<IActionResult> CallApiUsingUserAccessToken()
{
    var accessToken = await HttpContext.Authentication.GetTokenAsync("access_token");

    var client = new HttpClient();
    client.SetBearerToken(accessToken);
    var content = await client.GetStringAsync("http://localhost:5001/identity");

    ViewBag.Json = JArray.Parse(content).ToString();
    return View("json");
}
```
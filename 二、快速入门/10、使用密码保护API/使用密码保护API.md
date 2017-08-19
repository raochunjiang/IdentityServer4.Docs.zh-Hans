# IdentityServer4 中文文档 -10- （快速入门）使用密码保护API

------------------------------------------------------------------------------------

**OAuth 2.0 资源所有者密码授权** 允许一个客户端发送用户名和密码到令牌服务并获得一个表示该用户访问令牌。

（OAuth 2.0） **规范** 建议仅对“受信任”的应用程序使用资源所有者密码授权。一般来说，当你想要验证一个用户并请求访问令牌的时候，使用交互式 OpenID Connect 流通常会更好。

不过，这个授权类型允许我们在 IdentityServer 快速入门中引入 **用户** 的概念，这是我们要展示它的原因。

## 添加用户

就像基于内存存储的资源（即 范围 Scopes）和客户端一样，对于用户也可以这样做。

> 注意：查看基于 ASP.NET Identity 的快速入门以获得更多关于如何正确存储和管理用户账户的信息。

`TestUser` 类型表示一个测试用户及其身份信息。让我们向配置类（如果你有严格按照顺序进行演练，那么配置类应该在 QuickstartIdentityServer 项目的 Config.cs 文件中）中添加以下代码以创建一对用户：

首先添加以下 using 语句 到 Config.cs 文件中：

```CSharp
public static List<TestUser> GetUsers()
{
    return new List<TestUser>()
    {
        new TestUser
        {
            SubjectId="1",
            Username="爱丽丝",
            Password="password"
        },
        new TestUser
        {
            SubjectId="2",
            Username="博德",
            Password="password"
        }
    };
}
```

然后将测试用户注册到 IdentityServer：

```CSharp
public void ConfigureServices(IServiceCollection services)
{
    // 使用内存存储，密钥，客户端和资源来配置身份服务器。
    services.AddIdentityServer()
        .AddTemporarySigningCredential()
        .AddInMemoryApiResources(Config.GetApiResources())
        .AddInMemoryClients(Config.GetClients())
        .AddTestUsers(Config.GetUsers());
}
```

`AddTestUsers` 扩展方法在背后做了以下几件事：

* 为资源所有者密码授权添加支持
* 添加对用户相关服务的支持，这服务通常为登录 UI 所使用（我们将在下一个快速入门中用到登录 UI）
* 为基于测试用户的身份信息服务添加支持（你将在下一个快速入门中学习更多与之相关的东西）

## 为资源所有者密码授权添加一个客户端定义

你可以通过修改 `AllowedGrantTypes` 属性简单地添加对已有客户端授权类型的支持。

通常你会想要为资源所有者用例创建独立的客户端，添加以下代码到你配置中的客户端定义中：

```CSharp
public static IEnumerable<Client> GetClients()
{
    return new List<Client>
    {
        // 省略其他客户端定义...

        // 资源所有者密码授权客户端定义
        new Client
        {
            ClientId = "ro.client",

            AllowedGrantTypes = GrantTypes.ResourceOwnerPassword,

            ClientSecrets =
            {
                new Secret("secret".Sha256())
            },
            AllowedScopes = { "api1" }
        }
    };
}
```

## 使用密码授权请求一个令牌

客户端看起来跟之前 **客户端凭证授权** 的客户端是相似的。主要差别在于现在的客户端将会以某种方式收集用户密码，然后在令牌请求期间发送到令牌服务。

IdentityModel 的 `TokenClient` 在这里再次为我们提了供帮助：

```CSharp
// 请求以获得令牌
var tokenClient = new TokenClient(disco.TokenEndpoint, "ro.client", "secret");
var tokenResponse = await tokenClient.RequestResourceOwnerPasswordAsync("爱丽丝","password","api1");
if (tokenResponse.IsError)
{
    Console.WriteLine(tokenResponse.Error);
    return;
}
Console.WriteLine(tokenResponse.Json);
Console.WriteLine("\n\n");
```

当你发送令牌到身份 API 端点的时候，你会发现与客户端凭证授权
相比，资源所有者密码授权有一个很小但很重要的区别。访问令牌现在将包含一个 `sub` 信息，该信息是用户的唯一标识。`sub` 信息可以在调用 API 后通过检查内容变量来被查看，并且也将被控制台应用程序显示到屏幕上。

`sub` 信息的存在（或缺失）使得 API 能够区分代表客户端的调用和代表用户的调用。
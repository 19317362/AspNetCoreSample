 # ASP.NET Core 2.0 and Ubisecure SSO integration with OpenID Connect

## Discovery and client registration

### openid-configuration.json

The OpenID Connect Provider metadata of your Ubisecure SSO Server. 

Provider metadata is published at a well known URI derived from the name of the Provider by concatenating **/.well-known/openid-configuration**. For example, if the name of your Ubisecure SSO Server is **https://sso.example.com/uas** then the URI for Provider metadata is

https://sso.example.com/uas/.well-known/openid-configuration

### client-config.json

The OpenID Connect Client metadata. 

This file is generated by Ubisecure SSO Server at client registration time. You may use Ubisecure SSO Management Console or Management API for client registration.

### Managemet API and PowerShell automation

Run `sso-register-client.ps1` to create **openid-configuration.json** and **client-config.json**

## Code review

Most code files are as generated by the Visual Studio 2017 ASP.NET Core wizard. The files modified for this integration are

* [Startup.cs](https://github.com/psteniusubi/AspNetCoreSample/blob/master/Startup.cs)
* [Views/Home/Index.cshtml](https://github.com/psteniusubi/AspNetCoreSample/blob/master/Views/Home/Index.cshtml)

### Startup.cs

The following code reads the json configuration files and makes the configuration available in ProviderConfig an ClientConfig properties.

```c#
        public Startup(IHostingEnvironment env)
        {
            ProviderConfig = new ConfigurationBuilder()
                .SetBasePath(env.ContentRootPath)
                .AddJsonFile("openid-configuration.json")
                .Build();
            ClientConfig = new ConfigurationBuilder()
                .SetBasePath(env.ContentRootPath)
                .AddJsonFile("client-config.json")
                .Build();
        }

        public IConfigurationRoot ProviderConfig { get; set; }
        public IConfigurationRoot ClientConfig { get; set; }
```

This indicates [OpenID Connect](https://docs.microsoft.com/en-us/aspnet/core/api/microsoft.aspnetcore.authentication.openidconnect) is used to authenticate new anonymous users trying to access the application. Cookies are used to persist an authenticated session. Do review details of [ASP.NET Core cookie authentication](https://docs.microsoft.com/en-us/aspnet/core/api/microsoft.aspnetcore.authentication.cookies) before going into production: how large will the cookie or cookies become and how is their integrity protected?

```c#
            .AddAuthentication(sharedOptions =>
            {
                sharedOptions.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
                sharedOptions.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
            })
```

Here I'm setting up the built-in OpenID Connect client of ASP.NET Core. This is all quite easy to understand. I want to use Authorization Code flow, I adjust some built-in features and fetch information from my configuration files.

One thing to note is that when configured this way the OpenID Connect client will use http to fetch fresh copies of Provider metadata and Provider public keys. You should carefully consider if this satisfies the trust and threat model of your application.

```c#
            .AddOpenIdConnect(options =>
            {
                options.ResponseType = OpenIdConnectResponseType.Code;
                options.ResponseMode = null;
                options.DisableTelemetry = true;
                options.Authority = ProviderConfig["issuer"];
                options.ClientId = ClientConfig["client_id"];
                options.ClientSecret = ClientConfig["client_secret"];
                options.Scope.Clear();
                options.Scope.Add("openid");
            })
            .AddCookie(); 

```

The final piece from Configure method disables anonymous access and requires all users of the application be authenticated.

```c#

            app.UseAuthentication();

```

### Index.cshtml

The following generates a simple html list showing all claims received from Ubisecure SSO Server

```cshtml
@{
}
<h1>Welcome</h1>
<dl>
    @foreach (var claim in User.Claims)
    {
        <dt><b>@claim.Type</b></dt>
        <dd><i>@claim.Value</i></dd>
    }
</dl>
```

## Launching

Use Visual Studio 2017 to launch AspNetCoreSample application on http://localhost:19282

## References

* https://docs.microsoft.com/en-us/aspnet/core/api/microsoft.aspnetcore.authentication.openidconnect
* https://docs.microsoft.com/en-us/aspnet/core/api/microsoft.aspnetcore.authentication.cookies
* http://openid.net/specs/openid-connect-discovery-1_0.html
* http://openid.net/specs/openid-connect-registration-1_0.html
* http://openid.net/specs/openid-connect-core-1_0.html

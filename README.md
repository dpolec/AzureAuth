# AzureAuth

1 configure app in ad
1 configure authorizatoin in Web API 
1 autorize from one API to another in delegated scoped


ADD SOME INTRODUCTION

Create Web API, remove example endpoint, and add for now only two endpoint to veryfied authorization:
  ```
var builder = WebApplication.CreateBuilder(args);

var app = builder.Build();

app.UseHttpsRedirection();

app.MapGet("/no-auth", () => Results.Ok());
app.MapGet("/auth", () => Results.Ok()).RequireAuthorization();

app.Run();
  ```
When we send request to `no-auth` we received 200, but when we send to `auth` we received 500

![image](https://user-images.githubusercontent.com/11536139/231489913-90380c0d-474d-45b9-a025-299c7cfe1af3.png)

This is fine, because we don't setup authorization. Now we need create new application on portal.azure.com. 
Open service `Azure Active Directory`, in `Manage` section find `APP registration`, click on New registration

  ![image](https://user-images.githubusercontent.com/11536139/231072183-e44da553-01e1-45a3-8362-dd5e393fa597.png)

After creation in `Manage` section select `Exponse an API`, and set `Application ID URI`

![image](https://user-images.githubusercontent.com/11536139/231495931-f1b6cca7-f5de-4b40-94d9-f144cd1d760e.png)

Add scope

![image](https://user-images.githubusercontent.com/11536139/232051946-f795f794-6cab-4127-ab7a-2170a0357c26.png)

Assign scope to application, from `Overview` copy `Application (client) ID`, and assign scope to application

![image](https://user-images.githubusercontent.com/11536139/232052344-d7bad098-d818-4100-94d9-3f534ede2e43.png)

This configuration for now will be enought to autorize request.
In Web API add package:
<li>Microsoft.AspNetCore.Authentication</li>
<li>Microsoft.AspNetCore.Authentication.JwtBearer</li>

For builder service we need configure middelware for authorization
```
builder.Services
    .AddAuthorization()
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        var tenantId = "31dc6402-0301-47e7-906b-c64a780a7311";
        var audience = new List<string> { "api://3245fc3f-917a-4926-a3b6-778ce877047b" };

        var authority = string.Format(CultureInfo.InvariantCulture, "https://login.microsoftonline.com/{0}", tenantId);

        var configManager = new Microsoft.IdentityModel.Protocols.ConfigurationManager<OpenIdConnectConfiguration>(
            $"{authority}/.well-known/openid-configuration",
            new OpenIdConnectConfigurationRetriever());

        var config = configManager.GetConfigurationAsync().GetAwaiter().GetResult();

        var tokenValidationParameters = new TokenValidationParameters
        {
            ValidAudiences = audience,
            ValidIssuers = new List<string>
            {
                $"https://login.microsoftonline.com/{tenantId}/",
                $"https://login.microsoftonline.com/{tenantId}/v2.0",
                $"https://login.windows.net/{tenantId}/",
                $"https://login.microsoft.com/{tenantId}/",
                $"https://sts.windows.net/{tenantId}/"
            },
            IssuerSigningKeys = config.SigningKeys
        };

        options.TokenValidationParameters = tokenValidationParameters;
    });
```

Now add middelware:
```
app.UseAuthorization();
app.UseAuthentication();
```

To veryfied this we need add some user to AD. In Azure Active Directoryt in `Manage` section select `Users`

![image](https://user-images.githubusercontent.com/11536139/232052953-7ac54784-e82e-4be8-b2ac-80e84819f1da.png)

What next, we need capability to create token, again go to our application and we need create client secret, in `Manage` section select `Certificates & secrets`, save this, because this value will be availble to read only after creation, we will need this to create token.

I will be used PostMan and I add how configure this, but you can used any others rest client, or create in application.
<details>
  <summary>PostMan configuration</summary>
  
  In `Authorization` tab chose OAuth 2.0 type
  
  ![image](https://user-images.githubusercontent.com/11536139/231504571-1ec0fa29-6546-46df-ab25-04680e38afc2.png)

  In `Configure New Token` fill all data
  1. `Token name`
  2. `Grant type` -> `Authorization code`
  3. `Callback url` -> this will be setup automaticly, but to worked we need add this url to our application
  
  In `Manage` section, select `Authentication` -> `Add a platform` -> `Web` -> past redirect url -> `Configure`

  ![image](https://user-images.githubusercontent.com/11536139/231506576-6ad69661-9594-451e-8ece-f8c587e97773.png)
  
  ![image](https://user-images.githubusercontent.com/11536139/231507189-80bf3f9c-f5f9-4074-a4c0-13703472ff16.png)
  
  4. `Auth URL` -> https://login.microsoftonline.com/{TenantId}/oauth2/v2.0/authorize
  5. `Access Token URL` -> https://login.microsoftonline.com/{TenantId}/oauth2/v2.0/token
  6. `Client ID` -> you will find this on overwiev in application
  
  ![image](https://user-images.githubusercontent.com/11536139/231508007-1f27d89f-4fa4-47c2-b680-e163f23d1afc.png)

  7. `Client Secret` -> generated in application (`Manage` -> `Certificates & secrets`)
  8. `Scope` -> generated in application (`Manage` -> `Expose an API`), and you need add `/.default` (eg. `api://3245fc3f-917a-4926-a3b6-778ce877047b/.default`)
  9. `State` -> can be empty
  10. `Client Authentication` -> `Send as Basic Auth header`
  
  ![image](https://user-images.githubusercontent.com/11536139/231509021-314107e9-66f7-4bfd-90e4-fe506687cdbf.png)

  Now we can generated token, we need login to AD using user credential
</details>

Ok, we have API, we have configured application and user in AD, and setup rest client, you can check this, all should works fine.

![image](https://user-images.githubusercontent.com/11536139/232058726-aecdad31-e7e7-4f91-b2c0-154ff1ab183c.png)

Great! Usually this is enought. Now I will show how we can authorize in delegated scope (user scoope) from one API to second, lets call them `Can auth API`. We have one API, we need add new one, we need add in the same way how we added first. In our rest client change configuration for new API, we need change client id and secret, leve scope, beacue we want create token to authorize in `Main API`. We still can't create token, now we need told in first API that this second can authorize, copy application id from `Can auth API` and go to `Main API`. In `Manage` section, select `Expose an API` and add again client with scoped.

![image](https://user-images.githubusercontent.com/11536139/232060565-cb82aa40-98a4-48e7-a5a3-a18a587cea6c.png)

Somtimes we need wait some time, when this will be propagate, and start works. When you will try now create token, all will be works fine, and request will be authorized.

![image](https://user-images.githubusercontent.com/11536139/232060933-b64f59a8-805b-4ee7-b70e-21f1f0f36f84.png)

Next case it will be authorize token created for application, this is very common approach when one API need get some data from other API. Create token for application is very easy, we need just send request as `POST` to url `https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token` and in body as form data:
```
client_id: 8b001c8d-c64b-42b5-bf37-d8e97a58110e
client_secret: MHY8Q~Kp3kUQ~L1Rp2zWY0LpvW6xVZrzip_D3cKe
grant_type: client_credentials
scope: api://3245fc3f-917a-4926-a3b6-778ce877047b/.default
```

In response we received access token, and we can use it in request to authorize, but for now we have one secure breach, to undersand this we need create again new app, lets call them `Can't auth API`. Now we can generete in the same way application token, and we can authorize in `Main API`, where `Main API` don't know anything about `Can't auth API` application. To manage this we have something like `App roles`, role can be dedicated to permission only for user or application, or all. In app roles we can ach of this role we can use to separete endpoint. We can create mutiple app roles, each role can be assign to different endpoint, that we can decide which api has access to which endpoint.

Ok, so lets do this, and check.

First create app role

![image](https://user-images.githubusercontent.com/11536139/234760378-1c10b6bc-80bc-4b7d-8727-164cd8226643.png)

I call `FirstRole`, for both approach

![image](https://user-images.githubusercontent.com/11536139/234760881-738da450-dcaa-4cac-b27b-4119061ea964.png)

Ww need add this role to application where we want. To do this we need go to `Can Auth API` to section `API permissions` and add this role.

![image](https://user-images.githubusercontent.com/11536139/234761293-24b12f65-1eed-41d8-82cd-4d81616a14bd.png)

![image](https://user-images.githubusercontent.com/11536139/234761366-29bdec0f-1e9b-4fd3-a674-976ab88641aa.png)

![image](https://user-images.githubusercontent.com/11536139/234761461-122221f8-8520-460c-9277-70ba6f36267e.png)

in status we will see that is `Not granted for ...` 

![image](https://user-images.githubusercontent.com/11536139/234761569-a6de0090-8c80-4214-89b1-14246e91c72a.png)

We can add permission, but we need approved for active directory administrator, now if you are this adm then you can do this, you will be decide wich endpoit can have this role. But if you are working in bigger comapny, I supposed that you have dedicated administraotr, and we need ask him/her to grant this. In app where we need grant this in `API permissions` we will have special option to do this

![image](https://user-images.githubusercontent.com/11536139/234762405-2b93a77d-1c66-4c1b-af16-14209a206b96.png)

Ok, now we need to handle the role we added in the code, I will added new one endpoint to do this, we need register authorization policy, and add this policy to endpoint.

```
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthorization(auth =>
    {
        auth.AddPolicy("role-type-policy", policy => policy.RequireRole("FirstRole"));
    })
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        ...
    });

var app = builder.Build();

app.UseHttpsRedirection();

app.MapGet("/no-auth", () => Results.Ok());
app.MapGet("/auth", () => Results.Ok()).RequireAuthorization();
app.MapGet("/first-role", () => Results.Ok()).RequireAuthorization().RequireAuthorization("role-type-policy");

app.UseAuthentication();
app.UseAuthorization();

app.Run();
```

--------------------------------------------

### ToDo 

1. add app role (add to 1 app)
1. example how auth request base on role in scope user
1. example how auth request in scope of app
1. how secure request in scope of app
1. maybe section prepare I will don't split?


### what we can do more
1. certificate
1. 



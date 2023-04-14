# AzureAuth

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


--------------------------------------------

### ToDo 
1. in app add scope and client app 

![image](https://user-images.githubusercontent.com/11536139/231510925-9193a3b2-9158-4862-b865-edcb2bf721d3.png)

3. example how auth request in scope user
4. azure portal - configure 2 more app (can auth, can't auth)
5. example how auth request in scope user
6. add app role (add to 1 app)
7. example how auth request base on role in scope user
8. example how auth request in scope of app
9. how secure request in scope of app
10. maybe section prepare I will don't split?

#### Done
1. azure portal - configure app 
1. azure portal - create user
1. create web api
1. add auth

### what we can do more
1. certificate
1. 



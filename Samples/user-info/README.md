# Extracting a user's information from a JWT using ASP.NET Core Web API

This example demonstrates how you can retrieve a user's information from inside an ASP.NET Core controller. 

1. As a general rule, all the claims on the JWT which was passed in will be available in the Claims of the User object:

```csharp
public object Claims()
{
    return User.Claims.Select(c => new
    {
        Type = c.Type,
        Value = c.Value
    });
}
```

2. To get the unique identifer of the user in Auth0 you can get the value of the `NameIdentifier` claim. This is the value of the `sub` claim of the JWT which was passed in. You can use this to store a reference to the user in your own database.

```csharp
public object UserId()
{
    // The user's ID is available in the NameIdentifier claim
    string userId = User.Claims.FirstOrDefault(c => c.Type == ClaimTypes.NameIdentifier).Value;

    return new
    {
        UserId = userId
    };
}
```

3. Finally, if you need the full user profile, you can use the Auth0.NET SDK to call the `/userinfo` endpoint. First ensure that you have the Auth0.NET SDK installed:

```text
Install-Package Auth0.AuthenticationApi
```

Then, you need to ensure that you save the actual value of the JWT in a claim. You can do this in the `OnTokenValidated` event when registering the JWT middleware:

```csharp
var options = new JwtBearerOptions
{
    Audience = Configuration["auth0:apiIdentifier"],
    Authority = $"https://{Configuration["auth0:domain"]}/",
    Events = new JwtBearerEvents
    {
        OnTokenValidated = context =>
        {
            if (context.SecurityToken is JwtSecurityToken token)
            {
                if (context.Ticket.Principal.Identity is ClaimsIdentity identity)
                {
                    identity.AddClaim(new Claim("access_token", token.RawData));
                }
            }

            return Task.FromResult(0);
        }
    }
};
app.UseJwtBearerAuthentication(options);
```

Finally, inside your controller you can retreive the value of the access token from the claim, and then use that to call the `GetUserInfoAsync` method of the `AuthenticationApiClient`:

```csharp
public async Task<object> UserInformation()
{
    // Retrieve the access_token claim which we saved in the OnTokenValidated event
    string accessToken = User.Claims.FirstOrDefault(c => c.Type == "access_token").Value;

    // If we have an access_token, then retrieve the user's information
    if (!string.IsNullOrEmpty(accessToken))
    {
        var apiClient = new AuthenticationApiClient(_configuration["auth0:domain"]);
        var userInfo = await apiClient.GetUserInfoAsync(accessToken);

        return userInfo;
    }

    return null;
}
```

> **NB**: Retrieving the full user profile should be done in rare circumstances. You should definitely not make a call to the `/userinfo` endpoint every time a user calls one of your API endpoints, as you will be rate limited by Auth0. It is therefore suggested that you only retrieve the actual User Id as demonstrated in (2) above, and use that as a foreign key in your own database.

## To run this application:

1. This example uses RS256 JSON Web Tokens. Go to the Application in your Auth0 Dashboard and go to Settings > Advanced Settings > OAuth and ensure the **JsonWebToken Signature Algorithm** is set to **RS256**.
2. Ensure you have set the Auth0 settings in your `appsettings.json` file

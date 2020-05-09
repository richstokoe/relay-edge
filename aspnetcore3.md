## Building a Relay Edge client in [ASP.NET Core 3](https://asp.net)


Follow the steps from the official Microsoft docs [here](https://docs.microsoft.com/en-us/aspnet/core/getting-started/) to create a new ASP.NET Core web app.

Open the Startup.cs file in the root of your project folder.

## Add authentication

Add the following to the `ConfigureServices()` method:

```C#
services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
	.AddCookie(CookieAuthenticationDefaults.AuthenticationScheme, options =>
	{
		options.SlidingExpiration = true;

		// Set this to however long or short you wish your session to last
		options.ExpireTimeSpan = TimeSpan.FromDays(90);
	})
	.AddOpenIdConnect(OpenIdConnectDefaults.AuthenticationScheme, options =>
	{
		options.SignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
		options.Authority = "https://edgecastle.com";

		// This is required to allow the device to register itself.
		options.Scope.Add("create-devices");

		// This is required to allow the device to 'ping' the cloud service
		// even when you are not logged in
		options.Scope.Add("offline_access"); 
		options.ClientId = "relay-client";
		options.CallbackPath = "https://raspberrypi/auth/callback"; // set this to your callback path
		options.SignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
		options.AuthenticationMethod = Microsoft.AspNetCore.Authentication.OpenIdConnect.OpenIdConnectRedirectBehavior.FormPost;
		options.ResponseType = "code id_token";

		// This give us access to scoped services.
		options.EventsType.TokenResponseReceived = (context) => 
		{		
			/* Verify the incoming access and refresh tokens from the cloud service once
				the user has logged into their identity provider.
				You should use the "sid" claim to lock the device to only accept
				requests and commands from you.
			*/
			
			// Load the sid from your choice of data store:
			var existingSid = <get sid>;// Load the SID that logged in first, if any

			// Gets the Edgecastle User ID of the user attempting to login
			var sid = context.Principal.FindFirst("sid")?.Value;
			
			if(existingSid == default(string)
			{
				// Nobody's logged in before. Store the SID now
			}
			else if(sid != existingSid)
            {
				// Someone other than the owner is trying log in. Prevent this here:
                context.Fail(throw new System.Security.SecurityException($"User with sid {sid} does not match device owner sid."));
				
				// TODO: Log this?
            }
            else
            {				
                // The owner is logging in. Log this?
				// Continue
            }
			
			// Now we know the user logging in is allowed, save the latest access and refresh tokens

			var accessToken = context.TokenEndpointResponse.AccessToken;
			var refreshToken = context.TokenEndpointResponse.RefreshToken;

			if(string.IsNullOrWhiteSpace(refreshToken))
			{
				// Decide how you want to handle the fact that the offline_access scope was not allowed
				// You could fail the login attempt (context.Fail()).
			}

			// Save the accessToken and refreshToken so the device can use them for requests to the 
			// cloud service (such as heartbeat)
		};
	})
	.AddJwtBearer(options =>
	{
		// This configuration will be used to authenticate inbound requests 
		// from the Relay Cloud service

		options.Authority = "https://edgecastle.com";

		// The audience is set to the scope to allow 
		// more flexible client network scenarios.
		options.TokenValidationParameters.ValidateAudience = false;
	});
```  

Also add the following to the `ConfigureServices()` method to make it easier to secure cloud endpoints

```C#
services.AddAuthorization(config => 
{
	// Allow pings from cloud service to this device
	config.AddPolicy("pingdevice", policy =>
	{
		policy.AddAuthenticationSchemes(JwtBearerDefaults.AuthenticationScheme)
			.RequireClaim("scope", new [] { "pingdevice" });
	});

	// Allow cloud services to read disk information
	config.AddPolicy("read-disksinfo", policy =>
	{
		policy.AddAuthenticationSchemes(JwtBearerDefaults.AuthenticationScheme)
			.RequireClaim("scope", new [] { "read-disksinfo });
	});

	// Allow videostream access from cloud
	config.AddPolicy("read-videostream", policy =>
	{
		policy.AddAuthenticationSchemes(JwtBearerDefaults.AuthenticationScheme)
			.RequireClaim("scope", new [] { "read-videostream" });
	});
```

In the `Configure()` method in the Startup.cs file, add the following between 



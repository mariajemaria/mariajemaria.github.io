---
title: "Redirecting to different domains after Vipps login in an Optimizely multisite scenario"
date: 2023-03-10
---

Our customer wanted to have a landing page that logs the user in through Vipps and redirects them to a domain they choose before proceeding to Vipps login. This blogpost describes a solution we built to support this request.

![Login to a selected domain](/assets/images/login-choice.png)

## Intro

We are using [Vipps.Login.Episerver](https://nuget.optimizely.com/package/?id=Vipps.Login.Episerver) and [Vipps.Login.Episerver.Commerce](https://nuget.optimizely.com/package/?id=Vipps.Login.Episerver.Commerce) in our multisite environment to support login through Vipps. These packages are meant to simplify Optimizely configuration and setup with Owin.

More information about can be found in [Vipps github docs here](https://github.com/vippsas/vipps-login-api).

## Request

Additional request from the customer was to:
1. Create a landing page on the main domain
2. Present the user with a choice of domains they can be logged in to
3. Save all of their choices
4. Log the user in based on the first domain of their choice 
5. Forward to the chosen domain

## Solution

Sessions are of course based on the domain, so what we basically needed to do is:
1. Prior to redirecting to Vipps, we needed to redirect to the first chosen domain
2. There, we would save all of their choices to a cookie
3. Proceed with the usual login on that domain

To support that, we needed the following code:

```csharp
        [HttpGet]
        [Route("vippslanding")]
        public IHttpActionResult VippsLanding(string domains)
        {
            if (!ModelState.IsValid)
            {
                var message = string.Join(", ", ModelState.Values
                    .SelectMany(v => v.Errors)
                    .Select(e => e.ErrorMessage));

                return _errorFactory.CreateApiErrorResult(
                    HttpStatusCode.BadRequest,
                    message, this);
            }

            if (!string.IsNullOrWhiteSpace(domains))
            {
                _domainCookieHandler.SetDomains(domains);
            }

            var vippsLoginUrl =
                $"{HttpContext.Current.Request.Url.GetLeftPart(UriPartial.Authority)}/{VippsLoginConstants.LoginUrl}";

            return Redirect(vippsLoginUrl);
        }
```

**Pls note that this has to be an HTTP GET request!** CORS setup did not play out well for us with the combination of WebAPI and axios calls in frontend. In the end, using GET, we could remove all the CORS related setup.

## Cancelling Vipps login

Startup class would need to include handling the redirect URL in case the user cancels the login due to having multiple login pages - a standard login page and the abovementioned landing page.

```csharp
app.UseOpenIdConnectAuthentication(new VippsOpenIdConnectAuthenticationOptions(
                VippsLoginConfig.ClientId,
                VippsLoginConfig.ClientSecret,
                VippsLoginConfig.Authority)
            {
                // 1. Here you pass in the scopes you need
                Scope = string.Join(" ", new[]
                {
                    VippsScopes.ApiV2,
                    VippsScopes.OpenId,
                    VippsScopes.Email,
                    VippsScopes.Name,
                    VippsScopes.BirthDate,
                    VippsScopes.Address,
                    VippsScopes.PhoneNumber
                }),
                // Various notifications that we can handle during the auth flow
                // By default it will handle:
                // RedirectToIdentityProvider - Redirecting to Vipps using correct RedirectUri
                // AuthorizationCodeReceived - Exchange Authentication code for id_token and access_token
                // DefaultSecurityTokenValidated - Find matching CustomerContact
                Notifications = new VippsEpiNotifications
                {
                    AuthenticationFailed = context =>
                    {
                        var errorKey = "VippsSomethingWentWrong";
                        switch (context.Exception)
                        {
                            case VippsLoginDuplicateAccountException _:
                                errorKey = nameof(VippsLoginDuplicateAccountException);
                                break;
                            case VippsLoginSanityCheckException _:
                                errorKey = nameof(VippsLoginSanityCheckException);
                                break;
                            case VippsLoginLinkAccountException accountException:
                                if (accountException.UserError)
                                {
                                    errorKey = nameof(VippsLoginLinkAccountException);
                                }
                                break;
                        }

                        var domainCookieHandler = ServiceLocator.Current.GetInstance<IDomainCookieHandler>();
                        var isFromVippsLanding = domainCookieHandler.PopVippsLandingCookie(context.OwinContext);

                        var vippsLandingPageUrl = siteSettings.VippsLandingPage.GetFriendlyUrl(includeHost: true);
                        var redirectUrl = isFromVippsLanding ? vippsLandingPageUrl : loginPageUrl;

                        // 2. Redirect to login or error page and display message
                        context.HandleResponse();
                        context.Response.Redirect($"{redirectUrl}?errorKey={errorKey}");
                        return Task.FromResult(0);
                    }
                }
            });
```

## Failure when logging with Vipps

Similar code would need to be added in case of an error. I am including only the interesting parts here:

```csharp
// Trigger Vipps middleware on this path to start authentication
            app.Map($"/{VippsLoginConstants.LoginUrl}", map => map.Run(async ctx =>
            {
                var service = ServiceLocator.Current.GetInstance<IVippsLoginCommerceService>();

                var isAuthenticated = ctx.Authentication.User?.Identity?.IsAuthenticated ?? false;
                if (!isAuthenticated)
                {
                    // Regular log in
                    ctx.Authentication.Challenge(VippsAuthenticationDefaults.AuthenticationType);
                    return;
                }
                
                var registrationService = ServiceLocator.Current.GetInstance<IRegistrationService>();
                
                var success = await registrationService.RegisterOrLoginVippsUser(ctx.Authentication.User.Identity);

                if (success)
                {
                    // 3. Vipps log in and sync Vipps user info
                    if (service.HandleLogin(ctx, new VippsSyncOptions
                    {
                        SyncContactInfo = false,
                        SyncAddresses = false
                    }))
                        return;

                    // Link Vipps account to current logged in user account
                    bool.TryParse(ctx.Request.Query.Get("LinkAccount"), out var linkAccount);
                    if (linkAccount && service.HandleLinkAccount(ctx))
                        return;

                    // Return to this url after authenticating
                    var returnUrl = ctx.Request.Query.Get("ReturnUrl") ?? "/";
                    service.HandleRedirect(ctx, returnUrl);
                }
                else
                {
                    ctx.Authentication.SignOut(
                        VippsAuthenticationDefaults.AuthenticationType,
                        CookieAuthenticationDefaults.AuthenticationType);

                    var domainCookieHandler = ServiceLocator.Current.GetInstance<IDomainCookieHandler>();
                    var isFromVippsLanding = domainCookieHandler.PopVippsLandingCookie(ctx);

                    var vippsLandingPageUrl = siteSettings.VippsLandingPage.GetFriendlyUrl(includeHost: true);
                    var redirectUrl = isFromVippsLanding ? vippsLandingPageUrl : loginPageUrl;

                    ctx.Response.Redirect($"{redirectUrl}?errorKey={nameof(LoginViewResources.VippsUserAlreadyTaken)}");
                }

                return;
            }));
```

Hope this helps someone, especially trying to get this to work with an HTTP POST request!

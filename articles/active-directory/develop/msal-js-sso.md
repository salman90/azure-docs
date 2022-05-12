---
title: Single sign-on (MSAL.js) | Azure
titleSuffix: Microsoft identity platform
description: Learn about building single sign-on experiences using the Microsoft Authentication Library for JavaScript (MSAL.js).
services: active-directory
author: mmacy
manager: CelesteDG

ms.service: active-directory
ms.subservice: develop
ms.topic: conceptual
ms.workload: identity
ms.date: 10/25/2021
ms.author: marsma
ms.reviewer: saeeda
ms.custom: aaddev, has-adal-ref
#Customer intent: As an application developer, I want to learn about enabling single sign on experiences with MSAL.js library so I can decide if this platform meets my application development needs and requirements.
---

# Single sign-on with MSAL.js

Single sign-on (SSO) provides a more seamless experience by reducing the number of times your users are asked for their credentials. Users enter their credentials once, and the established session can be reused by other applications on the device without further prompting. 

Azure Active Directory (Azure AD) enables SSO by setting a session cookie when a user first authenticates. MSAL.js allows use of the session cookie for SSO between the browser tabs opened for one or several applications.

## SSO between browser tabs

When a user has your application open in several tabs and signs in on one of them, they're signed into the same app open on the other tabs without being prompted. MSAL.js caches the ID token for the user in the browser `localStorage` and will sign the user in to the application on the other open tabs.

By default, MSAL.js uses `sessionStorage`, which doesn't allow the session to be shared between tabs. To share authentication state between tabs, set the cacheLocation in MSAL.js to localStorage as shown below.

```javascript
const config = {
  auth: {
    clientId: "abcd-ef12-gh34-ikkl-ashdjhlhsdg",
  },
  cache: {
    cacheLocation: "localStorage",
  },
};

const msalInstance = new msal.PublicClientApplication(config);
```

## SSO between apps

When a user authenticates, a session cookie is set on the Azure AD domain in the browser. MSAL.js relies on this session cookie to provide SSO for the user between different applications. MSAL.js also caches the ID tokens and access tokens of the user in the browser storage per application domain.

MSAL.js relies on `ssoSilent` API to authorize the user without an interaction. As a result, the SSO behavior varies for different cases:

### With User Hint

If you already have the user's sign-in information, you can pass this into the `ssoSilent`API to improve performance and ensure that the authorization server will look for the correct account session. You can pass one of the following into the request object to successfully obtain the token silently.

- Session ID `sid` (which can be retrieved from `idTokenClaims` of an `account` object)
- `login_hint` (which can be retrieved from the `account` object userame property or the `upn` claim in the ID token) 
- `account` (which can be retrieved from using one the [account APIs](https://github.com/AzureAD/microsoft-authentication-library-for-js/edit/dev/lib/msal-browser/docs/login-user.md#account-apis))

**Using a session ID**

To use a SID, add `sid` as an [optional claim](active-directory-optional-claims.md) to your app's ID tokens. The `sid` claim allows an application to identify a user's Azure AD session independent of their account name or username. To learn how to add optional claims like `sid`, see [Provide optional claims to your app](active-directory-optional-claims.md). Use the session ID (SID) in silent authentication requests you make with `ssoSilent` in MSAL.js.

```javascript
const request = {
  scopes: ["user.read"],
  sid: sid,
};

try {
    const loginResponse = await msalInstance.ssoSilent(request);
} catch (err) {
    if (err instanceof InteractionRequiredAuthError) {
        const loginResponse = await msalInstance.loginPopup(request).catch(error => {
            // handle error
        });
    } else {
        // handle error
    }
}
```

**Using a login hint**

To bypass the account selection prompt typically shown during interactive authentication requests (or for silent requests when you haven't configured the sid optional claim), provide a loginHint. In multi-tenant applications, also include a domain_hint.

```javascript
const request = {
  scopes: ["user.read"],
  loginHint: preferred_username,
  extraQueryParameters: { domain_hint: "organizations" },
};

try {
    const loginResponse = await msalInstance.ssoSilent(request);
} catch (err) {
    if (err instanceof InteractionRequiredAuthError) {
        const loginResponse = await msalInstance.loginPopup(request).catch(error => {
            // handle error
        });
    } else {
        // handle error
    }
}
```
Get the values for `loginHint` and `domain_hint` from the user's **ID token**:

- `loginHint`: Use the ID token's `preferred_username` claim value.

- `domain_hint`: Use the ID token's `tid` claim value. Required in requests made by multi-tenant applications that use the */common* authority. Optional for other applications.

For more information about login hint and domain hint, see [Microsoft identity platform and OAuth 2.0 authorization code flow](v2-oauth2-auth-code-flow.md).

**Using the account APIs**

If you know the account information, you can also retrieve the user account by using the `getAccountByUsername()` or `getAccountByHomeId()` APIs:

```javascript
const username = "test@contoso.com";
const myAccount  = msalInstance.getAccountByUsername(username);
const request = {
    scopes: ["User.Read"],
    account: myAccount
};

try {
    const loginResponse = await msalInstance.ssoSilent(request);
} catch (err) {
    if (err instanceof InteractionRequiredAuthError) {
        const loginResponse = await msalInstance.loginPopup(request).catch(error => {
            // handle error
        });
    } else {
        // handle error
    }
}
```

### Without User Hint

If there is not enough information available about the user, you can attempt to use the `ssoSilent` API without passing an account, `sid` or `login_hint`.

```javascript
const request = {
    scopes: ["User.Read"]
};

try {
    const loginResponse = await msalInstance.ssoSilent(request);
} catch (err) {
    if (err instanceof InteractionRequiredAuthError) {
        const loginResponse = await msalInstance.loginPopup(request).catch(error => {
            // handle error
        });
    } else {
        // handle error
    }
}
```

However, be aware that if your application has code paths for multiple users in a single browser session, or if the user has multiple accounts for that single browser session, then there is a higher likelihood of silent sign-in errors. You may see the following error show up in the event of multiple account sessions found by the authorization server:

```txt
InteractionRequiredAuthError: interaction_required: AADSTS16000: Either multiple user identities are available for the current request or selected account is not supported for the scenario.
```

This indicates that the server could not determine which account to sign into, and will require either one of the parameters above (`account`, `login_hint`, `sid`) or an interactive sign-in to choose the account.

## Considerations when using `ssoSilent`

**RedirectUri Considerations**

When using popup and silent APIs we recommend setting the `redirectUri` to a blank page or a page that does not implement MSAL. This will help prevent potential issues as well as improve performance. If your application is only using popup and silent APIs you can set this on the `PublicClientApplication` config. If your application also needs to support redirect APIs you can set the `redirectUri` on a per request basis:

**3rd party cookies**

`ssoSilent` attempts to open a hidden iframe and reuse an existing session with AAD. This will not work in browsers that block 3rd party cookies such as Safari. Additionally, the request object is required when using the "Silent" type. If you already have the user's sign-in information, you can pass either the `loginHint` or `sid` optional parameters to sign-in a specific account. 


## SSO in ADAL.js to MSAL.js update

MSAL.js brings feature parity with ADAL.js for Azure AD authentication scenarios. To make the migration from ADAL.js to MSAL.js easy and to avoid prompting your users to sign in again, the library reads the ID token representing user’s session in ADAL.js cache, and seamlessly signs in the user in MSAL.js.

To take advantage of the SSO behavior when updating from ADAL.js, you'll need to ensure the libraries are using `localStorage` for caching tokens. Set the `cacheLocation` to `localStorage` in both the MSAL.js and ADAL.js configuration at initialization as follows:

```javascript

// In ADAL.js
window.config = {
  clientId: "g075edef-0efa-453b-997b-de1337c29185",
  cacheLocation: "localStorage",
};

var authContext = new AuthenticationContext(config);

// In latest MSAL.js version
const config = {
  auth: {
    clientId: "abcd-ef12-gh34-ikkl-ashdjhlhsdg",
  },
  cache: {
    cacheLocation: "localStorage",
  },
};

const msalInstance = new msal.PublicClientApplication(config);
```

Once the `cacheLocation` is configured, MSAL.js can read the cached state of the authenticated user in ADAL.js and use that to provide SSO in MSAL.js.

## Next steps

For more information about SSO, see:

- [HybridSample](https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/samples/msal-browser-samples/HybridSample)
- [A React single-page application using MSAL React to authorize users for calling a protected web API on Azure Active Directory](https://github.com/Azure-Samples/ms-identity-javascript-react-tutorial/tree/main/3-Authorizatio)
- [Configurable token lifetimes](active-directory-configurable-token-lifetimes.md)
- [Single Sign-On SAML protocol](single-sign-on-saml-protocol.md)

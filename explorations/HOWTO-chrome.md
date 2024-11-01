# HOWTO

This article explains how to try out FedCM in Google Chrome.

We will do our best to keep these instructions up to date as protocol and API
changes occur. It will subsequently roll into downstream channels, but with lag
in functionality and bug fixes since this is under development.

## Available functionality

As of December 2022, FedCM API is available by default on Chrome (versions 108+).
See [here](https://developer.chrome.com/docs/privacy-sandbox/fedcm/#use-api) for
a detailed guide on how to use the parts of the API that have been shipped!

## Testing the API

Testing can be easier in Incognito mode, Guest mode or with single-use profiles
if sign-up status persistence is not desired.

At the moment the only way to reset the sign-up status is to clear browsing data on the Chrome profile.
This can be done from the [ClearBrowsing Data (chrome://settings/clearBrowserData)] [Settings] dialog.
Under [Advanced] select the [Site Settings] checkbox. Also be sure the time range of data being cleared
includes the time when the sign-up status was set.

## Experimental functionality

To test experimental functionality:

1. Download Google Chrome Canary. It is best to experiment with the latest
   build possible to get the most up-to-date implementation.
2. FedCM is blocked if third party cookies are blocked. Ensure the Chrome
   version you're using is not blocking third party cookies by navigating to
   `chrome://settings/cookies`.
3. Enable your experimental FedCM feature. This can be done directly from
   `chrome://flags` and searching 'fedcm' to see all available features.

The list of experimental features can be found [here](/proposals/README.md).

### Logout

We have been experimenting with methods for helping session management
features continue to work that currently rely on third-party cookies. So far the
only implemented proposal is an API for Logout.

The Logout API, `IdentityCredential.logoutRPs()` which is being explored as a way
to preserve OIDC front-channel logout and SAML Single Signout with loss of
access to third-party cookies in embedded contexts. It is intended to replace
situations where an IDP logging out a user also must log out the user in RP
contexts and would normally do it using iframes with each RP's known logout URL.

The API takes an array of URLs as an argument. For each URL, the browser
determines if the user is known to have previously logged in to the RP using
that IDP, and if it has, it sends a credentialed GET request to that URL.

```js
IdentityCredential.logoutRPs([{
  url: "https://rp1.example/logout",
  accountId: "123",
}, {
  url: "https://rp2.example/logout",
  accountId: "456",
}]);
```

For security reasons, the IDP does not learn whether any of the network requests
succeeded or failed.

### LoginHint

To use the LoginHint API:

* Add an array of `hints` to the accounts described in the accounts endpoint:

```
{
  accounts: [{
    id: "accountId",
    email: "account@email.com",
    hints: ["hint", "otherHint"],
    ...
  }, ...]
}
```

* Invoke the API with the `loginHint` parameter like so:

```js
  return await navigator.credentials.get({
      identity: {
        providers: [{
          configURL: "https://idp.example/config.json",
          clientId: "123",
          nonce: nonce,
          loginHint : "hint"
        }]
      }
  });
```

Now, only accounts with the "hint" provided will show in the chooser.

### UserInfo

To use the UserInfo API:

* The RP must embed an IDP iframe, which will perform the query.
* The embedded iframe must receive permissions to invoke FedCM (via Permissions Policy).
* The user first needs to go through the FedCM flow once before invoking UserInfo.
* In a subsequent site visit, the IDP iframe may invoke UserInfo:

```js
const user_info = await IdentityProvider.getUserInfo({
    configUrl: "https://idp.example/config.json",
    clientId: "client1234"
});

user_info.forEach( info => {
  // It's up to the IDP regarding how to display the returned accounts.
  // Accounts are sorted based on RP registration status.
  const name = info.name;
  const given_name = info.given_name;
  const picture = info.picture;
  const email = info.email;
}
```

### RP Context

To use the RP Context API:

* Provide the `context` value in JS, like so:

```js
const {token} = await navigator.credentials.get({
  identity: {
    context: "signup", 
    providers: [{
          configURL: "https://idp.example/fedcm.json",
          clientId: "1234",
    }],
  }
});
```

Now, the browser UI will be different based on the value provided.

### IdP Sign-in Status API

To use the IdP Sign-in Status API:

1. Enable the experimental feature `FedCM with FedCM IDP sign-in status` in `chrome://flags`.
2. When the user logs-in to the IdP, use the following HTTP header `IdP-SignIn-Status: action=signin`.
3. When the user logs-out of all of their accounts in the IdP, use the following HTTP header `IdP-SignIn-Status: action=signed-out`.
4. Add a `signin_url": "/idp_login.html` property to the `configURL` configuration. 
5. The browser is going load the `signin_url` when the user is signed-out of the IdP.
6. Call `IdentityProvider.close()` when the user is done logging-in to the IdP.

### Error API

To use the Error API:

* Enable the experimental feature `FedCmError` in `chrome://flags`.
* Provide an `error` in the ID assertion endpoint instead of a `token`:
```
{
  "error" : {
     "code" : "access_denied",
     "url" : "https://idp.example/error?type=foo"
  }
}
```
Note that the `error` field in the response including both `code` and `url` is
optional. As long as the flag is enabled, Chrome will render an error UI when
the token request fails. The `error` field is used to customize the flow when an
error happens. Chrome will show a customized UI with proper error message if the
code is "invalid_request", "unauthorized_client", "access_denied", "server_error",
or "temporarily_unavailable". If a `url` field is provided and same-site with
the IdP's `configURL`, Chrome will add an affordance for users to open a new
page (e.g., via pop-up window) with that URL to learn more about the error on
that page.

### Auto-selected Flag API

To use the Auto-selected Flag API:
* Enable the experimental feature `FedCmAutoSelectedFlag` in `chrome://flags`.

The browser will send a new boolean to represent whether auto re-authentication
was triggered such that the account was auto selected by the browser in the flow
to both the IdP and the API caller.

For IdP, the browser will include `is_auto_selected` in the request sent to the
ID assersion endpoint:
```
POST /fedcm_assertion_endpoint HTTP/1.1
Host: idp.example
Origin: https://rp.example/
Content-Type: application/x-www-form-urlencoded
Cookie: 0x23223
Sec-Fetch-Dest: webidentity

account_id=123&client_id=client1234&nonce=Ct60bD&disclosure_text_shown=true&is_auto_selected=true
```

For the API caller, the browser will include a boolean when resolving the
promise:
```
const cred = await navigator.credentials.get({
  identity: {
    providers: [{
      configURL: "https://idp.example/manifest.json",
      clientId: "1234"
    }]
  }
});

const token = cred.token;
if (cred.isAutoSelected !== undefined) {
  const isAutoSelected = cred.isAutoSelected;
}
```

### DomainHint

To use the DomainHint:
* Ensure that chrome://version shows 121.0.6146.0 or higher.
* Enable the experimental feature `FedCmDomainHint` in `chrome://flags`.

* Add an array of `domain_hints` to the accounts described in the accounts endpoint:

```
{
  accounts: [{
    id: "karenCorp1",
    email: "karen@corp1.com",
    name: "Karen",
    domain_hints: ["corp1", "corp2"],
  }, {
    id: "otherId",
    email: "karen@mail.com",
    name: "Karen",
  }, {
    id: "karenCorp3",
    email: "karen@corp3.com,
    name: "Karen",
    domain_hints: ["corp3"],
  }
  }, ...]
}
```

* Invoke the API with the `domainHint` parameter like so:

```js
  // This will show the karenCorp1 account.
  return await navigator.credentials.get({
      identity: {
        providers: [{
          configURL: "https://idp.example/config.json",
          clientId: "123",
          nonce: nonce,
          domainHint : "corp1"
        }]
      }
  });
```

Now, only accounts matching the hint provided will show in the chooser.

You may also use "any" to show only accounts which list at least one domain hint.

```js
  // This will show the karenCorp1 and karenCorp3 accounts.
  return await navigator.credentials.get({
      identity: {
        providers: [{
          configURL: "https://idp.example/config.json",
          clientId: "123",
          nonce: nonce,
          domainHint : "any"
        }]
      }
  });
```

### Disconnect API

To use the Disconnect API:
* Ensure that chrome://version shows 121.0.6145.0 or higher.
* Enable the experimental feature `FedCmDisconnect` in `chrome://flags`.

Add a disconnect endpoint to the FedCM config file. It must be same-origin
with the config file.

```json
{
  "accounts_endpoint": "/accounts",
  "id_assertion_endpoint": "/assertion",
...
  "disconnect_endpoint": "/disconnect"
}
```

Implement the `disconnect_endpoint`. It is fetched using a POST credentialed
request with CORS mode:

```
POST /disconnect HTTP/1.1
Host: idp.example
Referer: rp.example
Content-Type: application/x-www-form-urlencoded
Cookie: 0x123
Sec-Fetch-Dest: webidentity

account_hint=account456&client_id=rp123
```

The IdP can then disconnect the account and respond with the
`Access-Control-Allow-Origin` and `Access-Control-Allow-Credentials` headers to
satisfy CORS, and in the body of the response it should include a JSON with the
account ID of the account that has been disconnected.

```json
{
  "account_id": "account456Id"
}
```

When a user goes through the FedCM flow, the browser stores that (RP, IDP, account)
knowledge in browser storage in order to allow auto reauthn and User Info API to
work correctly in the future. If the browser finds an account in local storage
matching the ID provided, it will note the disconnection of that account. If the
IdP fails by either returning some network error or saying that the disconnection
was unsuccessful, or if the `account_id` is nowhere to be found, the browser will
remove from local storage all of the federated accounts associated with the (RP,
IDP).

### Button Flow
The button flow differs from the widget flow in several ways. The most significant
difference is that the button flow requires a user gesture such as clicking on a
sign-in button. This means that a user must be able to successfully sign in with a
federated account using this flow. In contrast, the widget flow is an optimized
flow that can reduce sign-in friction. This means that if the widget flow is
unavailable, a user can still click a "Sign in with IdP" button to continue. See
illustrative [mocks here](https://docs.google.com/presentation/d/1iURrPakaHgBfQ6mAefKijjxToiTTgBSPz1rtaV0od98/edit?usp=sharing).

#### Button Mode API
To use the Button Mode API:
* Enable the experimental feature `FedCmButtonMode` in `chrome://flags`.
* Make sure to invoke the API behind [transient user activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation).
* Invoke the API with the `mode` parameter like so:
  ```js
    return await navigator.credentials.get({
        identity: {
          providers: [{
            configURL: "https://idp.example/config.json",
            clientId: "123",
            nonce: nonce
          }],
          mode: "button"
        }
    });
  ```

The browser will send a new parameter to the IdP representing the request type by including
`mode=button` in the request sent to the ID assersion endpoint:
```
POST /fedcm_assertion_endpoint HTTP/1.1
Host: idp.example
Origin: https://rp.example/
Content-Type: application/x-www-form-urlencoded
Cookie: 0x23223
Sec-Fetch-Dest: webidentity

account_id=123&client_id=client1234&nonce=Ct60bD&disclosure_text_shown=true&is_auto_selected=false
&mode=button
```
Note that we currently only include the new parameter in the button flow. Other flow
types such as "widget" (name TBD) will be added when Chrome ships this feature.

#### Use Other Account API
This API allows users to use other accounts in the account chooser when, for example, IdPs
support multiple accounts or replacing the existing account.

To use the Use Other Account API:
* Enable the experimental feature `FedCmUseOtherAccount` in `chrome://flags`.
* IdP specifies the following in the FedCM config file:
```
{
  "accounts_endpoint" : ...,
  "modes: {
    "button": {
      "supports_use_other_account": true|false,
    }
  }
}
```

### Continuation API
This API lets the IdP request that the authorization flow should continue
in a popup window that is controlled by the IdP. This can be used to request
additional permission, to ask a user to confirm their account details, or
for a variety of other use cases.

To use this feature:
* Enable the experimental feature `FedCmAuthz` in chrome://flags
* Return a "continue_on" field with a URL instead of a token
  from the ID assertion endpoint. For example:
  ```js
  {
    "continue_on": "https://idp.example/finish_login?account_id=123"
  }
  ```
* When the authorization flow finishes, call `IdentityProvider.resolve` to close the
  popup and provide the token that will be passed to the RP:
  ```js
    IdentityProvider.resolve("this is the token");
  ```
* If the account ID has changed (for example, if the popup provided a "Switch
  User" function), you can specify it in a second parameter:
  ```js
    IdentityProvider.resolve("this is the token", {accountId: "123"});
  ```
* If the user cancels the login flow, call `IdentityProvider.close` to close
  the popup and reject the promise that was returned from `navigator.credentials.get`:
  ```js
    IdentityProvider.close();
  ```

### Parameters API
This feature lets RPs specify additional key/value pairs that will get sent
to the ID assertion endpoint.

To use this feature:
* Enable the experimental feature `FedCmAuthz` in chrome://flags
* Add a `params` field to the `navigator.credentials.get` call:
  ```js
    navigator.credentials.get({
        identity: {
          providers: [{
            configURL: "https://idp.example/config.json",
            clientId: "123",
            nonce: nonce,
            params: {
              "key": "value",
              "anything_goes": "yes",
              "really": "yes",
              "scopes": "calendar.readonly",
              "dpop": "something",
              "moar": "sure",
            }
          }],
        }
    });
  ```
* These key/value pairs will be sent as-is in the ID assertion request:
  `account_id=123&key=value&anything_goes=yes&really=yes&scopes=calendar.readonly&dpop=something&moar=sure&...`
  

### Multiple configURLs
This feature lets you have multiple different config files under the same
eTLD+1 as long as they all have the same accounts_endpoint. This can be
useful to specify different branding or different ID assertion endpoints.

To use this feature:
* Enable the experimental feature `FedCmAuthz` in chrome://flags
* Add the login_url and accounts_endpoint to the .well-known/web-identity
  file:
  ```js
  {
    "provider_urls": [
      // keep this unchanged
    ],
    "accounts_endpoint": "https://fedcm.idp.example/accounts",
    "login_url": "https://fedcm.idp.example/login.html"
  }
  ```

### Account labels
The account labels API lets IdPs give a list of labels to an account and
lets different config files specify a filter for those labels.

To use the API:
* Enable the experimental feature `FedCmAuthz` in chrome://flags
* Add a `labels` field to accounts in the account endpoint:
  ```js
  {
    "name": "John Smith",
    //...
    "labels": ["label1"]
  }
  ```
* Add the desired label to the config file:
  ```js
  {
    "accounts_endpoint": "...",
    // ...
    "accounts": {
      "include": "label1"
    }
  }
  ```

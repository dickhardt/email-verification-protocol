# verified-email-auto-fill (VEAF)

Verified Email Auto Fill 

The process if broken down into the following steps:

1. Ahead of time **Provider Registration**
2. Just-in time **Request** for verified emails
    1. Email Suggestion **aggregation** from registered providers
    2. Email Suggestion **selection**
    3. Token **Acquisition** from Provider
    4. Token **Verification** from Verifier

## Ahead of Time Provider Registration

Ahead of time, a website registers itself as a third party autofill provider by:

1. User navigates to any sub-domain on `example.net`. Eg. `accounts.example.net`

2. The page can call to register an email address it is an issuer for by calling:

> Exploring doing this declaratively with HTTP headers and/or HTML metadata.

```javascript
// This prompts the user to accept "https://issuer.example.net" as an third party autofill provider.
const response = await IdentityProvider.register(
    "https://issuer.example.net/fedcm.json",
);
```

3. If the user accepts the prompt, the browser records "https://issuer.example.net/fedcm.json" in its local storage.

## Just-in-time Email Request

1. User navigates to a site that will act as the RP.

On page load, the RP page has the following HTML in the page:

```html

<input autocomplete="email webidentity">

```

And has made a call which returns a promise for a token:

```js
try {
  const {token} = await navigator.credentials.get({
    mediation: "conditional",
    identity: {
      providers: [{
        format: "sd-jwt",
        fields: ["name", "email", "picture"],
        nonce: "1234",
      }]
    }
  });
  // send to token to server
} catch ( e ) {
   // no providers or other error
}
```

### Aggregation

On page load, the browser:

1. Loops through the `configURL`s that were registered previously and stored in browser storage in the previous step to build a list of suggestions:

#### For each registered provider

1. The browser fetches the well-known file for `https://issuer.example.com` from `https://issuer.example.com/.well-known/web-identity` per https://w3c-fedid.github.io/FedCM/#idp-api-well-known

which returns `application/json` containing the IdentityProviderWellKnown JSON object https://w3c-fedid.github.io/FedCM/#idp-api-config-file

> Q: what are the restrictions on the hostname in the provider_urls?
> How does the browser know which one of the `provider_urls` to use if more than one in the array?

The well-known file MUST contain:

    - **accounts_endpoint** - what the browser calls to get the accounts from the provider
    - **jwks_uri** - a jwks file containing one or public keys

For example:

```json 
{
  "accounts_endpoint": "https://accounts.example.com/fedcm/accounts",
  "jwks_uri": "https://accounts.example.com/jwks.json" // current thinking is this is the `jku` in ID Token
} 
```

2. The browser fetches the config file listed in `provider_urls` which MUST contain:
    - **sd_assertion_endpoint** - the endpoint the browser calls passing 1P cookies to obtain an SD-JWT

> We could potentially move the `sd_assertion_endpoint` to the .well-known file, but that would force issuers to have just a single `sd_assertion_endpoint` per eTL+1. IdPs needed more than one `id_assertion_endpoint` per eTL+1, so I'd expect them to want too for the `sd_assertion_endpoint`, but if that's not true, it could move too to the .well-known file.

```json 
{
  "sd_assertion_endpoint": "https://accounts.example.com/fedcm/sd-jwt",
} 
```

3. Browser fetches `accounts_endpoint` from provider

```json
{
  "accounts": [{
    "id": "xyz",
    "name": "John Doe",
    "email": "john.doe@mydomain.example",
  }]
}
```

4. The browser checks if the following DNS records exists:
   
```
email._webidentity.mydomain.example   TXT   iss=https://issuer.example.com
```

Ensuring that `mydomain.example` has delegated the issuer `https://issuer.example.com` to be authoratative for issuing verified email SD-JWTs 

>Q: what are the restrictions on the hostname in each of these URIs? 

The browser adds a suggestion for a verified email address if the DNS entry is correct and fetching the `.well-known/web-identity` provided a `provider_urls` value that contains the required `accounts_endpoint`, `sd_assertion_endpoint`, and `jwks_uri` values. 

> Q: Perhaps the browser should call the `accounts_endpoint` and ensure it gets an email that matches the passed email, and then POSTS the corresponding account `id` to the `sd_assertion_endpoint` and then verifies the returned SD-JWT (since user should be logged in) verifying the `jwks_uri` has a key for the `iss` value in the SD-JWT.

### Selection

> Exploring not requiring JS and enabling this functionality declaratively by the page having a hidden field that the browser will fill with the SD-JWT that gets posted to the RP server.

1. User clicks in input field and browser displays the list of suggestions of available email addresses could be shared. Emails that would be verified are decorated for user to understand. User selects a verified email from browser selection.

2. User picks an johm.doe@mydomain.example.

## Token Acquisistion from Provider

1. browser POSTS to `sd_issuance_endpont` provider w/ 1P cookies to get SD-JWT

| Name          | Description |
| ------------- | ------------------------------------------------------------------------------------------- |
| account_id    | The `id` passed in the account in returned in the `accounts_endpoint`.                      |
| public_key    | A JWK public key generated by the browser to be bound to the issuer's JWT                   |
| format        | The format requested by the verifier and suppported by the browser. Currently, `vc+sd-jwt`  |

For example:

```
\\ cookies

account_id=xyz&format=SD-JWT&... key binding info
```

2. provider can decide to issue right away, or can tell browser to re-authenticate the user `continue_on` and browser displays popup window 

| Name          | Description |
| ------------- | ------------------------------------------------------------------------------------------- |
| token         | An SD-JWT                                                                                   |
| continue_on   | A URL that the browser uses to open a pop-up window where the user can continue on          |

```json
{"token":"eyssss...."}
```

or

```json
{ "continue_on": "https://issuer.example.net/login"}
```

3. if popup window, provider calls `IdentityProvider.resolve(sd_jwt)`

## Token Verification from Verifier

1. The `navigator.credentials.get()` call returns and `credential.token` is an SD-JWT+KB

``` 
// token example and payload
```

2. JS code sends `token` to RP server. 


3. RP retrieves `iss` value from SD-JWT and disclosed email address. RP checks DNS record for email domain contains `iss` value just as browser did. 

```
email._webidentity.mydomain.example   TXT   iss=https://issuer.example.com
```

4. RP Server fetches jwks values for `iss` value in SD-JWT+KB the same as in Provider Registration, and verifies the SD-JWT+KB.

> Alternative - SD-JWT `jku` has jwks url to get keys and `jku` MUST start with `iss` string  


# Open Questions

1) Would it be possible to relax the prompt necessary for registration?

For example, if we passed the user's email in the registration call, would it allow the browser to relax the prompt?

```javascript
const response = await IdentityProvider.register(
    "https://issuer.example.net/fedcm.json",
    "john.doe@mydomain.example" // is this useful though? Browser can learn which emails by calling accounts_endpoint
);
```














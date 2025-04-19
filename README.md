# verified-email-auto-fill (VEAF)

Verified Email Auto Fill 

## Provider Registration


1. User navigates to any sub-domain on `example.com`. Eg. `accounts.example.com`

2. The page can call to register an email address it is an issuer for by calling:

```javascript
const response = await IdentityProvider.register(
    "https://issuer.example.net",
    "john.doe@mydomain.example" // is this useful though? Browser can learn which emails by calling accounts_endpoint 
);
```
> Exploring doing this declaratively with HTTP headers and/or HTML metadata

3. The browser checks if the following DNS records exists:

```
email._webidentity.mydomain.example   TXT   iss=https://issuer.example.com
```

ensuring that `mydomain.example` has delegated the issuer `https://issuer.example.com` to be authoratative for issuing verified email SD-JWTs 

4. The browser fetches the well-known file for `https://issuer.example.com` from `https://issuer.example.com/.well-known/web-identity` per https://w3c-fedid.github.io/FedCM/#idp-api-well-known

which returns `application/json` containing the IdentityProviderWellKnown JSON object https://w3c-fedid.github.io/FedCM/#idp-api-config-file

```json
{
  "provider_urls": [
      "https://accounts.example.com/any_file_path.json"
  ]
}
```

> Q: what are the restrictions on the hostname in the provider_urls? How does the browser know which one of the `provider_urls` to use if more than one in the array?

5. The browser fetches the config file listed in `provider_urls` which MUST contain:

- **accounts_endpoint** - what the browser calls to get the accounts from the provider
- **sd_assertion_endpoint - the endpoint the browser calls passing 1P cookies to obtain an SD-JWT
- **jwks_uri** - a jwks file containing one or public keys

```json 
{
  "accounts_endpoint": "https://accounts.example.com/fedcm/accounts",
  "sd_assertion_endpoint": "https://accounts.example.com/fedcm/sd-jwt",
  "jwks_uri": "https://accounts.example.com/jwks.json"
}
```

>Q: what are the restrictions on the hostname in each of these URIs? 

The `IdentityProvider.register()` call succeeds if the DNS entry is correct and fetching the `.well-known/web-identity` provided a `provider_urls` value that contains the required `accounts_endpoint`, `sd_assertion_endpoint`, and `jwks_uri` values. 

> Q: Perhaps the browser should call the `accounts_endpoint` and ensure it gets an email that matches the passed email, and then POSTS the corresponding account `id` to the `sd_assertion_endpoint` and then verifies the returned sd-jwt (since user should be logged in) verifying the `jwks_uri` has a key for the `iss` value in the sd-jwt.

The browser has now registered the provider for the email `john.doe@domain.example`

## Token Release

1. User navigates to a site that will act as the RP.

RP page has the following HTML in the page:

```html

<input autocomplete="email webidentity">

```

And has made this call which has not returned:

```js
try {
  const {token} = await navigator.credentials.get({
    mediation: "conditional",
    identity: {
      providers: [{
        format: "vc+sd-jwt",
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

> Exploring not requiring JS and enabling this functionality declaratively by the page having a hidden field that the browser will fill with the SD-JWT that gets posted to the RP server.


2. User clicks in input field and browser displays selection of what could be shared. Emails that would be verified are decorated for user to understand. User selects a verified email from browser selection.

3. The `navigator.credentials.get()` call returns and `credential.token` is an SD-JWT+KB

``` 
// token example and payload
```

5. JS code sends `token` to RP server. 


6. RP retrieves `iss` value from sd-jwt and disclosed email address. RP checks DNS record for email domain contains `iss` value just as browser did. 

```
email._webidentity.mydomain.example   TXT   iss=https://issuer.example.com
```

8. RP Server fetches jwks values for `iss` value in SD-JWT+KB the same as in Provider Registration, and verifies the SD-JWT+KB.

> Alternative - `jku` has jwks url to get keys and `jku` MUST start with `iss` string  


## Token Acquisistion from Provider

1. Browser fetches `accounts_endpoint` from provider

```json
{
  "accounts": [{
    "id": "xyz",
    "name": "John Doe",
    "email": "john.doe@mydomain.example",
  }]
}
```

3. Browser shows accounts metadata to autofill

4. User picks an johm.doe@mydomain.example.

3. browser POSTS to `sd_issuance_endpont` provider w/ 1P cookies to get SD-JWT

```
\\ cookies

account_id=xyz&format=sd-jwt&... key binding info
```

4. provider can decide to issue right away, or can tell browser to re-authenticate the user `continue_on` and browser displays popup window 

```json
{"token":"eyssss...."}
```

or

```json
{ "continue_on": "https://issuer.example.net/login"}
```

5. if popup window, provider calls `IdentityProvider.resolve(sd_jwt)`
















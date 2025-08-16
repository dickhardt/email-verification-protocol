# Verified Email Autocomplete

Verifying control of an email address is a frequent activity on the web today and is used both to prove the user has provided a valid email address, and as a means of authenticating the user when returning to an application. 

Verification is performed by either:

1) sending the user a link they click on or a verification code. This requires the user to switch from the application they are using to their email address and having to wait for the email arrive, and then perform the verification action. This friction often causes drop off in users completing the task.
2) the user logs in with a social login provider such as Apple or Google that provide a verified email address. This requires the application to have set up a relationship with each social provider, and the user to be using one of those services and wanting to share the additional profile information that is also provided in the OpenID Connect flow.

Verified Email Autocomplete enables an application to obtain a verified email address from any Issuer the user wishes to use without any prior registration by the application, and to only share the verified email address improving the privacy aspects of the interaction.

The protocol aligns with the issuer->holder->verifier pattern where the holder is the browser and the verifier is the website requesting a verified email address. The issuer can be any service that a DNS record for the email domain delegates as being authoritative for the email domain.


## Key Concepts


- SD-JWT: A JWT per [SD-JWT link] that is signed by the Issuer and contains one or more hashes and matching email claims and has the browser's public key bound. Non-email claims are permitted but not addressed in this doc. Using an SD-JWT blinds the Issuer to the RP that is requesting the verified email, allows the browser to selectively disclose which claims to release, and allows separation between issuance of the token by the Issuer to the browser(holder) and presentation of the token to the RP (verifier).

- Issuer: a service that exposes an `accounts_endpoint` that returns the email addresses it can issue SD-JWTs for, a `issuance_endpoint` that is called to obtain an SD-JWT, and a `sd_jwt_uri` that contains the public keys used to verify the SD-JWT. The Issuer is identified by its domain, an eTLD+1 (eg `issuer.example`). THe hostname in all URLs from the Issuer's metadata MUST end with the Issuer's domain. This identifier is what binds the SD-JWT, the DNS delegation, with the Issuer.

> Restricting the Issuer to be an eTLD+1 may be too restrictive. Let's get feedback. Having a crisp identifier and a format different than OpenID Connect tokens (no leading https://) simplifies verification and has clean bindings between all the services, DNS record, and token.


## User Experience

Registration Process: The user navigates to the Issuer's website (the apex domain or any subdomain) and is prompted to enable the Issuer to issue verified emails, and the user accepts.

Verified Email Release: The user navigates to any website that requires a verified email address and an input field to enter the email address. The user focusses on the input field and the browser provides one or more verified emails for the user to provide. The user selects a verified email and the app proceeds having obtained the verified email.


# Processing Steps

1. Issuer **Registration**
2. Email **Request**
3. Email **Aggregation**
4. Email  **Selection**
6. Token **Issuance**
6. Token **Verification** 

## 1. Issuer Registration

Ahead of time, a website registers itself as a third party autofill provider by:
  
- **1.1** - User navigates to the apex or any sub-domain of the Issuer such as `issuer.example` or `www.issuer.example` and logs in. The page notifies the browser that the user is logged in with:

```javascript
navigator.login.setStatus("logged-in");
```

> The Issuer can also set the login status with a HTTP header `Set-Login: logged-in` on the way back from redirects.
> The Issuer calls `navigator.login.setStatus("logged-out");` when the user logs out.

- **1.2** - The page calls to register as an Issuer with:

```javascript
// This prompts the user to accept "https://issuer.example" as Issuer of verified emails.
const response = await IdentityProvider.register();
// Q: perhaps this should be IdentityProvider.registerEmailIssuer()  ???
```

> TODO: Explore doing this declaratively with HTTP headers and/or HTML metadata.

- **1.3** - The browser then confirms the Issuer will correctly issue SD-JWTs by performing steps (3), (5) and (6) below.

- **1.4** - If the Issuer has provided valid SD-JWTs for at least one email address, the browser prompts the user to accept the issuer by displaying the email addresses verified, and if the user accepts the prompt, the browser records `issuer.example.net` as an Issuer in its local storage.



## 2. Email Request

User navigates to a site that will act as the RP.

- **2.1** - The RP page has the following HTML in the page:

```html

<input autocomplete="email webidentity">

```

- **2.2** - The page has made this call which has not returned:

```js
try {
  const {token} = await navigator.credentials.get({
    mediation: "conditional",
    identity: {
      providers: [{
        format: "sd-jwt",
        fields: ["email"],
        nonce: "259c5eae-486d-4b0f-b666-2a5b5ce1c925",
      }]
    }
  });
  // send to token to server
} catch ( e ) {
   // no providers or other error
}
```


> Explore not requiring JS and enabling this functionality declaratively by the page having a hidden field that the browser will fill with the SD-JWT that gets posted to the RP server.


## 3. Email Suggestion Aggregation

On page load and detecting the RP has performed (2), for each registered Issuer that the user is [logged-in]([logged-out](https://w3c-fedid.github.io/login-status/#get-the-login-status)) it does the following:

> TODO: We have to introduce the ability for the browser to know the difference between (a) "logged-out" users and (b) users that are "logged-in" but actually don't have any accounts with verified emails to be provided.

- **3.1** - The browser loads `https://issuer.example/.well-known/web-identity` and MUST follow redirects to the same path but with a different subdomain of the Issuer, for example `https://accounts.issuer.example/.well-known/web-identity`. 

> Most apex domains redirect all HTTP calls to a subdomain

- **3.2** - The browser checks that the `.well-known/web-identity` file contains JSON that includes the following properties:

- *accounts_endpoint* - the API endpoint per FedCM that returns the accounts the issuer provides
- *sd_issuance_endpoint* - the API endpoint the browser calls to obtain an SD-JWT
- *login_url* - The URL that the browser can point the user to to login to the Issuer in case the user is logged out
- *sd_jwks_uri* - the URL where the issuer provides its public keys to verify the SD-JWT

Each of these properties MUST include the issuer domain as the root of their hostname. 

Following is an example `.well-known/web-identity` file

```json
{
  "accounts_endpoint": "https://accounts.issuer.example/fedcm/accounts",
  "sd_issuance_endpoint": "https://accounts.issuer.example/fedcm/issuance",
  "sd_jwks_uri": "https://accounts.issuer.example/fedcm/jwks.json"
}
```

- **3.3** - The browser fetches the `accounts_endpoint` from Issuer passing cookies. The `application/json` response MAY include an `accounts` property that is an array of objects that MUST contain `id` and `email`.

Eg:

```json
{
  "accounts": [{
    "id": "xyz",
    "email": "john.doe@email-domain.example",
  }]
}
```

- **3.4** - For each email domain, the browser confirms there is a DNS record for `email._webidentity.` with a TXT record containing the `iss=` set to the Issuer domain. Following is an example DNS record:

```
email._webidentity.email-domain.example   TXT   iss=issuer.example
```

This record confirms that `email-domain.example` has delegated Verified Email Autocomplete to the Issuer `issuer.example`.

Note this record MUST also exist for `issuer.example` to support Verified Email Autocomplete.

```
email._webidentity.issuer.example   TXT   iss=issuer.example
```
> Access to DNS records and email is often independent of website deployments. This provides assurance that an Issuer is truly authorized as an insider with only access to websites on `issuer.example` could setup an Issuer that would grant them verified emails for any email at `issuer.example`.

- **3.5** - The browser stores the list of verified emails is will offer for Verified Email Autocomplete, and MAY cache this list for future pages.

## 4. Email Selection 

- **4.1** - User focusses on input field with

- **4.2** - The browser displays the list of suggestions of available email addresses could be shared. Emails that would be verified are decorated for user to understand. 

- **4.3** - User selects a verified email from browser selection.


## 5. Token Issuance

- **5.1** - browser POSTS to the `sd_issuance_endpont` of the Issuer for the selected email w/ 1P cookies to get SD-JWT

```
\\ cookies

account_id=xyz&format=SD-JWT&... key binding info
```

- **5.2** - Issuer checks if there is a logged in user, and the logged in user has the account identified by `account_id`. If all good the Issuer creates a fresh SD-JWT and returns it as the value of `token` in an `application/json` response. If not, a response with an `error` can be passed to the browser.


```json
{"token":"eyssss...."}
```

5.3 If the user identified by account_id is not logged, the Issuer responds with `application/json` containing `continue_on` with the value of the url the browser should load in a popup window.

```json
{ "continue_on": "https://accounts.issuer.example/login"}
```


on successful login the Issuer calls `IdentityProvider.resolve(sd_jwt)`

> Is this best signature for this? Perhaps more specific of issuer and email?


## 6. Token Verification

- **6.1** - The `navigator.credentials.get()` call returns and `credential.token` is an SD-JWT+KB

``` 
// token example and payload
```

> Explore browser setting a hidden field instead so JS is not required

- **6.2** - JS code sends `token` to RP server. 

- **6.3** - RP Server MUST validate the SD-JWT as described [here](https://www.ietf.org/archive/id/draft-ietf-oauth-selective-disclosure-jwt-17.html#name-verification) (e.g. validating `nonce` and `aud` as described [here](https://www.ietf.org/archive/id/draft-ietf-oauth-selective-disclosure-jwt-17.html#section-7.3-4.5.2.6)).

- **6.4** - RP Server retrieves `iss` value from SD-JWT and disclosed email address and extracts email domain. 

- **6.5** - RP Server checks DNS record for email domain contains `iss` value just as browser did in 3.4. 

```
email._webidentity.email-domain.example   TXT   iss=issuer.example
```

- **6.6** - RP Server fetches `.well-known/web-identity` just as browser did in 3.1. and extracts `sd_jwks_uri` and verifies the host ends with the Issuer domain.

- **6.7** - RP Server verifies SD-JWT+KB token using keys from the `sd_jwks_uri` per [sd-jwt rfc]

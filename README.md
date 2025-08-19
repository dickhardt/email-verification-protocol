# Verified Email Autocomplete

Verifying control of an email address is a frequent activity on the web today and is used both to prove the user has provided a valid email address, and as a means of authenticating the user when returning to an application. 

Verification is performed by either:

1) sending the user a link they click on or a verification code. This requires the user to switch from the application they are using to their email address and having to wait for the email arrive, and then perform the verification action. This friction often causes drop off in users completing the task.
2) the user logs in with a social login provider such as Apple or Google that provide a verified email address. This requires the application to have set up a relationship with each social provider, and the user to be using one of those services and wanting to share the additional profile information that is also provided in the OpenID Connect flow.

Verified Email Autocomplete enables an application to obtain a verified email address from any Issuer the user wishes to use without any prior registration by the application, and to only share the verified email address improving the privacy aspects of the interaction.

The protocol aligns with the issuer->holder->verifier pattern where the holder is the browser and the verifier is the website requesting a verified email address. The issuer can be any service with a DNS record that the email domain delegates as being authoritative for the email domain.


## Key Concepts


- SD-JWT: A JWT per [SD-JWT link] that is signed by the Issuer and contains an email claim is bound to a public key managed by the . Non-email claims are permitted but not addressed in this doc. Using an SD-JWT blinds the Issuer to the RP that is requesting the verified email and allows separation between issuance of the token by the Issuer to the browser(holder) and presentation of the token to the RP (verifier).

- Issuer: a service that exposes a `issuance_endpoint` that is called to obtain an SD-JWT, and a `jwt_uri` that contains the public keys used to verify the SD-JWT. The Issuer is identified by its domain, an eTLD+1 (eg `issuer.example`). The hostname in all URLs from the Issuer's metadata MUST end with the Issuer's domain. This identifier is what binds the SD-JWT, the DNS delegation, with the Issuer.

> Having a crisp identifier and a format different than OpenID Connect tokens (no leading https://) simplifies verification and has clean bindings between all the services, DNS record, and token.


## User Experience

Verified Email Release: The user navigates to any website that requires a verified email address and an input field to enter the email address. The user focusses on the input field and the browser provides one or emails for the user to select based on emails the user has provided previously to the browser. The user selects a verified email and the app proceeds having obtained the verified email.

> Are emails that can be verified decorated by the browser in the autocomplete UI?
> What UX is presented to the user when the app gets a verified email so the user knows it is already verified?


# Processing Steps

1. **Email Request**
2. **Email Selection**
3. **Token Request**
4. **Token Issuance**
5. **Token Presentation**
6. **Token Verification**

## 1. Email Request

User navigates to a site that will act as the RP.

- **1.1** - The RP page has the following HTML in the page:

```html

<input autocomplete="email web-identity">

```

- **1.2** - The page has made this call which has not returned:

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


## 2. Email Selection 

- **2.1** - User focusses on input field with `autocomplete="email web-identity"`

- **2.2** - The browser displays the list of email addresses it has for the user. 

> Q: Are emails that could be verified decorated for user to understand? 

- **2.3** - User selects an email address from browser selection.


## 3. Token Request

If the RP has performed (1):

- **3.1** - the browser parses the email domain ($EMAIL_DOMAIN) from the email address, looks up the `TXT` record for `email._web-identity.$EMAIL_DOMAIN`, and looks for a string of the form `iss=$ISSUER` where $ISSUER is the issuer identifier. 

example record

```
email._web-identity.email-domain.example   TXT   iss=issuer.example
```

This record confirms that `email-domain.example` has delegated Verified Email Autocomplete to the issuer `issuer.example`.

If the email domain and the issuer are the same domain, then the record would be:

```
email._web-identity.issuer.example   TXT   iss=issuer.example
```

> Access to DNS records and email is often independent of website deployments. This provides assurance that an issuer is truly authorized as an insider with only access to websites on `issuer.example` could setup an issuer that would grant them verified emails for any email at `issuer.example`.

- **3.2** - if an issuer is found, the browser loads `https://$ISSUER$/.well-known/web-identity` and MUST follow redirects to the same path but with a different subdomain of the Issuer.

For example, `https://issuer.example/.well-known/web-identity` may redirect to `https://accounts.issuer.example/.well-known/web-identity`. 


- **3.3** - the browser confirms that the `.well-known/web-identity` file contains JSON that includes the following properties:

- *issuance_endpoint* - the API endpoint the browser calls to obtain an SD-JWT
- *jwks_uri* - the URL where the issuer provides its public keys to verify the SD-JWT

Each of these properties MUST include the issuer domain as the root of their hostname. 

Following is an example `.well-known/web-identity` file

```json
{
  "issuance_endpoint": "https://accounts.issuer.example/web-identity/issuance",
  "jwks_uri": "https://accounts.issuer.example/web-identity/jwks.json"
}
```

- **3.4** - the browser generates a private / public key and signs a JWT with the private key that has the public key in the JWT header in the JWK format as a `jwk` claim, and contains the following claims in the payload:

  - *iss* - a string for the browser, can be any value
  - *aud* - the issuer
  - *iat* - time when the JWT was signed
  - *nonce* - nonce provided by the RP
  - *email* - email address to be verified 

An example JWT header:
```json
{
  "alg": "EdDSA",
  "typ": "JWT",
  "jwk": {
    "kty": "OKP",
    "crv": "Ed25519",
    "x": "11qYAYdk9E6z7mT6rk6j1QnXb6pYq4v9wXb6pYq4v9w"  // base64url-encoded public key
  }
}
```
> do we want to register a new JWT `typ`

An example payload 
```json
{
  "iss": "user-agent",
  "aud": "issuer.example",
  "iat": 1692345600,
  "nonce": "259c5eae-486d-4b0f-b666-2a5b5ce1c925",
  "email": "user@example.com"
}
```


- **3.5** - the browser POSTs to the `issuance_endpoint` of the issuer with 1P cookies with a content-type of `application/json` containing a JSON string with a `request_token` property set to the signed JWT as a JSON string. 

```bash
\\ cookies
Content-type: application/x-www

request_token=eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVC... truncated for brevity

```
> Q: What is our mechanism for crypto agility and algorithm support discovery? 
> The web-identity file can have an array of supported algorithms

## 4. Token Issuance

On receipt of a token request:

- **4.1** - the issuer MUST verify the request_token by:

  - parsing the JWT into header, payload, and signature components
  - confirming the presence of, and extracting the `jwk` and `alg` fields from the JWT header, and the `aud`, `iat`, `email`, and `nonce` claims from the payload
  - verifying the JWT signature using the `jwk` with the `alg` algorithm
  - verifying the `aud` claim exactly matches the issuer's identifier
  - verifying the `iat` claim is within 60 seconds of the current time
  - verifying the `email` claim contains a syntactically valid email address


- **4.2** - the issuer checks if the cookies sent represent a logged in user, and if the logged in user has control of the email provided in the request_token. If so the issuer generates an SD-JWT with the following properties:

  - **Header**: MUST contain 
    - `alg`: signing algorithm
    - `kid`: key identifier of key used to sign
    - `typ` set to "web-identity+sd-jwt"
  - **Payload**: MUST contain the following claims:
    - `iss`: the issuer identifier
    - `iat`: issued at time 
    - `cnf`: confirmation claim containing the public key from the request_token's `jwk` field
`   - `email`: claim containing the email address from the request_token
    - `email_verified`: claim that email is verified per OpenID Connect 1.0
  - **Signature**: MUST be signed with the issuer's private key corresponding to a public key in the `jwks_uri` identified by `kid`


  Example header:
  ```json
  {
    "alg": "EdDSA",
    "kid": "2024-08-19",
    "typ": "web-identity+sd-jwt"
  }
  ```

  Example payload:
  ```json
  {
    "iss": "issuer.example",
    "iat": 1724083200,
    "cnf": {
      "jwk": {
        "kty": "OKP",
        "crv": "Ed25519",
        "x": "11qYAYdk9E6z7mT6rk6j1QnXb6pYq4v9wXb6pYq4v9w"
      }
    },
    "email": "user@example.com",
    "email_verified": true
  }
  ```


- **4.3** - the issuer returns the SD-JWT to the browser as the value of `issued_token` in an `application/json` response.

Example:
```bash
Content-type: application/json

{"issued_token":"eyssss...."}
```

> In future the issuer versions the issuer could prompt the user to login via a URL or with a Passkey request.


## 5. Token Presentation

On receiving the `issued_token`:

- ** 5.1 ** - the browser MUST verify the SD-JWT per (SD-JWT spec) by:

  - parsing the SD-JWT into header, payload, and signature components
  - confirming the presence of, and extracting the `alg` and `kid` fields from the SD-JWT header, and the `iss`, `iat`, `cnf`, `email`, and `email_verified` claims from the payload
  - parsing the email domain from the `email` claim and looking up the `TXT` record for `email._web-identity.$EMAIL_DOMAIN` to verify the `iss` claim matches the issuer identifier in the DNS record
  - fetching the issuer's public keys from the `jwks_uri` specified in the `.well-known/web-identity` file
  - verifying the SD-JWT signature using the public key identified by `kid` from the JWKS with the `alg` algorithm
  - verifying the `iat` claim is within 60 seconds of the current time
  - verifying the `email` claim matches the email address the user selected
  - verifying the `email_verified` claim is true

- ** 5.2 ** - the browser then creates an SD-JWT+KB by:

  - taking the verified SD-JWT from step 5.1 as the base token
  - creating a Key Binding JWT (KB-JWT) with the following structure:
    - **Header**: 
      - `alg`: same signing algorithm used by the browser's private key
      - `typ`: "kb+jwt"
    - **Payload**:
      - `aud`: the RP's origin
      - `nonce`: the nonce from the original `navigator.credentials.get()` call
      - `iat`: current time when creating the KB-JWT
      - `sd_hash`: SHA-256 hash of the SD-JWT
  - signing the KB-JWT with the browser's private key (the same key pair generated in step 3.4)
  - concatenating the SD-JWT and the KB-JWT separated by a tilde (~) to form the SD-JWT+KB

  Example KB-JWT header:
  ```json
  {
    "alg": "EdDSA",
    "typ": "kb+jwt"
  }
  ```

  Example KB-JWT payload:
  ```json
  {
    "aud": "https://rp.example",
    "nonce": "259c5eae-486d-4b0f-b666-2a5b5ce1c925",
    "iat": 1724083260,
    "sd_hash": "X9yH0Ajrdm1Oij4tWso9UzzKJvPoDxwmuEcO3XAdRC0"
  }
  ```

- ** 5.3 ** - the browser returns the `navigator.credentials.get()` call and `credential.token` is the SD-JWT+KB


> Explore browser setting a hidden field instead so JS is not required

## 6. Token Verification

The RP web page now has a SD-JWT+KB and sends it with JS code to the RP server. The RP server MUST verify the SD-JWT+KB by:

- **6.1** - the RP server receives the SD-JWT+KB from the web page

- **6.2** - the RP parses the SD-JWT+KB by separating the SD-JWT and KB-JWT components (separated by tilde ~)

- **6.3** - the RP verifies the KB-JWT by:
  - parsing the KB-JWT into header, payload, and signature components
  - confirming the presence of, and extracting the `alg` field from the KB-JWT header, and the `aud`, `nonce`, `iat`, and `sd_hash` claims from the payload
  - verifying the `aud` claim matches the RP's origin
  - verifying the `nonce` claim matches the nonce from the RP's session with the web page
  - verifying the `iat` claim is within a reasonable time window 
  - computing the SHA-256 hash of the SD-JWT and verifying it matches the `sd_hash` claim

- **6.4** - the RP verifies the SD-JWT by:
  - parsing the SD-JWT into header, payload, and signature components
  - confirming the presence of, and extracting the `alg` and `kid` fields from the SD-JWT header, and the `iss`, `iat`, `cnf`, `email`, and `email_verified` claims from the payload
  - parsing the email domain from the `email` claim and looking up the `TXT` record for `email._web-identity.$EMAIL_DOMAIN` to verify the `iss` claim matches the issuer identifier in the DNS record
  - fetching the issuer's public keys from the `jwks_uri` specified in the `.well-known/web-identity` file
  - verifying the SD-JWT signature using the public key identified by `kid` from the JWKS with the `alg` algorithm
  - verifying the `iss` claim exactly matches the issuer identifier from the DNS record
  - verifying the `iat` claim is within a reasonable time window
  - verifying the `email_verified` claim is true

- **6.5** - the RP verifies the KB-JWT signature using the public key from the `cnf` claim in the SD-JWT with the `alg` algorithm from the KB-JWT header


# Privacy Considerations

> Below are notes capturing some discussions of potential privacy implications.

1. The email domain operator no longer learns which applications the user is verifying their email address to as the applications are no longer sending an email verification code to the user. By using an SD-JWT+KB, the browser intermediates the request and response so that the issuer does not learn the identity of the RP. 

2. The RP can infer if a user is logged into the issuer as the RP receives a SD-JWT when the user is logged in, and does not when the user is not logged in. 

3. The issuer may learn the user has email at a mail domain it is authoritative for that it did not know the user had. 
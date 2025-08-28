# Email Verification Protocol

Verifying control of an email address is a frequent activity on the web today and is used both to prove the user has provided a valid email address, and as a means of authenticating the user when returning to an application. 

Verification is performed by either:

1) sending the user a link they click on or a verification code. This requires the user to switch from the application they are using to their email address and having to wait for the email arrive, and then perform the verification action. This friction often causes drop off in users completing the task.
2) the user logs in with a social login provider such as Apple or Google that provide a verified email address. This requires the application to have set up a relationship with each social provider, and the user to be using one of those services and wanting to share the additional profile information that is also provided in the OpenID Connect flow.

The Email Verification Protocol enables an application to obtain a verified email address from any Issuer the user wishes to use without any prior registration by the application, and to only share the verified email address improving the privacy aspects of the interaction.

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

1. [**Email Request**](#1-email-request)
2. [**Email Selection**](#2-email-selection)
3. [**Token Request**](#3-token-request)
4. [**Token Issuance**](#4-token-issuance)
5. [**Token Presentation**](#5-token-presentation)
6. [**Token Verification**](#6-token-verification)

## 1. Email Request

User navigates to a site that will act as the RP. 

- **1.1** - the RP Server generates a nonce and binds the nonce to the session.

- **1.2** - the RP Server returns a page that has an input field with the `autocomplete` property set to "email" and the `nonce` property set the the nonce. Following is an example of the HTML in the page:

```html

<input autocomplete="email"  verification="an_element_id"  nonce="12345677890..random">

```

> The exact HTML markup is still in flux.


## 2. Email Selection 

- **2.1** - User focusses on input field with `autocomplete="email web-identity"`

- **2.2** - The browser displays the list of email addresses it has for the user. 

> Q: Are emails that could be verified decorated for user to understand? 

- **2.3** - User selects an email address from browser selection, or the user types an email into the field.

> If we allow user to type in a field we allow learning about new emails, or if the user does not want the browser to remember emails, the Email Verification Protocol is still available. In the future when we allow the user to use a passkey to authenticate to the issuer, the user can provide a verified email at a public computer by authenticating with their passkey and not enter any secrets into the public computer.


## 3. Token Request

If the RP has performed (1):

- **3.1** - the browser parses the email domain ($EMAIL_DOMAIN) from the email address, looks up the `TXT` record for `_email-verification_.$EMAIL_DOMAIN`. The contents of the record is MUST start with `iss=` followed by the issuer identifier. There MUST be only one `TXT` record for `_email-verification_.$EMAIL_DOMAIN`.

example record

```
_email-verification_.email-domain.example   TXT   iss=issuer.example
```

This record confirms that `email-domain.example` has delegated email verification to the issuer `issuer.example`.

If the email domain and the issuer are the same domain, then the record would be:

```
_email-verification_.issuer.example   TXT   iss=issuer.example
```

> Access to DNS records and email is often independent of website deployments. This provides assurance that an issuer is truly authorized as an insider with only access to websites on `issuer.example` could setup an issuer that would grant them verified emails for any email at `issuer.example`.

- **3.2** - if an issuer is found, the browser loads `https://$ISSUER$/.well-known/email-verification` and MUST follow redirects to the same path but with a different subdomain of the Issuer.

For example, `https://issuer.example/.well-known/email-verification` may redirect to `https://accounts.issuer.example/.well-known/email-verification`. 


- **3.3** - the browser confirms that the `.well-known/email-verification` file contains JSON that includes the following properties:

- *issuance_endpoint* - the API endpoint the browser calls to obtain an SD-JWT
- *jwks_uri* - the URL where the issuer provides its public keys to verify the SD-JWT
- *signing_alg_values_supported* - OPTIONAL. JSON array containing a list of the JWS signing algorithms ("alg" values) supported by both the browser for request tokens and the issuer for issued tokens. The same algorithm MUST be used for both the `request_token` and `issuance` within a single issuance flow. Algorithm identifiers MUST be from the IANA "JSON Web Signature and Encryption Algorithms" registry. If omitted, "EdDSA" is the default. "EdDSA" SHOULD be included in the supported algorithms list. The value "none" MUST NOT be used.

Each of these properties MUST include the issuer domain as the root of their hostname. 

Following is an example `.well-known/email-verification` file

```json
{
  "issuance_endpoint": "https://accounts.issuer.example/email-verification/issuance",
  "jwks_uri": "https://accounts.issuer.example/email-verification/jwks",
  "signing_alg_values_supported": ["EdDSA", "RS256"]
}
```

- **3.4** - the browser generates a private / public key and signs a JWT with the private key that has the public key in the JWT header in the JWK format as a `jwk` claim, and contains the following claims in the payload:

  - *aud* - the issuer
  - *iat* - time when the JWT was signed
  - *jti* - unique identifier for the token
  - *nonce* - nonce provided by the RP
  - *email* - email address to be verified 

The browser SHOULD select an algorithm from the issuer's `signing_alg_values_supported` array, or use "EdDSA" if the property is not present.

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
  "aud": "issuer.example",
  "iat": 1692345600,
  "nonce": "259c5eae-486d-4b0f-b666-2a5b5ce1c925",
  "email": "user@example.com"
}
```


- **3.5** - the browser POSTs to the `issuance_endpoint` of the issuer with 1P cookies with a content-type of `application/x-www-form-urlencoded` containing a `request_token` parameter set to the signed JWT. 

```bash
POST /email-verification/issuance HTTP/1.1
Host: accounts.issuer.example
Cookie: session=...
Content-Type: application/x-www-form-urlencoded

request_token=eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVC...
```

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
    - `alg`: signing algorithm (SHOULD match the algorithm from the request_token)
    - `kid`: key identifier of key used to sign
    - `typ` set to "evp+sd-jwt"
  - **Payload**: MUST contain the following claims:
    - `iss`: the issuer identifier
    - `iat`: issued at time 
    - `cnf`: confirmation claim containing the public key from the request_token's `jwk` field
    - `email`: claim containing the email address from the request_token
    - `email_verified`: claim that email is verified per OpenID Connect 1.0
  - **Signature**: MUST be signed with the issuer's private key corresponding to a public key in the `jwks_uri` identified by `kid`


Example header:
  ```json
  {
    "alg": "EdDSA",
    "kid": "2024-08-19",
    "typ": "evp+sd-jwt"
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
The resulting JWT has the `~` appended to it, making it a valid SD-JWT.

- **4.3** - the issuer returns the SD-JWT to the browser as the value of `issuance_token` in an `application/json` response.

Example:
```bash
HTTP/1.1 200 OK
Content-Type: application/json

{"issuance_token":"eyJhbGciOiJFZERTQSIsImtpZCI6IjIwMjQtMDgtMTkiLCJ0eXAiOiJ3ZWItaWRlbnRpdHkrc2Qtand0In0..."}
```

## 4.4 Error Responses

If the issuer cannot process the token request successfully, it MUST return an appropriate HTTP status code with a JSON error response containing an `error` field and optionally an `error_description` field.

### 4.4.1 Authentication Required

When the request lacks valid authentication cookies, contains expired/invalid cookies, or the authenticated user does not have control of the requested email address:

**HTTP 401 Unauthorized**
```json
{
  "error": "authentication_required",
  "error_description": "User must be authenticated and have control of the requested email address"
}
```

### 4.4.2 Invalid Parameters

When the `request_token` is malformed, missing required claims, or contains invalid values:

**HTTP 400 Bad Request**
```json
{
  "error": "invalid_request", 
  "error_description": "Invalid or malformed request_token"
}
```

Specific cases include:
- Missing or invalid `email` claim
- Missing or invalid `nonce` claim  
- Missing or invalid `aud` claim that doesn't match the issuer identifier
- Missing or invalid `iat` claim (outside 60 second window)
- Missing or invalid `jwk` in header

### 4.4.3 Invalid Token

When the `request_token` signature verification fails or the token structure is invalid:

**HTTP 400 Bad Request**
```json
{
  "error": "invalid_token",
  "error_description": "Token signature verification failed or token structure is invalid"
}
```

### 4.4.4 Server Errors

For internal server errors or temporary unavailability:

**HTTP 500 Internal Server Error**
```json
{
  "error": "server_error",
  "error_description": "Temporary server error, please try again later"
}
```

### Error Response Requirements

All error responses MUST:
- Use appropriate HTTP status codes (400, 401, 500, etc.)
- Include `Content-Type: application/json` header
- Include proper CORS headers to allow browser access
- Contain an `error` field with a machine-readable error code
- Optionally contain an `error_description` field with human-readable details

The browser SHOULD handle these errors gracefully by either prompting the user to authenticate with the issuer (when that is specified)  or falling back to traditional email verification methods.

> In a future version of this spec, the issuer could prompt the user to login via a URL or with a Passkey request.


## 5. Token Presentation

On receiving the `issuance_token`:

- **5.1** - the browser MUST verify the SD-JWT per (SD-JWT spec) by:

  - parsing the SD-JWT into header, payload, and signature components
  - confirming the presence of, and extracting the `alg` and `kid` fields from the SD-JWT header, and the `iss`, `iat`, `cnf`, `email`, and `email_verified` claims from the payload
  - parsing the email domain from the `email` claim and looking up the `TXT` record for `_email-verification_.$EMAIL_DOMAIN` to verify the `iss` claim matches the issuer identifier in the DNS record
  - fetching the issuer's public keys from the `jwks_uri` specified in the `.well-known/email-verification` file
  - verifying the SD-JWT signature using the public key identified by `kid` from the JWKS with the `alg` algorithm
  - verifying the `iat` claim is within 60 seconds of the current time
  - verifying the `email` claim matches the email address the user selected
  - verifying the `email_verified` claim is true

- **5.2** - the browser then creates an SD-JWT+KB by:

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

- **5.3** - the browser sets a TBD hidden field and fires the TBD event ...

> details TBD

## 6. Token Verification

The RP web page now has the SD-JWT+KB from the event, and passes it to the RP server, or the token was posted to the RP server.

> details TBD

The RP server MUST verify the SD-JWT+KB by:

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
  - parsing the email domain from the `email` claim and looking up the `TXT` record for `_email-verification_.$EMAIL_DOMAIN` to verify the `iss` claim matches the issuer identifier in the DNS record
  - fetching the issuer's public keys from the `jwks_uri` specified in the `.well-known/email-verification` file
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
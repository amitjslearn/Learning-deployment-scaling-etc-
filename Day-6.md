
## Day 6 – API Security

**OAuth 2.0 (high level)**  
- Framework for delegated access: a client gets an access token from an authorization server to call an API on behalf of a user or itself.  
- Core roles:  
  - Resource owner (user)  
  - Client (your app)  
  - Authorization server (e.g., Azure AD)  
  - Resource server (your API)  
- Common flows:  
  - Authorization Code (frontends + backend)  
  - Client Credentials (service-to-service / daemons)  

**JWT (JSON Web Token)**  
- Compact, signed token format used as the bearer access token in many OAuth 2.0 implementations.  
- Structure: `header.payload.signature` (Base64URL-encoded).  
  - Header: algorithm, token type.  
  - Payload: claims (sub, exp, iss, aud, custom fields like roles).  
  - Signature: ensures integrity and authenticity.  
- Typical use: client sends `Authorization: Bearer <token>` header, API validates signature, issuer, audience, expiry, then trusts the claims.  

**Azure AD integration basics**  
- Register your API as an app in Azure AD (expose scopes/roles).  
- Register client app (e.g., SPA, backend, Postman) and grant it permission to call the API.  
- Client requests a token from Azure AD for your API’s scope; Azure AD returns a JWT access token.  
- Your API validates the token (issuer, audience, signature) using Microsoft’s public keys.  

***

## Build: Secure APIs with JWT (FastAPI-style notes)

Assume you already have a basic FastAPI app.

### 1. Core pieces you need

- A user store (in-memory or DB) with usernames + hashed passwords.  
- A `/token` endpoint where users send username/password and receive a JWT.  
- A token creation function that:  
  - Signs a JWT with a secret key or private key.  
  - Sets standard claims: subject (`sub`), expiration (`exp`), issued-at (`iat`), etc.  
- A dependency that:  
  - Reads the `Authorization: Bearer <token>` header.  
  - Decodes and verifies the JWT.  
  - Rejects invalid/expired tokens.  

### 2. Typical configuration values

- `SECRET_KEY`: long random string (for HS256).  
- `ALGORITHM`: e.g., `"HS256"`.  
- `ACCESS_TOKEN_EXPIRE_MINUTES`: e.g., 30 minutes.  

Example claims to include when creating the token:

- `sub`: the username or user ID.  
- `exp`: expiration time.  
- Optional: roles/permissions, tenant ID, etc.  

***

## Protecting Endpoints with JWT

Typical pattern (in words):

1. Client calls `/token` with credentials (e.g., form data: username, password).  
2. If valid, API returns `{ "access_token": "<jwt>", "token_type": "bearer" }`.  
3. Client stores token securely and sends it with subsequent requests:  
   - `Authorization: Bearer <jwt>`  
4. Protected endpoints declare a dependency that:  
   - Extracts token from header.  
   - Verifies the token.  
   - Retrieves the current user from the token’s claims or user store.  
5. If checks fail, return 401 with `WWW-Authenticate: Bearer`.  

You then:  
- Apply that dependency to endpoints you want to secure.  
- Keep public endpoints (like `/health`) without the dependency.  

***

## Azure AD Integration – Conceptual Notes

If/when you switch to Azure AD issuing tokens:

- Your API no longer generates JWTs; Azure AD does.  
- Steps:  
  - Register API in Azure AD, define scopes (e.g., `api://APP-ID/.default` or `api://APP-ID/read`).  
  - Register client, grant it access to API scopes.  
  - Client obtains access token from Azure AD (using OAuth 2.0 flow).  
  - API validates incoming token:  
    - Read from `Authorization` header.  
    - Validate signature using Azure AD’s JWKS (public keys).  
    - Validate audience (`aud` matches your API), issuer (`iss` matches tenant), and expiry.  
- You still protect endpoints via dependencies, but token validation logic uses Azure’s configuration (issuer, audience, keys).  

***

## Deliverable: Authenticated API Calls

How to demonstrate Day 6 is done:

1. **Unauthenticated call fails**  
   - Call a protected endpoint without `Authorization` header.  
   - API returns 401 Unauthorized (or 403 depending on design).  

2. **Obtain token**  
   - Call `/token` (or your auth endpoint) with valid credentials.  
   - Receive a JWT access token in the response.  

3. **Authenticated call succeeds**  
   - Call the same protected endpoint with:  
     - `Authorization: Bearer <access_token>`  
   - API returns protected data (e.g., current user info, private resource).  

4. **Expired/invalid token behavior**  
   - Wait until token expires or modify it.  
   - API should reject it with 401 and an appropriate error message.  


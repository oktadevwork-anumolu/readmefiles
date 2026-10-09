# IRIS A2A Integration Guide (Filer Path)

Scope: Stable Custody Group II LLC as the filer, submitting information returns through the IRIS Application-to-Application (A2A) channel using a JWT private key and JWKS.

Verification note: This guide is built from IRS working group Q&As, meeting decks, and the e-Services API Authorization guide. Items marked **[verify]** are not confirmed in those sources. Check Publication 5718 (IRIS Electronic Application to Application) and Publication 5903 (IRIS App for TCC) before relying on them.

---

## Part 1: Get a TCC (Transmitter Control Code)

### Step 1. Create and verify your ID.me account
- **What:** Each Responsible Official (RO) creates their own ID.me account and completes identity credentialing.
- **Why required:** IRS uses ID.me to verify the identity of anyone acting for an organization. The IRS guidance says ID.me and the IRIS TCC application are separate processes, and you must finish the ID.me credentialing before submitting the IRIS TCC application. Each person signs in with their own ID.me account for multi-factor authentication. Sharing logins is not allowed.
- **Timing:** Start this early. The IRS delay for ID.me emails has been a recurring question.

### Step 2. Apply for the IRIS TCC (not the FIRE TCC)
- **What:** Sign in to the IRIS Application for TCC link and start a new application. Select the IRIS option, not FIRE.
- **Why required:** A TCC is the identifier IRS uses for the transmitter. The IRS notes that the FIRE TCC and the IRIS TCC are separate applications. Current FIRE users must complete an IRIS Application for TCC, and the FIRE system is being retired.
- **Choose the transmission method carefully.** Applying with the A2A method when you need a different method, or the reverse, was a common error. Check the method before submitting.

### Step 3. Add two Responsible Officials
- **What:** List two ROs on the application. Each completes their own ID.me sign-in and sets their own 5-digit PIN when prompted.
- **Why required:** The IRIS TCC application requires two ROs. The PIN ties the TCC to a person who can act for the company.

### Step 4. Wait for TCC assignment and confirm
- **What:** IRS typically assigns the TCC within about 24 hours. Check the application status in the IRIS system.
- **Why required:** You cannot register an API Client ID without an active TCC. IRIS TCC is required to access the IRIS system and to submit testing.
- **Note:** If the TCC is not used for 3 consecutive years, IRS deletes it and you must reapply. A TCC is not renewed each year, but inactivity matters.

### Step 5. Confirm the business structure matches the TCC
- **Why this matters:** Your application uses EIN 01-0925870. The TCC is tied to the organization that registered it. One IRS answer said that if a TCC was registered with an EIN but a SSN is used on some forms, the registration may not match. Confirm the legal name and EIN on the TCC match the entity filing returns.

---

## Part 2: Register an API Client ID

You already started this in the screenshot (application saved, tracking number issued, no client IDs yet).

### Step 6. Create the API Client ID Application
- **What:** In the IRS portal, choose New Application, then API Client ID Application, then NEW API CLIENT ID.
- **Why required:** The IRIS A2A channel uses the API Client ID to authenticate and authorize access to IRIS. Anyone submitting through A2A must have one.

### Step 7. Set the API Client Info
- **API Label:** Any descriptive name (self-defined). Use something like `StableCustody-IRIS-Prod` or `-Test`.
- **Select API:** Check **IRIS**. Keep **SOR** selected (the screen shows SOR and IRIS checked).
- **Integration Type:** A2A.
- **API Status:** Active.
- **Why each matters:** The API label is how you identify the client later. Selecting IRIS scopes the client to IRIS only. A2A is the server-to-server path. Status Active allows it to be used.

### Step 8. Submit the JWKS
- **What:** Paste the JSON Web Key Set with at least one valid public RSA key. The JWKS is a JSON object with one property, `keys`, which is an array of JWK objects.
- **Why required:** The IRS uses this public key to verify the signature on the JWT your system sends. Without it, IRS cannot authenticate your requests.
- **Never upload the private key.** Only public key material goes here.
- **Save.** Save stores the application details. It does not create the client ID by itself. Click NEW API CLIENT ID to create the client ID record.

### Step 9. Retrieve the Client ID
- **What:** Sign in to the API Client ID Application and open the Application Details page. Copy the IRIS Client ID.
- **Why required:** This value is the `client_id` (or the `sub`/`iss` value, depending on the token request format **[verify]**) your app uses in every authentication request.
- **Important:** Customer ID is not the same as the API Client ID. The IRS states that the Customer ID is not the API Client ID.

### Step 10. Complete the A2A Consent process (Publication 5718)
- **What:** Enter your IRIS A2A Client ID on the A2A Authorization page and complete consent.
- **Why required:** Getting a client ID alone does not grant access. The IRS consent step connects your client ID to your TCC and authorizes the A2A channel.
- **Requirement:** The API application must be in Completed status for API, eFile, and TINM. Consenting back to the API application will not work. Check each status before trying to submit.

---

## Part 3: Build the JWT Private Key and JWKS Integration

### Why this flow
The IRS e-Services API guide describes token-based authentication under the OAuth authorization framework. Your app proves its identity by signing a JWT with a private key, and IRS verifies the signature using the public key you registered. The private key never leaves your system.

### Step 11. Generate an RSA key pair
- **Recommended:** RSA 2048-bit or larger. The IRS guidance specifies an RSA public key.
- **Why:** Your client ID is only as secure as the private key. RSA is the algorithm IRS expects.
- **Best practice:** Generate the key in a managed store (AWS KMS asymmetric key or an HSM) if your system supports signing without exporting the private key. The IRS guidance does not mention KMS **[verify]**, so confirm that your JWT signing library supports it.
- **Do not** reuse an SSL/TLS certificate key. A forum question in IRIS Q&A showed a problem with an SSL-based key. Use a dedicated signing key pair.

### Step 12. Build the public JWK with a `kid`
- **Include:** `kty` (RSA), `kid` (unique key ID), `alg` (RS256), `use` (sig), `n` and `e` (modulus and exponent).
- **Optional:** The e-Services guide also covers JWKs with `x5t` and `x5c` parameters, which carry a certificate fingerprint and chain. Use these if IRS requires a certificate **[verify]**.
- **Why required:** The `kid` lets IRS select the right public key when verifying your token. Without a `kid` match, verification fails.

### Step 13. Wrap the public JWK in a JWKS
```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "stable-custody-iris-2026-10",
      "use": "sig",
      "alg": "RS256",
      "n": "<base64url modulus>",
      "e": "AQAB"
    }
  ]
}
```
- **Why required:** The `keys` array is the format the form expects. Paste this JSON into the JWKS field, then save.

### Step 14. Sign a client assertion JWT
- **Header:** `alg: RS256`, `kid` (must match the JWKS `kid`), `typ: JWT`.
- **Payload claims [verify exact names in Pub 5718]:** issuer and subject set to your client ID, audience set to the IRS token endpoint, `iat` (issued at), `exp` (short expiry, a few minutes), and `jti` (unique ID to prevent replay).
- **Why required:** Each claim lets IRS confirm who you are, who the token is for, and that it is recent. Missing or mismatched claims cause an "invalid client" error.
- **Key rule:** The `kid` in the JWT payload must match the `kid` in the JSON key. Mismatch is a common cause of the "given JWT for client authentication is invalid" error.

### Step 15. Exchange the assertion for an access token
- **What:** POST the signed JWT to the IRS token endpoint using the client-credentials-with-JWT-assertion pattern **[verify endpoint and grant_type in Pub 5718]**.
- **Why required:** The access token is what authorizes IRIS API calls. Tokens are short-lived, so request a new one when needed.
- **Cache the token** until shortly before expiry. Requesting a new token on every submission adds latency and can hit rate limits.

### Step 16. Call the IRIS A2A endpoints
- **What:** Send the access token as a Bearer header with your submissions. Start with ATS (Application Testing System) testing before production.
- **Why required:** IRS requires ATS testing before a software ID moves from Test to Production. The October 2024 meeting noted a software ID moving from "Test" to "Production" status.
- **Software ID:** Each submission package needs a software ID. Confirm it is in Production before going live.

---

## Part 4: Operational Requirements

### Step 17. Store and rotate keys safely
- **Private key:** Store in KMS, HSM, or an encrypted secrets manager. Restrict access by IAM role. Never commit it to source control.
- **Rotation:** Add a new key with a new `kid` to the JWKS, switch signing to the new key, then remove the old one after it is no longer used. The exact IRS rotation procedure is **[verify]**.
- **Why:** A leaked private key lets anyone submit as your filer.

### Step 18. Manage authorized users
- **What:** The API Client ID Application has an Authorized Users tab.
- **Why:** Adding or removing users controls who can manage the client. Keep this list current when staff change.

### Step 19. Monitor TCC and API status
- **Check:** TCC status, API status (Active), and consent status.
- **Why:** Any one of these failing blocks submissions. A TCC unused for 3 years is deleted.

### Step 20. Plan for the FIRE retirement
- **Why:** FIRE is being retired. IRS targeted the end of December 2026, and later materials mention 2027. Plan to move all filings to IRIS, and make sure your software does not depend on FIRE.

---

## Part 5: Okta Setup (Internal Access Control)

Okta doesn't participate in IRS authentication. IRIS verifies only the JWT signed with the key you registered with IRS. Okta's job is to control which internal callers can reach your IRIS submission service. Keep the two key pairs separate, and never reuse the IRS signing key in Okta.

**Assumed architecture**

```
Internal caller --(Okta access token)--> IRIS submission service --(IRS-bound JWT)--> IRIS A2A
                                         (backend, IRS private key in KMS)
```

Menu names vary by Okta release and tenant. Confirm labels in your admin console.

### Step 21. Create or choose a custom authorization server
- **Where:** Security → API → Authorization Servers.
- **Why:** The submission service needs tokens with its own audience and scopes. A dedicated server keeps those separate from other apps. The default server works for a quick test, but a custom one gives a cleaner scope model.
- **Set:** Audience (for example `api://iris-submission`). Record the issuer URL.

### Step 22. Define scopes
- **Where:** Your authorization server → Scopes.
- **Examples:** `iris.submit`, `iris.status.read`.
- **Why:** Scopes limit who can submit. Only callers that need submission receive `iris.submit`.

### Step 23. Create an API Services application
- **Where:** Applications → Create App Integration → API Services.
- **Why:** This is Okta's app type for backend, client-credentials access without a user login.
- **Client authentication:** Public key / private key. This is the same pattern as the IRS side, but with a separate key pair.
- **Key:** Generate the key in Okta or register your own public key or JWKS URL. If Okta generates it, download the private key immediately and store it in your secrets manager or KMS. **[verify]** Okta may not show it again.

### Step 24. Set the key ID and rotation plan
- **Why:** Okta matches the `kid` in the caller's JWT header to the registered key. Use a clear naming scheme (for example `okta-svc-2026-10`) and document the rotation steps.

### Step 25. Add an access policy and rule
- **Where:** Your authorization server → Access Policies → Add policy → Add rule.
- **Rule settings:** Grant type Client Credentials, and assign the allowed scopes (`iris.submit`).
- **Why:** Without a rule that matches the client, Okta issues no token, even for a valid client.

### Step 26. Restrict which applications can use the server
- **Why:** Assign the app to the groups that need it. This limits the damage if a key leaks.

### Step 27. Store Okta credentials separately
- **Where:** Your secrets manager, kept apart from the IRS key.
- **Why:** A leak in one system shouldn't expose the other.

### Step 28. Validate tokens in the submission service
- **What:** On every request, verify signature, issuer, audience, expiry, and scope. Fetch signing keys from the authorization server's keys endpoint (`/v1/keys` under your server's path, **[verify]** exact URL in your tenant).
- **Why:** Okta protects the service only if the service checks each token it receives.

### What Okta does not do
- It does not register the IRS client ID.
- It does not sign the IRS assertion.
- It does not replace the TCC, ID.me, or A2A consent steps.

---

## Answer to "What is the best approach?"

1. Run the integration as a backend service, never in a browser. The private key must stay on your server.
2. Use a standard JWT library (for example `jose` in Node, `PyJWT` with `cryptography` in Python, or `nimbus-jose-jwt` in Java) to build the signed assertion.
3. Hold the private key in KMS or an HSM so the key is never exported.
4. Test end to end in ATS before production, then move the Software ID to Production.
5. Automate key rotation and token caching.
6. Protect the internal submission service with Okta (Part 5), using a separate key pair from the IRS key.

## Open questions to confirm with IRS
- Exact JWT claim names and the token endpoint URL (Publication 5718).
- Whether the x5c/x5t certificate form is required or optional.
- Key rotation procedure.
- Whether IRS accepts KMS-backed signing without a certificate.
- Whether Stable Custody Group II LLC can act as its own transmitter, or needs a transmitter partner.

## Links

**Primary IRS references**
- [Publication 5718: IRIS Electronic Application to Application (A2A)](https://www.irs.gov/pub/irs-pdf/p5718.pdf) — API Client ID, A2A consent, and JWT details. Start here for the [verify] items.
- [IRIS Application for TCC](https://www.irs.gov/tax-professionals/iris-application-for-tcc) — TCC application, Responsible Officials, and status checks.
- [Publication 5903: IRIS App for TCC](https://www.irs.gov/tax-professionals/iris-application-for-tcc) — TCC roles and the TCC tutorial (linked from the same IRS page; confirm the direct PDF link there).

**IRS e-Services API guidance**
- [IRS e-Services API Authorization User Guide (PDF)](https://content.govdelivery.com/attachments/USIRS/2022/11/21/file_attachments/2335011/IRS%20eServices%20API%20Authorization%20User%20Guide.pdf) — client ID, JWKs with `x5t` and `x5c`, RSA public keys.

**IRS IRIS working group materials**
- [IRIS Working Group Meeting, May 13, 2026](https://www.irs.gov/pub/irs-efile/iris-working-group-meeting-05132026.pdf) — current API Client ID setup steps and FIRE retirement.
- [IRIS 101 Q&A, August 5, 2025](https://www.irs.gov/pub/irs-efile/irs-101-q-and-a-08052025.pdf) — TCC, API Client ID, JWK, and FIRE retirement timing.
- [IRIS Working Group Q&A, June 11, 2025](https://www.eitc.irs.gov/pub/irs-efile/june-11-2025-iris-working-group-meeting-qa-updated.pdf) — Customer ID vs. API Client ID, TCC rules.
- [IRIS Working Group Q&A, July 9, 2025](https://www.stayexempt.irs.gov/pub/irs-efile/july-9-2025-iris-working-group-meeting-qas.pdf) — API Client ID steps, software developers and TCCs, 3-year TCC deletion.
- [IRIS Working Group Q&A, January 2025](https://irs.gov/pub/irs-efile/iris-working-group-qa-january%202025.pdf) — `kid` matching, Application Completed status, "invalid client" error.
- [IRIS Working Group Meeting, October 9, 2024](https://irs.gov/pub/irs-efile/iris-working-group-meeting-10092024.pdf) — API Client ID purpose, A2A consent, Software ID Test to Production.
- [IRIS Working Group Meeting, February 14, 2024](https://www.irs.gov/pub/irs-efile/iris-working-group-meeting-02142024.pdf) — TCC renewal, sole proprietor eligibility.
- [IRIS Working Group Meeting, January 17, 2024](https://www.irs.gov/pub/irs-efile/iris-working-group-meeting-january2024.pdf) — IRIS vs. FIRE TCC applications, Responsible Official sign-in.

**Third-party reference (JWKS format example)**
- [Oracle PeopleSoft: Generating a JSON Web Key for IRS Connection](https://docs.oracle.com/en/applications/peoplesoft/peopletools/8.63/security-administration/generate-json-web-key-irs-connection.html) — shows the JWK generation workflow for IRS A2A.

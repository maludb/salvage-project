# Auth

The ladder for authenticating users. Start at framework sessions; climb only
when a stated requirement forces it.

## framework-sessions

Your web framework's built-in session and cookie handling.

- **Use when:** you own the user table, support password (or single-method)
  login, and your framework's session primitives cover it.
- **Climb when:** you need flows the built-ins don't give you — password reset,
  email verification, MFA, OAuth logins, account linking.
- **Cost:** none beyond the framework. A login form and a session cookie; no
  extra dependency, no external service.
- **Rationale template:** `"chose framework sessions — {own user table},
  {single login method}, {built-ins sufficient}"`

## auth-library

A dedicated auth library handling flows, hashing, and providers in-process.

- **Use when:** you need standard flows (reset, verification, MFA, social
  login) but still want to own user data and run auth in your own app.
- **Climb when:** you need an external identity provider — SSO/SAML, enterprise
  directories, or you don't want to own credentials at all.
- **Cost:** a library dependency to configure and keep updated; you still own
  the user store and the security surface around it.
- **Rationale template:** `"chose auth library — {standard flows needed},
  {own user data}, {no external IdP}"`

## third-party-identity

A hosted identity service (Auth0, Clerk, Cognito, an SSO/SAML provider).

- **Use when:** you need SSO/SAML, enterprise directories, or want to offload
  credential storage and compliance to a managed provider.
- **Climb when:** (top of this ladder) — multi-tenant federation and custom
  identity infrastructure are a separate, later decision driven by stated needs.
- **Cost:** an external dependency and vendor lock-in to manage, plus integration
  and a recurring bill. Real burden — check it against the ceiling.
- **Rationale template:** `"chose third-party identity — {SSO/SAML required},
  {offload credentials}, {managed compliance}"`

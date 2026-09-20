# Authenticate to Authentik

A GitHub Action that exchanges the job's GitHub OIDC token for an Authentik
token, using the `client_credentials` grant with a `jwt-bearer` client
assertion. No client secret is involved: the assertion authenticates the client.

```yaml
permissions:
  contents: read
  id-token: write

steps:
  - uses: USA-RedDragon/action-authentik@v1
    id: authentik
    with:
      authentik-url: https://authentik.example.com
      client-id: my-provider
      scope: openid profile
```

The token is available as `steps.authentik.outputs.token` and is masked in logs.

## Inputs

| input | default | notes |
|---|---|---|
| `authentik-url` | required | Base URL, no trailing path |
| `client-id` | required | The provider's client ID |
| `scope` | `''` | See the note on `profile` below |
| `audience` | `''` | Audience for the GitHub OIDC token |
| `token-type` | `access_token` | `access_token` or `id_token` |

## Provider setup, and two things that fail quietly

On the Authentik OAuth2 provider:

- Enable the **client_credentials** grant.
- Add your GitHub Actions OAuth source under **Federated OIDC Sources**. This is
  the whole mechanism; with it empty the exchange fails with an authentication
  error that says nothing about federation.
- **Set a Signing Key.** With none, Authentik signs tokens HS256 using the client
  secret. Such a token carries no `kid` and the provider's JWKS endpoint serves
  `{}`, so any consumer that verifies through JWKS rejects it. This action emits a
  warning when it sees an HS256 token, because the failure otherwise surfaces far
  downstream.
- **Request `profile` in `scope`** if anything downstream keys on identity.
  Without it Authentik omits `preferred_username`, and a policy conditioned on a
  claim that is absent never matches: reads succeed and writes are denied, which
  looks like a permissions bug rather than a missing scope.

Authentik verifies the assertion's signature and expiry but **does not check
`aud`**, so audience is not a security boundary. Any token signed by a federated
source will authenticate. Scope the access downstream, on the identity the
property mapping produces.

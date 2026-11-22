# Password login (later phase, not in the current image)

The user wants a second authentication mode using pCloud username/password
for accounts without 2FA. This is a design note only; it is not implemented
yet, so the OAuth2 test can be evaluated first.

## Verified pCloud specification

There is **no** `/login` method. Login is done by passing the global
credentials to any method. The recommended login point is `userinfo`:

```
GET https://api.pcloud.com/userinfo?getauth=1&logout=1&username=<email>&password=<password>
```

Response contains:

```json
{ "result": 0, "auth": "AUTHENTICATION_TOKEN", ... }
```

The returned token is sent to subsequent calls as the global parameter
`auth=<token>` (OAuth2 tokens use the `Authorization: Bearer` header, which
our client already does). The client must therefore support **both** ways of
passing credentials.

### Optional digest mode
Instead of sending the password in clear text (only over TLS):

1. `GET /getdigest` → `digest`
2. `passworddigest = sha1(password + sha1(lowercase(username)) + digest)`
3. Login with `username`, `digest`, `passworddigest`, `getauth=1`.

We already have a streaming SHA-1 implementation, so this is easy to add.

### Token lifetime
- `authexpire` / `authinactiveexpire` set expiry in seconds.
- Tokens expire after inactivity; `listtokens` reports expiry.
- A 401 / result 1000 must trigger a clear "re-login" message.

## Planned client changes

- `PCloudAuth::LoginWithPassword(username, password, useDigest)`.
- Store an auth-type discriminator (OAuth2 vs password) with the token.
- `PCloudApi` sends either `Authorization: Bearer` or `auth=`, depending
  on the stored type.
- New command `pcloudctl login` that prompts for e-mail and password.
- Never log the password; log only the auth type and UID.
- Keep credentials in the existing 0600 settings file.
- Add a note in the docs that this mode is insecure if TLS is disabled and
  that 2FA accounts cannot use it.

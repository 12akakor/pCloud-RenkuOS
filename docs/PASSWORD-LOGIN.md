# Password login (shipped)

pCloud did not issue an OAuth2 client id for this project, so OAuth2 was
removed. Password login is the only mode, in both `pcloudctl` and the GUI.

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
`auth=<token>`. The password itself is kept in the Haiku key store, not in
the settings file.

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

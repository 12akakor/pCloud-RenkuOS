# Roadmap

## Phase 1 — read-only mount (current)

- [x] C++ project structure and Haiku style
- [x] OAuth2 configuration and token storage
- [x] HTTPS transport with `BSecureSocket`
- [x] JSON parser, SHA-1/SHA-256, base64url
- [x] Checksummed, atomic on-disk cache
- [x] FUSE low-level add-on with lookup/readdir/open/read/statfs
- [ ] CI build and test image via `test/pcloudfs`
- [ ] Tester feedback from `SYSLOG.TXT`

## Phase 2 — write support

- Write-ahead journal with CRC, replay on mount
- Upload to temporary remote name, verify checksum, then atomic rename
- Directory creation and rename using the same ordering rules
- Delete moves to a pending queue; local data removed only after remote
  confirmation
- Node monitor notifications so Tracker updates immediately

## Phase 3 — polish

- Preferences app with account, mount and cache settings
- Desktop volume icon and label via `FUSE_HAIKU_GET_DRIVE_INFO`
- Background refresh and conflict reporting
- Full test suite in CI

## Later — password login (no 2FA)

- [ ] Add `pcloudctl login` using `userinfo?getauth=1` (see [docs/PASSWORD-LOGIN.md](PASSWORD-LOGIN.md))
- [ ] Support `auth=` parameter in addition to Bearer tokens
- [ ] Optional digest mode with the existing SHA-1 implementation

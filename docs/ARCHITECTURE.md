# Architecture

## Why UserlandFS

A cloud filesystem must do HTTPS, JSON, OAuth2 and large streaming
transfers. None of that belongs in kernel code, and a kernel add-on cannot
use `BString`, `BMessage` or `BSecureSocket`. UserlandFS runs the filesystem
in a userspace server while exposing a real volume through the kernel VFS.
The result is a desktop icon like any other disk, with all code in ordinary
C++ using the native Be API.

## Layers

| Layer | Files | Responsibility |
|---|---|---|
| FUSE low-level add-on | `pcloudfs.cpp`, `PCloudFileSystem.*` | inode map, lookup, readdir, open/read; refuses writes |
| API | `PCloudApi.*` | `listfolder`, `checksumfile`, `getfilelink`, `userinfo` |
| Transport | `PCloudHttp.*` | HTTPS via `BSecureSocket`, redirects, chunked body, streaming to file |
| Authentication | `PCloudAuth.*` | OAuth2 tokens in a flattened `BMessage`, PKCE verifier |
| Integrity | `PCloudCrypto.*`, `PCloudCache.*` | SHA-1/SHA-256, base64url, atomic cache updates |
| Node model | `PCloudNode.*` | pCloud metadata as a portable record |

## Data integrity rules

1. A file is only exposed from the cache when its size and SHA-1 match the
   values returned by `checksumfile`.
2. Downloads go to a temporary file. The SHA-1 of the received bytes is
   computed while streaming; the file is renamed into place only after the
   checksum and size match.
3. Metadata and directory listings are written with write-temp/fsync/rename.
   A crash cannot produce a half-written record.
4. Phase 1 has no mutating operations at all. Write support will use a
   write-ahead journal before any remote mutation is attempted.

## Concurrency

`userlandfs_server` dispatches requests on several threads. All shared state
in `PCloudFileSystem` is guarded by `AutoLocker<BLocker>`. Network operations
hold that lock for the duration of the request; this serialises I/O, which is
acceptable for the first test image and keeps the state machine simple.

## Mount path

```
mount -t userlandfs -p pcloudfs /pCloud
```

`-p` selects the add-on named `pcloudfs` from
`/boot/system/add-ons/userlandfs`.

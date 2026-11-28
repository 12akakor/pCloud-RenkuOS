# pcloudfs

A native C++ pCloud filesystem for RenkuOS/Haiku. The filesystem is mounted as a
real volume through UserlandFS, so pCloud appears on the desktop like any other
disk. Browsing and reading files uses the native Be API; no FUSE port of C code
and no cloud SDK is used.

**Status: phase 1 (read-only) in development.** Writing is deliberately not
implemented yet: a read-only mount cannot corrupt remote data. See
[docs/ROADMAP.md](docs/ROADMAP.md).

## Design

```
Tracker / BFS / applications
            |
       kernel VFS
            |
      userlandfs kernel add-on
            |
   userlandfs_server process
            |
   libuserlandfs_fuse.so
            |
   pcloudfs add-on (C++)
      |        |        |
   Auth     Http      Cache
  (password) (BSecureSocket) (local, checksummed)
      \        |        /
             pCloud API
```

The add-on is a shared library loaded by `userlandfs_server`. It is written in
C++ and uses `BString`, `BMessage`, `BFile`, `BPath`, `BSecureSocket` and
`BNetworkAddress`. All remote data is verified with the SHA-1 reported by
pCloud before it is placed in the local cache. Cache updates are
write-temp/fsync/verify/rename, so a crash can never expose a half-written
file.

## Requirements

- A RenkuOS/Haiku tree with UserlandFS (kernel add-on, `userlandfs_server` and
  `libuserlandfs_fuse.so`).
- A pCloud account without two-factor authentication. Login uses your
  e-mail address and password.

## Building

The source tree is written to be dropped into a Haiku checkout. From the
`pcloudfs` directory (once the in-tree Jamfile has been included in the
surrounding build):

```
../configure --cross-tools-prefix <prefix>
jam -q pcloudfs pcloudctl
```

The GitHub Actions workflow builds the same targets with the RenkuOS
cross-compiler container.

## Installing and mounting

Copy the add-on to `/boot/system/add-ons/userlandfs/pcloudfs`, then:

```
pcloudctl login <email> <password>  # stores the auth token
mkdir /pCloud
mount -t userlandfs -p pcloudfs /pCloud
```

The volume shows up on the desktop. Files are downloaded on demand into
``/boot/home/config/cache/pcloudfs/`.

## Testing

See [docs/TESTING.md](docs/TESTING.md) and the step-by-step [docs/TEST-INSTRUCTIONS.md](docs/TEST-INSTRUCTIONS.md). The X270 test image writes the
kernel syslog and the end of `/boot/system/var/log/syslog` to `SYSLOG.TXT` on
the EFI (FAT) partition every ten seconds, so all diagnostics from this
add-on can be read on another computer.

## Coding style

Haiku coding guidelines are followed: tabs for indentation, 100-column lines,
InterCaps types and functions, `f`/`g`/`s`/`k` prefixes, `BString` over raw
strings, `status_t` for errors, and C++ casts. See
<https://www.haiku-os.org/development/coding-guidelines>.

## License

MIT. See [LICENSE](LICENSE).

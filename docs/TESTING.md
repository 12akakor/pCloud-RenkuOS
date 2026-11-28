# Testing

## Image build

The `test/pcloudfs` branch of `12akakor/Source` contains this source tree
plus the in-tree Jamfile hooks. Pushing to that branch triggers `x270-build`,
which builds a raw x86_64 image with:

- partition 1: EFI FAT partition with `BOOTX64.EFI` and a 512 KiB
  `SYSLOG.TXT`;
- partition 2: the Haiku BFS system volume.

The test kernel rewrites `SYSLOG.TXT` every ten seconds with the kernel
syslog and the tail of `/boot/system/var/log/syslog`. Messages from
`pcloudfs` are written with `syslog()` and therefore end up there.

The workflow publishes a `.zip` with the `.img` as a GitHub release asset so
it can be downloaded without a GitHub login.

## What to check on the machine

1. Boot the image. Confirm `pcloudfs` appears in
   `/boot/system/add-ons/userlandfs`.
2. Log in: `pcloudctl login <email> <password>`, or use the Account tab of
   the `pcloudfs_prefs` GUI app.
3. `pcloudctl status` — must show the account, UID and quota.
4. `mkdir /pCloud && mount -t userlandfs -p pcloudfs /pCloud`.
5. In Tracker: browse folders, open a file from each size class (small,
   several MiB), verify content with a checksum tool if available.
6. Try to delete or create a file: the operation must fail with "read-only"
   rather than silently doing nothing.
7. Read `SYSLOG.TXT` from the FAT partition on another computer and upload it
   together with your observations.

## Known limitations of phase 1

- Read-only. No mkdir, upload, rename or delete.
- Network operations are serialised behind one lock.
- Only the first host returned by `getfilelink` is used.
- Metadata is refreshed on a 60 second cache timeout, not by node monitor.

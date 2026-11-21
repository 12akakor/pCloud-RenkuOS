# Detailed test instructions (X270 image)

This document is written for the person holding the USB stick. Every command
is meant to be typed in a Terminal window on the booted RenkuOS image. All
results are also copied to `SYSLOG.TXT` on the FAT (EFI) partition, so the
log can be read on another computer.

## 1. Prepare

1. Download the `.zip` from the release asset link and unpack it.
2. Write the `.img` to a USB stick in **DD mode** (in Rufus choose "DD image",
   not ISO mode).
3. Boot the X270 from the stick (UEFI). The first boot may take a few minutes.

## 2. Confirm that the test tools are in the image

```
ls -l /boot/home/tests
```

You should see at least `pcloudfs_test`. Also check the add-on and server:

```
ls -l /boot/system/add-ons/userlandfs/pcloudfs
ls -l /boot/system/servers/userlandfs_server
```

## 3. Run the automated unit/integrity suite (no account needed)

``<arg_value:6124c78e>cd /boot/home/tests
./pcloudfs_test unit
```

This runs, without network access:

- SHA-1 and SHA-256 against published test vectors;
- streaming hashes split at 1, 7, 63, 64, 65 and 127 byte boundaries;
- JSON parsing of nested objects, arrays, escapes, UTF-16 surrogate pairs;
- rejection of seven malformed JSON inputs;
- cache metadata and directory-listing round trips;
- promotion of a verified download;
- rejection (and no data loss) when a download has the wrong SHA-1 or size.

Each check prints `PASS`, `FAIL` or `SKIP`. A full report is written to
`/boot/home/tests/pcloudfs-test-results.txt` and to the syslog.

**Expected:** no `FAIL` lines. If there are any, send the entire screen output
and the `SYSLOG.TXT` file.

## 4. Configure pCloud OAuth2

You need your own pCloud application credentials. They are **not** stored in
the repository and are not part of the image.

1. Register an application at <https://docs.pcloud.com/my_apps/> if needed.
2. Set its redirect URI to `http://localhost:8765/callback`.
3. In the Terminal:

   ```
   pcloudctl setup
   ```

4. Enter the client ID and client secret when asked.
5. A long URL is printed. Open it in WebPositive (or another computer), log in
   to pCloud and approve the application.
6. The setup tool listens on port 8765 and captures the redirect. On success
   it prints `Authorization complete` and your numeric UID.

If the browser runs on another machine, the redirect will not reach this
machine. In that case copy the `code=` value from the browser address bar
and tell me — I will add a manual-code option in the next image.

## 5. Check the account from the command line

```
pcloudctl status
```

Expected: `Configuration: yes`, `Access token: present`, your UID, the cache
path, your e-mail address and a non-zero quota.

## 6. Run the network suite

```
cd /boot/home/tests
./pcloudfs_test network
```

This checks `userinfo`, `listfolder` of the root, metadata validity, a
negative test with a non-existent folder id, and `checksumfile` for up to
three small files. It never modifies anything in pCloud.

## 7. Mount the volume

```
mkdir -p /pCloud
mount -t userlandfs -p pcloudfs /pCloud
```

The volume should appear on the desktop like an extra disk. Then:

```
ls -la /pCloud
```

Open a folder in Tracker and a small file in a text editor or media player.

## 8. Run the filesystem suite

``<arg_value:6124c78e>cd /boot/home/tests
./pcloudfs_test fs
```

This checks:

- `stat` of the mount point and of the first child;
- `opendir` / `readdir` of the volume root;
- opening a file read-only and reading bytes from it;
- that opening a file for writing fails with `EROFS`;
- that `mkdir` and file creation are refused and create nothing;
- `statvfs` reporting quota blocks.

**Expected:** every write attempt fails and no file or folder is created.

## 9. Run everything in one go

```
cd /boot/home/tests
./pcloudfs_test all
```

This executes all four suites in order and writes the final summary to
`/boot/home/tests/pcloudfs-test-results.txt`.

## 10. Send back the evidence

1. Copy `SYSLOG.TXT` from the FAT (EFI) partition to another computer.
2. Also copy `/boot/home/tests/pcloudfs-test-results.txt`.
3. Upload both, plus a short note with:
   - whether the X270 booted normally;
   - the exact commands that failed;
   - what Tracker showed on the desktop;
   - the image build number (visible in About this System).

## Troubleshooting

| Symptom | What to send |
|---|---|
| `mount` says the file system does not exist | `ls` of `/boot/system/add-ons/userlandfs` and the syslog |
| `pcloudctl status` shows no token | output of `pcloudctl setup`, syslog |
| Directory is empty but the account has files | syslog lines containing `API:` and `HTTP:` |
| Opening a file hangs | wait 60 s, then send `SYSLOG.TXT` (it is rewritten every 10 s) |
| Any `TEST FAIL` line | full `pcloudfs-test-results.txt` and `SYSLOG.TXT` |

# GUI plan (phase 3, not in the current image)

Everything works from the terminal today by design. That must keep working:
the CLI (`pcloudctl`) and the command-line tests are the automation and
diagnostics interface. The GUI is an additional front end, not a replacement.

## What already works through the GUI

Because pcloudfs is a real UserlandFS volume, Tracker already provides the
file-manager experience once phase 2 is enabled:

- browse folders and files as a normal disk on the desktop;
- open files with their preferred application;
- copy to/from other disks, drag and drop, context menus, icon previews;
- after phase 2: create folders, rename, delete, copy files — all through
  Tracker, because these are ordinary filesystem operations.

So "pCloud Drive-like browsing" needs no custom file manager.

## What still needs a GUI app

| Task | Today | Planned GUI |
|---|---|---|
| Enter e-mail and password | `pcloudctl login` | Preferences window, two text fields |
| Authorize | paste code in terminal | Open system browser + paste code field |
| Mount / unmount | `mount` command | Button in preferences + Deskbar menu |
| Account status / quota | `pcloudctl status` | Status view with quota bar |
| Cache size / pending ops | syslog | Status view + "Clear cache" button |
| Errors | `SYSLOG.TXT` | `BNotification` + status text |

## Native Be API design

A single `BApplication` (`pcloudfs_prefs`) with:

- `BWindow` + `BTabView`: `Account`, `Mount`, `Cache`, `About`.
- Account tab: `BTextControl` for client id/secret, `BButton` "Open browser",
  `BTextControl` for the authorization code, `BButton` "Authorize".
  The app calls the same `PCloudAuth::AuthorizeUrl()` and
  `ExchangeCode()` as the CLI; no embedded web view is needed — use
  `BUrl::OpenWithPreferredApplication()` to open the system browser.
- Mount tab: mount point `BTextControl`, Mount/Unmount `BButton`, current
  state from `statvfs()` and the volume roster (`BVolumeRoster`).
- Cache tab: path, size, `BButton` "Clear verified cache" (never clears
  pending journal records without confirmation).
- `BDeskbar::AddItem()` replica: small icon with a `BPopUpMenu` for
  status, mount/unmount, and "Open preferences".
- `BNotification` for auth failure, quota full, and pending operations.

## Architectural rule

The GUI must not contain filesystem or network logic of its own. It calls
`PCloudAuth`, `PCloudApi`, `PCloudCache` and (in phase 2) the journal —
the exact same classes the CLI and the add-on use. That keeps one
implementation of the integrity rules.

## Order

1. Finish and validate read-only mount + password login with the real account.
2. Phase 2 write support, validated read-only first, then enabled.
3. Preferences app (account + mount + status).
4. Deskbar replica and notifications.
5. Tracker integration polish (volume icon/label via the UserlandFS drive-info
   extension, preferred app for `pcloud://` links).

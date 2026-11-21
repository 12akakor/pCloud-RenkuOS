successfully downloaded text file (SHA: 88019650fc716cb7ef1d6156bfccc432aa76558e)
# Review notes (phase 1)

These items were found and corrected during static review of the first
implementation. They are recorded so the same mistakes are not repeated in
phase 2.

## Corrected before the first image build

| Problem | Fix |
|---|---|
| `BMessage::FromFlatBuffer()` and `Flatten(char**)` do not exist in the Haiku API | use `Unflatten()`, `FlattenedSize()` and `Flatten(buffer, size)` |
| `BFile::Fd()` is not public | download to a POSIX file descriptor from `open()` and `fsync()` it |
| `BDirectory::Rename()` does not exist | use `BEntry::Rename(newPath, true)` from the Storage Kit, which replaces atomically |
| `BObjectList(bool)` does not control ownership | ownership is the template parameter; default is owning |
| Missing `<errno.h>`, `<Entry.h>`, `<new>` includes | added |

## Integrity reasoning

- Every cache write is temp + `fsync()` + `rename()`; the previous version
  stays intact until the new one is complete.
- Downloaded bytes are hashed while streaming and compared with the SHA-1
  from `checksumfile` before promotion.
- `CachedFile()` re-verifies size and SHA-1 on every access. A corrupt cache
  file is never returned to the filesystem.
- Phase 1 refuses every mutating FUSE operation with `EROFS`, so no remote
  data can be changed.

## Remaining risks to verify with the tester

- The OAuth2 authorization URL may need adjustment to pCloud's exact
  authorize endpoint and redirect handling.
- The FUSE add-on name resolution depends on the add-on being installed in
  `/boot/system/add-ons/userlandfs`.
- `getfilelink` hosts may require the file name path exactly as returned.
- Serialised I/O means opening a large file blocks other filesystem requests.

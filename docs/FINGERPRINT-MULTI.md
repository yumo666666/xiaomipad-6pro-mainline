# Multiple fingers in one fingerprint template (experimental)

> **Status: experimental, not part of the packaged component.** The behaviour
> below was tested on a liuqin unit with a locally modified FPC1264 TOD driver,
> enrollment provider and an FP3-to-raw-database extractor. Those changes are
> not in the component shipped by this repository, which still refuses a second
> enrollment. The page records the design and the measured result for future
> work.

The FPC1264 stores its templates in an opaque database inside the TEE, not in
the file fprintd writes. That database has five slots, and the TA can append a
finger to an existing database. The problem is the libfprint/fprintd layer above
it, which assumes one template file per finger.

## Why one file per finger does not work

fprintd 1.94.5 `device.c: verify_start()` takes `gallery[0]` when the device does
not implement `IDENTIFY`, and retries only that template. With two template
files, a verify therefore always starts from the first finger; the second finger
never gets a chance to match, no matter how often it is presented.

Making the driver implement `IDENTIFY` is the other possible design, but the
`fpi_device_identify_report()` API allows one report per action, so a driver
cannot simply try the remaining databases and report the one that matched.

## One database, rewritten finger ID

The tested design keeps exactly one template file:

```text
enroll a new finger
  -> TOD driver captures the samples
  -> provider extracts the raw OEM database from the stored FpPrint
  -> oem_runtime --enrol-existing <raw-db> appends a slot in the TA
  -> driver rewrites the returned finger ID to the stored one (2)
  -> fprintd sees the same print and overwrites the same template file
```

fprintd's gallery therefore stays at one entry, so `verify_start()` loads the
database that contains every enrolled finger. Matching is still done by the TA:
the host never receives a template to compare.

The three pieces that differ from the packaged component are:

| Piece | Change |
| --- | --- |
| TOD driver | call `fp_print_set_finger()` with the stored finger ID after an append, so the print keeps its identity |
| Enrollment provider | use `--bio-pipeline-existing` / `--enrol-existing` instead of the single-finger paths, and stop refusing when a template already exists |
| Extractor | convert the stored FP3 container back to the raw `fpi-data` database before handing it to the TA; the TA rejects the FP3 container itself with `db_import_remaining_mismatch` |

The last point was the first failure mode: feeding the 772,578-byte FpPrint file
to the TA fails on the final 1458-byte fragment with `0xe853009f`, while the
772,495-byte raw database is accepted.

## Enrolling a second finger

With those pieces installed, the user workflow stays in Settings:

1. Close any open Settings window, then reopen **Settings → System → Users →
   Fingerprint Login**.
2. Authorize with the normal password dialog and scan the new finger, lift and
   reposition about 20 times.
3. The page still shows a single row, `left-index-finger`. That is expected:
   the new finger was appended to the same print and the stored finger ID was
   rewritten.
4. Lock the screen and unlock with either finger.

The template file grows from 772,578 bytes to about 1,162,014 bytes after the
second finger. The journal records the append and the canonical ID:

```text
OEM enroll database extended under stored finger 2 (requested finger 7)
native_enrol=OK database=extended canonical_finger=2
```

## Limits

- **Five slots.** The TA allocates five fingerprint slots; a sixth append fails
  with `0xe56004c4`. Enrolling the same finger twice consumes a slot.
- **One Settings row.** fprintd knows one print, so the UI shows the first
  finger's name for all of them. There is no per-finger status.
- **Delete means delete all.** Removing the row removes the whole database,
  including every appended finger. This is safer than leaving orphaned
  templates, but it is not selective.
- **The template changes on every successful unlock.** The OEM fprintd patch
  performs an adaptive update, so comparing file hashes is not a way to detect
  tampering or re-enrollment.
- **Settings claims the reader.** While the Fingerprint Login page is open,
  `gnome-control-center` holds the device; `sudo` and the lock screen may fall
  back to the password until it is closed.

## Rolling back

Restoring the packaged single-finger component also needs the matching template
backup; the tested procedure was:

```sh
systemctl stop fprintd
tar xzf /root/fpc-backup-2026-10-30-pre-round2/liuqin-fpc-oem-101725.tar.gz -C /usr/local/lib
systemctl start fprintd
```

Back up `/var/lib/fprint` and `/usr/local/lib/liuqin-fpc-oem` before
experimenting. Re-enrolling from scratch is the only supported way to remove an
appended finger.

## Provenance

The reverse engineering and the modified components were developed for this
tablet. The TA protocol work builds on the OEM component in this repository; no
vendor binaries are distributed here. The measured two-finger result is from the
10-27 build (`build=20261027-round2 database=per-account extract=1`).

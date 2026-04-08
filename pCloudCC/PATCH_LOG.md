# Patch Log — pCloudCC GCC Compilation Fixes

**Date:** 2026-04-08  
**Branch:** master  
**Build target:** `cmake . && make` (full build)  
**Result:** SUCCESS — all targets built (`sqlite3`, `pcloudcc_lib`, `pcloudcc`)

---

## Summary of Changes

### 1. `lib/pclsync/ptools.h` — line 89
**Problem:** `parse_os_path` declared with `char* delim` but callers passed char literals (`DELIM_SEMICOLON ';'`, `DELIM_DIR '/'`).  
**Fix:** Changed parameter type to `char delim`.

### 2. `lib/pclsync/ptools.c` — `parse_os_path` definition + other fixes
**Problems:**
- Function signature had `char* delim` (mismatch with callers).
- Line 180: `P_BOOL(charBuff, ...)` — missing array index, should be `charBuff[i]`.
- Line 415: `fName[k] = NULL` — assigning pointer to char; should be `'\0'`.
- `set_be_file_dates`: `msgErr` declared as `char[1024]` but `backend_call` expects `char**`; both call sites passed `msgErr` instead of `&msgErr`.
- Missing `#include <stdio.h>` for Linux build (caused implicit `sprintf` declaration).

**Fixes:**
- Changed `char* delim` → `char delim` in function definition.
- Fixed `charBuff` → `charBuff[i]`.
- Fixed `fName[k] = NULL` → `fName[k] = '\0'`.
- Changed `char msgErr[1024]` → `char* msgErr = NULL` and both usages to `&msgErr`.
- Added `#include <stdio.h>` inside `#if defined(P_OS_LINUX)` block.

### 3. `lib/pclsync/psynclib.c` — lines 2870, 2883, 2899, 2913, 2918

**Problems:**
- Line 2870: `psync_syncid_t syncId = (psync_syncid_t*)ptr` — cast pointer to uint32_t.
- Line 2883: `int eventId = (int*)ptr` — cast pointer to int.
- Lines 2899/2913/2918 (`psync_delete_sync_by_folderid`): `psync_syncid_t* syncId` used to hold a DB value; `syncId = row[0]` assigned `uint64_t` to pointer; `syncIdT = syncId` copied pointer instead of value into newly allocated memory.

**Fixes:**
- Line 2870: `*(psync_syncid_t*)ptr` (dereference the pointer).
- Line 2883: `(int)(uintptr_t)ptr` (value-in-pointer pattern, matching the call site in pfs.c).
- Lines 2899/2913/2918: replaced `psync_syncid_t* syncId` with `psync_syncid_t syncIdVal`; `syncId = row[0]` → `syncIdVal = (psync_syncid_t)row[0]`; `syncIdT = syncId` → `*syncIdT = syncIdVal`.

### 4. `lib/pclsync/pcallbacks.c` — line 438
**Problem:** `data_event_fptr = (data_event_callback*)ptr` — assigned `void*` cast to pointer-to-function-pointer to a function pointer variable.  
**Fix:** `(data_event_callback)ptr` — cast directly to function pointer type.

### 5. `lib/pclsync/pdiff.c` — line 700
**Problem:** `create_backend_event(..., res)` — passed `binresult*` as the `char** err` last argument. The `res` variable was already freed at this point.  
**Fix:** Pass `NULL` (the error value is not used after the call).

### 6. `lib/pclsync/pssl.h` + `lib/pclsync/pssl.c` — `psync_ssl_rsa_decrypt_symm_key_lock`
**Problem:** Function declared and defined with `psync_rsa_privatekey_t*` and `const psync_encrypted_symmetric_key_t*` (pointer-to-pointer types), but all callers pass the typedef values directly (which are already pointer types).  
**Fix:** Changed signature to match callers — `psync_rsa_privatekey_t rsa, const psync_encrypted_symmetric_key_t enckey` (by value, same as non-lock variant). Updated `pssl.c` definition accordingly.

### 7. `lib/pclsync/pp2p.c` — line 629
**Problem:** `psync_ssl_rsa_decrypt_symm_key_lock(psync_rsa_private, ekey)` — was previously reporting errors after the signature was mistakenly changed to pointer params. Reverted to original by-value call.  
**Fix:** Kept original call syntax (no `&` prefix) after signature fix.

### 8. `lib/pclsync/publiclinks.c` — forward declaration
**Problem:** `process_bres` is defined at the end of the file but called earlier; GCC treats implicit function declaration as error.  
**Fix:** Added forward declaration at top of file.

### 9. `lib/pclsync/pfs.c` — line 2443
**Problem:** `psync_run_thread1(..., PEVENT_BKUP_F_DEL_DRIVE)` — integer constant passed as `void*` argument.  
**Fix:** Cast to `(void*)(uintptr_t)PEVENT_BKUP_F_DEL_DRIVE`.

### 10. `lib/mbedtls` — build prerequisite
**Problem:** `libmbedtls.a` was not built; cmake build failed with missing target.  
**Fix:** Ran `cmake . -DCMAKE_C_FLAGS="-fPIC" && make` inside `lib/mbedtls` to produce the static library.

---

## Build Commands

```sh
# Step 1 — pclsync library
cd lib/pclsync && make clean && make fs

# Step 2 — mbedtls (prerequisite, if not already built)
cd lib/mbedtls && cmake . -DCMAKE_C_FLAGS="-fPIC" && make

# Step 3 — full pCloudCC build
cd pCloudCC && cmake . && make
```

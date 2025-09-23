# WA-SQlite reconcile effort

## Checkpoint 1: 55f7054 Merge branch 'reconcile/11' into reconcile/master

<details>
<summary>Files changed:</summary> 

.github/ISSUE_TEMPLATE/bug_report.md
Makefile
README.md
src/examples/IDBBatchAtomicVFS.js
src/exported_functions.json
src/extra_exported_runtime_methods.json
src/libvfs.c
src/libvfs.js
yarn.lock

</details>

Notes: 

Checkpoint 1 is a save point, applying minor updates before some (potentially) breaking changes.

- add `JSFILES` as deps for make rules for release targets (they're expected to be there, as part of git repo, but this is the update in the original `Makefile`, so I've included them here as well)
- IDBBatchAtomic - support asyncify interprocess communication
- minor updates to exported functions (add `_sqlite3_bind_parameter_index`)
- update `libvfs.c` - call vfs methods (`vfsRead`, `vfsWrite`, `vfsTruncate` with `i64` offset values, instead of pointers to those values -- I would assume some change in 64 bit legalization)
- update `libvfs.js`:
  - handle delegalization of `i64` params (passed as two `i32`s)
  - use string names to access properties on Module (`Module["method"]` instead of `Module.method`) to prevent issues with minifying of method names
  - allow async handling (by exposing `Module["handleAsync"]` if asyncify build)

## Checkpoint 2: 931a0d3 Merge branch 'reconcile/12' into reconcile/master

<details>
<summary>Files changed:</summary>

.github/workflows/ci.yml
.gitignore
.yarn/releases/yarn-3.1.1.cjs
.yarn/releases/yarn-4.0.2.cjs
.yarnrc.yml
Makefile
README.md
demo/ahp-contention.html
demo/ahp-contention.js
demo/ahp-demo.html
demo/ahp-demo.js
demo/ahp-worker.js
demo/benchmarks.js
demo/benchmarks/benchmark1.sql
demo/benchmarks/benchmark10.sql
demo/benchmarks/benchmark11.sql
demo/benchmarks/benchmark12.sql
demo/benchmarks/benchmark13.sql
demo/benchmarks/benchmark14.sql
demo/benchmarks/benchmark15.sql
demo/benchmarks/benchmark16.sql
demo/benchmarks/benchmark2.sql
demo/benchmarks/benchmark3.sql
demo/benchmarks/benchmark4.sql
demo/benchmarks/benchmark5.sql
demo/benchmarks/benchmark6.sql
demo/benchmarks/benchmark7.sql
demo/benchmarks/benchmark8.sql
demo/benchmarks/benchmark9.sql
demo/benchmarks/benchmarks.html
demo/benchmarks/benchmarks.js
demo/benchmarks/index.html
demo/clean-worker.js
demo/contention-sharedworker.js
demo/contention.html
demo/contention.js
demo/contention/contention-worker.js
demo/contention/contention.html
demo/contention/contention.js
demo/contention/index.html
demo/demo-worker.js
demo/demo.html
demo/demo.js
demo/file/index.js
demo/file/service-worker.js
demo/file/verifier.js
demo/hello.html
demo/hello.js
demo/index.html
demo/index.js
demo/retry/RetryVFS.js
demo/retry/retry-worker.js
demo/write-hint/index.html
demo/write-hint/index.js
demo/write-hint/worker.js
y/releases/yarn-4.0.2.cjs
y/sdks/integrations.yml
y/sdks/typescript/bin/tsc
y/sdks/typescript/bin/tsserver
y/sdks/typescript/lib/tsc.js
y/sdks/typescript/lib/tsserver.js
y/sdks/typescript/lib/tsserverlibrary.js
y/sdks/typescript/lib/typescript.js
y/sdks/typescript/package.json
yarn.lock
docs/assets/highlight.css
docs/assets/icons.css
docs/assets/icons.png
docs/assets/icons@2x.png
docs/assets/main.js
docs/assets/navigation.js
docs/assets/search.js
docs/assets/style.css
docs/assets/widgets.png
docs/assets/widgets@2x.png
docs/index.html
docs/interfaces/SQLiteAPI.html
docs/interfaces/SQLiteModule.html
docs/interfaces/SQLiteModuleIndexInfo.html
docs/interfaces/SQLitePrepareOptions.html
docs/interfaces/SQLiteVFS.html
docs/types/SQLiteCompatibleType.html
jsconfig.json
package.json
src/FacadeVFS.js
src/VFS.js
src/WebLocksMixin.js
src/asyncify_exports.json
src/asyncify_imports.json
src/examples/AccessHandlePoolVFS.js
src/examples/ArrayAsyncModule.js
src/examples/ArrayModule.js
src/examples/IDBBatchAtomicVFS.js
src/examples/IDBContext.js
src/examples/IDBMinimalVFS.js
src/examples/IDBVersionedVFS.js
src/examples/MemoryAsyncVFS.js
src/examples/MemoryVFS.js
src/examples/OPFSAdaptiveVFS.js
src/examples/OPFSCoopSyncVFS.js
src/examples/OPFSPermutedVFS.js
src/examples/OriginPrivateFileSystemVFS.js
src/examples/README.md
src/examples/WebLocks.js
src/exported_functions.json
src/libadapters.h
src/libadapters.js
src/libauthorizer.c
src/libauthorizer.js
src/libfunction.c
src/libfunction.js
src/libmodule.c
src/libmodule.js
src/libprogress.c
src/libprogress.js
src/libvfs.c
src/libvfs.js
src/main.c
src/sqlite-api.js
src/sqlite-constants.js
src/types/index.d.ts
test/AccessHandlePoolVFS.test.js
test/IDBBatchAtomicVFS.test.js
test/MemoryAsyncVFS.test.js
test/MemoryVFS.test.js
test/OPFSCoopSyncVFS.test.js
test/OPFSPermutedVFS.test.js
test/OriginPrivateVFS.test.js
test/TestContext.js
test/WebLocksMixin.test.js
test/api.test.js
test/api_exec.js
test/api_misc.js
test/api_statements.js
test/callbacks.test.js
test/obsolete/GOOG.js
test/obsolete/IDBBatchAtomicVFS.test.js
test/obsolete/IDBMinimalVFS.test.js
test/obsolete/MemoryAsyncVFS.test.js
test/obsolete/MemoryVFS.test.js
test/obsolete/OPFSProxy.js
test/obsolete/OPFSWorker.js
test/obsolete/OriginPrivateFileSystemVFS.test.js
test/obsolete/VFS.test.js
test/obsolete/VFSTests.js
test/obsolete/WebLocks.test.js
test/obsolete/api-instances.js
test/obsolete/module.test.js
test/obsolete/sqlite-api.test.js
test/obsolete/tag.test.js
test/sql.test.js
test/sql_0001.js
test/sql_0002.js
test/sql_0003.js
test/sql_0004.js
test/sql_0005.js
test/test-worker.js
test/vfs_xAccess.js
test/vfs_xClose.js
test/vfs_xOpen.js
test/vfs_xRead.js
test/vfs_xWrite.js
typedoc.json
web-test-runner.config.mjs


</details>

Notes:

Includes a large change in original repo (merge of `dev` branch -> v1.0)

- bump emscripten version `3.1.45` -> `3.1.47` (done in CI job, but also applicable to wrapping processes, e.g. `crsqlite-wasm`)
- some changes re yarn (too many to break down, shouldn't really affect the cr-sqlite usage)
- updates to demo (too many to break down, shouldn't really affect the cr-sqlite usage)
- updates to docs (too many to break down, shouldn't really affect the cr-sqlite usage)
- updates to tests (too many to break down, shouldn't really affect the cr-sqlite usage)

- Makefile:
  - add `ASYNCIFY_EXPORTS` (asyncifying methods exported from SQLite to JS glue)
  - add JSPI support
  - refactor JS libraries flag: single library (`libadapters.js`), multiple post js files (all the rest)

- VFS updates:
  - add FacadeVFS - a scaffold for VFS implementations, exposes `xMethod`s and ports them to `jMethods` (e.g. `xRead` calls to `jRead` on JS implemented VFS)
  - add WebLocksMixin - a mixin implementing SQLite VFS locking protocol for FacadeVFS (providing defaults for all VFSes extending FacadeVFS and not implementing locking overrides)
  - replace the `xMethod` (e.g. `xRead`) in the JS interface with `jMethod` (e.g. `jRead`) -- `FacadeVFS` handles the porting from one to the other
  - switch from `new VFSImplementation()` to `VFSImplementation.create()` as prefered way to initialise an instance of VFS adapter (e.g. `IDBBatchAtomic.create(...)`)
  - remove VFS adapter: IDBMinimalVFS
  - remove VFS adapter: IDBVersionedVFS
  - remove VFS adapter: OriginPrivateFileSystemVFS
  - add VFS adapter: OPFSAdaptiveVFS
  - add VFS adapter: OPFSCoopSyncVFS
  - add VFS adapter: OPFSPermutedVFS

- major change in the way JS functions get called by WASM code:
  - replace explicitly exposed JS functions (e.g. `xRead`, callbacks for `updateHook`, etc.) with function signatures, e.g. `ipippj`:
    - a function returning `i32`
    - 1 `p` (pointer for function pointer)
    - accepting `ppij` as parameters: (pointer, pointer, i32, i64)
  - all JS functions "exposed" to the WASM code are no exposed directly, but rather registered with the central lookup table (on the JS side) and a key (i32) passed to the WASM code
  - when WASM code calls to a function, it does so by the registered (i32) key and params the function signature expects
  - every signature also exposes the async version (e.g. `ipippij_async` for `ipippij`) with async flags passed to the WASM code when registering the function
  - `libadapters.js` and `libadapters.h` define both sides of cross environment function calls
  - following this update, I've refactored the way `updateHook` (defined by Matt) communicates with the WASM code to stick to the new convention (this gets replaced downstream as update hook is defined in the original repo as well)

- updates to SQLiteAPI (a wrapper around WASM glue code):
  - remove `.declare_vtab` - seemed unused (and not present in the original `wa-sqlite` repo)
  - remove `.user_data` - seemed unused (and not present in the original `wa-sqlite` repo)
  - move `.prepare_v2`, `.str_new`, `.str_apendall`, `.str_finish`, `.str_value` to a backwards compatibility section:
    - all of these functions have been replaced (with `.statements` method) in the original `wa-sqlite` code, but kept as wrappers from `vlcn.io/js` still depend on them being exposed

## Checkpoint 3: ac43e2e Merge branch 'reconcile/21' into reconcile/master

<details>
<summary>Files changed:</summary>

README.md
demo/demo-worker.js
demo/file/index.js
demo/hello.html
demo/hello.js
demo/hello/README.md
demo/hello/hello.html
demo/hello/hello.js
demo/hello/index.html
package.json
src/WebLocksMixin.js
src/examples/OPFSAnyContextVFS.js
src/examples/README.md
test/IDBBatchAtomicVFS.test.js
test/OPFSAdaptiveVFS.test.js
test/OPFSAnyContextVFS.test.js
test/api.test.js
test/data/idbv5.json
test/obsolete/GOOG.js
test/obsolete/IDBBatchAtomicVFS.test.js
test/obsolete/IDBMinimalVFS.test.js
test/obsolete/MemoryAsyncVFS.test.js
test/obsolete/MemoryVFS.test.js
test/obsolete/OPFSProxy.js
test/obsolete/OPFSWorker.js
test/obsolete/VFSTests.js
test/obsolete/WebLocks.test.js
test/sql.test.js
test/test-worker.js
yarn.lock

</details>

Notes:

This was a milestone for us as we wanted to be able to use OPFSAnyContextVFS, so I've marked a checkpoint here. 
The patch adds OPFSAnyContextVFS implementation, along with some updates (testing mostly)

## Checkpoint 4: 7591a62 Merge branch 'reconcile/35' into reconcile/master

<details>
<summary>Files changed:</summary>

Makefile
README.md
demo/contention/contention-worker.js
demo/demo-worker.js
demo/hello/hello.html
demo/hello/hello.js
docs/assets/search.js
docs/interfaces/SQLiteAPI.html
package.json
src/asyncify_imports.json
src/examples/IDBMirrorVFS.js
src/examples/OPFSPermutedVFS.js
src/examples/README.md
src/libadapters.h
src/libadapters.js
src/libfunction.c
src/libfunction.js
src/libhook.c
src/libhook.js
src/sqlite-api.js
src/types/index.d.ts
test/IDBMirrorVFS.test.js
test/TestContext.js
test/api.test.js
test/api_statements.js
test/callbacks.test.js
test/sql.test.js
test/test-worker.js
yarn.lock

</details>

Notes:

I can't remember why this was a checkpoint, probably due to SQLite version bump:
- bump SQLITE_VERSION `3.45.0` -> `3.46.0`
- add `libhook.c` and `libhook.js` - replacing `.updateHook` (previously in libfunction) with `.update_hook`
- add VFS adapter: IDBMirrorVFS

## Final: 9c3f7cb Merge branch 'master' into reconcile/master

<details>
<summary>Files changed:</summary>

.github/ISSUE_TEMPLATE/-do-not-post-anything-other-than-a-bug-report.md
.github/workflows/ci.yml
Makefile
README.md
demo/demo.js
demo/file/index.js
demo/file/service-worker.js
demo/file/verifier.js
demo/hello/hello.js
docs/assets/search.js
docs/interfaces/SQLiteAPI.html
package.json
src/FacadeVFS.js
src/WebLocksMixin.js
src/examples/IDBBatchAtomicVFS.js
src/examples/IDBMirrorVFS.js
src/examples/MemoryVFS.js
src/examples/OPFSAdaptiveVFS.js
src/examples/OPFSCoopSyncVFS.js
src/examples/OPFSPermutedVFS.js
src/examples/README.md
src/extra_exported_runtime_methods.json
src/jspi_exports.json
src/libadapters.js
src/libhook.c
src/libhook.js
src/sqlite-api.js
src/types/globals.d.ts
src/types/index.d.ts
test/api_statements.js
test/callbacks.test.js
web-test-runner.config.mjs
yarn.lock

</details>

Notes:

- bump Emscripten version `3.1.47` -> `3.1.61`
- bump SQLITE_VERSION `3.45.0` -> `3.50.1`
- add `.commit_hook` method to the API

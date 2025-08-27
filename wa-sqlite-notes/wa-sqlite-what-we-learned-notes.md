# CR-SQLite what we've learned: building from wa-sqlite

- CR-SQLite uses `wa-sqlite` as a base: a non-official wrapper around SQLite compiled to WebAssembly
- `wa-sqlite` was developed and is actively maintained by Roy Hashimoto in `rhashimoto/wa-sqlite` repo
- `@vlcn.io/wa-sqlite` is a fork of `rhashimoto/wa-sqlite` (hosted at `vlcn-io/wa-sqlite`) with some additional wrappers on top
- cr-sqlite uses the built WASM binary (potentially binaries) to initialise the WASM module, as well as the JS API wrapping the JS module of `wa-sqlite`
- cr-sqlite wraps the JS API further (in `@vlcn.io/crsqlite-wasm`), providing a high level JS API to interact with the DB

## rhashimoto/wa-sqlite

- `wa-sqlite` builds SQLite from source (C amalgamation) for WebAssembly using `emscripten`
- `emscripten` spits out WASM binary as well as JS glue code for every build
- `wa-sqlite` features three Emscripten builds: sync, async (asyncify), JSPI (newer async -- experimental)
- in addition to the core C code, `wa-sqlite` provides certain libs (most often on both sides: C and JS) to make the JS <-> C interop more convenient, e.g.:
  - `libhook.c` exposes a function:
  ```c
	void EMSCRIPTEN_KEEPALIVE libhook_update_hook(sqlite3* db, int xUpdateHook, void* pApp) {
		sqlite3_update_hook(db, xUpdateHook ? &libhook_xUpdateHook : NULL, pApp);
	}
  ```
  - while the WASM build exposes all (relevant*) SQLite functions for JS usage for calling using `ccall` or `cwrap`, e.g.:
	```js
	ccall("sqlite3_update_hook", ["number", ["number", "number", "number"], [db, updateHookPointer, pApp])
	```
	`libhook.c` exposes `libhook_update_hook` which wraps `sqlite3_update_hook` and can be called from JS like so:
	```js
	ccall("libhook_update_hook", ["void", ["number", "number", "number"], [db, updateHookPointer, pApp])
	```
	Notice that the function call is nearly the same as calling `sqlite3_update_hook`, however, function calling in between environments (JS <-> WASM) is not trivial, especially unnamed functions, like callbacks (and I'm not sure it would even work in this case without some additional sugar), that's why, the `libhook.c` defines additional functions (defined in the file, in C context) for easier passing to (e.g.) `sqlite3_update_hook`, which we won't get into too much here, see more below.
  - `libhook.js` defines the JS side of this communication, e.g.:
  ```js
	// NOTE: this is a simplified example, see the full example using actual source below
	(function() {
	  Module['update_hook'] = function(db, xUpdateHook) {
		const key = Math.floor(Math.random()) * (2^32) // u32
		const hasCallback = Number(Boolean(xUpdateHook)) // true = 1

		// Register callback: not getting in to this, see below for full breakdown

	    ccall('libhook_update_hook', 'void', ['number', 'number', 'number'], [db, hasCallback, key]);
	  };
	})();
  ```
  Now, as stated, there's some additional magic around registering `xUpdateHook` (and facilitating calling back into it), but for the purpose of this example, it's apparent how `Module.update_hook` calls to C-defined `libhook_update_hook`, passing the DB pointer (number), a flag indicating whether a callback is passed (number) and a key (number) to identify the callback later on.
  - this, in a nutshell is how `lib*.(c|js)` fills bridge the gap between JS and C (WASM) contexts
  - those libs are passed to `emcc` during the build, so the build process:
    - compiles the C amalgamation
	- compiles the `lib*.c` files
	- links them
	- produces the core glue code (internally)
	- extends the glue code: runs `lib*.js` files one after the other (hence the IIFE including the additional method assignment `(function() { Module["update_hook"] = {...} })()`)
- finally, `wa-sqlite` exposes a `sqlite-api.js` - a JS code which isn't passed through escripten, but is to be used as is to wrap the JS glue code:
  ```js
    // Import the JS glue factory
	import ModuleFactory from "dist/wa-sqlite-async.mjs"
	// Import the WASM binary as URL
	import wasmUrl from "dist/wa-sqlite-async.wasm?url"
	// Import the JS API wrapper
	import SQLiteAPI from "src/sqlite-api.js"

	const module = ModuleFactory({ locateFile() { return wasmUrl }})
	// Wrap the JS glue code in the high level API
	const api = SQLiteAPI.Factory(module)
  ```

## @vlcn.io/wa-sqlite

- this is a fork of `rhashimoto/wa-sqlite` with some additional functionality
- the base is the same (see previous section)
- additionally, the package extends the build process by loading the Rust-based `crsql` extension, updated process:
	- compile the `crsql` extension from Rust source (into a static lib)
    - append `crsqlite.c` (initialising the `crsql` code) file to the amalgamation (producing: `sqlite3-extra.c`):
	- use `sqlite3-extra.c` as the full amalgamation (and compile)
	- compile the `lib*.c` files
	- link everything (incl. `crsql` exension -- thanks to `crsqlite.c`)
	- produce the core glue code (internally -- Emscripten -- same as in `wa-sqlite`)
	- extend the glue code: same as with `wa-sqlite`
- one small distinction: the output files are named differently in `@vlcn.io` version:
  - `wa-sqlite.(wasm|mjs)` -> `crsqlite-sync.(wasm|mjs)`
  - `wa-sqlite-async.(wasm|mjs)` -> `crsqlite.(wasm|mjs)` (asyncify build - main one being used in `vlcn-io` chain)
  - `wa-sqlite-jspi.(wasm|mjs)` -> `crsqlite-jspi.(wasm|mjs)` (JSPI build -- experimental in `rhashimoto/wa-sqlite`, we build it in `vlcl-io/wa-sqlite`, but haven't yet tested it, NOTE: when I say we, I mean our fork of `wa-sqlite`, brought up-to-date with `rhashimoto/wa-sqlite`)

## @vlcn.io/crsqlite-wasm: how the JS API gets built to the high level one we use

- `@vlcn.io/crsqlite-wasm` depends on `@vlcn.io/wa-sqlite` and provides some wrappers around the JS glue and the wrapping SQLiteAPI:
  - it provides initialisation abstraction:
	```ts
	import { initWasm } from "@vlcn.io/crsqlite-wasm"
	import wasmUrl from "@vlcn.io/wa-sqlite/dist/crsqlite.wasm"

	const db = await initWasm(() => wasmUrl)
	```
  - when initialised, the JS interface hirarchy is the following: `crsqlite-wasm`-defined `DBAsync` --wraps--> `SQLiteAPI` --wraps--> `wa-sqlite` JS glue --produced by--> Emscripten (core glue) + `lib*.js` (js libs defined in `wa-sqlite` as inputs to build process)
  - to show an example:
  ```ts
  // top level (crsqlite-wasm defined DB)
  const db = await initWasm(() => wasmUrl)
  // Every time an update happens, log a message to the console
  const handleUpdate = () => { console.log("db updated") }
  db.onUpdate(handleUpdate)

  // crsqlite-wasm (src/DB.ts)
  class DB implements DBAsync {
	// ...
	api: SQLiteAPI

	onUpdate(cb: (type: UpdateType, dbName: string, tblName: string, rowid: bigint) => void): () => void {
		if (this.#updateHooks == null) {
			this.api.update_hook(this.db, this.#onUpdate);
			this.#updateHooks = new Set();
		}
		this.#updateHooks.add(cb);

		return () => this.#updateHooks?.delete(cb);
	}
  }

  // SQLiteAPI (wa-sqlite/src/sqlite-api.js)
  export function Factory(Module) {
	sqlite3.update_hook = function(db, xUpdateHook) {
  	  verifyDatabase(db);

  	  // Convert SQLite callback arguments to JavaScript-friendly arguments.
	  // (this handles the difference between "raw" libhook.js <-> libhook.c communication to friendlier JS api)
  	  function cvtArgs(iUpdateType, dbName, tblName, lo32, hi32) {
  	    return [iUpdateType, Module.UTF8ToString(dbName), Module.UTF8ToString(tblName), cvt32x2ToBigInt(lo32, hi32];
  	  };
  	  function adapt(f) {
  	    return f instanceof AsyncFunction ?
  	      (async (iUpdateType, dbName, tblName, lo32, hi32) => f(...cvtArgs(iUpdateType, dbName, tblName, lo32, hi32))) :
  	      ((iUpdateType, dbName, tblName, lo32, hi32) => f(...cvtArgs(iUpdateType, dbName, tblName, lo32, hi32)));
  	  }

  	  Module.update_hook(db, adapt(xUpdateHook));
  	};
  }

  // wa-sqlite/src/libhook.js
  (function() {
    const AsyncFunction = Object.getPrototypeOf(async function(){}).constructor;
    let pAsyncFlags = 0;

    // Register the method to the WASM glue module
    Module['update_hook'] = function(db, xUpdateHook) {
      if (pAsyncFlags) {
        Module['deleteCallback'](pAsyncFlags);
        Module['_sqlite3_free'](pAsyncFlags);
        pAsyncFlags = 0;
      }

      pAsyncFlags = Module['_sqlite3_malloc'](4);
      setValue(pAsyncFlags, xUpdateHook instanceof AsyncFunction ? 1 : 0, 'i32');

      ccall(
        'libhook_update_hook',
        'void',
        ['number', 'number', 'number'],
        [db, xUpdateHook ? 1 : 0, pAsyncFlags]);
      if (xUpdateHook) {
        Module['setCallback'](pAsyncFlags, (_, iUpdateType, dbName, tblName, lo32, hi32) => {
          return xUpdateHook(iUpdateType, dbName, tblName, lo32, hi32);
        });
      }
    };
  })();
  ```

## Function signature as function name: JS <-> C (WASM) interop designed in a reusable and generic manner

- `wa-sqlite` uses function signatures as function names in the JS side of things
- this is used for all C -> JS calls (callbacks, VFS interface, etc.)
- `libadapters.js` defines these function names (e.g. ``):
	```ts
	const SIGNATURES = [
  	  'ipp', // xProgress, xCommitHook
  	  'ippp', // xClose, xSectorSize, xDeviceCharacteristics
  	  // ...
  	  'ipppiiip', // xShmMap
  	  'vppippii', // xUpdateHook
  	];

  	// Define each of the function from SIGNATURES e.g., for 'ipp' we'd get (NOTE: this is the oversimplification of what libadapters.js actually does):
  	function ipp(pApp: number): number
  	// Only in asynficy (and maybe JSPI) builds (I'm not sure how JSPI handles async)
  	async function ipp_async(pApp: number): number
  	```
- on the C side, `libadapters.h` defines:
	```c
	// Define helpers

	// Pointer
	#define P const void *
	// int = i32
	#define I int
	// int64 - this is what we see as 2 i32s when being passed to JS side
	#define J int64_t
	// A macro for declaring an extern function type
	// extern meaning:
	//	- this function is defined somewhere else,
	//	- it's safe to call it (as it will be provided by compiler/linker)
	//	- it's actually a JS function which gets bridged to C during Emscripten compilation/linking
	//
	// Example of this macro:
	// DECLARE(I, ipp, P, P)
	//   - TYPE = I
	//   - NAME = "ipp"
	//   - __VA_ARGS__ = P, P
	// This expands to:
	// extern I ipp(P, P) -- equivalent to 'int ipp(const void *, const void *)'
	// extern I ipp_async(P, P) -- equivalent to 'int ipp_async(const void *, const void *)'
	#define DECLARE(TYPE, NAME, ...)                                               \
	  extern TYPE NAME(__VA_ARGS__);                                               \
	  extern TYPE NAME##_async(__VA_ARGS__);

	// Use helpers (macro and type aliases) to declare extern function types, e.g.:
	DECLARE(I, ipp, P, P);
	DECLARE(I, ippp, P, P, P);
	// ...
	DECLARE(I, ipppiiip, P, P, P, I, I, I, P);
	DECLARE(void, vppippii, P, P, I, P, P, I, I);
	```
	With these definitions, the C code can call to a JS function, like e.g.:
	```c
	void *p1, *p2;
	int rc = ipp(pApp, p1);
	```
- to address the elephant in the room: what does this signature mean
	- `v` - void (used only for return)
	- `i` - int (i32)
	- `j` - sqlite3_int64 (passed as two i32s) -> `j` == `ii`
	- `p` - `const void *` pointer (this is essentially a pointer pointing to an unknown type -- needs to be cast to a type in order to be used: quite unsafe)
	- so, e.g. `vppippj` would be declared as:
		```ts
		/**
		 * @param key - pointer -- passed as a number
		 * @param p1 - pointer -- passed as number
		 * @param i1 - int (32 bit) -- regular number
		 * @param p2 - pointer -- passed as number
		 * @param p3 - pointer -- passed as number
		 * @param j1lo - lower 32 bits of sqlite3_int64 -- passed as int (32)
		 * @param j1hi - higher 32 bits of sqlite3_int64 -- passed as int (32)
		 */
		function vppippj(key: number, p1: number, i1: number, p2: number, p3: number, j1lo: number, j1hi: number): void
		```
- a caveat: when a JS callback is defined as (e.g.) `vpii`, the C counterpart has to call to `vppii` (adding extra `p` as the first param):
	```ts
	// vpippii
	type UpdateHookCb = (pApp: number, updateType: number, dbName: number, tblName: number, rowLo: number, rowHi: number) => void
	Module["update_hook"] = function (cb: UpdateHookCb): void
	```
  However, the C code calls to `vppippii` (additional `p`):
	```c
	static void libhook_xUpdateHook(void* pApp, int iUpdateType, const char* dbName, const char* tblName, sqlite3_int64 rowid) {
		int hi32 = ((rowid & 0xFFFFFFFF00000000LL) >> 32);
		int lo32 = (rowid & 0xFFFFFFFFLL);
		const int asyncFlags = pApp ? *(int *)pApp : 0;
		vppippii(pApp, pApp, iUpdateType, dbName, tblName, lo32, hi32)
	}
  ```
  This is because the first param (`p`) is the pointer to the function, the `wa-sqlite` code utilises that pointer part to make a pseudo-pointer: uses it as a key for the lookup table
- the full flow (using `update_hook` as an example is as follows):
	- we register a callback to `onUpdate`:
	```ts
	// Top level (app code)
	const handleUpdate = (
		updateType: number,
		dbName: string,
		tblName: string,
		rowid: number
	) => console.log(`update: type: ${updateType}, dbName: ${dbName}, tblName: ${tblName}, rowid: ${rowid}`)
	db.onUpdate(handleUpdate)
	```
	- this gets passed to `crsqlite-wasm`-defined DB, which, aside from additional reuse (multicasting), passes it along to `SQLiteAPI.update_hook`
	- `SQLiteAPI` modifies the args (wraps a function to adapt to C interop):
	```ts
	sqlite3.update_hook = function(db, xUpdateHook) {
		verifyDatabase(db);

		// Convert SQLite callback arguments to JavaScript-friendly arguments.
		function cvtArgs(iUpdateType, dbName, tblName, lo32, hi32) {
			return [
				iUpdateType,
				Module.UTF8ToString(dbName),
				Module.UTF8ToString(tblName),
				cvt32x2ToBigInt(lo32, hi32)
			];
		};
		function adapt(f) {
			return f instanceof AsyncFunction ?
				(async (iUpdateType, dbName, tblName, lo32, hi32) => f(...cvtArgs(iUpdateType, dbName, tblName, lo32, hi32))) :
				((iUpdateType, dbName, tblName, lo32, hi32) => f(...cvtArgs(iUpdateType, dbName, tblName, lo32, hi32)));
		}

	    Module.update_hook(db, adapt(xUpdateHook));
	  };;
	```
	- finally, `libhook.js`-defined `Module.update_hook` gets called:
	```ts
	Module['update_hook'] = function(db, xUpdateHook) {
		if (pAsyncFlags) {
			Module['deleteCallback'](pAsyncFlags);
			Module['_sqlite3_free'](pAsyncFlags);
			pAsyncFlags = 0;
		}

		// A clever way of getting a unique int (32) to use as a key for the callback in the lookup table
		// Get a pointer (address -- next available offset in the linear memory)
		// Additionally, store the async flags (is async -- 1 or 0) in the allocated memory
		pAsyncFlags = Module['_sqlite3_malloc'](4);
		setValue(pAsyncFlags, xUpdateHook instanceof AsyncFunction ? 1 : 0, 'i32');

		ccall(
			'libhook_update_hook',
			'void',
			['number', 'number', 'number'],
			// Params:
			// - db - pointer to the DB (memory address)
			// has update hook - (1 or 0)
			// pAsyncFlags - address of async flags on the heap - used as both callback key as well as to read async flags on the WASM side
			[db, xUpdateHook ? 1 : 0, pAsyncFlags]);

		// If we have a callback, register it in the lookup table
		// adding entry: [pAsyncFlags] => xUpdateHook
		if (xUpdateHook) {
			Module['setCallback'](pAsyncFlags, (_, iUpdateType, dbName, tblName, lo32, hi32) => {
				return xUpdateHook(iUpdateType, dbName, tblName, lo32, hi32);
			});
		};
	}
	```
	- important note: `libadapters.js` adds this `.setCallback` method (along with `.getCallback` and `.deleteCallback`): used to register the callback in the central callback lookup table
		```ts
		// libadapters.js
		const targets = new Map();
    	Module['setCallback'] = (key, target) => targets.set(key, target);
    	Module['getCallback'] = key => targets.get(key);
    	Module['deleteCallback'] = key => targets.delete(key);
		```
		- by convention, the `key` is always `pAsyncFlags` pointer (convenience: always unique -- associated with the function, needed to be passed to WASM side anyway: to determine whether or not the function should be called asynchronously)
	- finally, calling `ccall("libhook_update_hook", ...)` calls to C-defined `libhook_update_hook`:
		```c
		// Prepare a single function to handle all update callbacks
		// Signature is the one required by `sqlite3_update_hook`
		static void libhook_xUpdateHook(
		  void* pApp,
		  int iUpdateType,
		  const char* dbName,
		  const char* tblName,
		  sqlite3_int64 rowid) {
			// Legalize rowid into two 32 bit integers
			int hi32 = ((rowid & 0xFFFFFFFF00000000LL) >> 32);
			int lo32 = (rowid & 0xFFFFFFFFLL);

			// Read async flags (determining whether to call 'vppippii' or 'vppippii_async')
			const int asyncFlags = pApp ? *(int *)pApp : 0;

			// Call the appropriate JS defined callback
			CALL_JS(vppippii, pApp, pApp, iUpdateType, dbName, tblName, lo32, hi32);
		  }

		// JS code calls this function directly, it:
		// - calls `sqlite3_update_hook` with the appropriate params
		// - reads `xUpdateHook` (1 or 0 flag) to determine whether or not the callback should be registered
		// - if callback exists, passes the predefined `libhook_xUpdateHook` as the callback to `sqlite3_update_hook`
		// - passes `pApp` (pAsyncFlags in JS code above) - this is used to identify the appropriate callback back in JS side
		void EMSCRIPTEN_KEEPALIVE libhook_update_hook(sqlite3* db, int xUpdateHook, void* pApp) {
		  sqlite3_update_hook(db, xUpdateHook ? &libhook_xUpdateHook : NULL, pApp);
		}
		```

	- now, to show this from bottom up:
	  - `libhook_update_hook` had been called, registering a callback (`libhook_xUpdateHook`) with `pApp`
	  - an update happens, so the DB calls `libhook_xUpdateHook`, passing `pApp`
	  - `libhook_xUpdateHook` reads `pApp` determining which JS handler to call (`vppippii` or `vppippii_async`, depending on async flag)
		```c
		// libhook.c -> libhook_xUpdateHook
		const int asyncFlags = pApp ? *(int *)pApp : 0;
		```
	  - `libhook_xUpdateHook` calls the appropriate function, passing `pApp` (along with other params provided by SQLite)
		```c
		// libhook.c -> libhook_xUpdateHook
		CALL_JS(vppippii, pApp, pApp, iUpdateType, dbName, tblName, lo32, hi32);
		```
	  - JS defined `vppippii` (or `vppippii_async`) gets called, reading the `pApp` (key) and `...params`
	  - `vppippii` looks into the lookup table (using `pApp`) to get the registered JS callback, registered by `Module.update_hook` (`pApp` will match `pAsyncFlags` in JS code):
		```ts
		// libhook.js -> update_hook
		Module['update_hook'] = function(db, xUpdateHook) {
			// ...
			pAsyncFlags = Module['_sqlite3_malloc'](4);
			// ccall("libhook_update_hook", ...)
			Module['setCallback'](pAsyncFlags, (_, iUpdateType, dbName, tblName, lo32, hi32) => {
				return xUpdateHook(iUpdateType, dbName, tblName, lo32, hi32);
			});
		}
		```
	  - the callback is found, called and has the rest of the args passed up
	  - `xUpdateHook` (in `libhook.js`-defined `update_hook`) calls to the callback passed from `SQLiteAPI` (the `adapt` wrapper), which then unwraps the inter-context-comm-friendly params and transforms them to expected (JS values):
		```ts
		// sqlite-api.js -> 'update_hook'
		function cvtArgs(iUpdateType, dbName, tblName, lo32, hi32) {
			return [
				iUpdateType,
				// Replace the pointer to the string with actual string (read from the heap)
				Module.UTF8ToString(dbName),
				// Replace the pointer to the string with actual string (read from the heap)
				Module.UTF8ToString(tblName),
				// Delegalize the 64bit int
				cvt32x2ToBigInt(lo32, hi32)
			];
		};
		function adapt(f) {
			return f instanceof AsyncFunction ?
				(async (iUpdateType, dbName, tblName, lo32, hi32) => f(...cvtArgs(iUpdateType, dbName, tblName, lo32, hi32))) :
				((iUpdateType, dbName, tblName, lo32, hi32) => f(...cvtArgs(iUpdateType, dbName, tblName, lo32, hi32)));
		}
		```
	- the `cvtArgs` here transforms the args to JS friendly ones effectively calling the callback passed from parent `crsqlite-wasm`-defined `DB.onUpdate`
	- finally, the original `handleUpdate` passed from the app code gets called with expected JS values (which get logged to the console)
- this pattern is used throughout the codebase, whenever a C -> JS call is needed (e.g. VFS interface, etc.)
- this allows for compact memory footprint (only a handful of functions get defined in both contexts, whereas all listeners, handlers get defined in their own context, namely JS and -- in case of JS -- GCd when no longer needed)



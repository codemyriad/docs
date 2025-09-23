# Dynamic JS Callbacks in WASM: A Production Grade Design Pattern

## Intro

Calling into WASM code from JS (and vice versa) is made trivial using tools like Emscripten, which abstract away the plumbing of cross-environment communication.
That is: in a trivial case of statically defined functions (known at build/compile/link time) this exchange is made easy. 
However, things start to get significantly more complicated when we want to use (and manage) dynamic callbacks, such as logs, notifications, progress hooks, etc.
That is, indeed, a challenging endeavour as it introduces a lot of bookkeeping, e.g. 
Emscripten allows us to grow function tables at runtime (`ALLOW_TABLE_GROWTH=1`) and register functions dynamically (`Module.addFunction`), but for every such function we need to imperatively allocate memory and release it after we no longer need it.
Every memory-aware developer out there will recognise this as a slippery slope and one that adds more points where things could go wrong as the codebase grows.

Recently, while working with [rhashimoto/wa-sqlite] repo we stumbled upon a really nice design pattern that tackles challenges of managing of such callbacks and makes it incredibly easy to:
- reuse a lot of cross-environment functions: using trampolines and multiplexers combined with in-memory (JS side) lookup tables
- make the process of adding new callbacks incredibly easy, so much so that the ergonomic, client-side API doesn't need to concern itself with how the data/callbacks are passed back and forth between JS <-> WASM sides of the stack.

In this article we expore this pattern, starting with the basics of cross-environment communication using statically defined functions and Emscripten's interop. 
After that we go into simple case of using dynamically registered callbacks: both accepting function params from C side as well as registering and passing the same functions from JS side (and detail challenges that make it cumbersome to manage as the project grows).
We then continue to expore ways in which calling back could be simplified by reusing functions (both on JS anc C side) scoped to single functionality (e.g. logging, but doing so across different calls), and introduce concepts like trampolines, multiplexers and lookup tables.
Finally, we generalise the reusability of callback functions, detailing the use of function signature as function definition (as seen in [rhashimoto/wa-sqlite]) and showcase the incredible ease of use after the pattern had been set up.

In order to make this scoped and digestible, we're a fictitious example which we extend throughout the article. 
This article assumes basic proior knowledge of WASM and Emscripten. Additionally, some (very basic) C knowledge is a plus, but the examples should be simple enough for a non-C developer to understand the code.

Finally, we can't say if using function signature as definition is a wide-spread convention, or crafted in [rhashimoto/wa-sqlite], but we give credit to the author of the repo either way.

---

## 1. Foundations: Calling Statically Defined WASM Functions From JS And Vice Versa

Let's imagine we wanted to perform an expensive, multi-step calculation in C and we would like to log back to JavaScript at each step.

For starters, we can use a statically defined logging function.

Let's start by preparing a JS library. We'll call it `libshims.js` _(note: the idiomatic way to organise Emscripten shims is to separate the modules by functinoality, but for the sake of this example, we'll cram everything into a single file as long as there's no explicit need to separate, e.g. due to different stages of the build process)_

```js
// libshims.js

// Define the (static) log function - logging out to console
function js_log(pMsg) {
    // Read the string from linear memory
	const msg = Module.UTF8ToString(pMsg);
	console.log(msg)
};

// Register the function with Emscripten's library manager
mergeInto(LibraryManager.library, { js_log });
```

Here we:
- define the `js_log` function
- use Emscripten's `mergeInto` to register it to the `LibraryManager.Library` object

This file is an instruction that will be **run (as a script) during the Emscripten build process** (at link time).
When Emscripten is running JS lib files, it makes the `mergeInto` and `LibraryManager` available in global scope (hence no import necessary).
Those helpers are used (in this way) to register the `js_log` function to WASM library, making it available to C code as external functions.
On the C side we can do something like this:

```c
// calc.c
#include <emscripten.h>

// Declare the JS-defined function (when called at runtime, it will find the right funciton through Emscripten's interop)
extern void js_log(const char *msg);

// Define the (expensive) data processing function
EMSCRIPTEN_KEEPALIVE
int process_data_f32(const float *data, int len, float *out) {
    int out_length; // This sould be updated during the run

    js_log("Initialising work...");
    // initialise work

    js_log("Preprocessing...");
    // preprocess data

    js_log("Calculating result...");
    // calculate the result
    
    js_log("Done!");
    return out_length;
}
```

Here we declare the external `js_log` function (previously defined in JS library code and registered to Emscripten's library), so we know it will be made available to the WASM compiled C code.
The rest is self-explanatory: `process_data_f32` is a function defining a fictitious expensive calculation, logging at each processing step.
The processing step is not important (let's imagine that every step , e.g. `// preprocess data` performs some expensive computation).

Now we compile with Emscripten:

```bash
emcc calc.c \
  --js-library libshims.js \
  -s EXPORTED_FUNCTIONS=<exported_functions> \
  -s EXPORTED_RUNTIME_METHODS=<exported_runtime_methods> \
  -o wasm_lib.mjs
```

Here we:
- instruct Emscripten to compile `calc.c` code (and produce WASM binary)
- while doing so, it should run `libshims.js` to extend the library with the `js_log` function (`--js-library libshims.js`)
- `-s EXPORTED_FUNCTIONS=<exported_functions>` - instructs `emcc` to include certain (Emscripten) functions (e.g. `_malloc`, `_free`, etc.)\*
- `-s EXPORTED_RUNTIME_METHODS=<exported_runtime_methods>` - instructs `emcc` to include certain (Emscripten) runtime methods (e.g. `ccall`)\*
- `-o wasm_lib.mjs`: output the JS glue code as an `wasm_lib.mjs` ES module (and `wasm_lib.wasm` binary)
_\*see the appendix for exported functions and exported runtime methods required to run this example_

We can now call the `process_data_f32` function from our JS app:

```ts
// index.ts
import Module from "./wasm_lib.mjs"
import wasmURL from "./wasm_lib.wasm?url"

const wasm = await Module({ locateFile: () => wasmURL });
function processData(data: number[]): number[] {
    // bolierplate to allocate mem and copy the data
	const pData = wasm._malloc(data.length * 4);
	wasm.HEAPF32.set(data, pData / 4);
	const pOut = wasm._malloc(data.length * 4);

	const outLen = wasm.ccall("process_data_f32", ["int"], ["number", "number", "number"], [pData, data.length, pOut]);

    // builerplate to read the res from linear mem
	const res = Array.from(wasm.HEAPF32.subarray(pOut / 4, pOut / 4 + outLen));

    // boilerplate to free the mem
	wasm._free(pData);
	wasm._free(pOut);

	return res
}
```

Running `processData(some_data_array)`, we get the following printed to console:

```
Initialising work...
Preprocessing...
Calculating result...
Done!
```

---

## 2. Making the Callback Logic More Configurable By Dynamically Specifying Callback Logic

The base implementation above is fine if we need hardcoded logging (`console.log`), but oftentimes we want to make the logging logic more configurable.
In order to achieve that, we refactor our code to allow for specifying of callbacks in a dynamic manner.

First, let's refactor our code a bit:
- let's specify a `libshims.c` module (we'll keep our client-exposed, WASM interop, functionality there)
- let's use `calc.c` as a core library module, where we specify the underlaying logic, without any knowledge of the environment (native complied, WASM, etc.)

Refactored core logic:

```c
// calc.h
#ifndef CALC_H
#define CALC_H
typedef void (*log_fn_t)(const char *msg);
int process_data_f32(const float *pData, int len, float *pOut, log_fn_t log_fn);
#endif

// calc.c
#include "calc.h"

int process_data_f32(const float *pData, int len, float *pOut,
                     log_fn_t log_fn) {
  log_fn("Initialising work...");
  // initialise work

  log_fn("Preprocessing...");
  // preprocess data

  log_fn("Calculating result...");
  // calculate the result

  for (int i = 0; i < len; i++) {
    pOut[i] = pData[i] + 0.2f;
  }

  log_fn("Done!");

  return len;
}
```

Here we've specified the `log_fn_t` (a type alias for logging function, passed as a parameter to `process_data_f32`) and extended the `process_data_f32` definition to accept dynamically passed logging function (`log_fn`).

In the simplest scenario (pure C code), we would call this function like so:

```c
// main.c
#include "calc.h"
#include <stdio.h>

// A C-native function used to log the data
void c_log(const char *msg) {
    printf("%s\n", msg);
}

int main() {
    // ... 
    // Call the process_data_f32 with C-environemnt logger
    process_data_f32(data, len, out, c_log);
    return 0;
}
```

To fit our use-case, we specify `libshims.c` to wrap the underlaying logic and expose it to client code (via Emscripten):

```c
// libshims.c
#include <emscripten.h>
#include "calc.h" // include the core library

// Define the (expensive) data processing
EMSCRIPTEN_KEEPALIVE
int libshims_process_data_f32(const float *data, int len, float *out, log_fn_t log_fn) {
    // NOTE: we're using the core library function with the same signature (notice the extra 'log_fn' parameter),
    // this will change in following iterations
    return process_data_f32(data, len, out, log_fn);
}
```

Here we've defined a wrapper function `libshims_process_data_f32` that calls the core library function `process_data_f32`, passing along the `log_fn` function pointer.
In its current state, this wrapping logic is completely useless (save for `EMSCRIPTEN_KEEPALIVE`) as we're calling the underlaying function with the same signature, but this will change in following iterations.

We need to change our build command slightly (to include `libshim.c` as well as allow adding of functions at runtime -- `ALLOW_TABLE_GROWTH=1`):

```bash
emcc libshims.c calc.c \
  --js-library libshims.js \
  -s EXPORTED_FUNCTIONS=<exported_functions> \
  -s EXPORTED_RUNTIME_METHODS=<exported_runtime_methods> \
  -s ALLOW_TABLE_GROWTH=1 \  # Allow dynamic growth of function table (needed for addFunction)
  -o wasm_lib.mjs
```

With the updated functionality, we can call `libshims_process_data_f32` from our JS app, passing along a dynamically defined logging function, like so:

```ts
// index.ts
import Module from "./wasm_lib.mjs"
import wasmURL from "./wasm_lib.wasm?url"

const wasm = Module({ locateFile: () => wasmURL });
function processData(data: number[], log: (msg: string) => void): number[] {
    // ...omitting boilerplate for data transfer (same as in previous example)

    // Wrap the log fuction with string reading logic
    const _log = (msgPtr: number) => {
        const msg = wasm.UTF8ToString(msgPtr);
        log(msg);
    }
    // Register the function (allocating mem for it)
	const pLogFn = wasm.addFunction(_log, "vp")

    // Updated call: notice the pLogFn argument at the end of the list
	const outLen = wasm.ccall("libshims_process_data_f32", ["int"], ["number", "number", "number", "number"], [pData, data.length, pOut, pLogFn]);

    // ...omitting boilerplate for result data transfer
    // ...omitting boilerplate for freeing mem

    // Remove the function (freeing the mem it occupied)
	wasm.removeFunction(pLogFn);

	return res
}
```

Now, when implementing `processData(data, log)`, we're accepting the `data` parameter (a `number` array -- we're casting it to `f32`), as well the `log` function.

The `log` function has a signature `(msg: string) => void`, whereas the raw logging callback (in WASM) expects signature `(pMsg: number) => void`, where `pMsg` is a pointer to the start of the message string, 
so we wrap the `log` function in `_log` (handling low-level data retrieval) and working as an adapter to `log`.

When using the `_log` function, we need to register it with Emscripten (using `wasm.addFunction`) to get a pointer that can be passed to C code. After the execution we call `wasm.removeFunction(pLogFn)` to free the memory occupied by the registered callback.

Running `processData(some_data_array, (msg) => console.log("[dyn_logger]", msg))`, we get the following printed to console:

```
[dyn_logger] Initialising work...
[dyn_logger] Preprocessing...
[dyn_logger] Calculating result...
[dyn_logger] Done!
```

The two things are important to note here:

Firstly, the registered function needs to be freed manually (it doesn't get GC'd, as a JS callback would), with which we need to be careful so as to avoid memory leaks.

The second really important thing to notice is the `vp` part: the second argument of `wasm.addFunction` call: 
**This is the function signature passed to Emscripten**, where:
  - `v` stands for void
  - `i` stands for int (32-bit integer)
  - `p` stands for pointer (32-bit integer)
  - `f` stands for float (32-bit float)
  - in this example we have a function with the following signature:
    - JS: `(pMsg: number) => void`
    - C: `void (*log_fn_t)(const char*)`
    - Emscripten shorthand: `vp` (returns: `void`, params: [`pointer`])


---

## 3. Reusing Callback Functions: Trampolines and Multiplexers

Passing of a dynamic logger in previous example is well and all if we're dealing with a function where we know the start of the execution and the end,
so we can register and deregister the callback function explicitly, however:
- this introduces the need for additional bookkeeping, which can be cumbersome as the complexity of the code grows 
- in this simple case we can do a synchronous register/deregister, but imagine this code was asynchronous and called from multiple places, or that the calc is a long-running process, where it's not trivial to know when to deregister the function

To address these concerns, we can combine the first and second approach and introduce some additinal details to make it easier to manage callbacks:

First let's update both `process_data_f32` and `log_fn_t` signatures to accept `id` as first param:

```c
// calc.h
#ifndef CALC_H
#define CALC_H
typedef void (*log_fn_t)(const void *id, const char *msg);
int process_data_f32(const void *id, const float *pData, int len, float *pOut, log_fn_t log_fn);
#endif

// calc.c
#include "calc.h"

int process_data_f32(const void *id, const float *pData, int len, float *pOut, log_fn_t log_fn) {
  log_fn(id, "Initialising work...");
  // initialise work

  log_fn(id, "Preprocessing...");
  // preprocess data

  log_fn(id, "Calculating result...");
  // calculate the result

  for (int i = 0; i < len; i++) {
    pOut[i] = pData[i] + 0.2f;
  }

  safe_log(log_fn, id, "Done!");

  return len;
}
```

Here we've updated the (core) `calc.(h|c)` library files:
- added a `const void *id` parameter to the `process_data_f32` function, which will be used to identify the calc run _(for our use-case, this could have well been an `int` -- pointers are ints anyway -- but in production this will often be a pointer to some object)_
- updated the calls to `log_fn`: with each call, we also pass the `id` parameter, alongside the string message

We need to reflect the updates in the shim code as well:

```c
// libshims.c
#include "calc.h"
#include <emscripten.h>
#include <stdio.h>

// Our good old friend js_log
extern void js_log(const void *id, const char *msg);

// A "trampoline" function that will call the actual logger
void libshimsLog(const void *id, const char *msg) { 
    // Don't call the log function if 'id' = 0 
    if (id) {
        js_log(id, str); 
    }
}

EMSCRIPTEN_KEEPALIVE
int libshims_process_data_f32(const void *id, const float *pData, int len, float *pOut) {
  return process_data_f32(id, pData, len, pOut, libshimLog);
}
```

We've updated the `libshims_process_data_f32` function as well to accept the calc `id` parameter. 
It no longer accepts a log function pointer (dynamic callback), instead, it calls the underlaying `process_data_f32`, passing `libshimsLog`.
`libshimsLog` is a trampoline function:
- every call to a dynamic callback passes through `libshimsLog`
- it determines whether or not the run is identifiable (if `id` is non-zero), if it is, it calls the logging function on the JS side (`js_log`), passing along the `id`, as well as the message (pointer).

What we have so far:
- when calling `libshims_process_data_f32`, we need to pass the `id` parameter (to identify the run)
- if the run is identifiable (`id` != 0), the extern loging function will be called with parameters: `id`, `pMsg` (through the hoops we set up)

Since `js_log` is now called with `id` identifying the calc run, we can use it as a single function (multiplexer) to handle all logging needs (identifying the appropriate callback based on `id`).

First we need to update the `libshims.js` (and `js_log` definition):

```js
// libshims.js
function js_log(id, pMsg) {
	const msg = Module.UTF8ToString(pMsg);
	// Get the registered callback from the callback table
	// NOTE: getCallback is defined in libshims-support.js
	Module["getCallback"](id)?.(msg)
};

mergeInto(LibraryManager.library, { js_log });
```

Now, instead of logging out (`console.log`), `js_log` will try to retrieve the (registered) callback from the lookup table (`Module["getCallback"]`) and pass the message to it (if one such exists).
It's important to notice that the lookup table isn't defined here (nor the `getCallback` function), this is fine for now as `libshims.js` will merely register the function at build time and we don't need `getCallback` until the function is called at runtime.
However, we do need to specify the lookup table and related functionality which we do in `libshims-support.js`:

```js
// libshims-support.js
(function() {
	const callbacks = new Map()
	let nextId = 1

	// Attach the following methods used to register, retrieve and remove callbacks from
	// lookup table, as used by:
	// - 'js_log' - multiplexer defined in libshims.js ('getCallback')
	// - client code ('registerCallback' and 'removeCallback')
	Module["registerCallback"] = function(cb) {
		const id = nextId++
		callbacks.set(id, cb)
		return id
	}
	Module["getCallback"] = function(id) {
		return callbacks.get(id)
	}
	Module["removeCallback"] = function(id) {
		callbacks.delete(id)
	}
})();
```

libshims support file uses the IIFE pattern to:
- create a lookup table for callback functions (keyed by `id`)
- expose methods to work with callbacks (`registerCallback`, `getCallback`, `removeCallback`)

Important thing to note is that `libshims.js` runs at link time (attaching `js_log` function to WASM library, making it accessible from C code, but having access to JS glue module), 
while `libshims-support.js` runs at runtime (when the JS glue factory is executed -- creating the runtime module object), attaching functionality (lookup table methods) accessible from both the client as well as `js_log`.

_It is also worth mentioning that this isn't the idiomatic way of defining support for some functionality: there is a more conventional Emscripten API to do so, but for simplicity's sake, we're doing it this way for the purpose of this example._

With these updates, we build the WASM module using the following command:

```bash
emcc libshims.c calc.c \
  --js-library libshims.js \
  --post-js libshims-support.js \
  -s EXPORTED_FUNCTIONS=<exported_functions> \
  -s EXPORTED_RUNTIME_METHODS=<exported_runtime_methods> \
  -o wasm_lib.mjs
```

Notice that we've added `libshims-support.js` and that it (unlike `libshims.js`) runs as `--post-js` - being appended to the generated JS glue factory code.

Finally, in the client app, we do something like this:

```ts
// index.ts
import Module from "./wasm_lib.mjs"
import wasmURL from "./wasm_lib.wasm?url"

const wasm = Module({ locateFile: () => wasmURL });

function processData(msg: number[], log: (msg: string) => void): number[] {
    // ...omitting boilerplate for data transfer (same as in previous example)

    // Register the callback (no need to wrap it as 'js_log' takes care of reading of string value)
	const id = wasm["registerCallback"](log)

    // Instead of passing the callback directly, we're passing the run id
	const outLen = wasm.ccall("libshims_process_data_f32", "number", ["number", "number", "number", "number"], [id, pData, data.length, pOut]);

    // ...omitting boilerplate for result data transfer
    // ...omitting boilerplate for freeing mem

	return res
}

```

To show examples of possible execution, we run `processData` with different params:

```js
processData(data_1, (msg) => console.log("[data 1 logger]", msg))
processData(data_2, (msg) => sendMessageToTelemetry("data 2", msg))
processData(data_3, (msg) => persistLogToFile("data_3.log", msg))
```

Running `processData(some_data_array)`, we get the following printed to console:

```
[data 1 logger] Initialising work...
[data 1 logger] Preprocessing...
[data 1 logger] Calculating result...
[data 1 logger] Done!
```

We get logs for run 2 sent to telemetry service.

Logs for run 3 get persisted to a file (`data_3.log`):

```
Initialising work...
Preprocessing...
Calculating result...
Done!
```

--- 

## 4. Making It Production Grade: Function Signature as Function Definition Pattern

Now that we've covered the plumbing, let's zoom out a bit. Let's extend the functionality to also include some event generating process, e.g.:

- a process listening to keyboard input and emitting events to JS, via some callback
- some additional data processing function that emits results as they are ready

We could define the underlaying functionality (core library) like so:

```c
// events.h
typedef void (*emit_fn_t) (const void *id, const char* code);
void add_event_listener(const void *id, emit_fn_t emit_fn);

// events.c
#include "events.h"

// A placeholder for keyboard IO
extern char* wait_for_keypress();

void add_event_listener(const void *id, emit_fn_t emit_fn) {
  while (true) {
    const char *code = wait_for_keypress();
    emit_fn(id, code);
  }
}

// datapipeline.h
typedef void (*send_fn_t) (const void *id, const char *data);
void start_process_pipeline(const void *id, const char *pData, send_fn_t send_fn);

// datapipeline.c
#include "datapipeline.h"

void start_process_pipeline(const void *id, const char *pData, send_fn_t send_fn) {
    // Step 1
    char* processed = process_step_1(pData);
    send_fn(id, processed);

    // Step 2
    processed = process_step_2(pData);
    send_fn(id, processed);

    // ...
}
```

One more "Let's imagine", but bear with me: **Let's imagine all of those processes run asynchronously** _(yes, the way previous example was set up - it runs in a blocking manner, but for simplicity's sake let's treat all of those blocking processes as some concurrent/parallel processes)_

Now, we should also define the shim code:

```c
// libshims.c
#include <emscripten.h>
#include <stdio.h>

#include "calc.h"
#include "events.h"
#include "datapipeline.h"

// Statically defined (JS-side) multiplexers
extern void js_log(const void *id, const char *msg);
extern void js_emit(const void *id, const char *code);
extern void js_send(const void *id, const char *data);

// Calc
void libshimsLog(const void *id, const char *msg) { 
    js_log(id, msg); 
}

EMSCRIPTEN_KEEPALIVE
int libshims_process_data_f32(const void *id, const float *pData, int len,
                             float *pOut) {
  return process_data_f32(id, pData, len, pOut, libshimsLog);
}

// Events
void libshimsEmit(const void *id, const char *code) { 
    js_emit(id, code); 
}

EMSCRIPTEN_KEEPALIVE
void libshims_add_event_listener(const void *id) {
    add_event_listener(id, pData, len, pOut, libshimsEmit);
}

// Data pipeline
void libshimsSend(const void *id, const char *data) { 
    js_send(id, data); 
}

EMSCRIPTEN_KEEPALIVE
void libshims_start_process_pipeline(const void *id) {
    start_process_pipeline(id, pData, libshimsSend);
}
```

Following C implementation, we could do the same on the JS side:
- define `js_log` multiplexer + callback lookup table (with related methods)
- repeat for `js_emit`
- repeat for `js_send`

This is a lot of repetition:
- there's some repeated C boilerplate
- there's even more repeated boilerplate on JS (and interop) side (callback tables and so)

This repetition only gets more cumbersome when adding additional functionality.

Now, notice that the callback signature is the exact same for all three cases:

```c
typedef void (*log_fn_t) (const void *id, const char *msg);
typedef void (*emit_fn_t) (const void *id, const char *code);
typedef void (*send_fn_t) (const void *id, const char *data);
```

This is an important observation as there's a nice pattern utilising just that: function signatures.

How does this work? 

We group (callback) functions by their signature: in this example all callback functions share the same signature:
- TS: `(id: number, msg: string) => void`
- C: `void (*fn)(const void*, const char*)`

Now, we utilise the shorthand for function signature as mentioned in `Module.addFunction` example (above):
- one letter per type:
  - `v` - void
  - `i` - int (32-bit integer)
  - `p` - pointer (32-bit integer)
  - `f` - float (32-bit float)
- follow C order: return type [... parameter types]
- the callbacks in our example:
  - return `void`
  - accept params: id - `pointer`, msg - `pointer`
  - the shorthand: `vpp`

Now we declare an `extern` generic multiplexer for vpp (calling literally `vpp`, let's call the module `libadapters`):

```c
// libadapters.h
extern void vpp(const void *, const char *);
```

Here we've defined an extern function (JS + Emscripten will provide the binding) of the same signature as our callbacks (in all three cases).

We update our `libshims.c` so that all three shims use the same `vpp` function (instead of respective `extern` callbacks):

```c
// libshims.c
#include <emscripten.h>
#include <stdio.h>

// Include 'vpp' extern declaration
#include "libadapters.h"

#include "calc.h"
#include "events.h"
#include "datapipeline.h"

// Calc
void libshimsLog(const void *id, const char *msg) { 
    vpp(id, msg); 
}

EMSCRIPTEN_KEEPALIVE
int libshims_process_data_f32(const void *id, const float *pData, int len,
                             float *pOut) {
  return process_data_f32(id, pData, len, pOut, libshimsLog);
}

// Events
void libshimsEmit(const void *id, const char *code) { 
    vpp(id, code); 
}

EMSCRIPTEN_KEEPALIVE
void libshims_add_event_listener(const void *id) {
    add_event_listener(id, pData, len, pOut, libshimsEmit);
}

// Data pipeline
void libshimsSend(const void *id, const char *data) { 
    vpp(id, str); 
}

EMSCRIPTEN_KEEPALIVE
void libshims_start_process_pipeline(const void *id) {
    start_process_pipeline(id, pData, libshimsSend);
}
```

We've left the trampoline functions (as is the convention), as those might contain some additional transforming functionality, but we're reusing the same extern `vpp` callback.
_(I would also note that the more conventional approach would be to also split different functionalities into separate modules, but we're keeping it simple here)._

On the JS side, we define a single multiplexer function `vpp` just as we did with `js_log`:

```js
// libadapters.js

// Define all unique callback signatures we're using so far
const signatures = [
    "vpp"
    // In the future we might add more here
]

const callbacks = {}
for (const signature of signatures) {
    // Register each signature using the same logic:
    // - read 1st param - id and find the registered callback
    // - call the registered callback with ...rest params
    callbacks[signature] = (id, ...params) => {
        return Module["getCallback"](id)?.(...params)
    }
}

mergeInto(LibraryManager.library, callbacks);
```

Following our (simplified) `js_log` example, we need to also define the callback lookup table and related methods:

```js
// libadapters-support.js
(function() {
	const callbacks = new Map()
	let nextId = 1

    // Register a callback (regardless of signature)
	Module["registerCallback"] = function(cb) {
		const id = nextId++
		callbacks.set(id, cb)
		return id
	}
    // Get callback (from the table of all callbacks)
	Module["getCallback"] = function(id) {
		return callbacks.get(id)
	}
	Module["removeCallback"] = function(id) {
		callbacks.delete(id)
	}
})();
```

And with this design pattern we only need the following build flags / files:

```bash
emcc calc.c events.c datapipeline.c libshims.c \
  --js-library libadapters.js \
  --post-js libadapters-support.js \
  -s EXPORTED_FUNCTIONS=<exported_functions> \
  -s EXPORTED_RUNTIME_METHODS=<exported_runtime_methods> \
  -o wasm_lib.mjs
```

And finally, the client code can look something like this:

```ts
import Module from "./wasm_lib.mjs"
import wasmURL from "./wasm_lib.wasm?url"

const wasm = Module({ locateFile: () => wasmURL });

// A wrapper around (str: string) => void callback:
// - reading data from pointer
// - calling the callback (in more ergonomic manner)
function adaptStrCb(cb: (str: string) => void) {
    return (pStr: number) => {
        const str = wasm.UTF8ToString(pStr);
        cb(str);
    }
}

function processData(data: number[], log: (msg: string) => void) {
    // ... allocate and copy memory boilderplate
    const id = wasm["registerCallback"](adaptStrCb(log));

    wasm.ccall("libshims_process_data_f32", "number", ["number", "number", "number", "number"], [id, pData, data.length, pOut]);

    // ... read result + release memory
    return res
}

function onKeyboardEvent(data: number[], onEvent: (code: string) => void) {
    const id = wasm["registerCallback"](adaptStrCb(onEvent));
    wasm.ccall("libshims_add_event_listener", "void", ["number"], [id]);
    return res
}

function startDataPipeline(data: string, onData: (data: string) => void) {
    // ... allocate and copy memory boilderplate

    const id = wasm["registerCallback"](adaptStrCb(onData));
    wasm.ccall("libshims_start_process_pipeline", "void", ["number", "number"], [id, pData]);

    // ... release memory
}
```

Now all functions register callbacks, seemingly independently, but every call from C (sharing the same signature) passes through `vpp`, using the lookup table to find its way to the appropriate callback.

To drive the point home let's introduce additional functionality:

- add a new function: `add_alert_listener` - a function that registers listeners to every alert event (`onAlert: (msg: string) => void`)
- extend `start_data_pipeline` to include additional callback reporting progress (`onProgress: (nProcessed: number, nTotal: number) => number`) returning:
  - 0 - continue processing
  - 1 - stop processing

The former becomes trivial, we extend our `libshims.c` :

```c
// libshims.c

// ... rest is the same as before

// defines add_alert_listener(const void *id, emit_fn_t emit);
#include "libalerts.h"

void libshimsAlert(const void *id, const char *msg) { 
    vpp(id, msg); 
}

EMSCRIPTEN_KEEPALIVE
void libshims_add_alert_listener(const void *id) {
    add_alert_listener(id, libshimsAlert);
}
```

Et voila, aside from including `libalerts.h` and defining the `libshimsAlert` trampoline function, nothing else changes (yet everything works), e.g.:

```ts
// index.ts

// ... rest is the same as before

// Add alert function
function onAlert(onAlert: (msg: string) => void) {
    const id = wasm["registerCallback"](adaptStrCb(onAlert));
    wasm.ccall("libshims_add_alert_listener", "void", ["number"], [id]);
}
```

Now extending `start_data_pipeline` with progress reporting, we're definig a new signature - `ipii`:
- returns `int` 
- accepts params: `id: pointer`, `nProcessed: int`, `nTotal: int`

We need to declare `ipii` extern function:

```c
// libadapters.h
extern void vpp(const void *, const char *);
extern int ipii(const void *, int *, int *);
```

Extend the core library (`datapipeline.c`) with additional functionality:

```c
// datapipeline.h
typedef void (*send_fn_t) (const void *id, const char *pData);
typedef int (*prog_fn_t) (const void *id, int nDone, int nTotal);
void start_process_pipeline(const void *data_id, const void *prog_id, const char *pData, send_fn_t send_fn, prog_fn_t prog_hook);

// datapipeline.c
#include "datapipeline.h"

void start_process_pipeline(const void *data_id, const void *prog_id, const char *pData, send_fn_t send_fn, prog_fn_t prog_hook) {
    int done, total_steps; // Dummy progress variables

    // Step 1
    char* processed = process_step_1(pData);
    send_fn(id, processed);
    if (prog_hook(prog_id, done++, total_steps)) {
        // Stop if progress_hook returns 1
        return;
    }

    // Step 2
    processed = process_step_2(pData);
    send_fn(id, processed);
    if (prog_hook(prog_id, done++, total_steps)) {
        return;
    }

    // ...
}
```

Create a trampoline and extend wrapper around `start_process_pipeline`

```c
// libshims.c

// ... rest

// Data pipeline
void libshimsSend(const void *id, const char *data) { 
    vpp(id, str); 
}

int libshimsProgress(const void *id, int nDone, int nTotal) { 
    return ipii(id, nDone, nTotal); 
}

EMSCRIPTEN_KEEPALIVE
void libshims_start_process_pipeline(const void *data_id, const void *prog_id, const char *pData) {
    start_process_pipeline(data_id, prog_id, pData, libshimsSend, libshimsProgress);
}
```

Finally, we need to extend `libadapters.js` to register `ipii` callback multiplexer:

```js
// libadapters.js

// Define all unique callback signatures we're using so far
const signatures = [
    "vpp", // log, emit, send, alert
    "ipii" // prog_hook
]

const callbacks = {}
for (const signature of signatures) {
    // Register each signature using the same logic:
    // - read 1st param - id and find the registered callback
    // - call the registered callback with ...rest params
    callbacks[signature] = (id, ...params) => {
        return Module["getCallback"](id)?.(...params)
    }
}

mergeInto(LibraryManager.library, callbacks);
```

Notice how easy it is to merely add additional signature (`ipii`), everything else is already set up (table lookups, registrations, etc.)

With these updates, in the app we can do something like:

```ts
// index.ts
import Module from "./wasm_lib.mjs"
import wasmUrl from "./wasm_lib.wasm?url"

const wasm = await Module({ locateFile: () => wasmUrl })

function startDataPipeline(
    data: string, 
    onData: (data: string) => void, 
    progressHook: (nDone: number, nTotal: number) => number
) {
    // ... allocate and copy memory boilderplate

    const data_id = wasm["registerCallback"](adaptStrCb(onData));
    const prog_id = wasm["registerCallback"](progressHook);

    wasm.ccall("libshims_start_process_pipeline", "void", ["number", "number"], [data_id, prog_id, pData]);

    // ... release memory
}
```

We can now call:

```js
const log = (msg) => console.log("processed chunk of data, res:", msg)
const progressHook = (nDone, nTotal) => {
    console.log(`progress: ${nDone} / ${nTotal}`)

    // Stop after step 2 (for whichever reason)
    if (nDone == 2) return 1

    return 0
}

startDataPipeline(data, log, progressHook)
```

---

## Conclusion

Specifying dynamic callbacks in JS <-> WASM interop is a non-trivial task and one that potentially includes a lot of bookkeeping lest we introduce memory leaks.
Reusing trampolines (on WASM side) and multiplexers + lookup tables (on JS side) greatly simplifies this process, but still leaves us with repeated boilerplate whenever we introduce new section of funcitonality.
Using function signature as function definition pattern makes the entire process generic: we can use a single lookup table (on JS side, without polluting the shared memory) and, when introducing new signature, we simply have to add it to a list (everything else just works).

If you're interested in seeing this in action, I can't recommend [rhashimoto/wa-sqlite] enough, where this pattern is coupled with SQLite's callback logic: used to notify of changes, but also tap into the execution process using this callback pattern.

## Appendix

In order to be able to build this example, one needs to specify `EXPORTED_FUNCTIONS` and `EXPORTED_RUNTIME_METHODS` during `emcc` build.

Emscripten exported functions necessary for this example:
- `_malloc` - allocate (linear) memory
- `_free` - free allocated memory

Emscripten exported runtime methods necessary for this example:
- `ccall` - used to call into C-defined WASM functions (e.g. `libshims_process_data_f32`)
- `HEAPF32` - used to copy / read `f32` data to / from allocated linear memory (we're using this in `libshims_process_data_f32`)
- `stringToUTF8` - used to store UTF8 string into linear memory
- `UTF8ToString` - used to read UTF8 string from linear memory
- `addFunction` - used (temporarily -- in section 2) to register dynamic callbacks at runtime (can only be used with `ALLOW_TABLE_GROWTH=1`)
- `removeFunction` - used (temporarily -- in section 2) to deregister dynamic callbacks at runtime 


[rhashimoto/wa-sqlite]: https://github.com/rhashimoto/wa-sqlite


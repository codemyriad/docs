# Bring the repo (vlcn-io/wa-sqlite) up to speed with the latest in rhashimoto/wa-sqlite

_note: I'm opening this issue to provide some context on PR we've already prepared: #PR_

There's a lot of diff between this (vlcn-io) version of wa-sqlite and the rhashimoto one. Since the divergence, the rhashimoto version had some breaking changes:
- some restructuring of `lib*.(js|h|c)` shims
- the refactor of JS callbacks (e.g. progress hook, update hook, VFS methods etc.)
- the refactor of the way VFS adapters are defined/used 
- dep updates (`emcc`, `sqlite`, etc.)
- addition of JSPI
- etc.

We're most interested in OPFS support, as it would provide significant benefits to the way we use cr-sqlite in our (browser) app:
- FS transparency is a significant benefit
- we've observed significant performance improvements when syncing
- as a second order effect, FS transparency makes it even easier for initial sync (if setting up a new node, one can simply download the `.sqlite3` file, store it in OPFS and open using the `crsqlite-wasm` stack)
- as a second order effect, it's easy to manage import/export (same inherent workflow as above)

Since the divergence, the rhashimoto version added multiple OPFS adapters:
- `opfs-coop-sync-vfs`
- `opfs-any-context-vfs`
- `opfs-adaptive-vfs`
- `opfs-permuted-vfs`

We understand those are examples, but we'd really like to use them as beasline and build on top of them if necessary. 
However, due to significant divergence those aren't compatible with the current version of this repo.

As stated above, we've already gone through the process of reconciling against the original and have done so within the context of [our project](link::librocco),
and would now like to merge this in so that:
- the dependency mgmt is easier (we could depend on the package directly, without need for submodules and our own fork)
- we believe the rest of the community might benefit from the efforts made in this regard


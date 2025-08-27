# Setting up build environment for 'wa-sqlite' development

- in order to work on wa-sqlite, we resolved to use it as a submodule within our monorepo (librocco) so that we can immediately test the changes and get a fast back and forth
- the goal was to be able to build (and load) the whole toolchain as we did, with little to no changes made to the main app
- the toolchain in question consists of:
  - `@vlcn.io/crsqlite-wasm` - (part of `vlcn-io/js` monorepo) exposes the `wa-sqlite` binary and wraps the JS API with additional functionality, exposing the interface we all know and love
  - `@vlcn.io/wa-sqlite` - the actual `wa-sqlite` package
  - the latter is the dependency of the former, so we didn't need to explicitly add the submodule (for `wa-sqlite`)
- one of the challenges here was that `crsqlite-wasm` (as part of `vlcn-io/js`) is not available as a standalone, so we had to add the full monorepo as a git submodule:
  - since our monorepo is managed by `rush` and `vlcn-io/js` is managed by `pnpm`, there's some similarity, but some distinct differences
  - furthermore, there are a lot of packages in `vlcn-io/js` and we needed only a small subset of them, we've decided to not include the (`vlcn-io/js`) packages in our monorepo, but rather use other trickery to use the submoduled source in our app
  - as mentioned, `vlcn-io/js` lists `vlcn-io/wa-sqlite` as a submodule, so we didn't need to add an additional submodule for it
  - there were some additional challenges like `vlcn-io/js` referencing packages outside the repo root (using relative paths), but those (missing deps) were easily found as their own packages in `vlcn-io/*` git repos.
  - the final (relevant) tree looks something like this:
  ```
  librocco/
  ├── 3rd-party/
  │   └── js/ (submodule)
  │       └── deps/
  │           └── wa-sqlite/ (submodule)
  │           └── ... (other deps: emsdk, crsql -- rust code)
  ```
  - in summary: we've strung up the `vlcn-io/js` submodule, listing `vlcn-io/wa-sqlite` as a submodule so that we can work on the `vlcn-io/wa-sqlite` and use the setup, with updates, with minimal intervention on the main app (librocco) side, this way we have 3 levels of git repos to manage:
    - `vlcn-io/wa-sqlite` (our fork to be precise) - active work
	- `vlcn-io/js` (our fork to be precise) - updating the (`wa-sqlite` submodule) hash and committing it
	- `librocco` (our monorepo) - following the commits in `js` and updating the submodule hash
	- of course, we had to add some very minor changes to `vlcn-io/js` to make it work in our monorepo, but those were minimal
- since we didn't include the submodules in our monorepo (`rush` tracked packages) and had a monorepo (`vlcn-io/js`) nested in another monorepo (`librocco`) we needed to use some trickery:
  - for dev, I've used TypeScript and Vite aliases to alias all imports from `@vlcn.io/*` to their respective packages (only the packages we were using, each package having its own logic conforming to respective `package.json` exports/files logic)
  - this didn't work in CI, it was failing for some reason and, after some debugging, we've decided to let it be and use further trickery to make it work in CI
- in order to make our dependency chain work in CI, as well as allow other developers (not actively working on this) to not have to build everything from source each time, we've decided to build out relevant packages for publishing:
  - use `pnpm pack` - to produce `.tgz` files for each of the relevant packages
  - move the `.tgz` files to a common folder (`3rd-party/artefacts`)
  - utilise `pnpm-config.json` (in rush config folder) to create global overrides for built packages, e.g.:
    ```json
	"globalOverrides": {
		"@vlcn.io/crsqlite-wasm": "file:../../3rd-party/artefacts/vlcn.io-crsqlite-wasm-0.16.0.tgz",
	}
	```
  - `pnpm-config.json` overrides also solves one subtlety around package resolution:
    - suppose we have a package depending on another local package (e.g. `@vlcn.io/ws-client` depends on `@vlcn.io/crsqlite-wasm`) using workspace dependency:
    ```jsonc
	// @vlcn.io/ws-client/package.json
	"dependencies": {
		"@vlcn.io/crsqlite-wasm": "workspace:*",
	}
	```
	- when running `pnpm pack` in `@vlcn.io/ws-client`, it will resolve local `@vlcn.io/crsqlite-wasm`'s package.json, get the version and rewrite the dependency to that version:
	```jsonc
	// @vlcn.io/ws-client/package.json (after pnpm pack)
	"dependencies": {
		"@vlcn.io/crsqlite-wasm": "0.16.0",
	}
	```
	- now, when installing from local tarballs, the package resolution would not use the local tarball, but rather try to fetch the package (in this case `@vlcn.io/crsqlite-wasm@0.16.0`) from the registry, and succeed (a valid version exists and is deployed), but this is not our WIP version and installing this would conflict with our local tarball build
	- using `pnpm-config.json` global overrides, we circumvent this issue as pnpm will ignore dependency version (or file or anything) and override all `@vlcn.io/crsqlite-wasm` dep resolutions to our local tarball
- furthermore, we've set up an environment variable to determine whether we want to use the submodules or build from tarballs:
	- `USE_SUBMODULES=true` - uses Vite aliases to alias imports to `@vlcn.io/*` packages (using submodules) - fast updating for dev
	- `USE_SUBMODULES=false` - uses installed `@vlcn.io/*` packages from `node_modules` (build from tarballs) - convenient when not actively working on submodules, but slow to update when working on them (need to rebuild and reinstall)
- finally, we managed to solve the issue in CI re working with submodules packages, so the full CI builds everything from scratch:
  - checks out the repo with submodules
  - builds tarballs from source (running `make` where `make` is due and `pnpm pack` afterwards)
  - upload built tarballs (for in-between jobs) as artifacts






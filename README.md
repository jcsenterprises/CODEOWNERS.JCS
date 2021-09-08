# Bare Module Specifier Resolution in node.js

# Maybe: Generate Import Maps From `package.json` Metadata

```js
{
  "imports": {
    "graceful-fs": "?", // package.json#main
    "graceful-fs/no-side-effects": "?" // package.json#exports
  },
  "scopes": [
    {
      "prefix": "fs://project/node_modules/graceful-fs",
      "imports": { /* ? */ }, // package.json#imports
    },
    {
      "prefix": "fs://project/node_modules/gofer",
      "imports": {
        "./lib/request.js": [
          "./lib/request.node.js",
          // or: { node: true, value: "./lib/request.node.js" },
          "./lib/request.browser.js",
          // or: { browser: true, value: "./lib/request.node.js" },
          "./lib/request.react-native.js",
          "./lib/request.electron.js",
        ],
      },
    },
  ]
}
```

```js
{
  "main": "./old-common.js",
  "exports": {
    "/": "/",
    // vanity support for nicer import specifier
    "/entry": "/entry.js"
  }
}
```

**Contributors:** Guy Bedford, Geoffrey Booth, Jan Krems, Saleh Abdel Motaal

## Outstanding Issues

* How do we prevent a "same specifier, different instance" issue.
  Can we prevent it?
  Can we discourage it?
  `import 'foo/cjs'` still allows getting two copies (it's less implicit though).

* How would imports/exports merging work when things get hoisted.

* Call out how "browser vs. node" implementation may look like.

* Ecosystem pressure to allow complex resolution.

* The specifier itself can only be mapped using the `main` field to prevent collision.
* `exports` changes the meaning of `main`:
  With `exports`, the absence of `main` means "no mapping of bare bare specifier",
  without `exports`, `main` implicitly defaults to `index`.
* What about `"main": "./index.js"` with `"type": "module"`.
  Would that mean `require('x')` would load the same file as a script
  (and supposedly fail unless it's a global side effect file)?
  Would it fail because `require._extensions` is adjusted based on `type`?
  Myles says he doesn't care about this case. It's mostly about the practical
  impact, not about 100% theoretical correctness.

* Can we still provide shorthands without expansive changes to the CJS loader?
  If we want to support `@quinnjs/cli/runner`,
  the CJS loader would be made to be `package.json` boundary aware because it
  would need to check the package directory for a potential `exports` spec.
  This would be a clear departure from how the resolution algorithm works today
  (`package.json` and `node_modules` have no connection from the CJS perspective).

* What needs to change in the CJS resolution algorithm?

* What are the risks of future browser compat / spec compliance?
  How can we reduce those risks?

* `PACKAGE_EXPORT_RESOLVE(packageName, packageSubpath, parentURL)`
  - Replaces `PACKAGE_MAIN_RESOLVE`
  - If packageSubpath is `""`, it uses the `main` field.
  - Otherwise, it uses the `exports` field (or appends if there is none)

* Using `~/public-entry` may allow us to cleanly support both hoisted and internal
  mappings. Alternatively, we could split this proposal into `imports` and `exports`.
  The `~` syntax could also be used to express "import from myself"
  as a shorthand for the package name.
  It would only be valid for its own scope.
  This should be treated as a future enhancement and maybe should also be opt-in
  because it requires duplicating all import map entries.

* Since it would apply to CJS as well, it would be less confusing to make lock-down
  an independent feature that maybe should not be included in the first version.
  The default value for `exports` would be `{ "./", "./" }`, effectively preserving
  the current behavior of `require`.

Scrap book:

```js
{
  "name": "gofer",
  "main": "lib/gofer.js",
  // General intuition: This is what this package contributes to the import map.
  // It currently includes both information of how it would expect to be used from the outside
  // and how the import map for its internal files ("import map scope") would look like.
  // A possible improvement would be to split these into two separate entries:
  // `exports` (hoisted) and `imports` (internal).
  "exports": { // default "exports": { "/": "/" },
    // These entries will be included in the scope of the user/referrer of this package.
    "/esm": "./lib/gofer.mjs",
    // "my-pkg/esm.mjs" containing "export * from './lib/gofer.mjs'" <- this is the alternative
  },
  "imports": {
    // Entries that start with a "." refer to URLs relative to the package root.
    // When importing the given URL from inside of this scope, it will be rewritten.
    // This process is not recursive (see: import maps).
    "./lib/fetch.js": "./lib/fetch.node.js",
    // This isn't restricted to relative URLs. The same would also work for absolute URLs.
    // Ignore the fallback syntax if it's confusing, this is modelled after:
    // https://github.com/WICG/import-maps#for-built-in-modules-in-browsers-without-import-maps
    "https://cdnjs.cloudflare.com/ajax/libs/fetch/3.0.0/fetch.min.js": [
      "std:fetch",
      "node:fetch",
      "https://cdnjs.cloudflare.com/ajax/libs/fetch/3.0.0/fetch.min.js",
    ],

    "./lib/request.js": [
      "./lib/request.node.js",
      "./lib/request.browser.js",
    ],
    "./build/Release/native.node": [
      "./build/Release/native.windows.node",
      "./build/Release/native.osx.node",
      "./build/Release/native.linux.node",
    ],

    // Since everything that doesn't start with `~` is just included in the import map scope,
    // we can also include arbitrary entries. Somebody wants to remap `fs`? They *could*.
    "fs": "https://unpkg.com/graceful-fs@4.1.15/graceful-fs.js",
  },

  // Now for the crazy bit: We *could*, in theory, replace #dependencies as well.
  // Downside: One loader wouldn't be able to resolve the same semver range
  // in different ways. Multiple copies still work.
  "imports": {
    "express": "https://unpkg.com/express@4.x",
    // Potential sugar:
    // Though it would be interesting how it would affect hoisting of import maps in express.
    // E.g. would that be part of installing it / generating the lock file?
    // Simplest case would be to have 1-2 entries and just use the slash expansion:
    "express/": "https://unpkg.com/express@4.x/",
    // But that isn't actually the same thing, it uses redirects instead of rewrites before fetching.
    // It *could* also mean that the file at https://unpkg.com/express@4.x/ contains something
    // along the lines of:
    //    export * from 'https://unpkg.com/express@4.x/';
    // Although that wouldn't properly handle default exports..?
    // https://github.com/guybedford/proposal-export-star-default never happened. :(

    // One possible way out of this: Use a custom protocol that must be replaced before real use,
    // e.g. when generating the import map.
    // This means the package would (de-facto) not work with the default loader
    // unless it has special support for this.
    // The following may load meta data from the given URL, determine
    // the exports, and generate multiple entries in the import map accordingly.
    "figgy-pudding": "js-bundle+https://legacy@registry.entropic.dev/figgy-pudding",
    // While the above isn't usable in a browser as-is (yet?), the final composed
    // import map would be compatible. It would unlikely to be anything but sugar
    // for package managers and bundlers (and/or node).

    // The easiest solution may be to keep using a separate field (or file) for dependency lists.
  },
}
```

## Motivating Examples

* A package (`react-dom`) has a dedicated entrypoint `react-dom/server` for code that isn't compatible with a browser environment.
* A package (`angular`) exposes multiple independent APIs, modeled via import paths like `angular/common/http`.
* A package (`lodash`) allows to import individual functions, e.g. `lodash/map`.
* A package is exclusively exposing an ESM interface.
* A package is exclusively exposing a CJS interface.
* A package is exposing both an ESM and a CJS interface.
* A project wants to mix both ESM and CJS code, with CJS running as part of the ESM module graph.
* A package wants to expose multiple entrypoints as its public API without leaking internal directory structure.

## High Level Considerations

* The baseline behavior of relative imports should match a browser's with a simple file server.
  This implies that `./x` will only ever import exactly the sibling file "x" without appending paths or extensions.
  `"x"` is never resolved to `x.mjs` or `x/index.mjs` (or the `.js` equivalents).
* The primary compatibility boundary are bare specifiers. Relative and absolute imports can follow simpler rules.
* Resolution should not depend on file extensions, allowing ESM syntax in `.js` files.
* The directory structure of a module should be treated as private implementation detail.

## `package.json` Interface

We propose a field in `package.json` to specify one or more entrypoint locations when importing bare specifiers.

> **The key is TBD, the examples use `"exports"` as a placeholder.**
> **Neither the name nor the fact that it exists top-level is final.**

The `package.json` `"exports"` interface will only be respected for bare specifiers, e.g. `import _ from 'lodash'` where the specifier `'lodash'` doesn’t start with a `.` or `/`.

`"exports"` works in concert with the `package.json` `"type": "module"` signifier that a package can be imported as ESM by Node.
`"exports"` by itself does not signify that a package should be treated as ESM, but `"exports"` is currently ignored for packages `require`d as CommonJS.
Extending this feature to CommonJS may occur in the future, but issues of backward compatibility would need to be addressed.
At the moment `"exports"` is limited to ESM packages or dual ESM/CommonJS packages that are imported as ESM.

For `"type": "module"` packages with both `"main"` and `"exports"`, a main entrypoint defined by `"exports"` takes precedence over one defined by `"main"`.
This allows a package to be importable as either ESM or CommonJS.
If a `package.json` lacks `"exports"` but includes `"type": "module"`, `"main"` defines the package’s ESM entrypoint.

### Example

Here’s a complete `package.json` example, for a hypothetical module named `@momentjs/moment`:

```js
{
  "name": "@momentjs/moment",
  "version": "0.0.0",
  "type": "module",
  "main": "./dist/index.js",
  "exports": {
    ".": "./src/moment.mjs",
    "./": "./src/util/",
    "./timezones/": "./data/timezones/",
    "./timezones/utc": "./data/timezones/utc/index.mjs"
  }
}
```

Within the `"exports"` object, the key string after the `'.'` is concatenated on the end of the name field, e.g. `import utc from '@momentjs/moment/timezones/utc'` is formed from `'@momentjs/moment'` + `'/timezones/utc'`. Note that this is string manipulation, not a file path: `"./timezones/utc"` is allowed, but just `"timezones/utc"` is not. The `.` is a placeholder representing the package name. The main entrypoint is therefore the dot string, `".": "./src/moment.mjs"`.

Keys that end in slashes can map to folder roots, following the [pattern in the browser import maps proposal](https://github.com/WICG/import-maps#packages-via-trailing-slashes): `"./timezones/": "./data/timezones/"` would allow `import pdt from "@momentjs/moment/timezones/pdt.mjs"` to import `./data/timezones/pdt.mjs`.

- Using `"./"` maps the root, so `"./": "./src/util/"` would allow `import tick from "@momentjs/moment/tick.mjs"` to import `./src/util/tick.mjs`.

- Mapping a key of `"./"` to a value of `"./"` exposes all files in the package, where `"./": "./"` would allow `import privateHelpers from "@momentjs/moment/private-helpers.mjs"` to import `./private-helpers.mjs`.

- When mapping to a folder root, both the left and right sides must end in slashes: `"./": "./dist/"`, not `".": "./dist"`.

- Unlike in CommonJS, there is no automatic searching for `index.js` or `index.mjs` or for file extensions. This matches the [behavior of the import maps proposal](https://github.com/WICG/import-maps#packages-via-trailing-slashes).

The value of an export, e.g. `"./src/moment.mjs"`, must begin with `.` to signify a relative path (e.g. "./src" is okay, but `"/src"` or `"src"` are not). This is to reserve potential future use for `"exports"` to export things referenced via specifiers that aren’t relatively-resolved files, such as other packages or other protocols.

There is the potential for collisions in the exports, such as `"./timezones/"` and `"./timezones/utc"` in the example above (e.g. if there’s a file named `utc` in the `./data/timezones` folder).
Rough outline of a possible resolution algorithm:

1. Find the package matching the base specifier, e.g. `@momentjs/moment` or `request`.
1. Load its exports map.
1. If there is an exact match for the requested specifier, return the resolution.
1. Otherwise, find the longest matching path prefix. If there is a path prefix, return the resolution by applying the prefix.
1. Return an error - no mapping found.

In the future, the algorithm might be adjusted to align with work done in the [import maps proposal](https://github.com/domenic/import-maps).

### Usage

For a consumer, the above `@momentjs/moment` and `request` packages can be used as follows, assuming the user’s project is in `/app` with `/app/package.json` and `/app/node_modules`:

```js
import request from 'request';
// Loads file:///app/node_modules/request/request.mjs

import request from './node_modules/request/request.mjs';
// Loads file:///app/node_modules/request/request.mjs

import request from 'file:///app/node_modules/request/request.mjs';
// Loads file:///app/node_modules/request/request.mjs

import utc from '@momentjs/moment/timezones/utc';
// Loads file:///app/node_modules/@momentjs/moment/timezones/utc/index.mjs
```

The following don’t work - please note that **error messages and codes are TBD**:

```js
import request from 'request/';
// Error: trailing slash not mapped

import request from 'request/request.mjs';
// Error: no such mapping

import moment from '@momentjs/moment/';
// Error: trailing slash not allowed (cannot import folders, only files)

import request from 'file:///app/node_modules/request';
// Error: folders cannot be imported, package.json only considered for bare imports

import request from 'file:///app/node_modules/request/';
// Error: folders cannot be imported, package.json only considered for bare imports

import utc from '@momentjs/moment/timezones/utc/'; // Note trailing slash
// Error: folders cannot be imported (there is no index.* magic)
```

### Prior Art

* [`package.json#browser`](https://github.com/defunctzombie/package-browser-field-spec)
* [Import Maps](https://github.com/domenic/import-maps)
* [`package.json#mimes`](https://github.com/nodejs/modules/pull/160)
* [node.js ESM resolver spec](https://github.com/nodejs/ecmascript-modules/pull/12)

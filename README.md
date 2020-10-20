# dwebstore
[![Build Status](https://travis-ci.com/distributedweb/dwebstore.svg?branch=master)](https://travis-ci.com/distributedweb/dwebstore)

This module is the canonical implementation of the "dwebstore" interface, which exposes a DDatabase factory and a set of associated functions for managing generated Hypercores.

A dwebstore is designed to efficiently store and replicate multiple sets of interlinked Hypercores, such as those used by [DDrive](https://github.com/distributedweb/ddrive) and [mountable-dwebtrie](https://github.com/distributedweb/dwebtrie), removing the responsibility of managing custom storage/replication code from these higher-level modules.

In order to do this, dwebstore provides:
1. __Key derivation__ - all writable DDatabase keys are derived from a single master key.
2. __Caching__ - Two separate caches are used for passively replicating cores (those requested by peers) and active cores (those requested by the owner of the dwebstore).
3. __Storage bootstrapping__ - You can create a `default` DDatabase that will be loaded when a key is not specified, which is useful when you don't want to reload a previously-created DDatabase by key.
4. __Namespacing__ - If you want to create multiple compound data structures backed by a single dwebstore, you can create namespaced corestores such that each data structure's `default` feed is separate.

### Installation
`npm i dwebstore --save`

### Usage
A dwebstore instance can be constructed with a random-access-storage module, a function that returns a random-access-storage module given a path, or a string. If a string is specified, it will be assumed to be a path to a local storage directory:
```js
const DWebstore = require('dwebstore')
const ram = require('random-access-memory')
const store = new DWebstore(ram)
await store.ready()
```

Hypercores can be generated with both the `get` and `default` methods. If the first writable core is created with `default`, it will be used for storage bootstrapping. We can always reload this bootstrapping core off disk without your having to store its public key externally. Keys for other hypercores should either be stored externally, or referenced from within the default core:
```js
const core1 = store1.default()
```
_Note: You do not have to create a default feed before creating additional ones unless you'd like to bootstrap your dwebstore from disk the next time it's instantiated._

Additional hypercores can be created by key, using the `get` method. In most scenarios, these additional keys can be extracted from the default (bootstrapping) core. If that's not the case, keys will have to be stored externally:
```js
const core2 = store1.get({ key: Buffer(...) })
```
All hypercores are indexed by their discovery keys, so that they can be dynamically injected into replication streams when requested.

Two corestores can be replicated with the `replicate` function, which accepts ddatabase's `replicate` options:
```js
const store1 = new DWebstore(ram)
const store2 = new DWebstore(ram)
await Promise.all([store1.ready(), store2.ready()]

const core1 = store2.get()
const core2 = store2.get({ key: core1.key })
const stream = store1.replicate(true, { live: true })
stream.pipe(store2.replicate(false, { live: true })).pipe(stream) // This will replicate all common cores.
```

### API
#### `const store = dwebstore(storage, [opts])`
Create a new dwebstore instance. `storage` can be either a random-access-storage module, or a function that takes a path and returns a random-access-storage instance.

Opts is an optional object which can contain any DDatabase constructor options, plus the following:
```js
{
  cacheSize: 1000 // The size of the LRU cache for passively-replicating cores.
}
```

#### `store.default(opts)`
Create a new default ddatabase, which is used for bootstrapping the creation of subsequent hypercores. Options match those in `get`.

#### `store.get(opts)`
Create a new ddatabase. Options can be one of the following:
```js
{
  key: 0x1232..., // A Buffer representing a ddatabase key
  discoveryKey: 0x1232..., // A Buffer representing a ddatabase discovery key (must have been previously created by key)
  ...opts // All other options accepted by the ddatabase constructor
}
```

If `opts` is a Buffer, it will be interpreted as a ddatabase key.

#### `store.on('feed', feed, options)`

Emitted everytime a feed is loaded internally (ie, the first time get(key) is called).
Options will be the full options map passed to .get.

#### `store.replicate(isInitiator, [opts])`
Create a replication stream that will replicate all cores currently in memory in the dwebstore instance.

When piped to another dwebstore's replication stream, only those cores that are shared between the two corestores will be successfully replicated.

#### `store.list()`
Returns a Map of all cores currently cached in memory. For each core in memory, the map will contain the following entries:
```
{
  discoveryKey => core,
  ...
}
```

#### `const namespacedStore = store.namespace('some-name')`
Create a "namespaced" dwebstore that uses the same underlying storage as its parent, and mirrors the complete dwebstore API. 

`namespacedStore.default` returns a different default core, using the namespace as part of key generation, which makes it easier to bootstrap multiple data structures from the same dwebstore. The general pattern is for all data structures to bootstrap themselves from their dwebstore's default feed:
```js
const store = new DWebstore(ram)
const drive1 = new DDrive(store.namespace('drive1'))
const drive2 = new DDrive(store.namespace('drive2'))
```

Namespaces currently need to be saved separately outside of dwebstore (as a mapping from key to namespace), so that data structures remain writable across restarts. Extending the above code, this might look like:
```js
async function getDrive (opts = {}) {
  let namespace = opts.key ? await lookupNamespace(opts.key) : await createNamespace()
  const namespacedCorestore = store.namespace(namespace)
  const drive = new DDrive(namespacedCorestore)
  await saveNamespace(drive.key, namespace)
}
```

#### `store.close(cb)`
Close all hypercores previously generated by the dwebstore.

### License
MIT

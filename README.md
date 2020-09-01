# dwebstore
[![Build Status](https://travis-ci.com/andrewosh/dwebstore.svg?token=WgJmQm3Kc6qzq1pzYrkx&branch=master)](https://travis-ci.com/andrewosh/dwebstore)

__Note: These APIs are still in progress and are subject to change prior to the DDrive v10 release.__

This module is the canonical implementation of the "dwebstore" interface, which exposes a DDatabase factory and a set of associated functions for managing generated Hypercores.

A dwebstore is designed to efficiently store and replicate multiple sets of interlinked Hypercores, such as those used by [DDrive](https://github.com/distributedweb/ddrive) and [mountable-dwebtrie](https://github.com/andrewosh/dwebtrie), removing the responsibility of managing custom storage/replication code from these higher-level modules.

In order to do this, dwebstore provides:
1. __Key derivation__ - all writable DDatabase keys are derived from a single master key.
2. __A dependency graph__ - you can specify parent/child relationships between Hypercores (a DDrive's content feed is a child of its metadata feed). The dependency graph is used to find the minimal number of Hypercores that should be replicated over a given stream. The graph can also be replicated (it is a dwebtrie).
3. __Caching__ - Two separate caches are used for passively replicating cores (those requested by peers) and active cores (those requested by the owner of the dwebstore).
4. __Storage bootstrapping__ - You can create a `default` DDatabase that will be loaded when a key is not specified, which is useful when you don't want to reload a previously-created DDatabase by key.
5. __Namespacing__ - If you want to create multiple compound data structures backed by a single dwebstore, you can create namespaced corestores such that each data structure's `default` feed is separate.

### Installation
`npm i dwebstore --save`

### Usage
A dwebstore instance can be constructed with a random-access-storage module, a function that returns a random-access-storage module given a path, or a string. If a string is specified, it will be assumed to be a path to a local storage directory:
```js
const DWebstore = require('dwebstore')
const ram = require('random-access-memory')
const raf = require('random-access-file')
const store1 = new DWebstore(ram)
const store2 = new DWebstore(path => raf('store2/' + path))
const store3 = new DWebstore('my-storage-dir')
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

Two corestores can be replicated with the `replicate` function, which accepts ddatabase's `replicate` options, as well as an optional starting node in the dependency graph (a discovery key). When specifying a starting node, only that node's children will be replicated into the stream:
```js
const store2 = dwebstore(ram)
const core3 = store2.get(core1.key)
const core4 = store2.get({ parents: [core1.key] })
const stream = store2.replicate(true, core3.discoveryKey, { live: true }) // This will replicate core3 and core4.
stream.pipe(store2.replicate(false, { live: true })).pipe(stream) // This will replicate all common cores.
```

### API
#### `const store = dwebstore(storage, [opts])`
Create a new dwebstore instance. `storage` can be either a random-access-storage module, or a function that takes a path and returns a random-access-storage instance.

Opts is an optional object which can contain the following:
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
  parents: [ 0x1234, 0xabba, ...], // A list of ddatabase keys specifying the core's parent dependencies.
  ...opts // All other options accepted by the ddatabase constructor
}
```

If `opts` is a Buffer, it will be interpreted as a ddatabase key.

#### `store.on('feed', feed, options)`

Emitted everytime a feed is loaded internally (ie, the first time get(key) is called).
Options will be the full options map passed to .get.

#### `store.replicate(isInitiator, [discoveryKey], [opts])`
Create a replication stream for either all managed hypercores, or for all hypercores that are children of the one with the specified `discoveryKey`. The replication options are passed directly to DDatabase.

#### `store.list()`
Returns a Map of all cores currently cached in memory. For each core in memory, the map will contain the following entries:
```
{
  discoveryKey => core,
  ...
}
```

#### `store.close(cb)`
Close all hypercores previously generated by the dwebstore.

### License
MIT

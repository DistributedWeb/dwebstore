## 5.7.0
- Simplifies internal reference counting and namespace handling. This removes the `NamespacedCorestore` class, but does not alter the interface.
- Uses `refpool` for reference handling.
- Removes `Nanoguard` and the undocumented `this.guard` property on DWebstore.
- Removes the private `_name` option to `DWebstore.get` in favor of a public `name` option.

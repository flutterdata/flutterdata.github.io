---
title: "Local Adapters"
weight: 21
menu: docs
---

Local adapters access the local storage, which for now is only Hive.

It is extremely rare to have to override a local adapter, so use with caution if you do.

A particularly useful use-case is data migration as `LocalAdapter`'s `deserialize` will be called after loading raw data from the Hive box and before the `json_serializable` call.

Example:

```dart
mixin TaskLocalAdapter on LocalAdapter<Task> {
  @override
  Task deserialize(Map<String, dynamic> map) {
    // transform map from old format to new format
  }
}
```

Activate the use of the overridden adapter with:

```dart
@DataRepository(
  [TaskAdapter],
  localAdapters: [TaskLocalAdapter],
)
class Task extends DataModel<Task> {
  // ...
}
```
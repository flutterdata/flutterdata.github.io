---
title: Models
date: 2020-04-20T19:01:08-03:00
weight: 30
menu: docs
---

Flutter Data models are data classes that extend `DataModel` and are annotated with `@DataRepository`:

```dart {hl_lines=[3]}
@DataRepository([TaskAdapter])
@JsonSerializable()
class Task extends DataModel<Task> {
  @override
  final int? id;
  final String title;
  final bool completed;

  Task({this.id, required this.title, this.completed = false});
}
```

`DataModel` automatically registers new data classes within the framework and enforces the implementation of an `id` getter. Use the type that better suits you: `int?` and `String?` are the most common.

{{< notice >}}
The `json_serializable` library is helpful but not required.

- Model with `@JsonSerializable`? You don't need to declare `fromJson` or `toJson`
- Model without `@JsonSerializable`? You must declare `fromJson` and `toJson`

If you choose it, you can make use of `@JsonKey` and other configuration parameters as usual. A common use-case is having a different remote `id` attribute such as `objectId`. Annotating `id` with `@JsonKey(name: 'objectId')` takes care of it.
{{< /notice >}}

## Freezed support

Here's an example:

```dart
@freezed
@DataRepository([TaskAdapter])
class Task extends DataModel<Task> with _$Task {
  Task._();

  factory Task({
    int? id,
    required String name,
    required BelongsTo<User> user,
  }) = _Task;

  factory Task.fromJson(Map<String, dynamic> json) => _$TaskFromJson(json);
}
```

Unions haven't been tested yet.

## Omitting attributes

In order to omit an attribute simply use `@JsonKey(ignore: true)`.

## Extension methods

In addition, various useful methods become available on the class:

### save

```dart
final user = User(id: 1, name: 'Frank');
await user.save();
```

The call is syntax-sugar for [Repository#save](/docs/repositories/#save) and takes the same arguments (except the model).

Or, saving locally (i.e. `remote: false`) with a sync API:

```dart
final user = User(id: 1, name: 'Frank');
user.saveLocal();
```

### delete

```dart
final user = await repository.findOne(1);
await user.delete();
```

It is syntax-sugar for [Repository#delete](/docs/repositories/#delete) and takes the same arguments (except the model).

Or, deleting locally (i.e. `remote: false`) with a sync API:

```dart
final user = User(id: 1, name: 'Frank');
user.deleteLocal();
```

### find

```dart
final updatedUser = await user.find();
```

It's syntax-sugar for [Repository#findOne](/docs/repositories/#findone) and takes the same arguments (except the model/ID).

Or, reloading locally (i.e. `remote: false`) with a sync API:

```dart
final user = User(id: 1, name: 'Frank');
final user2 = user.reloadLocal();
```

### withKeyOf

Used whenever we need to transfer identity to a model without identity (that is, without an ID).

```dart
final user = User(id: 1, 'Parker');
final user2 = user.copyWith(name: 'Frank').withKeyOf(user);
```

`id` will still be `null` but saving and retreiving will work:

```dart
await user2.save(remote: false);
final user3 = await user2.find();
// user3.id == 1
```

{{< notice >}}
Any Dart file that wants to use these extensions must import the library.

```dart
import 'package:flutter_data/flutter_data.dart';
```

**VSCode protip!** Type `Command + .` over the missing method and choose to import!

You can also disable them by hiding the extension:

```dart
import 'package:flutter_data/flutter_data.dart' hide DataModelExtension;
```

{{< /notice >}}


## Polymorphic models

An example where `Staff` and `Customer` are both `User`s:

```dart
abstract class User<T extends User<T>> extends DataModel<T> {
  final String id;
  final String name;
  User({this.id, this.name});
}

@JsonSerializable()
@DataRepository([JSONAPIAdapter, BaseAdapter])
class Customer extends User<Customer> {
  final String abc;
  Customer({String id, String name, this.abc}) : super(id: id, name: name);
}

@JsonSerializable()
@DataRepository([JSONAPIAdapter, BaseAdapter])
class Staff extends User<Staff> {
  final String xyz;
  Staff({String id, String name, this.xyz}) : super(id: id, name: name);
}
```

{{< contact >}}
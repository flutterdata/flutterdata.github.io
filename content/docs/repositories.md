---
title: "Repositories"
weight: 10
menu: docs
---

Flutter Data is organized around the concept of [models](/docs/models) which are data classes extending `DataModel`.

```dart
@DataRepository([TaskAdapter])
class Task extends DataModel<Task> {
  @override
  final int? id;
  final String title;
  final bool completed;

  Task({this.id, required this.title, this.completed = false});

  // ...
}
```

When annotated with `@DataRepository` (and [adapters](/docs/adapters) as arguments, as we'll see later) a model gets its own fully-fledged repository.

`Repository` is the API used to interact with models, whether local or remote.

Assuming a `Task` model and its corresponding `Repository<Task>`, let's see how to retrieve such resources from an API.

## Finders

### findAll

Using `ref.tasks` (short for `ref.watch(tasksRepositoryProvider)`) to obtain a repository we can find all resources in the collection.

```dart
Repository<Task> repository = ref.tasks;
final tasks = await repository.findAll();

// GET http://base.url/tasks
```

This async call triggered a request to `GET http://base.url/tasks`.

{{< notice >}}
  {{< partial "magic1.md" >}}
{{< /notice >}}

Method signature:

```dart
Future<List<T>?> findAll({
  bool? remote,
  bool? background,
  Map<String, dynamic>? params,
  Map<String, String>? headers,
  bool? syncLocal,
  OnSuccessAll<T>? onSuccess,
  OnErrorAll<T>? onError,
  DataRequestLabel? label,
});
```

The `remote`, `background`, `params`, `headers`, `onSuccess`, `onError` and `label` arguments are detailed in [common arguments](#common-arguments) below.

The `syncLocal` argument instructs local storage to synchronize the exact resources returned from the remote source (for example, to reflect server-side deletions).

```dart
final tasks = await ref.tasks.findAll(syncLocal: true);
```

Consider this example:

If a first call to `findAll` returns data for task IDs `1`, `2`, `3` and a second call updated data for `2`, `3`, `4` you will end up in your local storage with: `1`, `2` (updated), `3` (updated) and `4`.

Passing `syncLocal: true` to the second call will leave the local storage state with `2`, `3` and `4`.

📚 [See API docs](https://pub.dev/documentation/flutter_data/latest/flutter_data/Repository/findAll.html)

### findOne

Finds a resource by ID and saves it in local storage.

```dart
final task = await ref.tasks.findOne(1);

// GET http://base.url/tasks/1
```

{{< notice >}}

Similar to what's shown above in [findAll](#findall), Flutter Data resolves the URL by using the `urlForFindOne` function. We can override this in an [adapter](/docs/adapters).

For example, use path `/tasks/something/1`:

```dart
mixin TaskURLAdapter on RemoteAdapter<Task> {
  @override
  String urlForFindOne(id, params) => '$type/something/$id';
}

// would result in GET http://base.url/tasks/something/1
```

{{< /notice >}}

It can also take a `T` with an ID:

```dart
final task = await ref.tasks.findOne(anotherTaskWithId3);

// GET http://base.url/tasks/3
```

Method signature:

```dart
Future<T?> findOne(
  Object id, {
  bool? remote,
  bool? background,
  Map<String, dynamic>? params,
  Map<String, String>? headers,
  OnSuccessOne<T>? onSuccess,
  OnErrorOne<T>? onError,
  DataRequestLabel? label,
});
```

The `remote`, `background`, `params`, `headers`, `onSuccess`, `onError` and `label` arguments are detailed in [common arguments](#common-arguments) below.

📚 [See API docs](https://pub.dev/documentation/flutter_data/latest/flutter_data/Repository/findOne.html)

## Save and delete

### save

Persists a model to local storage and remote.

```dart
final savedTask = await repository.save(task);
```

{{< notice >}}

Want to use the `PUT` verb instead of `PATCH`? Use this [adapter](/docs/adapters):

```dart
mixin TaskURLAdapter on RemoteAdapter<Task> {
  @override
  String methodForSave(id, params) => id != null ? DataRequestMethod.PUT : DataRequestMethod.POST;
}
```
{{< /notice >}}

Method signature:

```dart
Future<T> save(
  T model, {
  bool? remote,
  Map<String, dynamic>? params,
  Map<String, String>? headers,
  OnSuccessOne<T>? onSuccess,
  OnErrorOne<T>? onError,
  DataRequestLabel? label,
});
```

The `remote`, `params`, `headers`, `onSuccess`, `onError` and `label` arguments are detailed in [common arguments](#common-arguments) below.

📚 [See API docs](https://pub.dev/documentation/flutter_data/latest/flutter_data/Repository/save.html)

### delete

Deletes a model from local storage and sends a `DELETE` HTTP request.

```dart
await repository.delete(model);
```

Method signature:

```dart
Future<T?> delete(
  Object model, {
  bool? remote,
  Map<String, dynamic>? params,
  Map<String, String>? headers,
  OnSuccessOne<T>? onSuccess,
  OnErrorOne<T>? onError,
  DataRequestLabel? label,
});
```

The `remote`, `params`, `headers`, `onSuccess`, `onError` and `label` arguments are detailed in [common arguments](#common-arguments) below.

📚 [See API docs](https://pub.dev/documentation/flutter_data/latest/flutter_data/Repository/delete.html)


## Watchers

`DataState` is a class that holds state related to resource fetching and is practical in UI applications. It is returned in `watchAll` and `watchOne`.

```dart
class DataState<T> with EquatableMixin {
  T model;
  bool isLoading;
  DataException? exception;
  StackTrace? stackTrace;
  // ...
}
```

It's typically used in a widget's `build` method like:

```dart
class MyApp extends HookConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.tasks.watchAll();
    if (state.isLoading) {
      return CircularProgressIndicator();
    }
    if (state.hasException) {
      return ErrorScreen(state.exception, state.stackTrace);
    }
    return ListView(
      children: [
        for (final task in state.model)
          Text(task.title),
    // ...
  }
}
```

{{< notice >}}

Why not used a [Freezed union](https://pub.dev/packages/freezed#union-types-and-sealed-classes) instead?

Because without forcing to branch, `DataState` easily allows rebuilding widgets when multiple substates happen simultaneously – a very common pattern. The tradeoff is having to remember to check for the `loading` and `error` substates.

{{< /notice >}}


### watchAll

Watches all models of a given type in local storage (through `ref.watch` and `watchAllNotifier`).

For updates to any model of type `Task` to prompt a rebuild we can use:

```dart
class TasksScreen extends HookConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.tasks.watchAll();
    if (state.isLoading) {
      return CircularProgressIndicator();
    }
    // use state.model which is a List<Task>
  }
);
```

By default when first rendered it triggers a background [`findAll`](#findall) call with `remote`, `params`, `headers`, `syncLocal` and `label` arguments. See [common arguments](#common-arguments).

Method signature:

```dart
DataState<List<T>?> watchAll({
  bool? remote,
  Map<String, dynamic>? params,
  Map<String, String>? headers,
  bool? syncLocal,
  String? finder,
  DataRequestLabel? label,
});
```

But this can easily be overridden. Any method in the [adapter](/docs/adapters) with the **exact [`findAll`](#findall) method signature** and annotated with `@DataFinder()` will be available to supply to the `finder` argument as a string (method name).

Pass `remote: false` to prevent any remote fetch at all.

{{< notice >}}
**Note:** Both [`watchAllProvider`](https://pub.dev/documentation/flutter_data/latest/flutter_data/Repository/watchAllProvider.html) and [`watchAllNotifier`](https://pub.dev/documentation/flutter_data/latest/flutter_data/Repository/watchAllNotifier.html) are also available.
{{< /notice >}}

📚 [See API docs](https://pub.dev/documentation/flutter_data/latest/flutter_data/Repository/watchAll.html)


### watchOne

Watches a model of a given type in local storage (through `ref.watch` and `watchOneNotifier`).

For updates to a given model of type `Task` to prompt a rebuild we can use:

```dart
class TaskScreen extends HookConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.tasks.watchOne(1);
    if (state.isLoading) {
      return CircularProgressIndicator();
    }
    // use state.model which is a Task
  }
);
```

By default when first rendered it triggers a background [`findOne`](#findone) call with `model`, `remote`, `params`, `headers` and `label` arguments. See [common arguments](#common-arguments).

Method signature:

```dart
DataState<T?> watchOne(
  Object model, {
  bool? remote,
  Map<String, dynamic>? params,
  Map<String, String>? headers,
  AlsoWatch<T>? alsoWatch,
  String? finder,
  DataRequestLabel? label,
});
```

But this can easily be overridden. Any method in the [adapter](/docs/adapters) with the **exact [`findOne`](#findone) method signature** and annotated with `@DataFinder()` will be available to supply to the `finder` argument as a string (method name).

Pass `remote: false` to prevent any remote fetch at all.

In addition, this watcher can react to relationships via `alsoWatch`:

```dart
watchOneNotifier(3, alsoWatch: (task) => [task.user]);
```

This feature is extremely powerful, actually any number of relationships can be watched:

```dart
watchOneNotifier(3, alsoWatch: (task) => [task.reminders, task.user, task.user.profile, task.user.profile.comments]);
```


{{< notice >}}
**Note:** Both [`watchOneProvider`](https://pub.dev/documentation/flutter_data/latest/flutter_data/Repository/watchOneProvider.html) and [`watchOneNotifier`](https://pub.dev/documentation/flutter_data/latest/flutter_data/Repository/watchOneNotifier.html) are also available.
{{< /notice >}}


### watch

This method takes a `DataModel` and watches its local changes.

```dart
class TaskScreen extends HookConsumerWidget {
  final Task model;
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final task = ref.tasks.watch(model);
    return Text(task.title);
  }
);
```

Note that it returns a model, not a `DataState`.

### notifierFor

Obtain the notifier for a model. Does not trigger a remote request.

```dart
final notifier = ref.tasks.notifierFor(task);

// equivalent to
ref.tasks.watchOneNotifier(task, remote: false);
```

{{< notice >}}
By default, changes will be notified immediately and trigger widget rebuilds.

For performance improvements, they can be throttled [by overriding `graphNotifierThrottleDurationProvider`](/docs/initialization).
{{< /notice >}}

## Common arguments

### remote

Request only models in local storage:

```dart
final tasks = await ref.tasks.findAll(remote: false);
```

Argument is of type `bool` and the default is `true`.

{{< notice >}}
In addition to adapters, the `@DataRepository` annotation can take a `remote` boolean argument which will make it the default for the repository.

```dart
@DataRepository([TaskAdapter], remote: false)
class Task extends DataModel<Task> {
  // by default no operation hits the remote endpoint
}
```

{{< /notice >}}

### params

Include query parameters (of type `Map<String, dynamic>`, in this case used for pagination and resource inclusion):

```dart
final tasks = await ref.tasks.findAll(
  params: {'include': 'comments', 'page': { 'limit': 20 }}
);

// GET http://base.url/tasks?include=comments&page[limit]=20
```

### headers

Include HTTP headers as a `Map<String, String>`:

```dart
final tasks = await ref.tasks.findAll(
  headers: { 'Authorization': 't0k3n' }
);
```

### onSuccess

Overrides the handler for the success state, useful when requiring a specific transformation of raw response data.

```dart
await ref.tasks.save(
  task,
  onSuccess: (data, label, adapter) async {
    final model = await adapter.onSuccess(data, label);
    return model as Task;
  },
);
```

[`RemoteAdapter#onSuccess`](https://pub.dev/documentation/flutter_data/latest/flutter_data/RemoteAdapter/onSuccess.html) is the default (overridable, of course). It esentially boils down to calls to `deserialize`.

### onError

Overrides the error handler:

```dart
await ref.tasks.save(
  task,
  onError: (error, label, adapter) async {
    throw WrappedException(error);
  },
);
```

[`RemoteAdapter#onError`](https://pub.dev/documentation/flutter_data/latest/flutter_data/RemoteAdapter/onError.html) is the default (overridable, of course). It essentially rethrows the error except if it is due to loss of connectivity or the remote resource was not found (HTTP 404).

### label

Optional argument of type `DataRequestLabel`. [See below](#logging-and-labels).

## Logging and labels

Labels are used in Flutter Data to easily track requests in logs. They carry an auto-generated `requestId` along with type and ID. They are provided by default and also by default finders log different events.

Some examples:

- `findAll/tasks@b5d14c`
- `findOne/users#3@c4a1bb`
- `findAll/tasks@b5d14c<c4a1bb`

Request IDs can be nested like the last one above, where the `b5d14c` call happened inside `c4a1bb`. In other words, during the request for `User` with ID `3`, a collection of tasks was also requested (presumably for that same user).

In the console these would be logged as:

```text
flutter: 05:961   [findAll/tasks@b5d14c] requesting
flutter: 05:973   [findOne/users#3@c4a1bb] requesting
flutter: 05:974     [findAll/tasks@b5d14c<c4a1bb] requesting
```

with the nested labels properly indented.

[Watchers](#watchers) also output useful logs in level 1 and 2.

Log levels can be set via the `logLevel` argument in `log`, and to adjust the global level:

```dart
ref.tasks.logLevel = 2;
```

In order to log custom information use something like:

```dart
final label = DataRequestLabel('save', type: 'users', id: '3');
ref.tasks.log(label, 'testing labels');

// or

final nestedLabel1 = DataRequestLabel('findAll',
  type: 'other', requestId: 'ff01b1', withParent: label);
ref.tasks.log(label, 'testing nested labels', logLevel: 2);
```

## Architecture overview

This is the dependency graph for an app with models `User` and `Task`:

<p class="py-6">
  <img src="/images/deps.png" style="max-width: 650px">
</p>

Clients should only interact with repositories and adapters, while using the [Adapter API](/docs/adapters) to customize behavior.

`LocalAdapter`, `GraphNotifier` and `LocalStorage` are internal concerns. **Do not use them.**

{{< contact >}}

{{< internal >}}
GRAPHVIZ

```
digraph g {
  rankdir="TB"
  "RemoteAdapter<Task>" -> "LocalAdapter<Task>" -> "LocalStorage"
  "RemoteAdapter<User>" -> "LocalAdapter<User>" -> "LocalStorage"
  "LocalAdapter<Task>" -> "GraphNotifier"
  "LocalAdapter<User>" -> "GraphNotifier"
  "GraphNotifier" -> "LocalStorage"
  "Repository<Task>" -> "RemoteAdapter<Task>"
  "Repository<User>" -> "RemoteAdapter<User>"
  "Repository<Task>" -> "RemoteAdapter<User>"
  "Repository<User>" -> "RemoteAdapter<Task>"
  "tasksRepositoryProvider" -> "Repository<Task>"
  "usersRepositoryProvider" -> "Repository<User>"
}
```

{{< /internal >}}

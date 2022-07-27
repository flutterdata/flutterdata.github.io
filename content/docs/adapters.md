---
title: "Adapters"
weight: 20
menu: docs
---

Flutter Data's building blocks are called adapters, making it extremely customizable and composable.

Adapters are essentially Dart mixins applied on `RemoteAdapter<T>`.

## Overriding basic behavior

Several pieces of information are required, for example, to construct a remote `findAll` call on a `Repository<Task>`. The framework takes a sensible guess and makes that `GET /tasks` by default.

Still, a base URL is necessary and the endpoint parts should be overridable.

The way we use these adapters is by declaring them on our `@DataRepository` annotation in the corresponding model. For example:

```dart {hl_lines=[1 2 3 4 7]}
mixin JSONServerTaskAdapter on RemoteAdapter<Task> {
  @override
  String get baseUrl => 'https://myapi.com/v1/';
}

@JsonSerializable()
@DataRepository([JSONServerTaskAdapter])
class Task extends DataModel<Task> {
  final int? id;
  final String title;
  final bool completed;

  Task({this.id, required this.title, this.completed = false});
}
```

What if the endpoint actually is at `https://myapi.com/v1/todos/all`?

```dart
mixin JSONServerTaskAdapter on RemoteAdapter<Task> {
  @override
  String get baseUrl => 'https://myapi.com/v1/';

  @override
  String urlForFindAll(Map<String, dynamic> params) => 'todos/all';
}
```

Here's a list of overridable members:

<br>

|          |      |
|--------------|-----------|
| `type` | defaults to a camel-cased, pluralized class name (`User` => `users`)      |
| `baseUrl` | must be implemented or it will throw an error |
| `urlForFindAll` | defaults to `type` |
| `methodForFindAll` | defaults to `DataRequestMethod.GET` |
| `urlForFindOne` | defaults to `${type}/${id}` |
| `methodForFindOne` | defaults to `DataRequestMethod.GET` |
| `urlForSave` | defaults to `${type}/${id}` if updating |
| `methodForSave` | defaults to `DataRequestMethod.PATCH` if updating |
| `urlForDelete` | defaults to `${type}/${id}` |
| `methodForDelete` | defaults to `DataRequestMethod.DELETE`|
| `defaultParams` | defaults to `{}` |
| `defaultHeaders` | defaults to `{'Content-Type': 'application/json'}` |
| `shouldLoadRemoteAll` | fine-grained control over the `remote` param on `findAll` |
| `shouldLoadRemoteOne` | fine-grained control over the `remote` param on `findOne` |
| `serialize` | can customize serialization (like the [JSON API Adapter](https://github.com/flutterdata/flutter_data_json_api_adapter/) does) |
| `deserialize` | can customize deserialization (like the [JSON API Adapter](https://github.com/flutterdata/flutter_data_json_api_adapter/) does) |
| `isNetworkError` | whether to retry a request when back online |

<br>

And if we have multiple models that all share the same base URL?

We can simply make the adapter generic and apply it to any `DataModel` in our app!

```dart {hl_lines=[1 2 3 4 7]}
mixin JsonServerAdapter<T extends DataModel<T>> on RemoteAdapter<T> {
  @override
  String get baseUrl => 'https://myapi.com/v1/';
}

@JsonSerializable()
@DataRepository([JsonServerAdapter])
class User extends DataModel<User> {
  final int? id;
  final String name;
  final String? email;

  Task({this.id, required this.name, this.email});
}
```

{{< notice >}}
**Important**: As the repository is generated, any change in the list of adapters **must** be followed by a build in order to take effect.

```bash
flutter pub run build_runner build
```

Trouble generating code? [See here](/docs/faq/#errors-generating-code).
{{< /notice >}}

Any number of adapters can be added and they will be applied in order.

That is:

```dart
@DataRepository([A, B, C, D, E])
```

after codegen will become:

```dart
RemoteAdapter<User> with A, B, C, D, E;
```

## Custom endpoints

Not every API perfectly aligns to CRUD endpoints. Here's an example on how to create a custom action, using the `sendRequest` API.

```dart
mixin PaymentAdapter on RemoteAdapter<Payment> {
  Future<Payment?> createManualPayment({
    required String paymentType,
    required double amount,
  }) async {
    final appConfig = read(appConfigProvider).instance;

    final payload = {
      'payment': {
        'app_id': appConfig.appId,
        'amount': amount,
        'name': paymentType,
        'provider': 'manual'
      },
    };

    return sendRequest(
      baseUrl.asUri / 'payments.json' & {'v': true},
      method: DataRequestMethod.POST,
      headers: await defaultHeaders & {'X-Client-Id': appConfig.appId},
      body: json.encode(payload),
      onSuccess: (data) {
        return deserialize(data as Map<String, dynamic>).model;
      },
    );
  }
}
```

Notice that a Riverpod `Reader` is available on every adapter as `read`, too.

The `createManualPayment` action can now be invoked like:

```dart
onPressed: () async {
  final payment = await ref.payments.paymentAdapter.createManualPayment(
    paymentType: PaymentType.LIGHTNING_NETWORK,
    amount: amount,
  );
  // ...
}
```

{{< notice >}}
This is the signature for the `sendRequest` method, that performs an HTTP request and returns the result of type `R` via `onSuccess`:

```dart
Future<R?> sendRequest<R>(
  final Uri uri, {
  DataRequestMethod method = DataRequestMethod.GET,
  Map<String, String>? headers,
  String? body,
  _OnSuccessGeneric<R>? onSuccess,
  _OnErrorGeneric<R>? onError,
  bool omitDefaultParams = false,
  DataRequestLabel? label,
});
```

- `uri` takes the full `Uri` (you must provider base URL and query parameters, too)
- `headers` takes the full headers (or `defaultHeaders` if omitted)

{{< /notice >}}

With all these building blocks adapters for Wordpress or Github REST access, or even JWT authentication are easy to build.

## Overriding watchers

Let's imagine our app has to list completed payments in different widgets.

```dart
final state = ref.payments.watchAll(params: {'filter': {'complete': true}});
```

We could use something like the above to only request completed payments from the backend API.

But non-completed payments in local storage would still show up through `watchAll`, so we would have to filter them every time in every widget.

Except if we override this behavior. Since the _meat_ of the watchers happens in the notifiers (`watchAllNotifier` in this case), that is what we are going to override:

```dart
@override
DataStateNotifier<List<Payment>?> watchAllNotifier({
  bool? remote,
  Map<String, dynamic>? params,
  Map<String, String>? headers,
  bool? syncLocal,
  String? finder,
  DataRequestLabel? label,
}) {
  return super
      .watchAllNotifier(
        remote: remote,
        params: params,
        headers: headers,
        syncLocal: syncLocal,
        finder: finder,
        label: label,
      )
      .where((payment) => payment.isComplete);
}
```

Both `where` and `map` are available as notifier extensions. In the future these could be turned into watcher arguments for easier access.

**Many more adapter examples can be found perusing the [articles](/articles).**

{{< contact >}}
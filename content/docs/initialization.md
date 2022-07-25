---
title: "Initialization"
weight: 55
menu: docs
---

Initializing Flutter Data consists of two parts: local storage initialization and repository initialization.

The former happens when wiring up providers and the latter during widget build.

### Local storage initialization

Here are the configuration options with their default arguments explicit:

```dart
ProviderScope(
  child: MyApp(),
  overrides: [
    configureRepositoryLocalStorage(
      // callback that returns a base directory where to place local storage
      // (if the path_provider package is present, otherwise you MUST override it)
      baseDirFn: () => getApplicationDocumentsDirectory().then((dir) => dir.path),
      // 256-bit key for AES encryption
      encryptionKey: null,
      // whether to clear all local storage during initialization
      clear: false,
    ),
    graphNotifierThrottleDurationProvider.overrideWithValue(Duration.zero),
  ],
),
```

Customizing the duration of the throttle on `GraphNotifier` will determine how often Flutter widgets are marked for rebuild when using [watchers](/docs/repositories#watchers).

### Repository initialization

Use `repositoryInitializerProvider` without arguments:

```dart
Container(
  child: ref.watch(repositoryInitializerProvider).when(
        error: (error, _) => Text(error.toString()),
        loading: () => const CircularProgressIndicator(),
        data: (_) => Text('Flutter Data is ready: ${ref.tasks}'),
      ),
),
```

## Flutter with Riverpod

```dart {hl_lines=[3 5 6 12 "23-29"]}
import 'package:flutter/material.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';
import 'package:flutter_data/flutter_data.dart';

import 'main.data.dart';
import 'models/task.dart';

void main() {
  runApp(
    ProviderScope(
      child: MyApp(),
      overrides: [configureRepositoryLocalStorage()],
    ),
  );
}

class MyApp extends HookConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return MaterialApp(
      home: Scaffold(
        body: Center(
          child: ref.watch(repositoryInitializerProvider).when(
                error: (error, _) => Text(error.toString()),
                loading: () => const CircularProgressIndicator(),
                data: (_) => Text('Flutter Data is ready: ${ref.tasks}'),
              ),
        ),
      ),
    );
  }
}
```

## Flutter with Provider

See [Configure Flutter Data to Work with Provider](/articles/configure-provider/)

## Flutter with GetIt

See [Configure Flutter Data to Work with GetIt](/articles/configure-get-it/)

## Dart

```dart
// lib/main.dart

late final Directory _dir;

final container = ProviderContainer(
  overrides: [
    // baseDirFn MUST be provided
    configureRepositoryLocalStorage(baseDirFn: () => _dir.path),
  ],
);

try {
  _dir = await Directory('tmp').create();
  _dir.deleteSync(recursive: true);

  await container.read(repositoryInitializerProvider.future);

  final usersRepo = container.read(usersRepositoryProvider);
  await usersRepo.findOne(1);
  // ...
}
```

## Re-initializing

It is possible to re-initialize Flutter Data, for example to perform a restart with [Phoenix](https://pub.dev/packages/flutter_phoenix) or simply a Riverpod `ref.refresh`:

```dart {hl_lines=[6]}
class MyApp extends HookConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return MaterialApp(
      home: RefreshIndicator(
        onRefresh: () async => ref.container.refresh(repositoryInitializerProvider.future),
        child: Scaffold(
          body: Center(
            child: ref.watch(repositoryInitializerProvider).when(
                  error: (error, _) => Text(error.toString()),
                  loading: () => const CircularProgressIndicator(),
                  data: (_) => Text('Flutter Data is ready: ${ref.tasks}'),
                ),
          ),
        ),
      ),
    );
  }
}
```

{{< contact >}}
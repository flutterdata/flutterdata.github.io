---
title: "Offline"
weight: 51
menu: docs
---

You can do this in your `Scaffold`

```dart
child: ref.watch(initializerProvider).when(
   error: (error, _) => Text(error.toString()),
   loading: () => const CircularProgressIndicator(),
   data: (_) => Text('App boot is ready, replace me with main UI widget'),
  ),
),
```

Then define your initializer where you initialize any number of services needed to display the main widget of your UI:

```dart
final initializerProvider = FutureProvider<void>((ref) async {
  // initialize FD
  await ref.container.refresh(repositoryInitializerProvider.future);
  
  // initialize other services
  
  // retry offline events
  final _sub = ref.listen(offlineRetryProvider, (_, __) {});
  
  // close offline retry subscription
  ref.onDispose(() {
    _sub.close();
  });
});
```

You could also place this offline retry logic in some more specific place (for example when a user logs in, and close the sub when the user logs out).
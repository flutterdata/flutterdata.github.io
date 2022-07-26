---
title: Creating a new task
weight: 20
menu: tutorial
---

First off let's add just one line during the initialization. This will enable very helpful logging of our tasks repository!

```dart {hl_lines=[6 7]}
// ...
child: ref.watch(repositoryInitializerProvider).when(
  error: (error, _) => Text(error.toString()),
  loading: () => const CircularProgressIndicator(),
  data: (_) {
    // enable verbose
    ref.tasks.logLevel = 2;
    return TasksScreen();
  }
),
// ...
```

When we restart we notice the following:

```text
flutter: 34:061 [watchAll/tasks@e20025] initializing
flutter: 34:100   [findAll/tasks@e2046b<e20025] requesting [HTTP GET] https://my-json-server.typicode.com/flutterdata/demo/tasks
flutter: 34:835   [findAll/tasks@e2046b<e20025] {1, 2, 3, 4, 5} (and 5 more) fetched from remote
```

Let's add a `TextField`, turn the input into a new `Task` and immediately save it.

```dart {hl_lines=[4 "12-18"]}
class TasksScreen extends HookConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final _newTaskController = useTextEditingController();
    final state = ref.tasks.watchAll();

    if (state.isLoading) {
      return CircularProgressIndicator();
    }
    return ListView(
      children: [
        TextField(
          controller: _newTaskController,
          onSubmitted: (value) async {
            Task(title: value).save();
            _newTaskController.clear();
          },
        ),
        for (final task in state.model!)
          ListTile(
            leading: Checkbox(
              value: task.completed,
              onChanged: (value) => task.toggleCompleted().save(),
            ),
            title: Text('${task.title} [id: ${task.id}]'),
          ),
      ],
    );
  }
}
```

For this we need to import `flutter_hooks`!

Hot-reloading once again we see our `TextField` ready to create new tasks:

{{< iphone "../w4.gif" >}}

It was that easy!

{{< notice >}}
You may have noticed that there was a flash with `[id: null]` (we didn't supply any ID upon model creation), until the server responds with one (in this case `11`) triggering an update.

Be aware that our [dummy JSON backend](https://my-json-server.typicode.com/flutterdata/demo) does not actually save new resources so **it will always respond with ID `11`**, causing a confusing situation if you keep adding tasks!

In the console:

```text
flutter: 25:084 [watchAll/tasks@68f651] updated models
flutter: 25:087 [save/tasks@6bb411] requesting [HTTP POST] https://my-json-server.typicode.com/flutterdata/demo/tasks
flutter: 25:713 [save/tasks@6bb411] saved in local storage and remote
flutter: 25:714 [watchAll/tasks@68f651] updated models
```

{{< /notice >}}

**NEXT: [Reloading the list](/tutorial/reloading)**

{{< contact >}}
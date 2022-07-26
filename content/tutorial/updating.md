---
title: Marking tasks as done
weight: 15
menu: tutorial
---

A read-only tasks app is not very practical! Let's add the ability to update the `completed` state and mark/unmark our tasks as done.

First, though, we'll extract the tasks-specific code to a separate screen named `TasksScreen`:

```dart
class TasksScreen extends HookConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.tasks.watchAll();
    if (state.isLoading) {
      return CircularProgressIndicator();
    }
    return ListView(
      children: [
        for (final task in state.model!) Text(task.title),
      ],
    );
  }
}
```

Remember to return this new widget from `TasksApp`:

```dart {hl_lines=[10]}
class TasksApp extends HookConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return MaterialApp(
      home: Scaffold(
        body: Center(
          child: ref.watch(repositoryInitializerProvider).when(
                error: (error, _) => Text(error.toString()),
                loading: () => const CircularProgressIndicator(),
                data: (_) => TasksScreen(),
              ),
        ),
      ),
      debugShowCheckedModeBanner: false,
    );
  }
}
```

Back to our `TasksScreen` we are going to wrap our title text widget in a `ListTile` prefixing it with a checkbox which, upon clicking, will toggle task completion:


```dart {hl_lines=["11-17"]}
class TasksScreen extends HookConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.tasks.watchAll();
    if (state.isLoading) {
      return CircularProgressIndicator();
    }
    return ListView(
      children: [
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

If only the `toggleCompleted()` method existed... 😀

Since `Task` is immutable, we return a new `Task` object with the inverse boolean value of `completed`:

```dart {hl_lines=[11 12 13]}
@JsonSerializable()
@DataRepository([JsonServerAdapter])
class Task extends DataModel<Task> {
  @override
  final int? id;
  final String title;
  final bool completed;

  Task({this.id, required this.title, this.completed = false});

  Task toggleCompleted() {
    return Task(id: this.id, title: this.title, completed: !this.completed).withKeyOf(this);
  }
}
```

{{< notice >}}
**What exactly is `withKeyOf(this)` for?**

When a new model is created, Flutter Data initializes it looking up its internal key based on its `id`.

This way, `Task(id: 4, title: 'a')` and `Task(id: 4, title: 'b')` are essentially two versions of the same model.

When there is no `id` (or potentially no `id`, as in the case above) we can use [`withKeyOf`](/docs/models/#withkeyof) to ensure the new model is treated as an updated version of the old one.
{{< /notice >}}

Regenerate code, hot-reload and check all boxes...

{{< iphone "../w2.gif" >}}

Done! **NEXT: [Creating a new task](/tutorial/creating)**

{{< contact >}}
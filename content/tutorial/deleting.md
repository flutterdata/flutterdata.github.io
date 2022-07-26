---
title: Deleting tasks
weight: 40
menu: tutorial
---

There's stuff we just don't want to do!

We can delete a `Task` on dismiss by wrapping the tile with a `Dismissible` and calling its `delete` method:

```dart {hl_lines=["24-27"]}
class TasksScreen extends HookConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final _newTaskController = useTextEditingController();
    final state = ref.tasks.watchAll(params: {'_limit': 5}, syncLocal: true);

    if (state.isLoading) {
      return CircularProgressIndicator();
    }

    return RefreshIndicator(
      onRefresh: () =>
          ref.tasks.findAll(params: {'_limit': 5}, syncLocal: true),
      child: ListView(
        children: [
          TextField(
            controller: _newTaskController,
            onSubmitted: (value) async {
              Task(title: value).save();
              _newTaskController.clear();
            },
          ),
          for (final task in state.model)
            Dismissible(
              key: ValueKey(task),
              direction: DismissDirection.endToStart,
              onDismissed: (_) => task.delete(),
              child: ListTile(
                leading: Checkbox(
                  value: task.completed,
                  onChanged: (value) => task.toggleCompleted().save(),
                ),
                title: Text('${task.title} [id: ${task.id}]'),
              ),
            ),
        ],
      ),
    );
  }
}
```

Hot-reload, swipe left and... they're gone!

{{< iphone "../w7.gif" >}}

{{< notice >}}
Remember to check out the debug console where you can find some Flutter Data activity logs like:

```text
flutter: 25:691 [watchAll/tasks@1744b4] updated models
flutter: 25:693 [delete/tasks#4@1936e7] requesting [HTTP DELETE] https://my-json-server.typicode.com/flutterdata/demo/tasks/4
flutter: 26:266 [delete/tasks#4@1936e7] deleted in local storage and remote
```
{{< /notice >}}

**NEXT: [Using relationships](/tutorial/relationships)**

{{< contact >}}
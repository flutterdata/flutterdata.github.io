---
title: Using relationships
weight: 100
menu: tutorial
draft: true
---

Let's now slightly rethink our query. Instead of **"fetching all tasks for user 1"** we are going to **"fetch user 1 with all their tasks"**.

Flutter Data has first-class support for [relationships](/data-support/relationships).

First, in `models/user.dart`, we'll create the `User` model with a `HasMany<Task>` relationship:

```dart {hl_lines=[16]}
import 'package:flutter_data/flutter_data.dart';
import 'package:json_annotation/json_annotation.dart';

import 'task.dart';

part 'user.g.dart';

@JsonSerializable()
@DataRepository([JsonServerAdapter])
class User extends DataModel<User> {
  @override
  final int? id;
  final String name;
  final HasMany<Task> tasks;

  User({this.id, required this.name, required this.tasks});
}
```

Time to run code generation and get a brand-new `Repository<User>`:

```text
flutter pub run build_runner build
```

Great. We are now going to issue the remote request via `watchOne()`, in order to list (_and watch for changes of_) user `1`'s `Task` models:

 - `params: {'_embed': 'tasks'},` tells the server to include this user's tasks (which our JSON adapter knows how to deserialize)
 - `alsoWatch: (user) => [user.tasks]` tells the [watcher](/docs/repositories/#watchone) to rebuild the widget any time user _or_ its tasks are updated or deleted; any number of relationships of any depth can be watched

```dart {hl_lines=[5 6 7 8 9 15 18 28]}
class TasksScreen extends HookConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final _newTaskController = useTextEditingController();
    final state = ref.users.watchOne(
        1, // user ID, an integer
        params: {'_embed': 'tasks'}, // HTTP param
        alsoWatch: (user) => [user.tasks] // watcher
      );

    if (state.isLoading) {
      return CircularProgressIndicator();
    }

    final tasks = state.model!.tasks.toList();

    return RefreshIndicator(
      onRefresh: () => ref.users.findOne(1, params: {'_embed': 'tasks'}),
      child: ListView(
        children: [
          TextField(
            controller: _newTaskController,
            onSubmitted: (value) async {
              Task(title: value).save();
              _newTaskController.clear();
            },
          ),
          for (final task in tasks)
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

Import the `user.dart` file, reload and watch it working!

{{< iphone "../w8a.png" >}}

{{< notice >}}

Note that tasks `4`, `5` and `9` for example were not loaded as they do not belong to user `1`!

This is the API response for https://my-json-server.typicode.com/flutterdata/demo/users/1?_embed=tasks that was parsed by the built-in `deserialize` method:

```json
{
  "id": 1,
  "name": "frank06",
  "tasks": [
    {
      "id": 1,
      "title": "Laundry 🧺",
      "completed": false,
      "userId": 1
    },
    {
      "id": 2,
      "title": "Groceries 🛒",
      "completed": true,
      "userId": 1
    },
    {
      "id": 3,
      "title": "Reservation at Malloys",
      "completed": true,
      "userId": 1
    },
    {
      "id": 7,
      "title": "Take Amanda to birthday",
      "completed": true,
      "userId": 1
    },
    {
      "id": 8,
      "title": "Get new surfboard 🏄‍♀️",
      "completed": false,
      "userId": 1
    },
    {
      "id": 10,
      "title": "Protest tyrannical mandates 👊",
      "completed": true,
      "userId": 1
    }
  ]
}
```

{{< /notice >}}

## Creating a task

As it is, adding a new task will not work. Why is that?

We are creating new `Task` models without any `User` associated to them:

```dart
onSubmitted: (value) async {
  Task(title: value).save();
  _newTaskController.clear();
},
```

Let's fix this. Add a `BelongsTo<User>` relationship in `models/task.dart` and regenerate our code:

```dart {hl_lines=[8 10 13]}
@JsonSerializable()
@DataRepository([JsonServerAdapter])
class Task extends DataModel<Task> {
  @override
  final int? id;
  final String title;
  final bool completed;
  final BelongsTo<User> user;

  Task({this.id, required this.title, this.completed = false, required this.user});
  
  Task toggleCompleted() {
    return Task(id: this.id, title: this.title, user: user, completed: !this.completed)
        .withKeyOf(this);
  }
}
```

Now we can provide the right user as a `BelongsTo`:

```dart {hl_lines=[15 16 25]}
class TasksScreen extends HookConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final _newTaskController = useTextEditingController();
    final state = ref.users.watchOne(
        1, // user ID, an integer
        params: {'_embed': 'tasks'}, // HTTP param
        alsoWatch: (user) => [user.tasks] // watcher
      );

    if (state.isLoading) {
      return CircularProgressIndicator();
    }

    final user = state.model!;
    final tasks = user.tasks.toList();

    return RefreshIndicator(
      onRefresh: () => ref.users.findOne(1, params: {'_embed': 'tasks'}),
      child: ListView(
        children: [
          TextField(
            controller: _newTaskController,
            onSubmitted: (value) async {
              Task(title: value, user: BelongsTo(user)).save();
              _newTaskController.clear();
            },
          ),
          for (final task in tasks)
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

And adding new tasks now works!

{{< iphone "../w8.png" >}}


{{< notice >}}
**Check out the source code: https://github.com/flutterdata/tutorial**
{{< /notice >}}

{{< contact >}}
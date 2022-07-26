---
title: Relationships
date: 2020-04-20T17:21:33-03:00
weight: 40
menu: docs
---

Flutter Data features an advanced relationship mapping system.

Use:

- `HasMany<T>` for to-many relationships
- `BelongsTo<T>` for to-one relationships

As an example, a `User` has many `Task`s:

```dart {hl_lines=[7]}
@JsonSerializable()
@DataRepository([JsonServerAdapter])
class User extends DataModel<User> {
  @override
  final int? id;
  final String name;
  final HasMany<Task>? tasks;

  User({this.id, required this.name, this.tasks});
}
```

and a `Task` belongs to a `User`:

```dart {hl_lines=[8]}
@JsonSerializable()
@DataRepository([JsonServerAdapter])
class Task extends DataModel<Task> {
  @override
  final int? id;
  final String title;
  final bool completed;
  final BelongsTo<User>? user;

  Task({this.id, required this.title, this.completed = false, this.user});
}
```

As long as the API responds correctly with relationship data (for example a `User` resource with a collection of `Task` models – or just IDs if models are already present in local storage) we can expect the following to work:

```dart
final user = await repository.findOne(1, params: {'_embed': 'tasks'});
final task = user!.tasks!.first;

print(task.title); // write Flutter Data docs
print(task.user!.value!.name); // Frank

// or

final house = House(address: '123 Main Rd');
final family = Family(surname: 'Assange', house: BelongsTo(house));

print(family.house.value.families.first.surname);  // Free Assange
```

We can infinitely navigate the relationship graph as it's based on a reactive graph data structure (`GraphNotifier`).

## Defaults

For relationships to work they must not be `null`.

```dart
final task = Task(title: 'do 1', user: BelongsTo());
```

If we don't want to supply a new relationship object like above, we may provide defaults like so:

```dart {hl_lines=[10 11]}
@JsonSerializable()
@DataRepository([JsonServerAdapter])
class Task extends DataModel<Task> {
  @override
  final int? id;
  final String title;
  final bool completed;
  late final BelongsTo<User> user;

  Task({this.id, required this.title, this.completed = false, BelongsTo<User>? user}) :
    user = user ?? BelongsTo();
}
```

## Inverses

Inverse relationships are guessed when unambiguous (one relationship of inverse type).

Not in this case, as `Family` has two `BelongsTo<House>`s:

```dart
@JsonSerializable()
@DataRepository([])
class Family extends DataModel<Family> {
  @override
  final String? id;
  final BelongsTo<House>? cottage;
  final BelongsTo<House>? residence;

  Family({
    this.id,
    this.cottage,
    this.residence,
  });
}

@JsonSerializable()
@DataRepository([])
class House extends DataModel<House> {
  @override
  final String? id;
  final BelongsTo<Family>? owner;

  House({
    this.id,
    BelongsTo<Family>? owner,
  }) : owner = owner ?? BelongsTo();
```

If you wish to disambuiguate or to be explicit, annotate your relationship in the `House` model:

```dart
@DataRelationship(inverse: 'residence')
final BelongsTo<Family>? owner;
```

Here's another example, a tree structure using custom inverses and Freezed:

```dart
@freezed
@DataRepository([], remote: false)
class Node extends DataModel<Node>, _$Node {
  Node._();
  factory Node(
      {int? id,
      String? name,
      @DataRelationship(inverse: 'children') BelongsTo<Node>? parent,
      @DataRelationship(inverse: 'parent') HasMany<Node>? children}) = _Node;
  factory Node.fromJson(Map<String, dynamic> json) => _$NodeFromJson(json);
}
```

## Remove a relationship

Given a `Post` with many `Comment`s we want to remove:

```dart
final postWithNoComments = post.copyWith(comments: HasMany.remove()).was(post);
```

Works with both `HasMany` and `BelongsTo`.

Removing a relationship does not delete its linked resources (the actual comments in this case).

## Disable relationship serialization

In order to keep the relationship working but avoid persisting it, use:

```dart
@DataRelationship(serialize: false)
final BelongsTo<Post>? post;
```

## Self-referential relationships

```dart
// in Post
@DataRelationship(serialize: false)
late BelongsTo<Post> post = asBelongsTo;
```

## Relationship extensions

A `User` with `Task`s could be created like this:

```dart
final t1 = Task(title: 'do 1');
final t2 = Task(title: 'do 2');
final user = User(name: 'Frank', tasks: HasMany({t1, t2}));

// or using an extension on Set<DataModel>

final user = User(name: 'Frank', tasks: {t1, t2}.asHasMany);
```

or a `Task` with `User`:

```dart
final user = User(name: 'Frank');
final task = Task(title: 'do 1', user: BelongsTo(user));

// or using an extension on DataModel

final task = Task(title: 'do 1', user: user.asBelongsTo);
```

{{< contact >}}
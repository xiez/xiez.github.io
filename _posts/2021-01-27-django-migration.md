---
title:  Django Migration 源码分析
classes: wide
categories:
  - 2021-02
tags:
  - django
---

本文从源码层次分析下 Django Migration 系统的内部原理。

## Make Migrations

`python manage.py makemigrations` 该命令通过对比现有的migration文件和所有APP的models字段，根据差异生成新的 migration 文件。

代码路径： [https://github.com/django/django/blob/stable/1.11.x/django/core/management/commands/makemigrations.py](https://github.com/django/django/blob/stable/1.11.x/django/core/management/commands/makemigrations.py)

### MigrationLoader

代码路径： [https://github.com/django/django/blob/stable/1.11.x/django/db/migrations/loader.py#L21](https://github.com/django/django/blob/stable/1.11.x/django/db/migrations/loader.py#L21)

`makemigration` 首先会[实例化](https://github.com/django/django/blob/stable/1.11.x/django/core/management/commands/makemigrations.py#L96)一个 `MigrationLoader`

```
loader = MigrationLoader(None, ignore_no_migrations=True)
```

`MigrationLoader` 在实例化过程中，会通过加载当前项目下所有的 migration 文件来构造出 `MigrationGraph`


```
class MigrationLoader(object):
    """
    Loads migration files from disk, and their status from the database.
    ...
    """
    def __init__(self, connection, load=True, ignore_no_migrations=False):
        ...
        if load:
            self.build_graph()

    def build_graph(self):
        self.load_disk()
        ...
        self.graph = MigrationGraph()
        ...

    def load_disk(self):
        """
        Loads the migrations from all INSTALLED_APPS from disk.
        """
        ...
```

`MigrationGraph` 本质上就是项目里所有 migration 文件的依赖关系图。图中的每一个节点`Node`代表一个app下的migration文件，例如 `('app_A, '0001_auto_20190822_0806')`

![dj_migration](https://github.com/xiez/xiez.github.io/raw/master/assets/images/2021/dj_mig.png)
```
class MigrationGraph(object):
    """
    Represents the digraph of all migrations in a project.
    ...
    """
    def __init__(self):
        self.node_map = {}
        self.nodes = {}
        self.cached = False

class Node(object):
    """
    A single node in the migration graph. Contains direct links to adjacent
    nodes in either direction.
    """
    def __init__(self, key):
        self.key = key
        self.children = set()
        self.parents = set()
```

### ProjectState

代码路径： [https://github.com/django/django/blob/stable/1.11.x/django/db/migrations/state.py#L88](https://github.com/django/django/blob/stable/1.11.x/django/db/migrations/state.py#L88)

有了上面的migration依赖关系图后，就可以[推导出整个项目的状态](https://github.com/django/django/blob/stable/1.11.x/django/core/management/commands/makemigrations.py#L150)，也就是上一次执行`makemigration`时的项目状态，项目状态可以理解为项目里所有APP里的model状态。


```
# Set up autodetector
autodetector = MigrationAutodetector(
    loader.project_state(),
    ProjectState.from_apps(apps),
    questioner,
)

```

其中， `loader.project_state()`就是通过依赖关系图推导出的项目状态，而`ProjectState.from_apps(apps)`是通过app推导出的当前实时的项目状态，把这两个新旧状态传给`MigrationAutodetector`，由它来对比出差异。

```
class MigrationLoader(object):

    def project_state(self, nodes=None, at_end=True):
        """
        Returns a ProjectState object representing the most recent state
        that the migrations we loaded represent.

        See graph.make_state for the meaning of "nodes" and "at_end"
        """
        return self.graph.make_state(nodes=nodes, at_end=at_end, real_apps=list(self.unmigrated_apps))
```

在这个函数里，loader没做什么事情，只是通过依赖关系图`graph.make_state`来推导状态。


```
class MigrationGraph(object):
    def make_state(self, nodes=None, at_end=True, real_apps=None):
        """
        Given a migration node or nodes, returns a complete ProjectState for it.
        If at_end is False, returns the state before the migration has run.
        If nodes is not provided, returns the overall most current project state.
        """
        if nodes is None:
            nodes = list(self.leaf_nodes())
        if len(nodes) == 0:
            return ProjectState()
        if not isinstance(nodes[0], tuple):
            nodes = [nodes]
        plan = []

        for node in nodes:
            for migration in self.forwards_plan(node):
                if migration not in plan:
                    if not at_end and migration in nodes:
                        continue
                    plan.append(migration)
        project_state = ProjectState(real_apps=real_apps)

        for node in plan:
            project_state = self.nodes[node].mutate_state(project_state, preserve=False)

        return project_state
```

`make_state`函数其实就做了一件事情，从空的ProjectState开始，把当前所有的migration应用一遍，这样就导出了对应的项目状态。

上面说的项目状态就是所有APP的model，对应的类为`ProjectState`

```
class ProjectState(object):
    """
    Represents the entire project's overall state.
    This is the item that is passed around - we do it here rather than at the
    app level so that cross-app FKs/etc. resolve properly.
    """

    def __init__(self, models=None, real_apps=None):
        self.models = models or {}
        # Apps to include from main registry, usually unmigrated ones
        self.real_apps = real_apps or []
        self.is_delayed = False

    @classmethod
    def from_apps(cls, apps):
        "Takes in an Apps and returns a ProjectState matching it"
        app_models = {}
        for model in apps.get_models(include_swapped=True):
            model_state = ModelState.from_model(model)
            app_models[(model_state.app_label, model_state.name_lower)] = model_state

        return cls(app_models)
```

ProjectState 借助了 `ModelState` 来应用migration, 之所以不用`db.Model`，是因为`db.Model.options`假定不可变。

```
class ModelState(object):
    """
    Represents a Django Model. We don't use the actual Model class
    as it's not designed to have its options changed - instead, we
    mutate this one and then render it into a Model as required.

    Note that while you are allowed to mutate .fields, you are not allowed
    to mutate the Field instances inside there themselves - you must instead
    assign new ones, as these are not detached during a clone.
    """
    def __init__(self, app_label, name, fields, options=None, bases=None, managers=None):
        self.app_label = app_label
        self.name = force_text(name)
        self.fields = fields
        self.options = options or {}
        self.options.setdefault('indexes', [])
        self.bases = bases or (models.Model, )
        self.managers = managers or []
```

### MigrationAutodetector

代码路径： [https://github.com/django/django/blob/stable/1.11.x/django/db/migrations/autodetector.py#L22](https://github.com/django/django/blob/stable/1.11.x/django/db/migrations/autodetector.py#L22)

autodetector 构造出来后，`makemigration`接下来调用 `changes` 函数来对比新旧状态的差异。

```
# Detect changes
changes = autodetector.changes(
    graph=loader.graph,
    trim_to_apps=app_labels or None,
    convert_apps=app_labels or None,
    migration_name=self.migration_name,
)
```

`MigrationAutodetector`的class如下：

```
class MigrationAutodetector(object):
    """
    Takes a pair of ProjectStates, and compares them to see what the
    first would need doing to make it match the second (the second
    usually being the project's current state).

    Note that this naturally operates on entire projects at a time,
    as it's likely that changes interact (for example, you can't
    add a ForeignKey without having a migration to add the table it
    depends on first). A user interface may offer single-app usage
    if it wishes, with the caveat that it may not always be possible.
    """

    def __init__(self, from_state, to_state, questioner=None):
        self.from_state = from_state
        self.to_state = to_state
        self.questioner = questioner or MigrationQuestioner()
        self.existing_apps = {app for app, model in from_state.models}

    def changes(self, graph, trim_to_apps=None, convert_apps=None, migration_name=None):
        """
        Main entry point to produce a list of applicable changes.
        Takes a graph to base names on and an optional set of apps
        to try and restrict to (restriction is not guaranteed)
        """
        changes = self._detect_changes(convert_apps, graph)
        changes = self.arrange_for_graph(changes, graph, migration_name)
        if trim_to_apps:
            changes = self._trim_to_apps(changes, trim_to_apps)
        return changes
```

`changes`为字典对象，包含每个app下的model变更，类似如下：

```
{'app1': [<Migration app1.0003_auto_20210120_1451>], 'app2': [<Migration app2.0005_auto_20210120_1451>]}
```

### Migration

代码路径： [https://github.com/django/django/blob/stable/1.11.x/django/db/migrations/migration.py#L10](https://github.com/django/django/blob/stable/1.11.x/django/db/migrations/migration.py#L10)

上一步生成的每个app的model变更都是一个`Migration`对象，class如下：

```
class Migration(object):
    """
    The base class for all migrations.
    """

    # Operations to apply during this migration, in order.
    operations = []

    # Other migrations that should be run before this migration.
    # Should be a list of (app, migration_name).
    dependencies = []

    def mutate_state(self, project_state, preserve=True):
        """
        Takes a ProjectState and returns a new one with the migration's
        operations applied to it. Preserves the original object state by
        default and will return a mutated state from a copy.
        """
        new_state = project_state
        if preserve:
            new_state = project_state.clone()

        for operation in self.operations:
            operation.state_forwards(self.app_label, new_state)
        return new_state
```

`Migration`有个方法`mutate_state`，该方法就是被`graph.make_state`用来应用migration，生成新的项目状态。

每个migration对象最后再通过`MigrationWriter`写入到各自app的`migration`目录。生成的migration文件如下：

```
# -*- coding: utf-8 -*-
# Generated by Django 1.11.2 on 2021-01-20 07:47
from __future__ import unicode_literals

from django.db import migrations, models


class Migration(migrations.Migration):

    dependencies = [
        ('app', '0004_auto_20210115_1133'),
    ]

    operations = [
        migrations.AlterField(
            model_name='model1',
            name='attribute_code',
            field=models.CharField(max_length=32, unique=True),
        ),
        migrations.AlterField(
            model_name='model2',
            name='parent_code',
            field=models.CharField(blank=True, default=''),
        ),
    ]
```

至此， `makemigration` 整个过程完成。


### Bonus

理论上说，`makemigrate`只检查项目里的migration文件，不应该访问数据库，但实际上实例化完`MigrationLoader`后，会做一次历史一致性的检查，该检查主要防止`django_migration`表的数据不一致，比如已经被应用的`migration 0002`，它所依赖的`migration 0001`却没被应用。

```
# Raise an error if any migrations are applied before their dependencies.
consistency_check_labels = set(config.label for config in apps.get_app_configs())

loader.check_consistent_history(connection)
```

这个问题在[ticket 25850](https://code.djangoproject.com/ticket/25850)里提出来，在[1.10版本里修复](https://github.com/django/django/commit/02ae5fd31a56ffb42feadb49c1f3870ba0a24869)。


## Migrate

[这篇](https://docs.djangoproject.com/en/3.1/ref/schema-editor/)关于`SchemaEditor`的文档里有说明，Django Migration 由两部分组成：

1. 计算出哪些`migration`需要应用。
2. 遍历这些`migration`的`operations`列表，然后交给 `SchemaEditor` 来执行更新数据库的操作。



### MigrationExecutor

首先，实例化一个`MigrationExecutor`, 并把数据库连接`connection`和一个回调函数`migration_progress_callback`传给这个 executor

```
executor = MigrationExecutor(connection, self.migration_progress_callback)
```

`migration_progress_callback`主要为了输出migration过程中的一些信息，在这个[commit](https://github.com/django/django/commit/315ab41e416c777d4f42932d42df07872e8f8895#diff-973e0df4f8467492bce02726075ea76ca3f574945cc3b9f81247758dce7e6b0fR11)里引入，使用callback函数的好处是，对`MigrationExecutor`的逻辑改动较少，也更灵活，如果需要输出不同的信息，无需改动executor, 只需要传入不同的callback函数即可。


`MigrationExecutor`代码如下：

```
class MigrationExecutor(object):
    """
    End-to-end migration execution - loads migrations, and runs them
    up or down to a specified set of targets.
    """

    def __init__(self, connection, progress_callback=None):
        self.connection = connection
        self.loader = MigrationLoader(self.connection)
        self.recorder = MigrationRecorder(self.connection)
        self.progress_callback = progress_callback
```

`MigrationLoader` 在上面已经了解过，主要用来加载磁盘上的migration文件，并生成migration文件的依赖关系图。

`MigrationRecorder` 主要为了把migration的执行记录在数据库里记下来，方便实现增量式的变更。

### MigrationRecorder

`MigrationRecorder` class 如下：

```
class MigrationRecorder(object):
    """
    Deals with storing migration records in the database.

    Because this table is actually itself used for dealing with model
    creation, it's the one thing we can't do normally via migrations.
    We manually handle table creation/schema updating (using schema backend)
    and then have a floating model to do queries with.

    If a migration is unapplied its row is removed from the table. Having
    a row in the table always means a migration is applied.
    """

    @python_2_unicode_compatible
    class Migration(models.Model):
        app = models.CharField(max_length=255)
        name = models.CharField(max_length=255)
        applied = models.DateTimeField(default=now)

        class Meta:
            apps = Apps()
            app_label = "migrations"
            db_table = "django_migrations"

        def __str__(self):
            return "Migration %s for %s" % (self.name, self.app)

    def __init__(self, connection):
        self.connection = connection
```

Django里所有的数据表都可以通过migratation创建，唯独migration自己所需要用到的表需要使用`SchemaEditor`手工创建。


```
class MigrationRecorder(object):
    ...

    def ensure_schema(self):
        """
        Ensures the table exists and has the correct schema.
        """
        # If the table's there, that's fine - we've never changed its schema
        # in the codebase.
        if self.Migration._meta.db_table in self.connection.introspection.table_names(self.connection.cursor()):
            return
        # Make the table
        try:
            with self.connection.schema_editor() as editor:
                editor.create_model(self.Migration)
        except DatabaseError as exc:
            raise MigrationSchemaMissing("Unable to create the django_migrations table (%s)" % exc)

```

`ensure_schema()`会在多个地方被调用，确保数据表`django_migrations`存在。

### Migration Plan

`MigrationExecutor` 实例化完成后，接下来需要生成`migration plan`, 也就是需要执行哪些migration

```
targets = executor.loader.graph.leaf_nodes()

plan = executor.migration_plan(targets)
```

```
class MigrationExecutor(object):

    def migration_plan(self, targets, clean_start=False):
        """
        Given a set of targets, returns a list of (Migration instance, backwards?).
        """
        plan = []

        for migration in self.loader.graph.forwards_plan(target):
            if migration not in applied:
                plan.append((self.loader.graph.nodes[migration], False))
                applied.add(migration)

```

`migration_plan` 主要就从 `loader` 的migration依赖图里，找到未应用的migration


### Pre-migrate/Post-migrate Signal

在migrate执行前后，各个app可以接收信号`pre_migrate/post_migrate`来做一些自定义的事情。

```
def emit_pre_migrate_signal(verbosity, interactive, db, **kwargs):
    # Emit the pre_migrate signal for every application.
    for app_config in apps.get_app_configs():
        models.signals.pre_migrate.send(
            sender=app_config,
            app_config=app_config,
            verbosity=verbosity,
            interactive=interactive,
            using=db,
            **kwargs
        )

def emit_post_migrate_signal(verbosity, interactive, db, **kwargs):
    # Emit the post_migrate signal for every application.
    for app_config in apps.get_app_configs():
        models.signals.post_migrate.send(
            sender=app_config,
            app_config=app_config,
            verbosity=verbosity,
            interactive=interactive,
            using=db,
            **kwargs
        )

```

### Migrate

`migrate` 分正向和反向(`forward/backward`)， 反向migrate主要用在回退一个migration, 例如：当前已经应用了0001, 0002, 0003 这3个migration， 重新应用0001, 就会回退后面两个migration。

有了需要应用的plan列表后，`migrate`就可以遍历这个列表，开始应用migration，也就是调用`migration.apply`


```
        post_migrate_state = executor.migrate(
            targets, plan=plan, state=pre_migrate_state.clone(), fake=fake,
            fake_initial=fake_initial,
        )
```

```
class MigrationExecutor(object):
    def migrate(self, targets, plan=None, state=None, fake=False, fake_initial=False):
        """
        Migrates the database up to the given targets.

        Django first needs to create all project states before a migration is
        (un)applied and in a second step run all the database operations.
        """

        ...
        elif all_forwards:
            state = self._migrate_all_forwards(state, plan, full_plan, fake=fake, fake_initial=fake_initial)

    def _migrate_all_forwards(self, state, plan, full_plan, fake, fake_initial):
        """
        Take a list of 2-tuples of the form (migration instance, False) and
        apply them in the order they occur in the full_plan.
        """
        migrations_to_run = {m[0] for m in plan}
        for migration, _ in full_plan:
            ...
            if migration in migrations_to_run:
                state = self.apply_migration(state, migration, fake=fake, fake_initial=fake_initial)

        return state

    def apply_migration(self, state, migration, fake=False, fake_initial=False):
        """
        Runs a migration forwards.
        """
        ...
                with self.connection.schema_editor(atomic=migration.atomic) as schema_editor:
                    state = migration.apply(state, schema_editor)

        return state
```

`apply`函数就是遍历migraiton里的 `operations` 列表，再交给 `SchemaEditor` 来执行具体的数据库变更。

```
    def apply(self, project_state, schema_editor, collect_sql=False):
        """
        Takes a project_state representing all migrations prior to this one
        and a schema_editor for a live database and applies the migration
        in a forwards order.

        Returns the resulting project state for efficient re-use by following
        Migrations.
        """
        for operation in self.operations:

                operation.database_forwards(self.app_label, schema_editor, old_state, project_state)

        return project_state

```

至此，整个 `migrate` 操作完成。


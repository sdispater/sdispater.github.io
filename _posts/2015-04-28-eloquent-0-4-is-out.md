---
layout: post
title:  "Eloquent 0.4 is out: Schema builder, mixins and scopes"
date:   2015-04-28 18:05:38
category: programming
tags: [python, eloquent, orm]
---

Eloquent 0.4 is now out. This version introduces the schema builder, soft deletes, mixins and scopes.

## Schema Builder

The new schema builder is here to help manipulate your databases structure
in an intuitive manner.

For that purpose you can use the ``Schema`` class which provides a database
agnostic way of manipulating tables.

{% highlight python %}
from eloquent import DatabaseManager, Schema

config = {
    'mysql': {
        'driver': 'mysql',
        'host': 'localhost',
        'database': 'database',
        'username': 'root',
        'password': '',
        'prefix': ''
    }
}

db = DatabaseManager(config)
schema = Schema(db)
{% endhighlight %}

### Creating and dropping tables

To create a new database table, the ``create`` method is used:

{% highlight python %}
with schema.create('users') as table:
    table.increments('id')
{% endhighlight %}

To rename an existing database table, the ``rename`` method can be used:

{% highlight python %}
schema.rename('from', 'to')
{% endhighlight %}

To specify which connection the schema operation should take place on, use the ``connection`` method:

{% highlight python %}
with schema.connection('foo').create('users') as table:
    table.increments('id')
{% endhighlight %}

To drop a table, you can use the ``drop`` or ``drop_if_exists`` methods:

{% highlight python %}
schema.drop('users')

schema.drop_if_exists('users')
{% endhighlight %}

### Adding columns

To update an existing table, you can use the ``table`` method:

{% highlight python %}
with schema.table('users') as table:
    table.string('email')
    table.integer('votes')
{% endhighlight %}

The table builder contains a variety of column types that you can use.
See the [official documentation](http://eloquent.readthedocs.org/en/latest/schema_builder.html#adding-columns) for the full list.

### Changing columns

Sometimes you may need to modify an existing column.
For example, you may wish to increase the size of a string column.
To do so, you can use the ``change`` method:

{% highlight python %}
with schema.table('user') as table:
    table.string('name', 50).nullable().change()
{% endhighlight %}

<div class="warning">
<p>The change column feature, while tested, is still considered in <strong>beta</strong> stage. Please report any encountered issue or bug on the <a href="https://github.com/sdispater/eloquent" title="Github Project">Github Project</a></p>
</div>

### Renaming columns

To rename a column, you can use use the ``rename_column`` method on the Schema builder:

{% highlight python %}
with schema.table('users') as table:
    table.rename('from', 'to')
{% endhighlight %}

<div class="warning">
<p>Prior to <strong>MySQL 5.6.6</strong>, foreign keys are <strong>NOT</strong> automatically updated when renaming columns. Therefore, you will need to <strong>drop</strong> the foreign key constraint, <strong>rename</strong> the column and <strong>recreate</strong> the constraint to avoid an error.</p>
<pre>
<code class="python">with schema.table('posts') as table:
    table.drop_foreign('posts_user_id_foreign')
    table.rename('user_id', 'author_id')
    table.foreign('author_id').references('id').on('users')
</code>
</pre>
<p>In future versions, Eloquent <strong>might</strong> handle this automatically.</p>
</div>

<div class="warning">
<p>The rename column feature, while tested, is still considered in <strong>beta</strong> stage (especially for SQLite). Please report any encountered issue or bug on the <a href="https://github.com/sdispater/eloquent" title="Github Project">Github Project</a></p>
</div>

### Dropping columns

To drop a column, you can use use the ``drop_column`` method on the ``Schema`` builder:

{% highlight python %}
# Dropping a single column
with schema.table('users') as table:
    table.drop_column('votes')

# Dropping multiple columns
with schema.table('users') as table:
    table.drop_column('votes', 'avatar', 'location')
{% endhighlight %}

### Foreign keys

Eloquent also provides support for adding foreign key constraints to your tables:

{% highlight python %}
table.integer('user_id').unsigned()
table.foreign('user_id').references('id').on('users')
{% endhighlight %}

In this example, we are stating that the ``user_id``
column references the ``id`` column on the ``users`` table.
Make sure to create the foreign key column first!

You may also specify options for the "on delete" and "on update" actions of the constraint:

{% highlight python %}
table.foreign('user_id')\
    .references('id').on('users')\
    .on_delete('cascade')
{% endhighlight %}

To drop a foreign key, you may use the ``drop_foreign`` method.
A similar naming convention is used for foreign keys as is used for other indexes:

{% highlight python %}
table.drop_foreign('posts_user_id_foreign')
{% endhighlight %}

There is more you can do with the schema builder, like handling indexes. Referer to the
[documentation](http://eloquent.readthedocs.org/en/latest/schema_builder.html) to see all the available features.


## Soft deleting

When soft deleting a model, it is not actually removed from your database.
Instead, a ``deleted_at`` timestamp is set on the record.
To enable soft deletes for a model, make it inherit from the ``SoftDeletes`` mixin:

{% highlight python %}
from eloquent import Model, SoftDeletes


class User(Model, SoftDeletes):

    __dates__ = ['deleted_at']
{% endhighlight %}

Now, when you call the ``delete`` method on the model, the ``deleted_at`` column will be
set to the current timestamp. When querying a model that uses soft deletes,
the "deleted" models will not be included in query results.

### Forcing soft deleted models into results

To force soft deleted models to appear in a result set, use the ``with_trashed`` method on the query:

{% highlight python %}
users = User.with_trashed().where('account_id', 1).get()
{% endhighlight %}

The ``with_trashed`` method may be used on a defined relationship:

{% highlight python %}
user.posts().with_trashed().get()
{% endhighlight %}


## Query Scopes

### Defining a query scope

Scopes allow you to easily re-use query logic in your models.
To define a scope, simply prefix a model method with ``scope``:

{% highlight python %}
from eloquent Model, scope


class User(Model):


    def scope_popular(self, query):
        return query.where('votes', '>', 100)

    def scope_women(self, query):
        return query.where_gender('W')
{% endhighlight %}

### Using a query scope

{% highlight python %}
    users = User.popular().women().order_by('created_at').get()
{% endhighlight %}

### Dynamic scopes

Sometimes you may wish to define a scope that accepts parameters.
Just add your parameters to your scope function:

{% highlight python %}
class User(Model):

    def scope_of_type(self, query, type):
        return query.where_type(type)
{% endhighlight %}

Then pass the parameter into the scope call:

{% highlight python %}
users = User.of_type('member').get()
{% endhighlight %}


## Global Scopes

Sometimes you may wish to define a scope that applies to all queries performed on a model.
In essence, this is how Eloquent's own "soft delete" feature works.
Global scopes are defined using a combination of mixins and an implementation of the ``Scope`` class.

First, let's define a mixin. For this example, we'll use the ``SoftDeletes`` that ships with Eloquent:


{% highlight python %}
from eloquent import SoftDeletingScope


class SoftDeletes(object):

    @classmethod
    def boot_soft_deletes(cls, model_class):
        """
        Boot the soft deleting mixin for a model.
        """
        model_class.add_global_scope(SoftDeletingScope())
{% endhighlight %}

If an Eloquent model inherits from a mixin that has a method matching the ``boot_name_of_trait``
naming convention, that mixin method will be called when the Eloquent model is booted,
giving you an opportunity to register a global scope, or do anything else you want.
A scope must be an instance of the ``Scope`` class, which specifies two methods: ``apply`` and ``remove``.

The apply method receives an ``Builder`` query builder object and the ``Model`` it's applied to,
and is responsible for adding any additional ``where`` clauses that the scope wishes to add.
The ``remove`` method also receives a ``Builder`` object and ``Model`` and is responsible
for reversing the action taken by ``apply``.
In other words, ``remove`` should remove the ``where`` clause (or any other clause) that was added.
So, for our ``SoftDeletingScope``, it would look something like this:

{% highlight python %}
from eloquent import Scope


class SoftDeletingScope(Scope):

    def apply(self, builder, model):
        """
        Apply the scope to a given query builder.

        :param builder: The query builder
        :type builder: eloquent.orm.builder.Builder

        :param model: The model
        :type model: eloquent.orm.Model
        """
        builder.where_null(model.get_qualified_deleted_at_column())

    def remove(self, builder, model):
        """
        Remove the scope from a given query builder.

        :param builder: The query builder
        :type builder: eloquent.orm.builder.Builder

        :param model: The model
        :type model: eloquent.orm.Model
        """
        column = model.get_qualified_deleted_at_column()

        query = builder.get_query()

        wheres = []
        for where in query.wheres:
            # If the where clause is a soft delete date constraint,
            # we will remove it from the query and reset the keys
            # on the wheres. This allows the developer to include
            # deleted model in a relationship result set that is lazy loaded.
            if not self._is_soft_delete_constraint(where, column):
                wheres.append(where)

        query.wheres = wheres
{% endhighlight %}


## What's next?

Finally, here are some features targeted for the 0.5 version:

* Database migrations based on the schema builder:

{% highlight python %}
from eloquent import Migration


class UserMigration(Migration):

    def up(self):
        """
        Run the migrations
        """
        with self.schema.table('users') as table:
            # ...

    def down(self):
        """
        Revert the migrations
        """
        with self.schema.table('users') as table:
            # ...
{% endhighlight %}

* Support for model accessors and mutators.

You can check the advancement on [the Github project](https://github.com/sdispater/eloquent/issues?q=is%3Aissue+milestone%3A0.5)

---
layout: post
title:  "Orator (former Eloquent) ORM 0.5 is out: Database migrations, attributes accessors and mutators"
date:   2015-05-24 18:05:38
category: programming
tags: [python, orator, orm]
---

Orator 0.5 is now out. This version introduces database migrations and attributes accessors and mutators.

## Why the change of name?

This ORM project was inspired by the [Eloquent](http://laravel.com/docs/master/eloquent) ORM of the [Laravel](http://laravel.com) PHP framework. So when I started it I took
the same name for the project. However, as I have been pointed out, it would likely lead to confusion and conflict
when searching information about it. That's why I decided to change the name to **Orator**.

What it means is that the `eloquent` package is now **deprecated** and that the 0.5 version will be the last. It is therefore highly encouraged
to use the `orator` package instead.

## Database migrations

Migrations are a type of version control for your database. They allow a team to modify the database schema and stay up to date on the current schema state. Migrations are typically paired with the Schema Builder to easily manage your database’s schema.

<div class="note">
<p>For the migrations to actually work, you need a configuration file describing your databases in a <code>DATABASES</code> dict, like so:</p>
<pre>
<code class="python">DATABASES = {
    'mysql': {
        'driver': 'mysql',
        'host': 'localhost',
        'database': 'database',
        'username': 'root',
        'password': '',
        'prefix': ''
    }
}
</code>
</pre>
<p>This file needs to be specified when using migrations commands.</p>
</div>

### Creating migrations

To create a migration, you can use the `migrations:make` command on the Orator CLI:

{% highlight bash %}
orator migrations:make create_users_table -c databases.py
{% endhighlight %}

This will create a migration file that looks like this:

{% highlight python %}
from orator.migrations import Migration


class CreateTableUsers(Migration):

    def up(self):
        """
        Run the migrations.
        """
        pass

    def down(self):
        """
        Revert the migrations.
        """
        pass
{% endhighlight %}

By default, the migration will be placed in a `migrations` folder relative to where the command has been executed, and will contain a timestamp which allows the framework to determine the order of the migrations.

If you want the migrations to be stored in another folder, use the `--path/-p` option:

{% highlight bash %}
orator migrations:make create_users_table -c databases.py -p my/path/to/migrations
{% endhighlight %}

The `--table` and `--create` options can also be used to indicate the name of the table, and whether the migration will be creating a new table:

{% highlight bash %}
orator migrations:make add_votes_to_users_table -c databases.py --table=users

orator migrations:make create_users_table -c databases.py --table=users --create
{% endhighlight %}

These commands would respectively create the following migrations:

{% highlight python %}
from orator.migrations import Migration


class AddVotesToUsersTable(Migration):

    def up(self):
        """
        Run the migrations.
        """
        with self.schema.table('users') as table:
            pass

    def down(self):
        """
        Revert the migrations.
        """
        with self.schema.table('users') as table:
            pass
{% endhighlight %}

{% highlight python %}
from orator.migrations import Migration


class CreateTableUsers(Migration):

    def up(self):
        """
        Run the migrations.
        """
        with self.schema.create('users') as table:
            table.increments('id')
            table.timestamps()

    def down(self):
        """
        Revert the migrations.
        """
        self.schema.drop('users')
{% endhighlight %}

### Running migrations

To run all outstanding migrations, just use the `migrations:run` command:

{% highlight bash %}
orator migrations:run -c databases.py
{% endhighlight %}

### Rolling back migrations

#### Rollback the last migration operation

{% highlight bash %}
orator migrations:rollback -c databases.py
{% endhighlight %}

#### Rollback all migrations

{% highlight bash %}
eloquent migrations:reset -c databases.py
{% endhighlight %}

### Getting migrations status

To see the status of the migrations, just use the migrations:status command:

{% highlight bash %}
orator migrations:status -c databases.py
{% endhighlight %}

This would output something like this:

{% highlight bash %}
+----------------------------------------------------+------+
| Migration                                          | Ran? |
+----------------------------------------------------+------+
| 2015_05_02_04371430559457_create_users_table       | Yes  |
| 2015_05_04_02361430725012_add_votes_to_users_table | No   |
+----------------------------------------------------+------+
{% endhighlight %}


## Accessors & mutators

Orator provides a convenient way to transform your model attributes when getting or setting them.

### Defining an accessor

Simply use the `accessor` decorator on your model to declare an accessor:

{% highlight python %}
from orator.orm import Model, accessor


class User(Model):

    @accessor
    def first_name(self):
        first_name = self.get_raw_attribute('first_name')

        return first_name[0].upper() + first_name[1:]
{% endhighlight %}

In the example above, the `first_name` column has an accessor.

<div class="note">
<p>The name of the decorated function <strong>must</strong> match the name of the column being accessed.</p>
</div>

### Defining a mutator

Mutators are declared in a similar fashion:

{% highlight python %}
from orator.orm import Model, mutator


class User(Model):

    @mutator
    def first_name(self, value):
        self.set_raw_attribute('first_name', value)
{% endhighlight %}

<div class="note">
<p>If the column being mutated already has an accessor, you can use it has a mutator:</p>
<pre>
<code class="python">from orator.orm import Model, accessor


    class User(Model):

        @accessor
        def first_name(self):
            first_name = self.get_raw_attribute('first_name')

            return first_name[0].upper() + first_name[1:]

        @first_name.mutator
        def set_first_name(self, value):
            self.set_raw_attribute(value.lower())
</code>
</pre>
<p>The inverse is also possible:</p>
<pre>
<code class="python">from orator.orm import Model, mutator


    class User(Model):

        @mutator
        def first_name(self, value):
            self.set_raw_attribute(value.lower())

        @first_name.accessor
        def get_first_name(self):
            first_name = self.get_raw_attribute('first_name')

            return first_name[0].upper() + first_name[1:]
</code>
</pre>
</div>

### Appendable attributes

When converting a model to a dictionary or a JSON string, you may occasionally need to add dictionary attributes that do not have a corresponding column in your database.
To do so, simply define an `accessor` for the value:

{% highlight python %}
class User(Model):

    @accessor
    def is_admin(self):
        return self.get_raw_attribute('admin') == 'yes'
{% endhighlight %}


Once you have created the accessor, just add the value to the `__appends__` property on the model:

{% highlight python %}
class User(Model):

    __append__ = ['is_admin']

    @accessor
    def is_admin(self):
        return self.get_raw_attribute('admin') == 'yes'
{% endhighlight %}

Once the attribute has been added to the `__appends__` list, it will be included in both the model's dictionary and JSON forms.
Attributes in the `__appends__` list respect the `__visible__` and `__hidden__` configuration on the model.


There is a lot more you can do with Orator, just give a look at the [documentation](http://orator.readthedocs.org)
to see all available features.


## What's next?

Finally, here are some features targeted for future versions (actual roadmap to be determined):

* Model events:

{% highlight python %}
from models import User


def check_user(user):
    if not user.is_valid():
        return False

User.creating(check_user)
{% endhighlight %}

* Extra functionalities, like caching.

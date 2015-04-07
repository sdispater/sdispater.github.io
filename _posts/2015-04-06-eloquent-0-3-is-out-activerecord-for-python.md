---
layout: post
title:  "Eloquent 0.3 is out: ActiveRecord for Python"
date:   2015-04-06 18:05:38
category: programming
tags: [python, eloquent, orm]
---

Eloquent 0.3 is now out. This is the first official release of the ORM.


## What is Eloquent

The Eloquent ORM provides a simple and beautiful ActiveRecord implementation
for Python.

It is inspired by the database part of the [Laravel framework](http://laravel.com)
but largely modified to be more pythonic.

The full documentation is available here: <http://eloquent.readthedocs.org>.


## Why

I started Eloquent because of a personal need for simplicity when defining
models and accessing databases, which is one of the things that i find quite
appealing in a framework like Ruby On Rails.

So I wanted Eloquent to follow the ActiveRecord pattern which provides less
configuration and verbosity compared to the data mapper pattern
which pretty much all of the Python ORMs follow.

I also wanted it to be really simple to read and write, with useful features
out of the box (like timestampable models, soft deletes, protection against mass assigment, model events).
Some of these features are not ready yet.


## Installation

You can install Eloquent in 2 different ways:

* The easier and more straightforward is to use pip

{% highlight python %}
pip install eloquent
{% endhighlight %}

* Install from source using the official repository (https://github.com/sdispater/eloquent)

The different dbapi packages are not part of the package dependencies,
so you must install them in order to connect to corresponding databases:

* Postgres: ``pyscopg2``
* MySQL: ``PyMySQL`` or ``MySQL-python``
* Sqlite: The ``sqlite3`` module is bundled with Python by default


## Basic Usage

### Configuration

All you need to get you started is the configuration describing your database connections
and passing it to a ``DatabaseManager`` instance.

{% highlight python %}
from eloquent import DatabaseManager, Model

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
Model.set_connection_resolver(db)
{% endhighlight %}


### Defining a model

{% highlight python %}
class User(Model):
    pass
{% endhighlight %}

Note that we did not tell the ORM which table to use for the ``User`` model. The plural "snake case" name of the
class name will be used as the table name unless another name is explicitly specified.
In this case, the ORM will assume the ``User`` model stores records in the ``users`` table.
You can specify a custom table by defining a ``__table__`` property on your model:

{% highlight python %}
class User(Model):

    __table__ = 'my_users'
{% endhighlight %}

<div class="note">
<p>The ORM will also assume that each table has a primary key column named ``id``.
You can define a <code>__primary_key__</code> property to override this convention.
Likewise, you can define a <code>__connection__</code> property to override the name of the database
connection that should be used when using the model.</p>
</div>

Once a model is defined, you are ready to start retrieving and creating records in your table.
Note that you will need to place ``updated_at`` and ``created_at`` columns on your table by default.
If you do not wish to have these columns automatically maintained,
set the ``__timestamps__`` property on your model to ``False``.


### Retrieving all models

{% highlight python %}
users = User.all()
{% endhighlight %}


### Retrieving a record by primary key

{% highlight python %}
user = User.find(1)
{% endhighlight %}


### Querying using models

{% highlight python %}
users = User.where('votes', '>', 100).take(10).get()

for user in users:
    print(user.name)
{% endhighlight %}


### Aggregates

You can also use the query builder aggregate functions:

{% highlight python %}
count = User.where('votes', '>', 100).count()
{% endhighlight %}

If you feel limited by the builder's fluent interface, you can use the ``where_raw`` method:

{% highlight python %}
users = User.where_raw('age > ? and votes = 100', [25]).get()
{% endhighlight %}


### Chunking Results

If you need to process a lot of records, you can use the ``chunk`` method to avoid
consuming a lot of RAM:

{% highlight python %}
for users in User.chunk(100):
    for user in users:
        # ...
{% endhighlight %}


### Specifying the query connection

You can specify which database connection to use when querying a model by using the ``on`` method:

{% highlight python %}
user = User.on('connection-name').find(1)
{% endhighlight %}

If you are using read / write connections, you can force the query to use the "write" connection
with the following method:

{% highlight python %}
user = User.on_write_connection().find(1)
{% endhighlight %}


## Mass assignment

When creating a new model, you pass attributes to the model constructor.
These attributes are then assigned to the model via mass-assignment.
Though convenient, this can be a serious security concern when passing user input into a model,
since the user is then free to modify **any** and **all** of the model's attributes.
For this reason, all models protect against mass-assignment by default.

To get started, set the ``__fillable__`` or ``__guarded__`` properties on your model.


### Defining fillable attributes on a model

The ``__fillable__`` property specifies which attributes can be mass-assigned.

{% highlight python %}
class User(Model):

    __fillable__ = ['first_name', 'last_name', 'email']
{% endhighlight %}


### Defining guarded attributes on a model

The ``__guarded__`` is the inverse and acts as "blacklist".

{% highlight python %}
class User(Model):

    __guarded__ = ['id', 'password']
{% endhighlight %}


You can also block **all** attributes from mass-assignment:

{% highlight python %}
__guarded__ = ['*']
{% endhighlight %}



## Insert, update and delete


### Saving a new model

To create a new record in the database, simply create a new model instance and call the ``save`` method.

{% highlight python %}
user = User()

user.name = 'John'

user.save()
{% endhighlight %}

You can also use the ``create`` method to save a model in a single line, but you will need to specify
either the ``__fillable__`` or ``__guarded__`` property on the model since all models are protected against
mass-assigment by default.

After saving or creating a new model with auto-incrementing IDs, you can retrieve the ID by accessing
the object's ``id`` attribute:

{% highlight python %}
inserted_id = user.id
{% endhighlight %}


### Using the create method

{% highlight python %}
# Create a new user in the database
user = User.create(name='John')

# Retrieve the user by attributes, or create it if it does not exist
user = User.first_or_create(name='John')

# Retrieve the user by attributes, or instantiate it if it does not exist
user = User.first_or_new(name='John')
{% endhighlight %}


### Updating a retrieved model

{% highlight python %}
user = User.find(1)

user.name = 'Foo'

user.save()
{% endhighlight %}

You can also run updates as queries against a set of models:

{% highlight python %}
affected_rows = User.where('votes', '>', 100).update(status=2)
{% endhighlight %}


### Deleting an existing model

To delete a model, simply call the ``delete`` model:

{% highlight python %}
user = User.find(1)

user.delete()
{% endhighlight %}


### Deleting an existing model by key

{% highlight python %}
User.destroy(1)

User.destroy(1, 2, 3)
{% endhighlight %}

You can also run a delete query on a set of models:

{% highlight python %}
affected_rows = User.where('votes', '>' 100).delete()
{% endhighlight %}


### Updating only the model's timestamps

If you want to only update the timestamps on a model, you can use the ``touch`` method:

{% highlight python %}
user.touch()
{% endhighlight %}


## Timestamps

By default, the ORM will maintain the ``created_at`` and ``updated_at`` columns on your database table
automatically. Simply add these ``timestamp`` columns to your table. If you do not wish for the ORM to maintain
these columns, just add the ``__timestamps__`` property:

{% highlight python %}
class User(Model):

    __timestamps__ = False
{% endhighlight %}


## Converting to dictionaries / JSON

### Converting a model to a dictionary

When building JSON APIs, you may often need to convert your models and relationships to dictionaries or JSON.
So, Eloquent includes methods for doing so. To convert a model and its loaded relationship to a dictionary,
you may use the ``to_dict`` method:

{% highlight python %}
user = User.with_('roles').first()

return user.to_dict()
{% endhighlight %}

Note that entire collections of models can also be converted to dictionaries:

{% highlight python %}
return User.all().to_dict()
{% endhighlight %}

### Converting a model to JSON

To convert a model to JSON, you can use the ``to_json`` method:

{% highlight python %}
return User.find(1).to_json()
{% endhighlight %}


There is a lot more you can do with Eloquent, just give a look at the [documentation](http://eloquent.readthedocs.org)
to see all available features.

And if you are interested by how it all works, you can just check out the [Github Repository](https://github.com/sdispater/eloquent), and feel free to contribute.


And, finally, here are some features targeted for the 0.4 version:

* A schema builder to create and modify tables:

{% highlight python %}
from eloquent import SchemaBuilder

schema = SchemaBuilder()

with schema.create('users') as table:
    table.increments('id')
    table.string('name').unique()
    table.string('email').unique()
    table.timestamps()
{% endhighlight %}

* Support for soft delete and other mixins:

{% highlight python %}
class User(Model, SoftDeletesMixin):
    pass
{% endhighlight %}

You can check the advancement on [the Github project](https://github.com/sdispater/eloquent/issues?q=is%3Aissue+milestone%3A0.4)

---
layout: post
title:  "Orator 0.7 is out: Relationship decorators, seeding, model factories and cache"
date:   2015-11-10 10:24:38
category: releases
tags: ['0.7']
---

Orator 0.7 is now out. This version introduces relationship decorators, seeding, model factories and cache support.
It also brings various performance improvements and fixes, especially in the way relationships are handled.

For the full list of changes, see the [CHANGELOG](https://github.com/sdispater/orator/blob/master/CHANGELOG.md)

## Relationship decorators

The way model relationships are declared has been largely modified and improved.

Now, relationships are declared via decorators.

<div class="note">
<p>Even though decorators are the preferred way the old notation is still supported.</p>
</div>

Here is an example:

{% highlight python %}
from orator.orm import has_many, belongs_to


class Post(Model):

    @has_many
    def comments(self):
        return Comment


class Comment(Model):

    @belongs_to
    def post(self):
        return Post
{% endhighlight %}

## Model Factories

When testing, it is common to need to insert a few records into your database before executing your test.
Instead of manually specifying the value of each column when you create this test data,
Orator allows you to define a default set of attributes for each of your models using "factories":

{% highlight python %}

from orator.orm import Factory

factory = Factory()

@factory.define(User)
def users_factory(faker):
    return {
        'name': faker.name(),
        'email': faker.email()
    }
{% endhighlight %}


Within the function (here ``users_factory``), which serves as the factory definition,
you can return the default test values of all attributes on the model.
The function will receive an instance of the [Faker](https://github.com/joke2k/faker) library,
which allows you to conveniently generate various kinds of random data for testing.


### Multiple Factory Types

Sometimes you may wish to have multiple factories for the same model class.
For example, perhaps you would like to have a factory for "Administrator" users in addition to normal users.
You can define these factories using the ``define_as`` method:

{% highlight python %}
@factory.define_as(User, 'admin')
def admins_factory(faker):
    return {
        'name': faker.name(),
        'email': faker.email(),
        'admin': True
    }
{% endhighlight %}

Instead of duplicating all of the attributes from your base user factory,
you can use the ``raw`` method to retrieve the base attributes.
Once you have the attributes, simply supplement them with any additional values you require:

{% highlight python %}
@factory.define_as(User, 'admin')
def admins_factory(faker):
    user = factory.raw(User)

    user.update({
        'admin': True
    })

    return user
{% endhighlight %}


### Using Factories In Tests

Once you have defined your factories, you can use them in your tests or database seed files
to generate model instances calling the ``Factory`` instance.
So, let's take a look at a few examples of creating models.
First, we'll use the ``make`` method, which creates models but does not save them to the database:

{% highlight python %}
def test_database():
    user = factory(User).make()

    # Use model in tests
{% endhighlight %}


If you would like to override some of the default values of your models,
you can pass keyword arguments to the ``make`` method.
Only the specified values will be replaced while the rest of the values remain
set to their default values as specified by the factory:

{% highlight python %}
def test_database():
    user = factory(User).make(name='John')
{% endhighlight %}

You can also create a ``Collection`` of many models or create models of a given type:

{% highlight python %}
# Create 3 User instances
users = factory(User, 3).make()

# Create a User "admin" instance
admin = factory(User, 'admin').make()

# Create three User "admin" instances
admins = factory(User, 'admin', 3).make()
{% endhighlight %}


### Persisting Factory Models

The ``create`` method not only creates the model instances,
but also saves them to the database using models' ``save`` method:

{% highlight python %}
def test_database():
    user = factory(User).create()

    # Use model in tests
{% endhighlight %}

Again, you can override attributes on the model by passing an array to the ``create`` method:

{% highlight python %}
def test_database():
    user = factory(User).create(name='John')
{% endhighlight %}


### Adding Relations To Models

You may even persist multiple models to the database.
In this example, we'll even attach a relation to the created models.
When using the ``create`` method to create multiple models, a ``Collection`` instance is returned,
allowing you to use any of the convenient functions provided by the collection, such as ``each``:

{% highlight python %}
users = factory(User, 3).create()
users.each(lambda u: u.save(factory(Post).make()))
{% endhighlight %}


## Seeding

Orator includes a simple method of seeding your database with test data using seed classes.
Seed classes may have any name you wish, but probably should follow some sensible convention,
such as ``UsersTableSeeder``, etc.
By default, a ``DatabaseSeeder`` class is defined for you the first time you create a seed class.
From this class, you can use the ``call`` method to run other seed classes,
allowing you to control the seeding order.


### Writing Seeders

To generate a seeder, you may issue the ``seeders:make`` command.
All seeders generated by the command will be placed in a ``seeders`` directory
relative to where the command has been executed:

{% highlight bash %}
orator seeds:make user_table_seeder
{% endhighlight %}

<div class="note">
<p>If you want the seed classes to be stored in another directory, use the ``-p/--path`` option</p>
</div>

A seeder class only contains one method by default: ``run``.
This method is called when the ``db:seed`` command is executed.
Within the ``run`` method, you can insert data into your database however you wish.
You can use the [QueryBuilder](/docs/query_builder.html) to manually insert data or you can use [ModelFactories](#model-factories).

As an example, let's modify the ``UsersTableSeeder`` class you just created.
Let's add a database insert statement to the ``run`` method:

{% highlight python %}
from orator.seeds import Seeder


class UsersTableSeeder(Seeder):

    def run(self):
        """
        Run the database seeds.
        """

        # Here you could just use random string generators
        # rather than hardcoded values
        self.db.table('users').insert({
            'name': 'john',
            'email': 'john@doe.com'
        })
{% endhighlight %}

#### Using Model Factories

Of course, manually specifying the attributes for each model seed is cumbersome.
Instead, you can use [ModelFactories](#model-factories) to conveniently generate large amounts of database records.
You can use an external factory or use the seeder class ``factory`` attribute.

For example, let's create 50 users and attach a relationship to each user:

{% highlight python %}
from orator.seeds import Seeder
from orator.orm import Factory


factory = Factory()


@factory.define(User)
def users_factory(faker):
    return {
        'name': faker.name(),
        'email': faker.email()
    }


@factory.define(Post)
def posts_factory(faker):
    return {
        'title': faker.name(),
        'content': faker.text()
    }


class UsersTableSeeder(Seeder):

    factory = factory  # This is only needed when using an external factory

    def run(self):
        """
        Run the database seeds.
        """
        self.factory(User, 50).create().each(
            lambda u: u.posts().save(self.factory(Post).make())
        )
{% endhighlight %}

Or using directly the ``factory`` attribute without an external factory:

{% highlight python %}
class UsersTableSeeder(Seeder):

    def run(self):
        """
        Run the database seeds.
        """
        self.factory.register(User, self.users_factory)
        self.factory.register(Post, self.posts_factory)

        self.factory(User, 50).create().each(
            lambda u: u.posts().save(self.factory(Post).make())
        )

    def users_factory(self, faker):
        return {
            'name': faker.name(),
            'email': faker.email()
        }

    def posts_factory(self, faker):
        return {
            'title': faker.name(),
            'content': faker.text()
        }
{% endhighlight %}

#### Calling Additional Seeders

Within the ``DatabaseSeeder`` class, you can use the ``call`` method to execute additional seed classes.
Using the ``call`` method allows you to break up your database seeding into multiple files
so that no single seeder class becomes overwhelmingly large.
Simply pass the seeder class you wish to run:

{% highlight python %}
def run(self):
    """
    Run the database seeds.
    """
    self.call(UsersTableSeeder)
    self.call(PostsTableSeeder)
    self.call(CommentsTableSeeder)
{% endhighlight %}

### Running Seeders

Once you have written your seeder classes, you may use the ``db:seed`` command to seed your database.
By default, the ``db:seed`` command runs the ``database_seeder`` file, which can be used to call other seed classes.
However, you can use the ``--seeder`` option to specify a specific seeder class to run individually:

{% highlight bash %}
orator db:seed

orator db:seed --seeder users_table_seeder
{% endhighlight %}

You can also seed your database using the ``migrations:refresh`` command,
which will also rollback and re-run all of your migrations.
This command is useful for completely re-building your database:

{% highlight bash %}
orator migrations:refresh --seed
{% endhighlight %}

## Cache

<div class="note">
<p>The cache functionality is not bundled in the core package but is provided
by the official extension <code>orator-cache</code></p>
</div>

The ``orator-cache`` package provides query results caching to Orator.
It uses the [cachy](https://github.com/sdispater/cachy) library to ease cache manipulation.

To activate the caching ability you just need to use the provided ``DatabaseManager`` class instead of
the default one and passing it a ``Cache`` instance:

{% highlight python %}
from orator_cache import DatabaseManager, Cache

stores = {
    'stores': {
        'redis': {
            'driver': 'redis',
            'host': 'localhost',
            'port': 6379,
            'db': 0
        },
        'memcached': {
            'driver' 'memcached',
            'servers': [
                '127.0.0.1:11211'
            ]
        }
    }
}

cache = Cache(stores)

db = DatabaseManager(config, cache=cache)
{% endhighlight %}

<div class="note">
<p>Since the <code>Cache</code> class is just a subclass of the Cachy <code>CacheManager</code> class. You can refer
to the Cachy <a href="http://cachy.readthedocs.org" title="documentation">documentation</a> to configure the underlying stores.</p>
</div>

<div class="warning">
<p>Even though, the extension provides a way to cache queries, the invalidation of the caches
is the responsability of the developer.</p>
</div>


### Caching queries

You can easily cache the results of a query using the ``remember`` or ``remember_forever`` methods:

{% highlight python %}
users = db.table('users').remember(10).get()
{% endhighlight %}

In this example, the results of the query will be cached for ten minutes.
While the results are cached, the query will not be run against the database,
and the results will be loaded from the default cache driver.

<div class="note">
<p>You can also specify which cache driver to use:</p>

{% highlight python %}
users = db.table('users').cache_driver('redis').remember(10).get()
{% endhighlight %}
</div>

If you are using a [supported cache driver](http://cachy.readthedocs.org/en/latest/cache_tags.html), you can also add tags to the caches:

{% highlight python %}
users = db.table('users').cache_tags(['people', 'authors']).remember(10).get()
{% endhighlight %}

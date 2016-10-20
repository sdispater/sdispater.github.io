---
layout: post
title:  "Building a REST API with Flask and the Orator ORM"
date:   2016-10-13 17:20:38
category: programming
tags: [python, orator]
---

In this article I'm going to show how easy it is to create a REST API with Flask and the Orator ORM.

The API will mainly consist in users that can post messages and follow each other.

It will not talk about REST concepts and will assume that you are familiar with the basics of REST. It will not talk about authentication either so that it remains simple. It's merely a tutorial to showcase some of the features of Orator.

<aside class="note">

<p>

The examples of API calls will use the excellent <a href="https://github.com/jkbrzt/httpie">httpie</a> library.

</p>

</aside>

## Set Up

First of, we download what we need: the [Flask](http://flask.pocoo.org/) micro framework and [flask-orator](http://flask-orator.readthedocs.io).

```shell
$ pip install flask flask-orator
```

Now that we have Flask installed let's create a simple web application, which we will put in a file called `app.py`:

```python
import os
from flask import Flask, request
from flask_orator import Orator, jsonify

# Configuration
DEBUG = True
ORATOR_DATABASES = {
    'default': 'twittor',
    'twittor': {
        'driver': 'sqlite',
        'database': os.path.join(os.path.dirname(__file__), 'twittor.db')
    }
}

# Creating Flask application
app = Flask(__name__)
app.config.from_object(__name__)

# Initializing Orator
db = Orator(app)

if __name__ == '__main__':
    app.run()
```

Let me explain in more details the code:

* `DEBUG = True` Since this is a tutorial, we activate the Flask debug mode to have auto reloading of the application when the file is modified and also to have more information when an error occurs.

* `ORATOR_DATABASES` This configuration section defines the database or databases you want to use. Basically, here we define a `sqlite` database named `twittor.db` that will live alongside our `app.py` file. For more information, you can refer to the official Orator [documentation](https://orator-orm.com/docs/)

* `app = Flask(__name__)` We create a new Flask application

* `app.config.from_object(__name__)` And we configure it. Basically, Flask will take all uppercase variables as a configuration element.

* `db = Orator(app)` Finally we create our `Orator` object which will act as a `DatabaseManager` with added benefits.


The last part is here to make `app.py` run the server when executed, so let's do that.

```shell
$ python app.py
* Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
* Restarting with stat
* Debugger is active!
* Debugger pin code: 313-220-660
```

Now we need a `db.py` file that will handle all database operations, like migrations.

```python
from app import db


if __name__ == '__main__':
    db.cli.run()
```

It will act exactly like the `orator` command but automatically configured to use the database specified in `app.py`.  



## Users

What we first need is to create the `users` table. We will use the `db.py` file we just created:  

```shell
$ python db.py make:migration create_users_table --table users --create
```

This will add a file in the `migrations` folder named `create_users_table` and prefixed by a timestamp:

```python
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
```

You need to modify this file to add the `name` and `email` columns:

```python
with self.schema.create('users') as table:
    table.increments('id')
    table.string('name').unique()
    table.string('email').unique()
    table.timestamps()
```

To see all types that are supported by the Orator schema builder refer to the [documentation](https://orator-orm.com/docs/schema_builder.html)

Then, you can run the migration:

```shell
$ python db.py migrate
```

Confirm and you database and the table will be created.

<aside class="note">

<p>You can also add the <code>-P/—pretend</code> option to see what queries will be executed without actually executing them.</p>

</aside>

Let's check if everything is ok:

```shell
$ python db.py migrate:status
```

And you should get something like this:

``` 
+--------------------------------------+------+
| Migration                            | Ran? |
+--------------------------------------+------+
| 2016_10_13_145432_create_users_table | Yes  |
+--------------------------------------+------+
```



Now that we have a newly created `users` table we can create our `User` model in `app.py`:

```python
class User(db.Model):

    __fillable__ = ['name', 'email']
```

Orator will automatically use the `users` table for the model. By convention, it uses the pluralized lowercased version of the model name as the table name. You can change this by using the `__table__` property.

```python
class User(db.Model):
    
    __table__ = 'users'
```

The `__fillable__` property tells Orator which attributes can be mass-assigned using the`create` and `update` methods.  

We can now add a route to create users:

```python
@app.route('/users', methods=['POST'])
def create_user():
    user = User.create(**request.get_json())
    
    return jsonify(user)
```

`jsonify` is an improved version of the `Flask` one. It will automatically detect if the value is a model or a collection and will automatically return it as a JSON string.

Let's test it out:

```shell
$ http POST http://127.0.0.1:5000/users name=twittor email=twittor@twittor.com
```

```http
HTTP/1.0 200 OK
Content-Length: 175
Content-Type: application/json
Date: Thu, 13 Oct 2016 15:24:50 GMT
Server: Werkzeug/0.11.11 Python/3.5.2

{
    "created_at": "2016-10-13T15:24:50.972840+00:00",
    "email": "twittor@twittor.com",
    "id": 1,
    "name": "twittor",
    "updated_at": "2016-10-13T15:24:50.972840+00:00"
}
```

As you can see, both `created_at` and `updated_at` columns have been automatically set. Orator will always populate timestamps unless told otherwise.  

Let's now add a route to retrieve our new user

```python
@app.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    user = User.find(user_id)

    return jsonify(user)
```

```shell
$ http http://127.0.0.1:5000/users/1
```

```http
HTTP/1.0 200 OK
Content-Length: 175
Content-Type: application/json
Date: Thu, 13 Oct 2016 15:27:13 GMT
Server: Werkzeug/0.11.11 Python/3.5.2

{
    "created_at": "2016-10-13T15:24:50.972840+00:00",
    "email": "twittor@twittor.com",
    "id": 1,
    "name": "twittor",
    "updated_at": "2016-10-13T15:24:50.972840+00:00"
}
```

The `find()` method will return `None` if no user was found, you can also return automatically a 404 error by using the `find_or_fail()` method. 

```python
user = User.find_or_fail(user_id)
```

And for the sake of it, we add an endpoint to retrieve all users:

```python
@app.route('/users', methods=['GET'])
def get_all_users():
    users = User.all()

    return jsonify(users)
```

```shell
$ http http://127.0.0.1:5000/users
```

```http
HTTP/1.0 200 OK
Content-Length: 193
Content-Type: application/json
Date: Thu, 13 Oct 2016 15:32:01 GMT
Server: Werkzeug/0.11.11 Python/3.5.2

[
    {
        "created_at": "2016-10-13T15:24:50.972840+00:00",
        "email": "sebastien@eustace.io",
        "id": 1,
        "name": "sdispater",
        "updated_at": "2016-10-13T15:24:50.972840+00:00"
    }
]
```

And to update them:

```python
@app.route('/users/<int:user_id>', methods=['PATCH'])
def update_user(user_id):
    user = User.find_or_fail(user_id)
    user.update(**request.get_json())

    return jsonify(user)
```

```shell
$ http PATCH http://127.0.0.1:5000/users/1 name=sebastien
```

```http
HTTP/1.0 200 OK
Content-Length: 175
Content-Type: application/json
Date: Thu, 13 Oct 2016 15:36:17 GMT
Server: Werkzeug/0.11.11 Python/3.5.2

{
    "created_at": "2016-10-13T15:24:50.972840+00:00",
    "email": "twittor@twittor.com",
    "id": 1,
    "name": "twittor",
    "updated_at": "2016-10-13T15:36:17.675494+00:00"
}
```

Since we are developing a new API, we will have to reset our database often to start fresh and see if everything works properly and creating users by hand can be cumbersome.

To fix this, Orator provides what are called seeders that can automatically populate our database.

Let's create a seeder for our `users` table: 

```shell
$ python db.py make:seed users_table_seeder
```

It will create the `seeds` folder with two files `database_seeder` and `users_table_seeder` . `database_seeder` is the main seeder that should call all other seeders, so let's modify it so that it looks like this: 

```python
from orator.seeds import Seeder
from .users_table_seeder import UsersTableSeeder


class DatabaseSeeder(Seeder):

    def run(self):
        """
        Run the database seeds.
        """
        self.call(UsersTableSeeder)
```

And `users_table_seeder` should look like this: 

```python
from orator.seeds import Seeder


class UsersTableSeeder(Seeder):

    def run(self):
        """
        Run the database seeds.
        """
        self.db.table('users').insert({
            'name': 'twittor',
            'email': 'twittor@twittor.com'
        })
```

now we can refresh the database. Refreshing the database will revert all migrations, reapply them and seed the database.

```shell
$ python db.py migrate:refresh
```

Confirm all and you should now have a fresh and seeded database.

<aside class="note">

<p>Calling <code>migrate:refresh</code> is the same thing as calling <code>migrate:reset</code>, <code>migrate</code> and <code>db:seed</code>.</p>

</aside>

Let's take a look

```shell
$ http http://127.0.0.1:5000/users
```

```http
HTTP/1.0 200 OK
Content-Length: 176
Content-Type: application/json
Date: Thu, 13 Oct 2016 15:48:10 GMT
Server: Werkzeug/0.11.11 Python/3.5.2

[
    {
        "created_at": "2016-10-13T15:48:03+00:00",
        "email": "twittor@twittor.com",
        "id": 1,
        "name": "twittor",
        "updated_at": "2016-10-13T15:48:03+00:00"
    }
]
```



## Messages

A user can have one or many messages. We will store them in the `messages` table that we must create: 

```shell
$ python db.py make:migration create_messages_table --table messages --create
```

Modify the newly created file:

```python
class CreateMessagesTable(Migration):

    def up(self):
        """
        Run the migrations.
        """
        with self.schema.create('messages') as table:
            table.increments('id')
            table.text('content')
            table.integer('user_id').unsigned()
            table.timestamps()

            table.foreign('user_id').references('id').on('users').on_delete('cascade')

    def down(self):
        """
        Revert the migrations.
        """
        self.schema.drop('messages')
```

Create a seeder for it:

```shell
$ python db.py make:seed messages_table_seeder
```

```python
class MessagesTableSeeder(Seeder):

    def run(self):
        """
        Run the database seeds.
        """
        user = self.db.table('users').where('name', 'twittor').first()

        self.db.table('messages').insert({
            'content': 'This is a message.',
            'user_id': user['id']
        })
```

```python
from orator.seeds import Seeder
from .users_table_seeder import UsersTableSeeder
from .messages_table_seeder import MessagesTableSeeder


class DatabaseSeeder(Seeder):

    def run(self):
        """
        Run the database seeds.
        """
        self.call(UsersTableSeeder)
        self.call(MessagesTableSeeder)
```

and, finally, refresh the database:

```shell
$ python db.py migrate:refresh
```



We create the `Message` model

```python
class Message(db.Model):
    
    __fillable__ = ['content']
```

We should now define the relations that exist between a `User` and a `Message`. A  user has one or many messages and a message only has one user. So let's import the corresponding decorators

```python
from orator.orm import belongs_to, has_many
```

And add the relations to our models:

```python
class User(db.Model):

    __fillable__ = ['name', 'email']
    
    @has_many
    def messages(self):
        return Message
```

```python
class Message(db.Model):

    __fillable__ = ['content']

    @belongs_to
    def user(self):
        return User
```

<aside class="note">

<p>The relation decorators accept arguments, as described in the <a href="https://orator-orm.com/docs/0.9/orm.html#relationships">documentation</a>, but we do not need them here since we use the Orator conventions.</p>

</aside>

We add a route to retrieve the messages of a specific user:

```python
@app.route('/users/<int:user_id>/messages', methods=['GET'])
def get_user_messages(user_id):
    user = User.find_or_fail(user_id)

    return jsonify(user.messages)
```

Let's check:

```shell
$ http http://127.0.0.1:5000/users/1/messages
```

```http
HTTP/1.0 200 OK
Content-Length: 172
Content-Type: application/json
Date: Thu, 13 Oct 2016 15:57:51 GMT
Server: Werkzeug/0.11.11 Python/3.5.2

[
    {
        "content": "This is a message.",
        "created_at": "2016-10-13T15:57:45+00:00",
        "id": 1,
        "updated_at": "2016-10-13T15:57:45+00:00",
        "user_id": 1
    }
]
```

We should also have a route to create new messages:

```python
@app.route('/users/<int:user_id>/messages', methods=['POST'])
def create_message(user_id):
    user = User.find_or_fail(user_id)
    message = user.messages().create(**request.get_json())

    return jsonify(message)
```

You can see that we are using the `messages` relation to create the new message. This is possible thanks to Orator [Dynamic Properties](https://orator-orm.com/docs/orm.html#dynamic-properties). Basically, each relation property can be called to create new objects or add conditions to retrieve a more fine grained portion of the relation.

```shell
$ http POST http://127.0.0.1:5000/users/1/messages content='A new message'
```

```http
HTTP/1.0 200 OK
Content-Length: 163
Content-Type: application/json
Date: Thu, 13 Oct 2016 16:00:48 GMT
Server: Werkzeug/0.11.11 Python/3.5.2

{
    "content": "A new message",
    "created_at": "2016-10-13T16:00:48.139841+00:00",
    "id": 2,
    "updated_at": "2016-10-13T16:00:48.139841+00:00",
    "user_id": 1
}
```

And finally, we create routes to get, update and delete a message:

```python
@app.route('/messages/<int:message_id>', methods=['GET'])
def get_message(message_id):
    message = Message.find_or_fail(message_id)

    return jsonify(message)


@app.route('/messages/<int:message_id>', methods=['PATCH'])
def update_message(message_id):
    message = Message.find_or_fail(message_id)
    message.update(**request.get_json())

    return jsonify(message)


@app.route('/messages/<int:message_id>', methods=['DELETE'])
def delete_message(message_id):
    message = Message.find_or_fail(message_id)
    message.delete()

    return app.response_class('No Content', 204)
```





## Followers

We want our users to be able to follow each other. To retain the information we need a table, let's call it `followers`: 

```shell
python db.py make:migration create_followers_table --table followers --create
```

We edit the migration file:

```python
class CreateFollowersTable(Migration):

    def up(self):
        """
        Run the migrations.
        """
        with self.schema.create('followers') as table:
            table.increments('id')
            table.integer('follower_id').unsigned()
            table.integer('followed_id').unsigned()
            table.timestamps()

            table.foreign('follower_id').references('id').on('users').on_delete('cascade')
            table.foreign('followed_id').references('id').on('users').on_delete('cascade')

    def down(self):
        """
        Revert the migrations.
        """
        self.schema.drop('followers')
```

Let's create the corresponding seeder:

```
$ python db.py make:seed followers_table_seeder
```

And call it from `DatabaseSeeder`:

```python
from orator.seeds import Seeder
from .users_table_seeder import UsersTableSeeder
from .messages_table_seeder import MessagesTableSeeder
from .followers_table_seeder import FollowersTableSeeder


class DatabaseSeeder(Seeder):

    def run(self):
        """
        Run the database seeds.
        """
        self.call(UsersTableSeeder)
        self.call(MessagesTableSeeder)
        self.call(FollowersTableSeeder)
```

We will add more users and messages:

```python
class UsersTableSeeder(Seeder):

    def run(self):
        """
        Run the database seeds.
        """
        # You can insert any number of users
        with self.db.transaction():
            self.db.table('users').insert({
                'name': 'twittor',
                'email': 'twittor@twittor.com'
            })
            self.db.table('users').insert({
                'name': 'john',
                'email': 'john@doe.com'
            })
            self.db.table('users').insert({
                'name': 'jane',
                'email': 'jane@doe.com'
            })
```



```python
class MessagesTableSeeder(Seeder):

    def run(self):
        """
        Run the database seeds.
        """
        with self.db.transaction():
            twittor = self.db.table('users').where('name', 'twittor').first()
            john = self.db.table('users').where('name', 'john').first()
            jane = self.db.table('users').where('name', 'jane').first()

            self.db.table('messages').insert({
                'content': 'Twittor\'s message.',
                'user_id': twittor['id']
            })

            self.db.table('messages').insert({
                'content': 'John\'s message.',
                'user_id': john['id']
            })

            self.db.table('messages').insert({
                'content': 'Jane\'s message.',
                'user_id': jane['id']
            })
```

And add followers data:

```python
class FollowersTableSeeder(Seeder):

    def run(self):
        """
        Run the database seeds.
        """
        with self.db.transaction():
            twittor = self.db.table('users').where('name', 'twittor').first()
            john = self.db.table('users').where('name', 'john').first()
            jane = self.db.table('users').where('name', 'jane').first()

            self.db.table('followers').insert([
                {'follower_id': twittor.id, 'followed_id': john.id},
                {'follower_id': john.id, 'followed_id': jane.id},
                {'follower_id': jane.id, 'followed_id': twittor.id},
            ])
```

We refresh:

```shell
$ python db.py migrate:refresh
```

The followers/followed relations are many to many relations so we need the `belongs_to_many` decorator: 

```python
from orator.orm import belongs_to, has_many, belongs_to_many
```

And we add the `followers` and `followed` relations to the `User` model:   

```python
class User(db.Model):

    __fillable__ = ['name', 'email']

    @has_many
    def messages(self):
        return Message

    @belongs_to_many(
        'followers',
        'followed_id', 'follower_id',
        with_timestamps=True
    )
    def followers(self):
        return User

    @belongs_to_many(
        'followers',
        'follower_id', 'followed_id',
        with_timestamps=True
    )
    def followed(self):
        return User
```

The `belongs_to_many` decorator accepts optional arguments and keyword arguments.

The first (`followers`) is the name of the table to use for the relation, we need to specify it here because it does not follow the Orator convention, which should be `messages_users` (derived from the alphabetical order of the related table names) . The second argument is the foreign key in the relation table (the column holding the id our user) and the third is the column holding the id of the related objects. The `with_timestamps` set to `True` means that we want to also retrieve the timestamps of the entries of our relation table `followers`.

You can have more information on it in the [documentation](https://orator-orm.com/docs/orm.html#many-to-many)

Now the routes to test it all out:

```python
@app.route('/users/<int:user_id>/following', methods=['GET'])
def get_user_followed(user_id):
    user = User.find_or_fail(user_id)

    return jsonify(user.followed)


@app.route('/users/<int:user_id>/followers', methods=['GET'])
def get_user_followers(user_id):
    user = User.find_or_fail(user_id)

    return jsonify(user.followers)
```

Lets' call the API:

```shell
$ http http://127.0.0.1:5000/users/1/following
```

```http
HTTP/1.0 200 OK
Content-Length: 333
Content-Type: application/json
Date: Thu, 13 Oct 2016 16:28:01 GMT
Server: Werkzeug/0.11.11 Python/3.5.2

[
    {
        "created_at": "2016-10-13T16:24:05+00:00",
        "email": "john@doe.com",
        "id": 2,
        "name": "john",
        "pivot": {
            "created_at": "2016-10-13T16:24:05+00:00",
            "followed_id": 2,
            "follower_id": 1,
            "updated_at": "2016-10-13T16:24:05+00:00"
        },
        "updated_at": "2016-10-13T16:24:05+00:00"
    }
]
```

```shell
$ http http://127.0.0.1:5000/users/1/followers
```

```http
HTTP/1.0 200 OK
Content-Length: 333
Content-Type: application/json
Date: Thu, 13 Oct 2016 16:28:30 GMT
Server: Werkzeug/0.11.11 Python/3.5.2

[
    {
        "created_at": "2016-10-13T16:24:05+00:00",
        "email": "jane@doe.com",
        "id": 3,
        "name": "jane",
        "pivot": {
            "created_at": "2016-10-13T16:24:05+00:00",
            "followed_id": 1,
            "follower_id": 3,
            "updated_at": "2016-10-13T16:24:05+00:00"
        },
        "updated_at": "2016-10-13T16:24:05+00:00"
    }
]
```

You can see here that a `pivot` attribute is present. Whenever a many-to-many relation is retrieved a `pivot` attribute is set that matches the database entry in the relation table (`followers` here). If you don't want it to appear in the response you can hide it by using the `__hidden__` property:

```python
class User(db.model):
    
    __hidden__ = ['pivot']
```

It would be great to be able to follow/unfollow users, so let's add some methods to the `User` to ease this: 

```python
def is_following(self, user):
    return self.followed().where('followed_id', user.id).exists()

def is_followed_by(self, user):
    return self.followers().where('follower_id', user.id).exists()

def follow(self, user):
    if not self.is_following(user):
        self.followed().attach(user)

def unfollow(self, user):
    if self.is_following(user):
        self.followed().detach(user)
```

And the corresponding routes:

```python
@app.route('/users/<int:user_id>/following/<int:followed_id>', methods=['PUT'])
def follow_user(user_id, followed_id):
    user = User.find_or_fail(user_id)
    followed = User.find_or_fail(followed_id)

    user.follow(followed)

    return app.response_class('No Content', 204)

@app.route('/users/<int:user_id>/following/<int:followed_id>', methods=['DELETE'])
def unfollow_user(user_id, followed_id):
    user = User.find_or_fail(user_id)
    followed = User.find_or_fail(followed_id)

    user.unfollow(followed)

    return app.response_class('No Content', 204)
```

We can now see if evrything works as expected:

```shell
$ http PUT http://127.0.0.1:5000/users/1/following/3
```

```http
HTTP/1.0 204 NO CONTENT
Content-Length: 0
Content-Type: text/html; charset=utf-8
Date: Thu, 13 Oct 2016 16:40:13 GMT
Server: Werkzeug/0.11.11 Python/3.5.2
```

```shell
$ http http://127.0.0.1:5000/users/1/following
```

```http
HTTP/1.0 200 OK
Content-Length: 330
Content-Type: application/json
Date: Thu, 13 Oct 2016 16:41:10 GMT
Server: Werkzeug/0.11.11 Python/3.5.2

[
    {
        "created_at": "2016-10-13T16:24:05+00:00",
        "email": "john@doe.com",
        "id": 2,
        "name": "john",
        "updated_at": "2016-10-13T16:24:05+00:00"
    },
    {
        "created_at": "2016-10-13T16:24:05+00:00",
        "email": "jane@doe.com",
        "id": 3,
        "name": "jane",
        "updated_at": "2016-10-13T16:24:05+00:00"
    }
]
```

```
$ http DELETE http://127.0.0.1:5000/users/1/following/3
```

```http
HTTP/1.0 204 NO CONTENT
Content-Length: 0
Content-Type: text/html; charset=utf-8
Date: Thu, 13 Oct 2016 16:41:37 GMT
Server: Werkzeug/0.11.11 Python/3.5.2
```

```
$ http http://127.0.0.1:5000/users/1/following
```

```http
HTTP/1.0 200 OK
Content-Length: 166
Content-Type: application/json
Date: Thu, 13 Oct 2016 16:42:08 GMT
Server: Werkzeug/0.11.11 Python/3.5.2

[
    {
        "created_at": "2016-10-13T16:24:05+00:00",
        "email": "john@doe.com",
        "id": 2,
        "name": "john",
        "updated_at": "2016-10-13T16:24:05+00:00"
    }
]
```


## Conclusion

These are the basics of building a (minimalistic) REST API with Flask and Orator.

I hope this has been useful and that it showcases the strength of Orator for such a use case.

I will probably make a follow up article later to introduce other features of Orator like query caching, polymorphic relations and the power of collections.

The complete code for this API is available here: [https://github.com/sdispater/twittor-api](https://github.com/sdispater/twittor-api)
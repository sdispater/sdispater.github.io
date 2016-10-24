---
layout: post
title:  "Please stop using Arrow"
date:   2016-10-21 17:20:38
category: programming
tags: [python, pendulum]
---

<aside class="note">

<p>Disclaimer: I am the author of <a href="https://pendulum.eustace.io">Pendulum</a> so this post might seem obviously biased. But the main reason I started it is that Arrow is so broken that I could no longer rely on it.</p>

</aside>



When Arrow appeared in the Python ecosystem like three years ago or so, it had the noble purpose of fixing the mess datetimes and timezones can be in Python. Here is its official description:

>  Arrow is a Python library that offers a sensible, human-friendly approach to creating, manipulating, formatting and converting dates, times, and timestamps.  It implements and updates the datetime type, plugging gaps in functionality, and provides an intelligent module API that supports many common creation scenarios.  Simply put, it helps you work with dates and times with fewer imports and a lot less code.
>
>  Arrow is heavily inspired by [moment.js](https://github.com/timrwood/moment) and [requests](https://github.com/kennethreitz/requests)



But noble principles do not always result in well executed code and API.



## Unintuitive API

The API of Arrow is all but intuitive. By taking inspiration in the famous [requests](https://github.com/kennethreitz/requests) library, which has a pretty well crafted API, it became the opposite. Basically, to create an Arrow object, you use the `get()` method which accepts a lot of different types and it will try to guess what you wanted to return an instance (sometimes erroneous). And if `get()` makes sense in the context of `requests` (after all you are executing a `GET` request) if does not make sense here.

Let's see some official examples:

```python
>>> arrow.get(1367900664)
# <Arrow [2013-05-07T04:24:24+00:00]>

>>> arrow.get('1367900664')
# <Arrow [2013-05-07T04:24:24+00:00]>

>>> arrow.get(1367900664.152325)
# <Arrow [2013-05-07T04:24:24.152325+00:00]>

>>> arrow.get('1367900664.152325')
# <Arrow [2013-05-07T04:24:24.152325+00:00]>

>>> arrow.get(datetime.utcnow())
# <Arrow [2013-05-07T04:24:24.152325+00:00]>

>>> arrow.get(datetime.now(), 'US/Pacific')
# <Arrow [2013-05-06T21:24:32.736373-07:00]>

>>> from dateutil import tz
>>> arrow.get(datetime.now(), tz.gettz('US/Pacific'))
# <Arrow [2013-05-06T21:24:41.129262-07:00]>

>>> arrow.get(datetime.now(tz.gettz('US/Pacific')))
# <Arrow [2013-05-06T21:24:49.552236-07:00]>

>>> arrow.get('2013-05-05 12:30:45', 'YYYY-MM-DD HH:mm:ss')
# <Arrow [2013-05-05T12:30:45+00:00]>

>>> arrow.get('2013-09-30T15:34:00.000-07:00')
# <Arrow [2013-09-30T15:34:00-07:00]>

>>> arrow.get(2013, 5, 5)
# <Arrow [2013-05-05T00:00:00+00:00]>
```

That's a lot. For newcomers it might seem practical, but it only encourages bad practices.

Another example of bad API design is the `replace()` method. The native method basically replaces some or all elements of a `datetime` object but with `Arrow` it's been overridden to also shift time. Singular units replaces them while plural units shift time.

```python
>>> arw = arrow.utcnow()
>>> arw
# <Arrow [2013-05-12T03:29:35.334214+00:00]>

>>> arw.replace(hour=4, minute=40)
# <Arrow [2013-05-12T04:40:35.334214+00:00]>

>>> arw.replace(weeks=+3)
# <Arrow [2013-06-02T03:29:35.334214+00:00]>
```

Its confusing at best and prone to errors.



## Bugs and strange behavior

But even more than bad API designs, Arrow is buggy and can behave unexpectedly.

Let's see some examples.

<aside class="note">

<p>These examples have been taken directly from the issues reported on the github project.</p>

</aside>

### Parsing

```python
>>> arrow.get('2016-1-17')
# <Arrow [2016-01-01T00:00:00+00:00]>

# Parsing of a date with wrong day
>>> arrow.get('2015-06-31')
# <Arrow [2015-06-01T00:00:00+00:00]>
```

### Instantiation

```python
# Wrong offset is set on instantation
>>> arrow.Arrow(1970, 1, 1, tzinfo=pytz.timezone('Europe/Paris'))
# <Arrow [1970-01-01T00:00:00+00:09]>

>>> arrow.Arrow.fromtimestamp(0, pytz.timezone('Europe/Paris'))
# <Arrow [1970-01-01T01:00:00+00:09]>
```

### Timezones

Timezones and DST transitions handling is where `arrow` is particularly broken.

```python
# Working with DST
>>> just_before = arrow.Arrow(2013, 3, 31, 1, 59, 59, 999999, 'Europe/Paris')
>>> just_after = just_before.replace(microseconds=1)
'2013-03-31T02:00:00+02:00'
# Should be 2013-03-31T03:00:00+02:00

>>> (just_after.to('utc') - just_before.to('utc')).total_seconds()
-3599.999999
# Should be 1e-06
```

```python
in_utc = arrow.get('2016-10-30 00:00')
# <Arrow [2016-10-30T00:00:00+00:00]>
in_amsterdam = in_utc.to('Europe/Amsterdam')
# <Arrow [2016-10-30T02:00:00+01:00]>
# Should be 2016-10-30T02:00:00+02:00
```

With `pendulum` these cases are handled correctly

```python
>>> pendulum.parse('2016-1-17')
# <Pendulum [2016-01-17T00:00:00+00:00]>

>>> pendulum.parse('2015-06-31')
# ValueError: day is out of range for month

>>> pendulum.Pendulum(1970, 1, 1, tzinfo=pytz.timezone('Europe/Paris'))
# <Pendulum [1970-01-01T00:00:00+01:00]>

>>> pendulum.fromtimestamp(0, pytz.timezone('Europe/Paris'))
# <Pendulum [1970-01-01T01:00:00+01:00]>

>>> just_before = pendulum.create(2013, 3, 31, 1, 59, 59, 999999, 'Europe/Paris')
>>> just_after = just_before.add(microseconds=1)
# <Pendulum [2013-03-31T03:00:00+02:00]>

>>> (just_after.in_timezone('utc') - just_before.in_timezone('utc')).total_seconds()
# 1e-06

>>> in_utc = pendulum.parse('2016-10-30 00:00')
# <Pendulum [2016-10-30T00:00:00+00:00]>
>>> in_amsterdam = in_utc.in_tz('Europe/Amsterdam')
# <Pendulum [2016-10-30T02:00:00+02:00]>
```

 

## Arrow seems no longer maintained

As of the writing of this post, it's been four months since the last commit on the [github project](https://github.com/crsmithdev/arrow).

Issues are piling up, some critical, and pull requests are left without responses or comments.

So, it's safe to say that it is pretty much abandoned at this point.



## Conclusion

Stop using Arrow.

There are alternatives to it, maintained and more complete, that you can use.

I want `pendulum` to be the most intuitive, complete and accurate library for Python but there are others you can choose.

But choose wisely.
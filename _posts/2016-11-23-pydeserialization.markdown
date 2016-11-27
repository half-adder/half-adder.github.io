---
layout: post
title:  "Deserializing nested dictionaries into complex, typed(!) python objects"
---

Background
----------
Serialization (also known as marshalling) and its inverse is a common task for many programmers. For the [unfamiliar](https://imgs.xkcd.com/comics/ten_thousand.png), serialization is, for the purposes of this post, the process of taking some object in memory, and creating a _serial_ representation of it in some standard format for the purposes of transferring the object to some other process, where the opposite (serial => memory) will take place. Some popular serialization formats include JSON, Google's Protocol Buffers, YAML, XML, Python's Pickle, and [many others](https://en.wikipedia.org/wiki/Category:Data_serialization_formats).

In the above list, there can be found two broad categories of serialization formats: human-readable (JSON, YAML, XML), and machine-readable (Protocol Buffers, Pickle). Machine-readable formats are great if you want _really fast_ de/serialization. They're pretty terrible for humans to read, though. For example, take a look at how Pickle serializes a simple dictionary:

{% highlight python %}
>>> import pickle
>>> favorite_color = {"lion": "yellow", "kitty": "red"}
>>> pickle.dumps(favorite_color)
"(dp0\nS'lion'\np1\nS'yellow'\np2\nsS'kitty'\np3\nS'red'\np4\ns."
{% endhighlight %}

While it's possible to _kind_ of make out what's going on, it's clear to see that once the objects start to become more complicated, and have multiple nested properties, a human reader would quickly be overwhelmed.

Human-readable serialization formats, on the other hand, are expectedly quite nice for humans to read. In JSON, favorite_color would look like this:
{% highlight json %}
{
    "lion": "yellow",
    "kitty": "red"
}
{% endhighlight %}

YAML is even easier for humans to read:

{% highlight yaml %}
lion: yellow
kitty: red
{% endhighlight %}

JSON and YAML are essentially textual representations of arbitrarily deeply-nested trees. Python represents such trees as `dicts`. The `json` module is included in Python's standard library, and `py-yaml` is easily installed with pip. These modules expose simple APIs that suck in some valid JSON/YAML and spit out a sweet sweet `dict`. So for this post, we will work with in-memory `dict`s.

With that, let's explore a technique to easily define complex Python objects which may be deserialized from nested dictionaries. 

Problem
-------
To set the stage, here is the problem we would like to solve. Let's say we have the following complex nested dictionary (as expressed in YAML for ease-of-reading):

{% highlight yaml %}
name: Tyrion
house: 
    name: Lannister
    age: 700
    colors: [Red, Gold]
    words: Hear Me Roar!
    seat: Casterly Rock
age: 15
sibling_names: [Jaime, Joffrey, Cersei]
{% endhighlight %}

We would like to define a set of Python objects to reprsent this nested structure. We would like to be able to access the various attributes naturally as properties, as such:

{% highlight python %}
# intialize the above object into a variable called tyrion...
>>> tyrion.name
"Tyrion"
>>> tyrion.house
<House 'Lannister'>
>>> tyrion.house.age
700
>>> tyrion.age
15
>>> tyrion.house.colors
["Red", "Gold"]
{% endhighlight %}

We would also like for it to be *easy* to intialize the object, given a dictionary, and we would like that initialization to be *typed*. That is to say, if we are given a `str` instead of an `int` for `tyrion`'s age, we would like an Exception to be raised. 

We should also be able to define methods on each of these nested objects. Perhaps one such example might be

{% highlight python %}
>>> tyrion.drink(liters=3, alcohol='wine')
{% endhighlight %}

Naive Implementation
--------------------
We can start with a naive way to just represent the classes we want.

{% highlight python %}

class House(object):

    def __init__(self, name, age, colors, words, seat):
        self.check_args(name, age, colors, words, seat)
        self.name = name
        self.age = age
        self.colors = colors
        self.words = words
        self.seat = seat

    def check_args(self, name, age, colors, words, seat):
        assert type(name) is str
        assert type(age) is int
        assert type(colors) is list
        assert all(type(c) is str for c in colors)
        assert type(words) is str
        assert type(seat) is str

    def fly_banner(self):
        print('flying banner')


class GoTPerson(object):

    def __init__(self, name, house, age, sibling_names):
        self.check_args(name, house, age, sibling_names)
        self.name = name
        self.house = house
        self.age = age
        self.sibling_names = sibling_names

    def check_args(self, name, house, age, sibling_names):
        assert type(name) is str
        assert type(house) is House
        assert type(age) is int
        assert type(sibling_names) is list
        assert all(type(s) is str for s in sibling_names)

    def drink(liters, alcohol):
        print('drinking %d liters of %s' % (liters, alcohol,))
{% endhighlight %}

Now that we've got a class structure, how do we get from a dictionary to a `GoTPerson`? We would like to write as little additional code as possible, because as you can see, the type checks already added a *lot* of overhead!

It'd be nice to use something like

{% highlight python %}
>>> a_dict
{
    'name': 'Tyrion',
    'house': {
        'name': 'Lannister',
        ...
    },
    ...
}
>>> tyrion = GoTPersion(**a_dict)
Traceback (most recent call last):
  ...
  File "test.py", line 26, in __init__
    self.check_args(name, house, age, sibling_names)
  File "test.py", line 34, in check_args
    assert type(house) is House
AssertionError
{% endhighlight %}

Why doesn't this work? We are assigning a the `dict` house to `tyrion`'s house attribute! We need to recursively create a `House` from that dict (and create any necessary objects from dicts if House has nested objects)...

So we are forced to make something like this for each of our classes...


{% highlight python %}
class GoTPerson(object):
    
    ...

    @classmethod
    def from_dict(cls, a_dict):
        name = a_dict['name'] 
        house = House.from_dict(a_dict['house'])
        age = a_dict['age']
        sibling_names = a_dict['sibling_names']

        return cls(name, house, age, sibling_names)
 
    ...
{% endhighlight %}

This works... But it's pretty ugly, and really verbose! There must be a better way.

Awesome Pythonic Implementation of Greatness
--------------------------------------------
There is a better way! The scaffolding of this implementation is *heavily* inspired from [The Python Cookbook](www.google.com), a fabulous resource for getting a large variety stuff done in a really nice way.

First, we define a base class.

{% highlight python %}
class Structure(object):

    _fields = []

    def _init_arg(self, expected_type, value):
        if isinstance(value, expected_type):
            return value
        else:
            return expected_type(**value)

    def __init__(self, **kwargs):
        field_names, field_types = zip(*self._fields)
        assert([isinstance(name, str) for name in field_names])
        assert([isinstance(type_, type) for type_ in field_types])

        for name, field_type in self._fields:
            setattr(self, name, self._init_arg(field_type, kwargs.pop(name)))

        # Check for any remaining unknown arguments
        if kwargs:
            raise TypeError('Invalid arguments(s): {}'.format(','.join(kwargs)))
{% endhighlight %}

Now, let's make `House` and `GoTPerson` subclass `Structure`.

{% highlight python %}
class House(Structure):

    _fields = [('house', str), ('age', int), ('colors', list), ('words', str), ('seat', str)]

    def fly_banner(self):
        print('flying banner')


class Tyrion(Structure):

    _fields = [('name', str), ('house', House), ('age', int), ('sibling_names', list)]

    def drink(liters, alcohol):
        print('drinking %d liters of %s' % (liters, alcohol,))
{% endhighlight %}

Awesome! We can now initialize a GoTPerson as such

{% highlight python %}
>>> a_dict
{
    'name': 'Tyrion',
    'house': {
        'name': 'Lannister',
        ...
    },
    ...
}
>>> tyrion = GoTPersion(**a_dict)
{% endhighlight %}

Our `Structure` class takes care of type checking, as well as recursively initializing any nested objects from dictionaries. This technique has massively simplified the process of deserializing JSON into Python objects in my own code, and I hope it does the same for yours!

There is one final challenge you might want to undertake. Try to cleanly establish type checks on the items of lists, which are not present in our current implementation.

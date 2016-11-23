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

Implementation 1
----------------
We can start with a naive way to accomplish what we want.

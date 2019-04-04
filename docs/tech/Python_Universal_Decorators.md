Universal Decorators
====================

Decorators a very useful tool in Python, but they are hard to implement in ways that don't have
some nasty gotchas. Overall I think that this is a spot in Python that could use a little rewrite
but that is a topic for another day. This article is documentation of one way through this that
tries to avoid almost all the potholes.

Edit 4/4/2019: I just stumbled across
[another article](https://pybit.es/decorator-optional-argument.html) that references the "Python 
cookbook 3rd edition" as having another solution similar to mine (search for "Allow for optional 
arguments" on that page).

Ways of using decorators
------------------------

Decorators can be invoked three different ways (please note the style identifiers):
```python
@my_decorator  # Style 1: without parentheses
def a():
    pass

@my_decorator()  # Style 2: with parentheses, but without any values
def b():
    pass

@my_decorator(bob=1)  # Style 3: with parentheses and some values
def c():
    pass
```

And they can be written in two forms: [Class](https://www.python.org/dev/peps/pep-3129/) style
and [Function](https://www.python.org/dev/peps/pep-0318/) style. When it comes to applying them
to functions, both work on all three styles above. However, for `methods` (functions attached to
Object instances), the first style does not work at all. For a discussion on that see [my page on
the problem](Python_Object_Decorators.md).

But it is possible to make a decorator that works on all three of these styles, and works on
`methods` as well as `functions`, but it takes a some thinking. Rather than go through all that
work, you an just start with my version:

```python
import functools


def my_decorator(*args, **kwargs):
    func = None
    if len(args) == 1 and callable(args[0]) and not kwargs:
        func = args[0]
        args = tuple()

    def outer(func):
        # note: `args` and `kwargs` are available here
        @functools.wraps(func)
        def inner(*inner_args, **inner_kwargs):
            # note: `args` and `kwargs` are available here too
            func(*inner_args, **inner_kwargs)
        return inner

    if func:
        return outer(func)
    else:
        return outer
```

Limitations
-----------

The only limitation I am aware of is that you can't call call the decorator with a single
`callable`. So you can't do this:
```python
@my_decorator(lambda x: x)
def my_function(a):
    pass
```

I could not figure out a way of seperating that from the no-parentheses case. However, you
could use a throw-away second value, or keyword to get around this, so:
```python
@my_decorator(lambda x: x, waste=None)
def my_function(a):
    pass
```

Annoying, but probably not a limitation for most uses.

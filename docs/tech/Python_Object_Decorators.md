Python Class-based decorators don't work on classes
===================================================

There are two different ways of creating decorators: Classes or functions. Unfortunately, the 
Classes form does not work when you are trying to use it on Object methods (on functions it is
fine). An example:

```python
import inspect

class MyDecorator(object):
    func = None
    def __init__(self, *args, **kwargs):
        print('Init args: %r kwargs: %r' % (args, kwargs))
        if args and callable(args[0]):
            print('  Is method: %s' % inspect.ismethod(args[0]))
            self.func = args[0]

    def __call__(self, *args, **kwargs):
        print('Call func: %r args: %r kwargs: %r' % (self.func, args, kwargs))
        if self.func:
            return self.func(*args, **kwargs)  # no access to the proper `self`
        else:
            return args[0]  # note: return a wrapper func if you need more

class MyClass(object):
    color = 'Not set'

    def __init__(self, color):
        self.color = color

    @MyDecorator  # without parentheses
    def green(self):
        print('Green worked: %s' % self.color)

    @MyDecorator()  # with parentheses, but without any value(s)
    def red(self):
        print('Red worked: %s' % self.color)

    @MyDecorator('a')  # with parentheses and value(s)
    def blue(self):
        print('Blue worked: %s' % self.color)
    
    print('---- Parsing complete ----')


a = MyClass('orange')
try:
    a.green()
except Exception as e:
    print('Green failed: %s' % e)  # this gets called here
try:
    a.red()
except Exception as e:
    print('Red failed: %s' % e)
try:
    a.blue()
except Exception as e:
    print('Blue failed: %s' % e)
```

The output:
```
Init args: (<function green at 0x7f8db2957230>,) kwargs: {}
  Is method: False
Init args: () kwargs: {}
Call func: None args: (<function red at 0x7f8db29572a8>,) kwargs: {}
Init args: ('a',) kwargs: {}
Call func: None args: (<function blue at 0x7f8db2957320>,) kwargs: {}
---- Parsing complete ----
Call func: <function green at 0x7f8db2957230> args: () kwargs: {}
Green failed: green() takes exactly 1 argument (0 given)
Red worked: orange
Blue worked: orange
```

This is a bit much to digest all at once, but a point summary of what is going on, first for `red`
and `blue`, which work fine:
- First, during parse time, the `__init__` method on the decorator is called with the information
  from the decorator line, but with no information on the method being called. Nothing is returned.
- Immediately afterwards (still during parse time) the `__call__` method is invoked. This time
  with no arguments, and is expected to return a callable.
- This callable is invoked in runtime, with `self` as the only argument. Life is good.

For the `green` path (without parentheses) things are very different:
- `__init__` is immediately (during parse time) called with only the method to work with. As this
  is during parse time, there is no `self` for us to store.
- At runtime the `__call__` method is invoked with no args provided. So there is no `self`
  available, so we can't get access to any instance variables (`color` in this case).

Summary
-------

This is only going to affect you if you use a) Class-style decorators, b) on Object methods, c)
without using parentheses. But it is still an annoying problem. I am seeing this on Python 2.7.6,
and 3.4.3.

All of this seems overly-complicated, and non-Pythonic to me. But if you want decorators that work
in all of these cases, use my [universal decorator example](Python_Universal_Decorators.html) as a
starting point.

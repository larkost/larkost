While figuring some code at work I was a bit surprised at a detail of multiple inheritance in Python. I was surprised that when using the `super` method that all superclasses wind up in the chain, not just the primary (first) parents. This is best shown in an example:

```python
#!/usr/bin/env python

class Base(object):
    def __init__(self):
        print('Class init: Base')
        super(Base, self).__init__()

class Alpha1(Base):
    def __init__(self):
        print('Class init: Alpha1')
        super(Alpha1, self).__init__()

class Alpha2(Alpha1):
    def __init__(self):
        print('Class init: Alpha2')
        super(Alpha2, self).__init__()

class Beta1(Base):
    def __init__(self):
        print('Class init: Beta1')
        super(Beta1, self).__init__()

class Beta2(Beta1):
    def __init__(self):
        print('Class init: Beta2')
        super(Beta2, self).__init__()

class Omega(Alpha2, Beta2):
    def __init__(self):
        print('Class init: Omega')
        super(Omega, self).__init__()

Omega()
```

I would have assumed that this would print:
```
Class init: Omega
Class init: Alpha2
Class init: Alpha1
Class init: Base
```
But the actual output is:
```
Class init: Omega
Class init: Alpha2
Class init: Alpha1
Class init: Beta2
Class init: Beta1
Class init: Base
```

The difference is the `Beta2` and `Beta1` entries just before the `Base` entry. I had not expected those to be be included in this chain because in my mind the `Alpha` chain had "priority" (my own concept). 

In retrospect Python is just walking down the [`mro` (method resolution order)](https://www.python.org/download/releases/2.3/mro/) chain. When thinking about it that way it makes perfect sense, but it is not what I had intuitively expected.

When designing class hierarchies I am going to have to keep this in mind. Usually this is not an issue since in normal cases multiple inheritance is only used for "mixin" purposes, and the additional chains don't have `__init__` methods (the most likely one to be chained with `super`). But it was an interesting find to me.

Preferences Class
=========================

I had a need for an easy to use class to hold preferences. Specifically I needed:
- A fixed structure with multiple levels including `list`, `set`, and `dict` items
- In the case of `dict` items I need to be able to enforce both the keys, and the types on the values
- Optional default values on `dict`s keys

I also had a list of things I wanted:
- Easy printing using `pprint` to make debugging easy
- The ability to create quickly from the object structures produced by YAML or JSON parsers
- Be able to access/create items using both the `a.b` and `a['b']` notations
- Auto-create intermediate levels when accessing items deeper in
- Do this all without a lot of boilerplate on the individual classes
- Work like normal `dict`s for iterating and things like `keys()`
- Ability to use `__slots__` to improve space useage and speed
- Default all unset (and undefaulted) values to `None`

The example here provides all of those. There were some thorny problems to overcome:
- `pprint` explicitly checks for `dict`, so you have to be a subclass to get recursive printing
- `dict` actually impliments most of its functions in C, so you have to override everything
- `collections.MutableMapping` provides some of this, but does not understand slots
- `__slots__` are great, but don't play well with universal ways of listing things

Note: At present this is not compatible with Python 3, but that may be doable.

Examples
--------

These examples all asume you have `AttrDict` in scope (see [Source] section).

```python
import pprint


class Alpha(AttrDict):
    _allowed_values = {'__all__': int}  # the special key `__all__` will apply to any undefined key
    green = 'something else'


class Beta(AttrDict):
    __slots__ = ['a', 'b']  # slots work to help save space, but are not required
    _allowed_values = {'red': str, 'orange': Alpha}


a = Alpha(red=5)
a.red  # result: 5
a['green']  # result: "something else"

b = Beta(red='one')
b.orange.green  # result: "something else"
```

Source
------

```python
import collections
import inspect
import types


class AttrDict(collections.MutableMapping, dict):
    _allowed_values = {}
    __keys = None

    def __init__(self, **kwargs):
        '''Use the object dict'''
        self.__keys = {x[0] for x in inspect.getmembers(self) if not x[0].startswith('_') and not callable(x[1])}
        if self._allowed_values:
            self.__keys.update(self._allowed_values.keys())
        if hasattr(self, '__slots__'):
            self.__keys.update(self.__slots__)
        for key, value in kwargs.items():
            self[key] = value

    def __setitem__(self, key, value):
        if key in self._allowed_values:
            expected_type = self._allowed_values[key]
        elif hasattr(self, key) or not self._allowed_values:
            expected_type = '__any__'
        else:
            raise AttributeError("%s has no attribute '%s'" % (self.__class__.__name__, key))

        if expected_type != '__any__' and not isinstance(value, expected_type):
            if isinstance(value, dict) and issubclass(expected_type, AttrDict):
                value = expected_type(value)  # ToDo: handle unions of types
            else:
                raise ValueError("%s requires the value to be %s, got: %r (%s)" % (
                    key, expected_type, value, type(value)))

        object.__setattr__(self, key, value)
        if not str(key).startswith('_'):
            self.__keys.add(key)

    __setattr__ = __setitem__

    def __getitem__(self, key):
        if hasattr(self, key):
            try:
                return object.__getattribute__(self, key)
            except AttributeError:
                return None
        elif key in self._allowed_values:
            allowed_types = self._allowed_values[key]
            if not isinstance(allowed_types, (list, set, tuple)):
                allowed_types = (allowed_types,)
            for allowed_type in allowed_types:
                if issubclass(allowed_type, (dict, list, set)):
                    # if we have a modifable container defined for this key, auto-create it
                    self.__setattr__(key, allowed_type())
                    return object.__getattribute__(self, key)
            else:
                return None
        else:
            raise AttributeError("%s object has no attribute `%s`" % (self.__class__.__name__, key))

    __getattr__ = __getitem__

    def __delitem__(self, key):
        raise NotImplementedError('This class does not support deleting items')

    def __iter__(self):
        return (x for x in self.__keys)

    def __len__(self):
        return len(self.__keys)
```
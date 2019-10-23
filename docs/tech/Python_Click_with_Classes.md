Click commands on members of a Class
====================================

A recent project involved adding a CLI to an existing project. Unusually this project
has its natural entry points living as member functions on Classes, including a lot of
them that are inherited from one of a couple of base classes. Notably these were regular
methods, not classmethods, so you have to have an instance on hand to run them on. Luckily
these instances were easy to create, and the `__init__` methods require no arguments.

I decided to use the [Click](https://click.palletsprojects.com) module both because it matched
some of my requirements, and it was the library used in other projects in the codebase I am
working in.

So the requirements:
- Apply the cli with as small a change to the existing code as possible
- The project has the concept of `command`/`subcommand` and arguments, which are mapped to
  Python's `Class`es and methods on them
- Have the CLI configuration (e.g. commands/subcommands/argument parsing) be close to the code 
  that handles it (this drives the choice of `click`)
- Minimise the amount of work (and internals knowledge) that other developers need to do to
  get new commands working
- Handle subclassing in a reasonable way
- Everything needs to work on Python3.6+

Oct. 23, 2019 update: I found that there were problems with multiple `click.argument`s, and
fixed that.

Problems encountered
--------------------

There were a number of hurdles that I did not see coming along the way:
1. `Click` is designed to be wrapped around functions, and does not have a natural way of
   instantiating an object to then call the methods on.
2. `@click.command()` is the natural way of decorating entry points, and works on methods
   (minus problem #1), but when you inherit it is points at the method referencing only the
   inherited class making it hard to divine the final class at runtime.
3. If you use the `@click.command` decorator without parens it creates a function/closure
   style wrapper that is hard to see though.


Solution
--------

My solution:
1. A `metaclass` which uses `__new__` to modify sublasses as those classes are done being
   parsed. Specifically it looks for `click.Command` objects on the new subclass and wrapps
   them with a class with a `__call__` method that instantiates the correct class before
   calling the `callback` on that command.
2. This `metaclass` also collects all of these `click.Command` objects in a `click.Group`
   object attached to the class as `click_group`. This allows the `__main__` to group all of
   these into another `click.Group` which is then run to parse arguments and run the entry
   point.
3. In finding the `class.Command` instances the `metaclass` has to see if they have already
   has their `callback` wrapped in my instantiaor. This happens on subclasses that inherit
   their methods from a class that has already been wrapped. In these cases a copy of the
   whole command needs to be made, and the class to be instantiated swapped in.
4. I am also grooming subclasses to see if the `@click.command` decorator was used, and since
   those can not really be inspected without silly measures (looking at source code), the code
   instead posts a warning that hopefully will prompt developers to change their code.

All of this work is confined to 3 classes: the Instantiator, the Metaclass, and a technically
unnecessary class to inherit from the Metaclass to make other subclasses easy (this can be used
to put other inherited methods on as well).

The Code
--------

```python
import click
import copy
import inspect
import warnings


class ClickInstantiator:
    klass = None
    command = None

    def __init__(self, command, klass):
        self.command = command
        self.klass = klass

    def __call__(self, *args, **kwargs):
        return self.command(self.klass(), *args, **kwargs)


class ClickCommandMetaclass(type):
    def __new__(mcs, name, bases, dct):
        klass = super().__new__(mcs, name, bases, dct)

        # create and populate the click.Group for this Class
        klass.click_group = click.Group(name=klass.__name__.lower())

        # warn about @click.command decorators missing the parens
        for name, command in inspect.getmembers(klass, inspect.isfunction):
            if repr(command).startswith('<function command.'):
                warnings.warn(
                    '%s.%s is wrapped with click.command without parens, please add them' % (klass.__name__, name))

                for name, command in inspect.getmembers(klass, lambda x: isinstance(x, click.Command)):
            if name == 'click_group':
                continue

            def find_final_command(target):
                """Find the last call command at the end of a stack of click.Command instances"""
                while isinstance(target.callback, click.Command):
                    target = target.callback
                return target

            command_target = find_final_command(command)

            if not isinstance(command_target.callback, ClickInstantiator):
                # the top class to implement this
                command_target.callback = ClickInstantiator(command_target.callback, klass)
            else:
                # this is a subclass function, copy it and replace the klass
                setattr(klass, name, copy.deepcopy(command))
                command = getattr(klass, name)
                find_final_command(getattr(klass, name)).callback.klass = klass

            # now add it to the group
            klass.click_group.add_command(command, name)
        return klass


class ClickCommandBase(metaclass=ClickCommandMetaclass):
    pass


# == Example code

class Alpha(ClickCommandBase):
    @click.command()
    def red(self) -> None:
        """Lets see if this works"""
        print('This works on %s!' % self.__class__.__name__)

class Beta(ClickCommandBase):
    @click.command()
    def red(self) -> None:
        """Do something blue"""
        print('Beta works as well!')

class Gamma(Alpha):
    pass

if __name__ == '__main__':
    cli = click.Group()
    for group in (x.click_group for x in {Alpha, Beta, Gamma} if hasattr(x, 'click_group')):
        cli.add_command(group, name=group.name)
    cli()
```

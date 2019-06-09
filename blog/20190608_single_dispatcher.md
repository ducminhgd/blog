# Single dispatcher

You can find a _same-same_ [Vietnamese version on my blog](https://ducminhgd.wordpress.com/2019/06/08/single-dispatcher-trong-python/)

My team is developing a project, which has many events, we try something new (to us) to handle event in the simplest way for developers. We aim for group handlers into a module or a class, and events are value objects only.

Consider we have some event classes and a handler class like this

```python
class EventA:
    ...


class EventB:
    ...


class EventC:
    ...


class EventHandler:

    def apply(self, event):
        ...
```


First opion was create a dictionary as handler registry

```python
HANDLER_REGISTRY = {
    'EventA': 'handle_a',
    'EventB': 'handle_b',
    'EventC': 'handle_c',
}
```

Then the handler should be

```python
class EventHandler:

    def handle_a(self, event):
        ...
    
    def handle_b(self, event):
        ...
    
    def handle_b(self, event):
        ...

    def apply(self, event):
        cls_name = event.__class__.__name__
        handler_name = HANDLER_REGISTRY[cls_name]
        handler_method = getattr(self, handler_name)


# Usage
a = EventA()
b = EventB()

eh = EventHandler()
eh.appy(a)
eh.appy(b)
```

_But, a teammate suggested my to use **Single Dispatcher** to handle events_

So that, source code should be

```python
import functools


def method_dispatch(func):
    """Support single dispatch of a class's method
    """
    dispatcher = functools.singledispatch(func)
    def wrapper(*args, **kwargs):
        return dispatcher.dispatch(args[1].__class__)(*args, **kwargs)

    wrapper.register = dispatcher.register
    functools.update_wrapper(wrapper, func)
    return wrapper

class EventHandler:

    @method_dispatch
    def apply(self, event):
        print(f'Not implemented or not registered for {event.__class__.__name__}')

    @apply.register(EventA)
    def _(self, event: EventA):
        print(f'Hope this is EventA and result is {event.__class__.__name__}')

    @apply.register(EventB)
    def _(self, event: EventB):
        print(f'Hope this is EventB and result is {event.__class__.__name__}')

# Usage
a = EventA()
b = EventB()
c = EventC()
eh = EventHandler()

eh.apply(a) # Hope this is EventA and result is EventA
eh.apply(b) # Hope this is EventB and result is EventB
eh.apply(c) # Not implemented or not registered for EventC
```

**From docstring of `functools.singledispatch(func)`**

> Single-dispatch generic function decorator.
> Transforms a function into a generic function, which can have different
> behaviours depending upon the type of its first argument. The decorated
> function acts as the default implementation, and additional
> implementations can be registered using the register() attribute of the
> generic function.

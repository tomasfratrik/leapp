# How to deprecate entities in leapp?

When you want to deprecate an entity in leapp projects, use the `deprecated`
decorator from `leapp.utils.deprecation` above the definition of the entity.
The decorator has three input parameters:

- `since` (mandatory) - specifying the start of the deprecation protection period
- `message` (mandatory) - explaining that particular deprecation
  (e.g. in case the deprecated functionality has a replacement, it is expected
  it will be mentioned in the msg.)
- `stack_level_offset` (optional) - useful to adjust the position of the reported
  usage in the deprecation message; e.g. in case of a base class or derived classes

**Warning: possible change:** *It's possible the `stack_level_offset` parameter
will be removed (or ignored) in future, if we discover a way to improve
the deprecation of derived classes.*

In case of a class deprecation, all derived classes are considered to be deprecated
as well. However, the current reporting could be a little bit confusing. To
improve that, the `stack_level_offset` option can be specified.
See [examples of the use of the @deprecated decorator for classes](deprecation.md#classes).

When you mark any entity as deprecated and this entity is then used
in the code, users will be notified about that via a terminal and report
messages (see the previous section). However, as the author of the deprecation,
you know that the entity is deprecated and you do not want
to notify people about the code that still uses the deprecated entity
just for the reason to retain the original functionality. To suppress
the deprecation messages in such cases, use the `suppress_deprecation`
decorator taking as arguments objects that should not be reported
by the deprecation. E.g. in case you use it above the definition
of an actor, any use of the deprecated entity inside the actor
will not be reported.

WARNING: It is strictly forbidden to use the `suppress_deprecation` decorator
for any purposes but one - retaining the original official functionality over
the deprecation protection period. If you see the message and you are not
the *provider* of the functionality, you have to update your code
to be independent on it.

## Examples of a model deprecation

Imagine we want to deprecate a *Foo* model that is produced in an actor called
*FooProducer* and consumed in an actor called *FooConsumer*. Let's keep this
example simple and say that we do not want to set any replacement of this
model. The first thing we have to do is to set the model definition as deprecated:

```python
from leapp.models import Model, fields
from leapp.topics import SomeTopic
from leapp.utils.deprecation import deprecated


@deprecated(since='2020-06-20', message='This model has been deprecated.')
class Foo(Model):
    topic = SomeTopic
    value = fields.String()
```

If we do only this and execute actors that produce/consume messages of this model,
we will obtain messages like these (just example from the actor
producing the message after execution by snactor):

```bash
# snactor run fooproducer
============================================================
                 USE OF DEPRECATED ENTITIES
============================================================

Usage of deprecated Model "Foo" @ /path/to/repo/actors/fooproducer/actor.py:17
Near:         self.produce(Foo(value='Answer is: 42'))

Reason: This model has been deprecated.
------------------------------------------------------------

============================================================
             END OF USE OF DEPRECATED ENTITIES
============================================================
```

Apparently, the `Reason` is not so good. It's just example. In real world
example, you would like to provide usually a little bit better explanation.
Anyway, much more interesting is the point, that the message is now printed
every time the actor is executed.

Obviously we do not want to remove the actor yet, because in such a case, the
model could be hardly called as deprecated - we need to keep the same
functionality during the protection period. But at the same time, we do not
want the deprecation message to be produced in this case, as it would be kind
of a spam for users who don't care about that model at all.

The warning messages are focused on a "custom" use of the deprecated
model - i.e. when a developer creates their own actor producing/consuming a message
of the model. To fix this, suppress the deprecation message in this actor.

To do it, the only thing that has to be done is to set the
`suppress_deprecation` decorator with the `Foo` as an argument (in this case)
before the actor, e.g.:

```python
from leapp.actors import Actor
from leapp.models import Foo  # deprecated model
from leapp.tags import IPUWorkflowTag, FactsPhaseTag
from leapp.utils.deprecation import suppress_deprecation


@suppress_deprecation(Foo)
class FooProducer(Actor):
    """
    Just produce the right answer to the world.
    """

    name = 'foo_producer'
    consumes = ()
    produces = (Foo,)
    tags = (IPUWorkflowTag, FactsPhaseTag)

    def process(self):
        self.produce(Foo(value='Answer is: 42'))
```

This is the most simple case. Let's do a small change and produce the message
inside the private actor's library instead. The library looks like this:

```python
from leapp.models import Foo  # deprecated model
from leapp.libraries.stdlib import api

def produce_answer():
    api.produce(Foo(value='Answer is: 42'))
```

And the updated actor looks like this:

```python
from leapp.actors import Actor
from leapp.libraries.actor import fooproducer_lib
from leapp.models import Foo  # deprecated model
from leapp.tags import IPUWorkflowTag, FactsPhaseTag
from leapp.utils.deprecation import suppress_deprecation


@suppress_deprecation(Foo)
class FooProducer(Actor):
    """
    Just produce the right answer to the world.
    """

    name = 'foo_producer'
    consumes = ()
    produces = (Foo,)
    tags = (IPUWorkflowTag, FactsPhaseTag)

    def process(self):
        fooproducer_lib.produce_answer()
```

Now, if you execute the actor again you still won't get any
deprecation message. So the `suppress_deprecation` decorator works transitively
as expected. However, even when the actor is treated well, the current
implementation could affect the result of unit tests. To explain the idea of what
could be wrong, imagine a unit test like this one:

```python
from leapp.libraries.actor import fooproducer_lib
from leapp.libraries.common.testutils import produce_mocked
from leapp.libraries.stdlib import api
from leapp.models import Foo  # deprecated model

def test_process(monkeypatch):
    produced_msgs = produce_mocked()
    monkeypatch.setattr(api, 'produce', produced_msgs)
    fooproducer_lib.produce_answer()
    assert Foo(value='Answer is: 42') in produced_msgs.model_instance
```

If you run the test, you will get output like this (shortened):

```python
| 21:48:01 | conftest | INFO | conftest.py | Actor 'foo_producer' context teardown complete

repos/system_upgrade/el7toel8/actors/fooproducer/tests/test_unit_fooproducer.py::test_process PASSED

================================================== warnings summary ==================================================
repos/system_upgrade/el7toel8/actors/fooproducer/tests/test_unit_fooproducer.py::test_process
  /tmp/leapp-repository/repos/system_upgrade/el7toel8/actors/fooproducer/libraries/fooproducer_lib.py:5: _DeprecationWarningContext: Usage of deprecated Model "Foo"
    api.produce(Foo(value='Answer is: 42'))
  /tmp/leapp-repository/repos/system_upgrade/el7toel8/actors/fooproducer/tests/test_unit_fooproducer.py:10: _DeprecationWarningContext: Usage of deprecated Model "Foo"
    assert Foo(value='Answer is: 42') in produced_msgs.model_instances

-- Docs: http://doc.pytest.org/en/latest/warnings.html
======================================== 1 passed, 2 warnings in 0.13 seconds ========================================
```

As you can see the warning have been generated again. This time on two places,
in `test_process` and `produce_answer` functions. Unless warning messages affect
results of tests, we do not require strictly to handle them. However, it's a good
practice. But if the warning log could affect the test (e.g. if a test function
checks logs) it should be treated by the `suppress_deprecation` decorator too.
In case of the library function, just add the `@suppress_deprecation(Foo)` line
before the definition of the `produce_answer` function. But if we do the same
for the test function, we will get an error (see that we have now just one
deprecation warning now):

```
| 21:59:57 | conftest | INFO | conftest.py | Actor 'foo_producer' context teardown complete
repos/system_upgrade/el7toel8/actors/fooproducer/tests/test_unit_fooproducer.py::test_process FAILED

====================================================== FAILURES ======================================================
____________________________________________________ test_process ____________________________________________________

args = (), kwargs = {'monkeypatch': <_pytest.monkeypatch.MonkeyPatch object at 0x7f21924b24d0>}
suppressed = <class 'leapp.models.foo.Foo'>

    @functools.wraps(target_item)
    def process_wrapper(*args, **kwargs):
        for suppressed in suppressed_items:
            _suppressed_deprecations.add(suppressed)
        try:
            return target_item(*args, **kwargs)
        finally:
            for suppressed in suppressed_items:
>               _suppressed_deprecations.remove(suppressed)
E               KeyError: <class 'leapp.models.foo.Foo'>

tut/lib/python3.7/site-packages/leapp/utils/deprecation.py:35: KeyError
================================================== warnings summary ==================================================
repos/system_upgrade/el7toel8/actors/fooproducer/tests/test_unit_fooproducer.py::test_process
  /tmp/leapp-repository/repos/system_upgrade/el7toel8/actors/fooproducer/tests/test_unit_fooproducer.py:13: _DeprecationWarningContext: Usage of deprecated Model "Foo"
    assert Foo(value='Answer is: 42') in produced_msgs.model_instances

-- Docs: http://doc.pytest.org/en/latest/warnings.html
======================================== 1 failed, 1 warnings in 0.21 seconds ========================================
```

It's because the mechanism of decorators in python and how pytest works.
In this case, we need to do a small workaround, like this:

```python
from leapp.libraries.actor import fooproducer_lib
from leapp.libraries.common.testutils import produce_mocked
from leapp.utils.deprecation import suppress_deprecation
from leapp.libraries.stdlib import api
from leapp.models import Foo  # deprecated model


@suppress_deprecation(Foo)
def _foo(value):
    """Small workaround to suppress deprecation messages in tests."""
    return Foo(value=value)

def test_process(monkeypatch):
    produced_msgs = produce_mocked()
    monkeypatch.setattr(api, 'produce', produced_msgs)
    fooproducer_lib.produce_answer()
    assert _foo(value='Answer is: 42') in produced_msgs.model_instances
```

That's the whole solution for the *FooProducer* actor. Analogically to this,
we need to treat the `FooConsumer` actor. You could notice that all imports
of the `Foo` model are commented. It's a good practice as it is more visible
to all developers that a deprecated entity is present.

## Example of a model replacement

This is analogous to the previous case. Take the same scenario, but extend it with
the case in which we want to replace the `Foo` model by the `Bar` model. What
will be changed in case of deprecation in the model definition? Just
the deprecation message and the new model definition:

```python
from leapp.models import Model, fields
from leapp.topics import SomeTopic
from leapp.utils.deprecation import deprecated


@deprecated(since='2020-06-20', message='The model has been replaced by Bar.')
class Foo(Model):
    topic = SomeTopic
    value = fields.String()


class Bar(Model):
    topic = SomeTopic
    value = fields.String()
```

You can see that in this case, the model has been just renamed, to keep
it simple. But it's sure that the new model can be different from the original
one (e.g. a different name of fields, a different purpose and set of fields,...).
The change in the `FooProducer` will be just extended by handling the new
model (it should include update of tests as well; I am skippin the example
as the change is trivial):

```python
from leapp.actors import Actor
from leapp.models import Bar
from leapp.models import Foo  # deprecated model
from leapp.tags import IPUWorkflowTag, FactsPhaseTag
from leapp.utils.deprecation import suppress_deprecation


@suppress_deprecation(Foo)
class FooProducer(Actor):
    """
    Just produce the right answer to the world.
    """

    name = 'foo_producer'
    consumes = ()
    produces = (Bar, Foo)
    tags = (IPUWorkflowTag, FactsPhaseTag)

    def process(self):
        self.produce(Foo(value='Answer is: 42'))
        self.produce(Bar(value='Answer is: 42'))
```

As you can see, the only thing that have been changed is the added production of the
`Bar` message. The original functionality is still present.

## Example of a derived model deprecation

**Warning: Known issue**: *The deprecation of a derived model is currently buggy and the documented
solution will end probably with traceback right now. The fix will be delivered
in the next release of leapp (current one is 0.11.0). As well, we will try to
simplify the final solution, so maybe this section will be more simple with
the next release.*

It's a common situation, that some models are derived from others. Typical
example is a base model which is actually not produced or consumed
by actors, but it is used as a base model for other models. For the sake of simplicity,
we'll use one of our previous solutions, but update the definition
of the `Foo` model (skipping imports):

```python
@deprecated(since='2020-01-1', message='This model has been deprecated.')
class BaseFoo(Model):
    topic = SystemInfoTopic
    value = fields.String()


class Foo(BaseFoo):
    pass
```

Previously, the content was handled completely, but with the new change, we
will see the warnings again:

```
================================================== warnings summary ==================================================
repos/system_upgrade/el7toel8/actors/fooproducer/tests/test_unit_fooproducer.py::test_process
  /tmp/leapp-repository/repos/system_upgrade/el7toel8/actors/fooproducer/libraries/fooproducer_lib.py:7: _DeprecationWarningContext: Usage of deprecated Model "BaseFoo"
    api.produce(Foo(value='Answer is: 42'))
  /tmp/leapp-repository/repos/system_upgrade/el7toel8/actors/fooproducer/tests/test_unit_fooproducer.py:10: _DeprecationWarningContext: Usage of deprecated Model "BaseFoo"
    return Foo(value=value)
```

See that the reported deprecated model is `BaseFoo`, however only
the `Foo` model is produced. This could be very confusing to users and developers. The
deprecation of the `BaseFoo` model could resolve our troubles, in the meaning that
the base class and all other classes are covered already, that the deprecated
entity has been used. But it is confusing, that with this solution, you have
to update the code in actor, to suppress the `BaseFoo` model instead of `Foo`,
even when `BaseFoo` is not used anywhere directly in the code. I mean
something like this:

```python
from leapp.models import Foo, BaseFoo
from leapp.libraries.stdlib import api
from leapp.utils.deprecation import suppress_deprecation

@suppress_deprecation(BaseFoo)
def produce_answer():
    api.produce(Foo(value='Answer is: 42'))
```

Now, I will put here just several ideas what could user do and why these are
wrong (if you are interested just about the working correct solution, skip
after the list):

1. *Deprecate all models (`BaseFoo`, `Foo`)* - The result will be two
   deprecation message per one use of `Foo`. One with the `Foo` msg, one with
   `BaseFoo`. That's not good, as we would like to get rid of `BaseFoo`
   completely in the messages ideally.
1. *Deprecate just the derived models (`Foo`)* - That could resolve the problem,
   but what if someone else derive a new model from the base one? They will not
   be notified about the deprecation and removal will break their code instantly.

If you want to ensure that all models (base and derived) are deprecated, the
best solution we are able to come up with it's little weird and it's going
slightly against our requirement (we are going to do an exception here), that
inside models cannot be defined any method or logic as they are supposed to be
used just as the *data container*. But this one is currently only possible way,
to deprecate such models correctly without confusing messages:

```python
from leapp.models import Model, fields
from leapp.topics import SystemInfoTopic
from leapp.utils.deprecation import deprecated
from leapp.utils.deprecation import suppress_deprecation


@deprecated(since='2020-01-01', message='This model has been deprecated.')
class BaseFoo(Model):
    topic = SystemInfoTopic
    value = fields.String()


@deprecated(since='2020-01-01', message='This model has been deprecated.')
class Foo(BaseFoo):
    @suppress_deprecation(BaseFoo)
    def __init__(self, *args, **kwargs):
        super(Foo, self).__init__(*args, **kwargs)
```

As you see, both models are deprecated. But in the derived one, there is the
`__init__` method defined, just for the purpose to be able to suppress the
deprecation message from the base model. Implementing this, the solution for
suppressing the deprecation warning in previous section will be working, without
any confusing messages. As well, this is the only possible usecase for a method
inside the models in official repositories managed by the OAMG team.

## Additional various outputs of snactor and leapp

### snactor warning message example

```
============================================================
                 USE OF DEPRECATED ENTITIES
============================================================

Usage of deprecated function "deprecated_function" @ /Users/vfeenstr/devel/work/leapp/leapp/tests/data/deprecation-tests/actors/deprecationtests/actor.py:19
Near:         deprecated_function()

Reason: This function is no longer supported.
------------------------------------------------------------
Usage of deprecated Model "DeprecatedModel" @ /Users/vfeenstr/devel/work/leapp/leapp/tests/data/deprecation-tests/actors/deprecationtests/actor.py:20
Near:         self.produce(DeprecatedModel())

Reason: This model is deprecated - Please do not use it anymore
------------------------------------------------------------
Usage of deprecated class "DeprecatedNoInit" @ /Users/vfeenstr/devel/work/leapp/leapp/tests/data/deprecation-tests/actors/deprecationtests/actor.py:21
Near:         DeprecatedNoInit()

Reason: Deprecated class without __init__
------------------------------------------------------------
Usage of deprecated class "DeprecatedBaseNoInit" @ /Users/vfeenstr/devel/work/leapp/leapp/tests/data/deprecation-tests/actors/deprecationtests/actor.py:22
Near:         DeprecatedNoInitDerived()

Reason: Deprecated base class without __init__
------------------------------------------------------------
Usage of deprecated class "DeprecatedWithInit" @ /Users/vfeenstr/devel/work/leapp/leapp/tests/data/deprecation-tests/actors/deprecationtests/actor.py:23
Near:         DeprecatedWithInit()

Reason: Deprecated class with __init__
------------------------------------------------------------

============================================================
             END OF USE OF DEPRECATED ENTITIES
============================================================
```

### leapp report example entries

```
----------------------------------------
Risk Factor: medium
Title: Usage of deprecated class "IsolatedActions" at /usr/share/leapp-repository/repositories/system_upgrade/el7toel8/libraries/repofileutils.py:38
Summary: IsolatedActions are deprecated
Since: 2020-01-02
Location: /usr/share/leapp-repository/repositories/system_upgrade/el7toel8/libraries/repofileutils.py:38
Near: def get_parsed_repofiles(context=mounting.NotIsolatedActions(base_dir='/')):

----------------------------------------
Risk Factor: medium
Title: Usage of deprecated class "IsolatedActions" at /usr/share/leapp-repository/repositories/system_upgrade/el7toel8/actors/scansubscriptionmanagerinfo/libraries/scanrhsm.py:8
Summary: IsolatedActions are deprecated
Since: 2020-01-02
Location: /usr/share/leapp-repository/repositories/system_upgrade/el7toel8/actors/scansubscriptionmanagerinfo/libraries/scanrhsm.py:8
Near:     context = NotIsolatedActions(base_dir='/')

----------------------------------------
Risk Factor: medium
Title: Usage of deprecated function "deprecated_method" at /usr/share/leapp-repository/repositories/system_upgrade/el7toel8/actors/deprecationdemo/actor.py:21
Summary: Deprecated for Demo!
Since: 2020-06-17
Location: /usr/share/leapp-repository/repositories/system_upgrade/el7toel8/actors/deprecationdemo/actor.py:21
Near:         self.deprecated_method()

----------------------------------------
```

## Additional simple usage examples of @deprecated

### Functions

```python
...
from leapp.utils.deprecation import deprecated


@deprecated(since='2020-06-20', message='This function has been deprecated!')
def some_deprecated_function(a, b, c):
    pass
```

### Models

```python
...
from leapp.utils.deprecation import deprecated


@deprecated(since='2020-06-20', message='This model has been deprecated!')
class MyModel(Model):
    topic = SomeTopic
    some_field = fields.String()
```

### Classes

```python
...
from leapp.utils.deprecation import deprecated


@deprecated(since='2020-06-20', message='This class has been deprecated!')
class MyClass(object):
    pass


# NOTE: Here we need to offset the stacklevel to get the report of the usage not in the derived class constructor
# but where the derived class has been created.
# How many levels you need, you will have to test, it depends on if you use the builtin __init__ methods or not,
# however it gives you the ability to go up the stack until you reach the position you need to.
@deprecated(since='2020-06-20', message='This class has been deprecated!', stack_level_offset=1)
class ABaseClass(object):
    def __init__(self):
        pass


class ADerivedClass(ABaseClass)
    def __init__(self):
        super(ADerivedClass, self).__init__()
```

=========================
rapidsms.contrib.handlers
=========================

.. module:: rapidsms.contrib.handlers

The `handlers` contrib application provides three classes- `BaseHandler`,
`KeywordHandler`, and `PatternHandler`- which can be extended to help you
create RapidSMS applications quickly.

.. _handlers-installation:

Installation
============

To define and use handlers for your RapidSMS project, you will need to
add ``"rapidsms.contrib.handlers"`` to :setting:`INSTALLED_APPS` in your
settings file::

    INSTALLED_APPS = [
        ...
        "rapidsms.contrib.handlers",
        ...
    ]

Then you'll also need to set :setting:`RAPIDSMS_HANDLERS`. The application
will load the handler classes listed in :setting:`RAPIDSMS_HANDLERS`,
as described in :ref:`handlers-discovery`.

.. code-block:: python

    RAPIDSMS_HANDLERS = [
        "rapidsms.contrib.handlers.KeywordHandler",
        "rapidsms.contrib.handlers.PatternHandler",
    ]

.. _handlers-usage:

Usage
=====

.. _keyword-handler:

KeywordHandler
--------------

Many RapidSMS applications operate based on whether a message begins with a
specific keyword. By subclassing `KeywordHandler`, you can easily create a
simple, keyword-based application::

    from rapidsms.contrib.handlers import KeywordHandler

    class LightHandler(KeywordHandler):
        keyword = "light"

        def help(self):
            self.respond("Send LIGHT ON or LIGHT OFF.")

        def handle(self, text):
            if text.upper() == "ON":
                self.respond("The light is now turned on.")

            elif text.upper() == "OFF":
                self.respond("Thanks for turning off the light!")

            else:
                self.help()

Your handler must define three things: `keyword`, `help()`, and `handle(text)`.
When a message is received that begins with the keyword (case insensitive;
leading whitespace is allowed), the remaining text is passed to the `handle`
method of the class. If no additional non-whitespace text is included with the
message, `help` is called instead. For example::

    > light
    < Send LIGHT ON or LIGHT OFF.
    > light on
    < The light is now turned on.
    > light off
    < Thanks for turning off the light!
    > light something else
    < Send LIGHT ON or LIGHT OFF.

The handler also treats ``,``, ``:``, and ``;`` after the keyword the same
as whitespace. For example::

    > light
    < Send LIGHT ON or LIGHT OFF.
    > light:on
    < The light is now turned on.
    > light, off
    < Thanks for turning off the light!
    > light :,; on
    < The light is now turned on.

.. TIP::
   Technically speaking, the incoming message text is compared to a regular
   expression pattern.

   The most common use case is to look for a single exact-match keyword.
   However, one could also match multiple keywords, for example
   ``keyword = "register|reg|join"``.

   However, due to how we build the final regular expression,
   capturing matches using grouping in the keyword regular expression
   won't work. If you need that, use the `PatternHandler`.

All non-matching messages are silently ignored to allow other applications and
handlers to catch them.

For example implementations of `KeywordHandler`, see

- `rapidsms.contrib.echo.handlers.echo.EchoHandler
  <https://github.com/rapidsms/rapidsms/blob/master/rapidsms/contrib/echo/handlers/echo.py>`_
- `rapidsms.contrib.registration.handlers.register.RegistrationHandler
  <https://github.com/rapidsms/rapidsms/blob/master/rapidsms/contrib/registration/handlers/register.py>`_
- `rapidsms.contrib.registration.handlers.language.LanguageHandler
  <https://github.com/rapidsms/rapidsms/blob/master/rapidsms/contrib/registration/handlers/language.py>`_

Here's documentation from the KeywordHandler class:

.. autoclass:: rapidsms.contrib.handlers.KeywordHandler()
    :members: keyword, help, handle


.. _pattern-handler:

PatternHandler
--------------

.. NOTE::
   Pattern-based handlers can work well for prototyping and simple use cases.
   For more complex parsing and message handling, we recommend writing a
   :doc:`RapidSMS application </topics/applications/index>` with a custom
   :ref:`handle phase <phase-handle>`.

The `PatternHandler` class can be subclassed to create applications which
respond to a message when a specific pattern is matched:

.. code-block:: python
    :emphasize-lines: 4,6,9

    from rapidsms.contrib.handlers import PatternHandler

    class SumHandler(PatternHandler):
        pattern = r"^(\d+) plus (\d+)$"

        def handle(self, a, b):
            a, b = int(a), int(b)
            total = a + b
            self.respond("%d + %d = %d" % (a, b, total))

Your handler must define `pattern` and `handle(*args)`. The pattern is
case-insensitive, but must otherwise be matched precisely as written (for
example, the handler pattern written above would not accept leading or
trailing whitespace, but the pattern ``r"^(\d+) plus (\d+)\s*$"`` would allow
trailing whitespace). When the pattern is matched, the `handle` method is
called with the captures as arguments. As an example, the above handler could
create the following conversation::

    > 1 plus 2
    < 1 + 2 = 3

Like `KeywordHandler`, each `PatternHandler` silently ignores all non-matching
messages to allow other handlers and applications to catch them.

Here's documentation from the PatternHandler class:

.. autoclass:: rapidsms.contrib.handlers.PatternHandler()
    :members: pattern, handle


.. _base-handler:

BaseHandler
-----------

All handlers, including the `KeywordHandler` and `PatternHandler`, are derived
from the `BaseHandler` class. When extending from `BaseHandler`, one must
always override the class method `dispatch`, which should return ``True`` when
it handles a message.

All instances of `BaseHandler` have access to `self.msg` and `self.router`, as
well as the methods `self.respond` and `self.respond_error` (which respond to
the instance's message).

`BaseHandler` also defines the class method `test`, which creates a simple
environment for testing a handler's response to a specific message text. If
the handler ignores the message then ``False`` is returned. Otherwise a list
containing the `text` property of each `OutgoingMessage` response, in the
order which they were sent, is returned. (Note: the list may be empty.) For
example::

    >>> from rapidsms.contrib.echo.handlers.echo import EchoHandler
    >>> EchoHandler.test("not applicable")
    False
    >>> EchoHandler.test("echo hello!")
    ["hello!"]

For an example implementation of a `BaseHandler`, see
`rapidsms.contrib.echo.handlers.ping.PingHandler
<https://github.com/rapidsms/rapidsms/blob/master/rapidsms/contrib/echo/handlers/ping.py>`_.

.. _calling-handlers:

Calling Handlers
================

When a message is received, the `handlers` application calls `dispatch` on
each of the handlers it loaded during :ref:`handlers discovery
<handlers-discovery>`.

The first handler to accept the message will block all others. The order in
which the handlers are called is not guaranteed, so each handler should be as
conservative as possible when choosing to respond to a message.

.. _handlers-discovery:

Handler Discovery
=================

.. versionchanged:: 0.15.0

Handlers may be any new-style Python class which extends from one of the
core handler classes, e.g. :ref:`BaseHandler <base-handler>`,
:ref:`PatternHandler <pattern-handler>`,
:ref:`KeywordHandler <keyword-handler>`, etc.

The Python package names of the handler classes to be loaded should be listed
in :setting:`RAPIDSMS_HANDLERS`.

Example:

.. code-block:: python

    RAPIDSMS_HANDLERS = [
        "rapidsms.contrib.handlers.KeywordHandler",
        "rapidsms.contrib.handlers.PatternHandler",
    ]

.. warning::

    The behavior described in the rest of this section is the old,
    deprecated behavior. If :setting:`RAPIDSMS_HANDLERS` is set,
    the older settings are ignored.

Handlers may be defined in the `handlers` subdirectory of any Django app
listed in :setting:`INSTALLED_APPS`. Each file in the `handlers` subdirectory
is expected to contain exactly one new-style Python class which extends from
one of the core handler classes.

Handler discovery, which occurs when the `handlers` application is loaded, can
be configured using the following project settings:

- :setting:`RAPIDSMS_HANDLERS_EXCLUDE_APPS` - The application will not load
  handlers from any Django app included in this list.

- :setting:`INSTALLED_HANDLERS` - If this list is not ``None``, the
  application will load only handlers in modules that are included in this
  list.

- :setting:`EXCLUDED_HANDLERS` - The application will not load any handler in
  a module that is included in this list.

.. NOTE::
   Prefix matching is used to determine which handlers are described in
   :setting:`INSTALLED_HANDLERS` and :setting:`EXCLUDED_HANDLERS`. The module
   name of each handler is compared to each value in these settings to see if
   it starts with the value. For example, consider the `rapidsms.contrib.echo`
   application which contains the `echo` handler and the `ping` handler:

      - "rapidsms.contrib.echo.handlers.echo" would match only `EchoHandler`,
      - "rapidsms.contrib.echo" would match both `EchoHandler` and
        `PingHandler`,
      - "rapidsms.contrib" would match all handlers in any RapidSMS contrib
        application, including both in `rapidsms.contrib.echo`.

=========================================
Parser methods, operators and combinators
=========================================

Parser methods
==============

Parser objects are returned by any of the builtin parser :doc:`primitives`. They
can be used and manipulated as below.

.. currentmodule:: parsy

.. class:: Parser

   The following methods are for actually **using** the parsers that you have
   created:

   .. method:: parse(string)

      Attempts to parse the given ``string``. If the parse is successful
      and consumes the entire string, the result is returned - otherwise, a
      ``ParseError`` is raised.

   .. function:: parse_partial(string)

      Similar to ``parse``, except that it does not require the entire
      string to be consumed. Returns a tuple of
      ``(result, rest_of_string)``, where ``rest_of_string`` is the part of
      the string that was left over.

   The following methods are essentially **combinators** that produce new
   parsers from existing ones. They are provided as methods on ``Parser`` for
   convenience. More combinators are documented below.

   .. function:: desc(string)

      Adds a desciption to the parser, which is used in the error message
      if parsing fails.

      >>> year = regex(r'[0-9]{4}').desc('4 digit year')
      >>> year.parse('123')
      ParseError: expected 4 digit year at 0:0

   .. function:: then(other_parser)

      Returns a parser which, if ``parser`` succeeds, will continue parsing
      with ``other_parser``. This will produce the value produced by
      ``other_parser``.

      .. code:: python

         >>> string('x').then(string('y')).parse('xy')
         'y'

      See also :ref:`parser-rshift`.

   .. function:: skip(other_parser)

      Similar to :meth:`Parser.then`, except the resulting parser will use
      the value produced by the first parser.

      .. code:: python

         >>> string('x').skip(string('y')).parse('xy')
         'x'

      See also :ref:`parser-lshift`.

   .. function:: many()

      Returns a parser that expects ``parser`` 0 or more times, and produces a
      list of the results. Note that this parser does not fail if nothing
      matches, but instead consumes nothing and produces an empty list.

      .. code:: python

         >>> parser = regex(r'[a-z]').many()
         >>> parser.parse('')
         []
         >>> parser.parse('abc')
         ['a', 'b', 'c']

   .. function:: times(min [, max=min])

      Returns a parser that expects ``parser`` at least ``min`` times, and
      at most ``max`` times, and produces a list of the results. If only
      one argument is given, the parser is expected exactly that number of
      times.

   .. function:: at_most(n)

      Returns a parser that expects ``parser`` at most ``n`` times, and
      produces a list of the results.

   .. function:: at_least(n)

      Returns a parser that expects ``parser`` at least ``n`` times, and
      produces a list of the results.

   .. function:: map(fn)

      Returns a parser that transforms the produced value of ``parser``
      with ``fn``.

      .. code:: python

         >>> regex(r'[0-9]+').map(int).parse('1234')
         1234

      This is the simplest way to convert parsed strings into the data types
      that you need.

   .. function:: result(val)

      Returns a parser that, if ``parser`` succeeds, always produces
      ``val``.

      .. code:: python

         >>> string('foo').result(42).parse('foo')
         42

   .. function:: bind(fn)

      Returns a parser which, if ``parser`` is successful, passes the
      result to ``fn``, and continues with the parser returned from ``fn``.
      This is the monadic binding operation.


.. _operators:

Parser operators
================

This section describes operators that you can use on :class:`Parser` objects to
build new parsers.


.. _parser-or:

``|`` operator
--------------

``parser | other_parser``

Returns a parser that tries ``parser`` and, if it fails, backtracks
and tries ``other_parser``. These can be chained together.

The resulting parser will produce the value produced by the first
successful parser.

.. code:: python

   >>> parser = string('x') | string('y') | string('z')
   >>> parser.parse('x')
   'x'
   >>> parser.parse('y')
   'y'
   >>> parser.parse('z')
   'z'

   >>> (string('x') >> string('y')).parse('xy')
   'y'

.. _parser-lshift:

``<<`` operator
---------------

``parser << other_parser``

The same as ``parser.skip(other_parser)`` - see :meth:`Parser.skip`.

(Hint - the arrows point at the important parser!)

.. code:: python

   >>> (string('x') << string('y')).parse('xy')
   'x'

.. _parser-rshift:

``>>`` operator
---------------

``parser >> other_parser``

The same as ``parser.then(other_parser)`` - see :meth:`Parser.then`.

(Hint - the arrows point at the important parser!)

.. code-block:: python

   >>> (string('x') >> string('y')).parse('xy')
   'y'


.. _parser-plus:

``+`` operator
--------------

``parser1 + parser2``

Requires both parsers to match in order, and adds the two results together using
the + operator. This will only work if the results support the plus operator
(e.g. strings and lists):


.. code-block:: python

   >>> (string("x") + regex("[0-9]")).parse("x1")
   "x1"

   >>> (string("x").many() + regex("[0-9]").map(int).many()).parse("xx123")
   ['x', 'x', 1, 2, 3]

The plus operator is a convenient shortcut for:

   >>> seq(parser1, parser2).map(lambda res: res[0] + res[1])

.. _parser-times:

``*`` operator
--------------

``parser1 * number``

This is a shortcut for doing :meth:`Parser.times`:

.. code-block:: python

   >>> (string("x") * 3).parse("xxx")
   ["x", "x", "x"]

You can also set both upper and lower bounds by multiplying by a range:

.. code-block:: python

   >>> (string("x") * range(0, 3)).parse("xxx")
   ParseError: expected EOF at 0:2

(Note the normal sematics of ``range`` are respected - the second number is an
*exclusive* upper bound, not inclusive).

Parser combinators
==================

.. function:: alt(*parsers)

   Creates a parser from the passed in argument list of alternative parsers,
   which are tried in order, moving to the next one if the current one fails, as
   per the :ref:`parser-or` - in other words, it matches any one of the
   alternative parsers.

   Example using `*arg` syntax to pass a list of parsers that have been
   generated by normal a normal ``map`` with :func:`string` over a list of
   characters:

   .. code-block:: python

      >>> hexdigit = alt(*map(string, "0123456789abcdef"))

.. function:: seq(*parsers)

   Creates a parser that runs a sequence of parsers in order and combines
   their results in a list.


   .. code-block:: python

      >>> seq(regex("[0-9]+").map(int) << string(" bottles of "),
              regex(r"\S+") << string(" on the "),
              regex(r"\S+")
              ).parse("99 bottles of beer on the wall")
      [99, 'bottles', 'wall']

Other combinators
=================

parsy does not try to include every possible combinator - there is no reason why
you cannot create your own for your needs using the builtin combinators and
primitives. If you find something that is very generic and would be very useful
to have as a builtin, please submit as a PR!

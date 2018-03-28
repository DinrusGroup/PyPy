===========================
What's new in PyPy2.7 5.10+
===========================

.. this is a revision shortly after release-pypy2.7-v5.10.0
.. startrev: 6b024edd9d12

.. branch: cpyext-avoid-roundtrip

Big refactoring of some cpyext code, which avoids a lot of nonsense when
calling C from Python and vice-versa: the result is a big speedup in
function/method calls, up to 6 times faster.

.. branch: cpyext-datetime2

Support ``tzinfo`` field on C-API datetime objects, fixes latest pandas HEAD


.. branch: mapdict-size-limit

Fix a corner case of mapdict: When an instance is used like a dict (using
``setattr`` and ``getattr``, or ``.__dict__``) and a lot of attributes are
added, then the performance using mapdict is linear in the number of
attributes. This is now fixed (by switching to a regular dict after 80
attributes).


.. branch: cpyext-faster-arg-passing

When using cpyext, improve the speed of passing certain objects from PyPy to C
code, most notably None, True, False, types, all instances of C-defined types.
Before, a dict lookup was needed every time such an object crossed over, now it
is just a field read.


.. branch: 2634_datetime_timedelta_performance

Improve datetime + timedelta performance.

.. branch: memory-accounting

Improve way to describe memory

.. branch: msvc14

Allow compilaiton with Visual Studio 2017 compiler suite on windows

.. branch: refactor-slots

Refactor cpyext slots.


.. branch: call-loopinvariant-into-bridges

Speed up branchy code that does a lot of function inlining by saving one call
to read the TLS in most bridges.

.. branch: rpython-sprint

Refactor in rpython signatures

.. branch: cpyext-tls-operror2

Store error state thread-locally in executioncontext, fixes issue #2764

.. branch: cpyext-fast-typecheck

Optimize `Py*_Check` for `Bool`, `Float`, `Set`. Also refactor and simplify
`W_PyCWrapperObject` which is used to call slots from the C-API, greatly
improving microbenchmarks in https://github.com/antocuni/cpyext-benchmarks

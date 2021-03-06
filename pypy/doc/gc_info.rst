Garbage collector documentation and configuration
=================================================


Incminimark
-----------

PyPy's default garbage collector is called incminimark - it's an incremental,
generational moving collector. Here we hope to explain a bit how it works
and how it can be tuned to suit the workload.

Incminimark first allocates objects in so called *nursery* - place for young
objects, where allocation is very cheap, being just a pointer bump. The nursery
size is a very crucial variable - depending on your workload (one or many
processes) and cache sizes you might want to experiment with it via
*PYPY_GC_NURSERY* environment variable. When the nursery is full, there is
performed a minor collection. Freed objects are no longer referencable and
just die, just by not being referenced any more; on the other hand, objects
found to still be alive must survive and are copied from the nursery
to the old generation. Either to arenas, which are collections
of objects of the same size, or directly allocated with malloc if they're
larger.  (A third category, the very large objects, are initially allocated
outside the nursery and never move.)

Since Incminimark is an incremental GC, the major collection is incremental,
meaning there should not be any pauses longer than 1ms.


Fragmentation
-------------

Before we discuss issues of "fragmentation", we need a bit of precision.
There are two kinds of related but distinct issues:

* If the program allocates a lot of memory, and then frees it all by
  dropping all references to it, then we might expect to see the RSS
  to drop.  (RSS = Resident Set Size on Linux, as seen by "top"; it is an
  approximation of the actual memory usage from the OS's point of view.)
  This might not occur: the RSS may remain at its highest value.  This
  issue is more precisely caused by the process not returning "free"
  memory to the OS.  We call this case "unreturned memory".

* After doing the above, if the RSS didn't go down, then at least future
  allocations should not cause the RSS to grow more.  That is, the process
  should reuse unreturned memory as long as it has got some left.  If this
  does not occur, the RSS grows even larger and we have real fragmentation
  issues.


gc.get_stats
------------

There is a special function in the ``gc`` module called
``get_stats(memory_pressure=False)``.

``memory_pressure`` controls whether or not to report memory pressure from
objects allocated outside of the GC, which requires walking the entire heap,
so it's disabled by default due to its cost. Enable it when debugging
mysterious memory disappearance.

Example call looks like that::
    
    >>> gc.get_stats(True)
    Total memory consumed:
    GC used:            4.2MB (peak: 4.2MB)
       in arenas:            763.7kB
       rawmalloced:          383.1kB
       nursery:              3.1MB
    raw assembler used: 0.0kB
    memory pressure:    0.0kB
    -----------------------------
    Total:              4.2MB

    Total memory allocated:
    GC allocated:            4.5MB (peak: 4.5MB)
       in arenas:            763.7kB
       rawmalloced:          383.1kB
       nursery:              3.1MB
    raw assembler allocated: 0.0kB
    memory pressure:    0.0kB
    -----------------------------
    Total:                   4.5MB
    
In this particular case, which is just at startup, GC consumes relatively
little memory and there is even less unused, but allocated memory. In case
there is a lot of unreturned memory or actual fragmentation, the "allocated"
can be much higher than "used".  Generally speaking, "peak" will more closely
resemble the actual memory consumed as reported by RSS.  Indeed, returning
memory to the OS is a hard and not solved problem.  In PyPy, it occurs only if
an arena is entirely free---a contiguous block of 64 pages of 4 or 8 KB each.
It is also rare for the "rawmalloced" category, at least for common system
implementations of ``malloc()``.

The details of various fields:

* GC in arenas - small old objects held in arenas. If the amount "allocated"
  is much higher than the amount "used", we have unreturned memory.  It is
  possible but unlikely that we have internal fragmentation here.  However,
  this unreturned memory cannot be reused for any ``malloc()``, including the
  memory from the "rawmalloced" section.

* GC rawmalloced - large objects allocated with malloc.  This is gives the
  current (first block of text) and peak (second block of text) memory
  allocated with ``malloc()``.  The amount of unreturned memory or
  fragmentation caused by ``malloc()`` cannot easily be reported.  Usually
  you can guess there is some if the RSS is much larger than the total
  memory reported for "GC allocated", but do keep in mind that this total
  does not include malloc'ed memory not known to PyPy's GC at all.  If you
  guess there is some, consider using `jemalloc`_ as opposed to system malloc.

.. _`jemalloc`: http://jemalloc.net/

* nursery - amount of memory allocated for nursery, fixed at startup,
  controlled via an environment variable

* raw assembler allocated - amount of assembler memory that JIT feels
  responsible for

* memory pressure, if asked for - amount of memory we think got allocated
  via external malloc (eg loading cert store in SSL contexts) that is kept
  alive by GC objects, but not accounted in the GC


.. _minimark-environment-variables:

Environment variables
---------------------

PyPy's default ``incminimark`` garbage collector is configurable through
several environment variables:

``PYPY_GC_NURSERY``
    The nursery size.
    Defaults to 1/2 of your last-level cache, or ``4M`` if unknown.
    Small values (like 1 or 1KB) are useful for debugging.

``PYPY_GC_NURSERY_DEBUG``
    If set to non-zero, will fill nursery with garbage, to help
    debugging.

``PYPY_GC_INCREMENT_STEP``
    The size of memory marked during the marking step.  Default is size of
    nursery times 2. If you mark it too high your GC is not incremental at
    all.  The minimum is set to size that survives minor collection times
    1.5 so we reclaim anything all the time.

``PYPY_GC_MAJOR_COLLECT``
    Major collection memory factor.
    Default is ``1.82``, which means trigger a major collection when the
    memory consumed equals 1.82 times the memory really used at the end
    of the previous major collection.

``PYPY_GC_GROWTH``
    Major collection threshold's max growth rate.
    Default is ``1.4``.
    Useful to collect more often than normally on sudden memory growth,
    e.g. when there is a temporary peak in memory usage.

``PYPY_GC_MAX``
    The max heap size.
    If coming near this limit, it will first collect more often, then
    raise an RPython MemoryError, and if that is not enough, crash the
    program with a fatal error.
    Try values like ``1.6GB``.

``PYPY_GC_MAX_DELTA``
    The major collection threshold will never be set to more than
    ``PYPY_GC_MAX_DELTA`` the amount really used after a collection.
    Defaults to 1/8th of the total RAM size (which is constrained to be
    at most 2/3/4GB on 32-bit systems).
    Try values like ``200MB``.

``PYPY_GC_MIN``
    Don't collect while the memory size is below this limit.
    Useful to avoid spending all the time in the GC in very small
    programs.
    Defaults to 8 times the nursery.

``PYPY_GC_DEBUG``
    Enable extra checks around collections that are too slow for normal
    use.
    Values are ``0`` (off), ``1`` (on major collections) or ``2`` (also
    on minor collections).

``PYPY_GC_MAX_PINNED``
    The maximal number of pinned objects at any point in time.  Defaults
    to a conservative value depending on nursery size and maximum object
    size inside the nursery.  Useful for debugging by setting it to 0.

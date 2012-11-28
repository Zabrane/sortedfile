
sortedfile
==========

`http://github.com/dw/sortedfile <http://github.com/dw/sortedfile>`_

.. toctree::
    :hidden:
    :maxdepth: 2

When handling large text files (for example, Apache logs), it is often
desirable to quickly access some subset without first splitting or import to a
database where a slow index creation process would be required.

When a file is already sorted (in the case of Apache logs, inherently so, since
they're generated in time order), we can exploit this using bisection search to
locate the beginning of the interesting subset.

Due to the nature of bisection this is O(log N) with the limiting factor being
the speed of a disk seek. Given a 1 terabyte file, 40 seeks are required,
resulting in an *expected* 600ms search time on a rusty old disk drive given
pessimistic constraints. Things look better on an SSD where less than 1ms seeks
are common, the same scenario could yield in excess of 25 lookups/second.


Common Parameters
#################

In addition to what is described below, each function takes the following
optional parameters:

``key``:
  If specified, indicates a function (in the style of ``sorted(..., key=)``)
  that maps each line in the file to an ordered Python object, which will then
  be used for comparison. Provide a key function to extract, for example, the
  unique ID or timestamp from the lines in your files.

  If no key function is given, lines are compared lexicographically.

``lo``:
  Lower search bound in bytes. Use this to skip e.g. undesirable header lines,
  or to constrain a search using a previously successful search. For line
  oriented files, search will actually include one byte prior to this offset
  in order to guarantee a complete line is seen.

``hi``:
  Upper search bound in bytes. If the file being searched is weird (e.g. it's a
  UNIX special device, or a file-like object or ``mmap.mmap``), specifies the
  highest bound that can be seeked.


Interface
#########

Five functions are provided in two variants, one for variable length lines and
one for fixed-length records. Fixed length versions are more efficient as they
require ``log2(length)`` fewer steps than a bytewise search.

Search Functions
++++++++++++++++

.. autofunction:: sortedfile.bisect_seek_left
.. autofunction:: sortedfile.bisect_seek_right
.. autofunction:: sortedfile.bisect_seek_fixed_left
.. autofunction:: sortedfile.bisect_seek_fixed_right


Iteration Functions
+++++++++++++++++++

.. autofunction:: sortedfile.iter_exclusive
.. autofunction:: sortedfile.iter_inclusive
.. autofunction:: sortedfile.iter_fixed_exclusive
.. autofunction:: sortedfile.iter_fixed_inclusive


Utility Functions
+++++++++++++++++

.. autofunction:: sortedfile.extents
.. autofunction:: sortedfile.extents_fixed
.. autofunction:: sortedfile.getsize



Example
#######

::

    def parse_ts(s):
        """Parse a UNIX syslog format date out of `s`."""
        return time.strptime(' '.join(s.split()[:3]), '%b %d %H:%M:%S')

    fp = file('/var/log/messages')
    # Copy a time range from syslog to stdout.
    it = sortedfile.iter_inclusive(fp,
        x=parse_ts('Nov 20 00:00:00'),
        y=parse_ts('Nov 25 23:59:59'),
        key=parse_ts)
    sys.stdout.writelines(it)


Cold Performance
################

Tests using a 100gb file containing 1.07 billion 100 byte records. Immediately
after running ``/usr/bin/purge`` on my 2010 Macbook with a SAMSUNG HN-M500MBB,
we get:

::

    sortedfile] python bigtest.py 
    46 recs in 5.08s (avg 110ms dist 31214mb / 9.05/sec)

A little while later:

::

    770 recs in 60.44s (avg 78ms dist 33080mb / 12.74/sec)

And the fixed record variant:

::

    sortedfile] python bigtest.py fixed
    85 recs in 5.01s (avg 58ms dist 33669mb / 16.96/sec)
    172 recs in 10.04s (avg 58ms dist 34344mb / 17.13/sec)
    ...
    1160 recs in 60.28s (avg 51ms dist 35038mb / 19.24/sec)

19 random reads per second on a 1 billion record data set, not bad for spinning
rust! ``bigtest.py`` could be tweaked to more thoroughly dodge the various
caches at work, but seems a realistic enough test as-is.

Reading the 100 consecutive records following each search provides some
indication of throughput in a common use case:

::

    sortedfile] python bigtest.py fixed span100
    ...
    101303 recs in 60.40s (avg 0.596ms / 1677.13/sec)


Hot Performance
###############

``bigtest.py warm`` is a more interesting test: instead of uniformly
distributed load over the full set, readers are only interested in recent data.
Without straying too far into kangaroo benchmark territory, it's fair to say
this is a common case.

Requests are randomly generated for the most recent 4% of the file (i.e. 4GB or
43 million records), with an initial warming that pre-caches the range most
reads are serviced by. ``mmap.mmap`` is used in place of ``file`` for its
significant performance benefits when disk IO is fast (i.e. cached).

After warmup it ``fork()`` s twice to make use of both cores.

::

    sortedfile] python bigtest.py warm mmap smp
    warm 0mb
    ...
    warm 4000mb
    done cache warm in 9159 ms
    48979 recs in 5.00s (avg 102us dist 0mb / 9793.93/sec)
    99043 recs in 10.00s (avg 100us dist 0mb / 9902.86/sec)
    ...
    558801 recs in 55.02s (avg 98us dist 0mb / 10156.87/sec)
    611674 recs in 60.00s (avg 98us dist 0mb / 10194.00/sec)

And the fixed variant:

::

    sortedfile] python bigtest.py fixed warm mmap smp
    warm 0mb
    ...
    warm 4000mb
    done cache warm in 56333 ms
    57496 recs in 5.01s (avg 87us dist 0mb / 11472.44/sec)
    118194 recs in 10.00s (avg 84us dist 0mb / 11818.55/sec)
    ...
    687029 recs in 55.00s (avg 80us dist 0mb / 12490.89/sec)
    751375 recs in 60.01s (avg 79us dist 0mb / 12521.16/sec)

Around 6250 random reads per second per core on 43 million hot records from a 1
billion record set, all using a plain text file as our "database" and a 23 line
Python function as our engine! Granted it only parses an integer from the
record, however even if the remainder of the record contained, say, JSON, a
single string split operation to remove the key would not overly hurt these
numbers.

And for the consecutive sequential read case:

::

    sortedfile] python bigtest.py fixed mmap smp warm span100
    ...
    15396036 recs in 60.01s (avg 0.004ms / 256578.04/sec)

There is an unfortunate limit: as ``mmap.mmap`` does not drop the GIL during a
read, page faults are enough to hang a process attempting to serve clients
using multiple threads. ``file`` does not have this problem, nor does forking a
new process per client, or maintaining a process pool.


A note on buffering
###################

When using ``file``, performance may vary according to the buffer size set for
the file and the target workload. For random reads of single records, a buffer
size that approximates the average record length will work better, whereas for
quick seeks followed by long sequential reads, a larger size is probably
better.


Interesting uses
################

Since the ``bisect`` functions re-check the input file's size on each call when
``hi`` isn't specified, it is trivial to have concurrent readers and writers,
so long as writers take care to open the file as ``O_APPEND``, and emit records
no larger than the maximum atomic write size for the operating system. On
Linux, since ``write()`` holds a lock, it should be possible to write records
of arbitrary size.

However since each region's midpoint will change while the file grows, this
mode may not interact well with OS caching without further mitigation.


Room for improvement
####################

It should be possible to squeeze better performance out of the ``file`` case by
paying more attention to the operating system's needs, in particular with
regard to read alignment, and the use of ``posix_fadvise``. The ``file``
performance above is significantly worse than ``mmap.mmap``, this is almost
certainly not inherent, and more likely due to a badly designed test.
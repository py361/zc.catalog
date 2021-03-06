==================
 Normalized Index
==================

The index module provides a normalizing wrapper, a DateTime normalizer, and
a set index and a value index normalized with the DateTime normalizer.

The normalizing wrapper implements a full complement of index interfaces--
zope.index.interfaces.IInjection, zope.index.interfaces.IIndexSearch,
zope.index.interfaces.IStatistics, and zc.catalog.interfaces.IIndexValues--
and delegates all of the behavior to the wrapped index, normalizing values
using the normalizer before the index sees them.

The normalizing wrapper currently only supports queries offered by
zc.catalog.interfaces.ISetIndex and zc.catalog.interfaces.IValueIndex.

The normalizer interface requires the following methods, as defined in the
interface:

    def value(value):
        """normalize or check constraints for an input value; raise an error
        or return the value to be indexed."""

    def any(value, index):
        """normalize a query value for a "any_of" search; return a sequence of
        values."""

    def all(value, index):
        """Normalize a query value for an "all_of" search; return the value
        for query"""

    def minimum(value, index):
        """normalize a query value for minimum of a range; return the value for
        query"""

    def maximum(value, index):
        """normalize a query value for maximum of a range; return the value for
        query"""

The DateTime normalizer performs the following normalizations and validations.
Whenever a timezone is needed, it tries to get a request from the current
interaction and adapt it to zope.interface.common.idatetime.ITZInfo; failing
that (no request or no adapter) it uses the system local timezone.

- input values must be datetimes with a timezone.  They are normalized to the
  resolution specified when the normalizer is created: a resolution of 0
  normalizes values to days; a resolution of 1 to hours; 2 to minutes; 3 to
  seconds; and 4 to microseconds.

- 'any' values may be timezone-aware datetimes, timezone-naive datetimes,
  or dates.  dates are converted to any value from the start to the end of the
  given date in the found timezone, as described above.  timezone-naive
  datetimes get the found timezone.

- 'all' values may be timezone-aware datetimes or timezone-naive datetimes.
  timezone-naive datetimes get the found timezone.

- 'minimum' values may be timezone-aware datetimes, timezone-naive datetimes,
  or dates.  dates are converted to the start of the given date in the found
  timezone, as described above.  timezone-naive datetimes get the found
  timezone.

- 'maximum' values may be timezone-aware datetimes, timezone-naive datetimes,
  or dates.  dates are converted to the end of the given date in the found
  timezone, as described above.  timezone-naive datetimes get the found
  timezone.

Let's look at the DateTime normalizer first, and then an integration of it
with the normalizing wrapper and the value and set indexes.

The indexed values are parsed with 'value'.

    >>> from zc.catalog.index import DateTimeNormalizer
    >>> n = DateTimeNormalizer() # defaults to minutes
    >>> import datetime
    >>> import pytz
    >>> naive_datetime = datetime.datetime(2005, 7, 15, 11, 21, 32, 104)
    >>> date = naive_datetime.date()
    >>> aware_datetime = naive_datetime.replace(
    ...     tzinfo=pytz.timezone('US/Eastern'))
    >>> n.value(naive_datetime)
    Traceback (most recent call last):
    ...
    ValueError: This index only indexes timezone-aware datetimes.
    >>> n.value(date)
    Traceback (most recent call last):
    ...
    ValueError: This index only indexes timezone-aware datetimes.
    >>> n.value(aware_datetime) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 11, 21, tzinfo=<DstTzInfo 'US/Eastern'...>)

If we specify a different resolution, the results are different.

    >>> another = DateTimeNormalizer(1) # hours
    >>> another.value(aware_datetime) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 11, 0, tzinfo=<DstTzInfo 'US/Eastern'...>)

Note that changing the resolution of an indexed value may create surprising
results, because queries do not change their resolution.  Therefore, if you
index something with a datetime with a finer resolution that the normalizer's,
then searching for that datetime will not find the doc_id.

Values in an 'any_of' query are parsed with 'any'.  'any' should return a
sequence of values.  It requires an index, which we will mock up here.

    >>> class DummyIndex(object):
    ...     def values(self, start, stop, exclude_start, exclude_stop):
    ...         assert not exclude_start and exclude_stop
    ...         six_hours = datetime.timedelta(hours=6)
    ...         res = []
    ...         dt = start
    ...         while dt < stop:
    ...             res.append(dt)
    ...             dt += six_hours
    ...         return res
    ...
    >>> index = DummyIndex()
    >>> tuple(n.any(naive_datetime, index)) # doctest: +ELLIPSIS
    (datetime.datetime(2005, 7, 15, 11, 21, 32, 104, tzinfo=<...Local...>),)
    >>> tuple(n.any(aware_datetime, index)) # doctest: +ELLIPSIS
    (datetime.datetime(2005, 7, 15, 11, 21, 32, 104, tzinfo=<...Eastern...>),)
    >>> tuple(n.any(date, index)) # doctest: +NORMALIZE_WHITESPACE +ELLIPSIS
    (datetime.datetime(2005, 7, 15, 0, 0, tzinfo=<...Local...>),
     datetime.datetime(2005, 7, 15, 6, 0, tzinfo=<...Local...>),
     datetime.datetime(2005, 7, 15, 12, 0, tzinfo=<...Local...>),
     datetime.datetime(2005, 7, 15, 18, 0, tzinfo=<...Local...>))

Values in an 'all_of' query are parsed with 'all'.

    >>> n.all(naive_datetime, index) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 11, 21, 32, 104, tzinfo=<...Local...>)
    >>> n.all(aware_datetime, index) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 11, 21, 32, 104, tzinfo=<...Eastern...>)
    >>> n.all(date, index) # doctest: +ELLIPSIS
    Traceback (most recent call last):
    ...
    ValueError: ...

Minimum values in a 'between' query as well as those in other methods are
parsed with 'minimum'.  They also take an optional exclude boolean, which
indicates whether the minimum is to be excluded.  For datetimes, it only
makes a difference if you pass in a date.

    >>> n.minimum(naive_datetime, index) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 11, 21, 32, 104, tzinfo=<...Local...>)
    >>> n.minimum(naive_datetime, index, exclude=True) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 11, 21, 32, 104, tzinfo=<...Local...>)

    >>> n.minimum(aware_datetime, index) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 11, 21, 32, 104, tzinfo=<...Eastern...>)
    >>> n.minimum(aware_datetime, index, True) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 11, 21, 32, 104, tzinfo=<...Eastern...>)

    >>> n.minimum(date, index) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 0, 0, tzinfo=<...Local...>)
    >>> n.minimum(date, index, True) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 23, 59, 59, 999999, tzinfo=<...Local...>)

Maximum values in a 'between' query as well as those in other methods are
parsed with 'maximum'.  They also take an optional exclude boolean, which
indicates whether the maximum is to be excluded.  For datetimes, it only
makes a difference if you pass in a date.

    >>> n.maximum(naive_datetime, index) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 11, 21, 32, 104, tzinfo=<...Local...>)
    >>> n.maximum(naive_datetime, index, exclude=True) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 11, 21, 32, 104, tzinfo=<...Local...>)

    >>> n.maximum(aware_datetime, index) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 11, 21, 32, 104, tzinfo=<...Eastern...>)
    >>> n.maximum(aware_datetime, index, True) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 11, 21, 32, 104, tzinfo=<...Eastern...>)

    >>> n.maximum(date, index) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 23, 59, 59, 999999, tzinfo=<...Local...>)
    >>> n.maximum(date, index, True) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 0, 0, tzinfo=<...Local...>)

Now let's examine these normalizers in the context of a real index.

    >>> from zc.catalog.index import DateTimeValueIndex, DateTimeSetIndex
    >>> setindex = DateTimeSetIndex() # minutes resolution
    >>> data = [] # generate some data
    >>> def date_gen(
    ...     start=aware_datetime,
    ...     count=12,
    ...     period=datetime.timedelta(hours=10)):
    ...     dt = start
    ...     ix = 0
    ...     while ix < count:
    ...         yield dt
    ...         dt += period
    ...         ix += 1
    ...
    >>> gen = date_gen()
    >>> count = 0
    >>> while True:
    ...     try:
    ...         next_ = [next(gen) for i in range(6)]
    ...     except StopIteration:
    ...         break
    ...     data.append((count, next_[0:1]))
    ...     count += 1
    ...     data.append((count, next_[1:3]))
    ...     count += 1
    ...     data.append((count, next_[3:6]))
    ...     count += 1
    ...
    >>> print(data) # doctest: +ELLIPSIS +NORMALIZE_WHITESPACE
    [(0,
      [datetime.datetime(2005, 7, 15, 11, 21, 32, 104, ...<...Eastern...>)]),
     (1,
      [datetime.datetime(2005, 7, 15, 21, 21, 32, 104, ...<...Eastern...>),
       datetime.datetime(2005, 7, 16, 7, 21, 32, 104, ...<...Eastern...>)]),
     (2,
      [datetime.datetime(2005, 7, 16, 17, 21, 32, 104, ...<...Eastern...>),
       datetime.datetime(2005, 7, 17, 3, 21, 32, 104, ...<...Eastern...>),
       datetime.datetime(2005, 7, 17, 13, 21, 32, 104, ...<...Eastern...>)]),
     (3,
      [datetime.datetime(2005, 7, 17, 23, 21, 32, 104, ...<...Eastern...>)]),
     (4,
      [datetime.datetime(2005, 7, 18, 9, 21, 32, 104, ...<...Eastern...>),
       datetime.datetime(2005, 7, 18, 19, 21, 32, 104, ...<...Eastern...>)]),
     (5,
      [datetime.datetime(2005, 7, 19, 5, 21, 32, 104, ...<...Eastern...>),
       datetime.datetime(2005, 7, 19, 15, 21, 32, 104, ...<...Eastern...>),
       datetime.datetime(2005, 7, 20, 1, 21, 32, 104, ...<...Eastern...>)])]
    >>> data_dict = dict(data)
    >>> for doc_id, value in data:
    ...     setindex.index_doc(doc_id, value)
    ...
    >>> list(setindex.ids())
    [0, 1, 2, 3, 4, 5]
    >>> set(setindex.values()) == set(
    ...     setindex.normalizer.value(v) for v in date_gen())
    True

For the searches, we will actually use a request and interaction, with an
adapter that returns the Eastern timezone.  This makes the examples less
dependent on the machine that they use.

    >>> import zope.security.management
    >>> import zope.publisher.browser
    >>> import zope.interface.common.idatetime
    >>> import zope.publisher.interfaces
    >>> request = zope.publisher.browser.TestRequest()
    >>> zope.security.management.newInteraction(request)
    >>> from zope import interface, component
    >>> @interface.implementer(zope.interface.common.idatetime.ITZInfo)
    ... @component.adapter(zope.publisher.interfaces.IRequest)
    ... def tzinfo(req):
    ...     return pytz.timezone('US/Eastern')
    ...
    >>> component.provideAdapter(tzinfo)
    >>> n.all(naive_datetime, index).tzinfo is pytz.timezone('US/Eastern')
    True

    >>> set(setindex.apply({'any_of': (datetime.date(2005, 7, 17),
    ...                                datetime.date(2005, 7, 20),
    ...                                datetime.date(2005, 12, 31))})) == set(
    ...     (2, 3, 5))
    True

Note that this search is using the normalized values.

    >>> set(setindex.apply({'all_of': (
    ...     datetime.datetime(
    ...         2005, 7, 16, 7, 21, tzinfo=pytz.timezone('US/Eastern')),
    ...     datetime.datetime(
    ...         2005, 7, 15, 21, 21, tzinfo=pytz.timezone('US/Eastern')),)})
    ...     ) == set((1,))
    True
    >>> list(setindex.apply({'any': None}))
    [0, 1, 2, 3, 4, 5]
    >>> set(setindex.apply({'between': (
    ...     datetime.datetime(2005, 4, 1, 12), datetime.datetime(2006, 5, 1))})
    ...     ) == set((0, 1, 2, 3, 4, 5))
    True
    >>> set(setindex.apply({'between': (
    ...     datetime.datetime(2005, 4, 1, 12), datetime.datetime(2006, 5, 1),
    ...     True, True)})
    ...     ) == set((0, 1, 2, 3, 4, 5))
    True

'between' searches should deal with dates well.

    >>> set(setindex.apply({'between': (
    ...     datetime.date(2005, 7, 16), datetime.date(2005, 7, 17))})
    ...     ) == set((1, 2, 3))
    True
    >>> len(setindex.apply({'between': (
    ...     datetime.date(2005, 7, 16), datetime.date(2005, 7, 17))})
    ...     ) == len(setindex.apply({'between': (
    ...     datetime.date(2005, 7, 15), datetime.date(2005, 7, 18),
    ...     True, True)})
    ...     )
    True

Removing docs works as usual.

    >>> setindex.unindex_doc(1)
    >>> list(setindex.ids())
    [0, 2, 3, 4, 5]

Value, Minvalue and Maxvalue can take timezone-less datetimes and dates.

    >>> setindex.minValue() # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 15, 11, 21, ...<...Eastern...>)
    >>> setindex.minValue(datetime.date(2005, 7, 17)) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 17, 3, 21, ...<...Eastern...>)

    >>> setindex.maxValue() # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 20, 1, 21, ...<...Eastern...>)
    >>> setindex.maxValue(datetime.date(2005, 7, 17)) # doctest: +ELLIPSIS
    datetime.datetime(2005, 7, 17, 23, 21, ...<...Eastern...>)

    >>> list(setindex.values(
    ... datetime.date(2005, 7, 17), datetime.date(2005, 7, 17)))
    ... # doctest: +ELLIPSIS +NORMALIZE_WHITESPACE
    [datetime.datetime(2005, 7, 17, 3, 21, ...<...Eastern...>),
     datetime.datetime(2005, 7, 17, 13, 21, ...<...Eastern...>),
     datetime.datetime(2005, 7, 17, 23, 21, ...<...Eastern...>)]

    >>> zope.security.management.endInteraction() # TODO put in tests tearDown

Sorting
=======

The normalization wrapper provides the zope.index.interfaces.IIndexSort
interface if its upstream index provides it. For example, the
DateTimeValueIndex will provide IIndexSort, because ValueIndex provides
sorting. It will also delegate the ``sort`` method to the value index.

    >>> from zc.catalog.index import DateTimeValueIndex
    >>> from zope.index.interfaces import IIndexSort

    >>> ix = DateTimeValueIndex()
    >>> IIndexSort.providedBy(ix.index)
    True
    >>> IIndexSort.providedBy(ix)
    True
    >>> ix.sort.__self__ is ix.index
    True

But it won't work for indexes that doesn't do sorting, for example
DateTimeSetIndex.

    >>> ix = DateTimeSetIndex()
    >>> IIndexSort.providedBy(ix.index)
    False
    >>> IIndexSort.providedBy(ix)
    False
    >>> ix.sort
    Traceback (most recent call last):
    ...
    AttributeError: 'SetIndex' object has no attribute 'sort'

---
layout: post
title:  "python GreenThreadPoolExecutor的工作原理"
date:   2017-04-01 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---


oepnstack的MessageHandlingServer中使用了GreenThreadPoolExecutor

这玩意回合了yeid和eventlet,网上相关文档很少,直接把源码读了,源码再下面两个包中


    python-futurist-0.13.0-1
    python-futures-3.0.3-1




```python

class GreenThreadPoolExecutor(_futures.Executor):
    """Executor that uses a green thread pool to execute calls asynchronously.

    See: https://docs.python.org/dev/library/concurrent.futures.html
    and http://eventlet.net/doc/modules/greenpool.html for information on
    how this works.

    It gathers statistics about the submissions executed for post-analysis...
    """

    threading = _green.threading

    def __init__(self, max_workers=1000, check_and_reject=None):
        """Initializes a green thread pool executor.

        :param max_workers: maximum number of workers that can be
                            simulatenously active at the same time, further
                            submitted work will be queued up when this limit
                            is reached.
        :type max_workers: int
        :param check_and_reject: a callback function that will be provided
                                 two position arguments, the first argument
                                 will be this executor instance, and the second
                                 will be the number of currently queued work
                                 items in this executors backlog; the callback
                                 should raise a :py:class:`.RejectedSubmission`
                                 exception if it wants to have this submission
                                 rejected.
        :type check_and_reject: callback
        """
        if not _utils.EVENTLET_AVAILABLE:
            raise RuntimeError('Eventlet is needed to use a green executor')
        if max_workers <= 0:
            raise ValueError("Max workers must be greater than zero")
        self._max_workers = max_workers
        self._pool = _green.Pool(self._max_workers)
        self._delayed_work = _green.Queue()
        self._check_and_reject = check_and_reject or (lambda e, waiting: None)
        self._shutdown_lock = self.threading.lock_object()
        self._shutdown = False
        self._gatherer = _Gatherer(self._submit,
                                   self.threading.lock_object)

    @property
    def alive(self):
        """Accessor to determine if the executor is alive/active."""
        return not self._shutdown

    @property
    def statistics(self):
        """:class:`.ExecutorStatistics` about the executors executions."""
        return self._gatherer.statistics

    def submit(self, fn, *args, **kwargs):
        """Submit some work to be executed (and gather statistics).

        :param args: non-keyworded arguments
        :type args: list
        :param kwargs: key-value arguments
        :type kwargs: dictionary
        """
        with self._shutdown_lock:
            if self._shutdown:
                raise RuntimeError('Can not schedule new futures'
                                   ' after being shutdown')
            self._check_and_reject(self, self._delayed_work.qsize())
            return self._gatherer.submit(fn, *args, **kwargs)

    def _submit(self, fn, *args, **kwargs):
        f = GreenFuture()
        work = _utils.WorkItem(f, fn, args, kwargs)
        if not self._spin_up(work):
            self._delayed_work.put(work)
        return f

    def _spin_up(self, work):
        """Spin up a greenworker if less than max_workers.

        :param work: work to be given to the greenworker
        :returns: whether a green worker was spun up or not
        :rtype: boolean
        """
        alive = self._pool.running() + self._pool.waiting()
        if alive < self._max_workers:
            self._pool.spawn_n(_green.GreenWorker(work, self._delayed_work))
            return True
        return False

    def shutdown(self, wait=True):
        with self._shutdown_lock:
            if not self._shutdown:
                self._shutdown = True
                shutoff = True
            else:
                shutoff = False
        if wait and shutoff:
            self._pool.waitall()
            self._delayed_work.join()

    # --------------下面是_futures.Executor的代码------------------
    def map(self, fn, *iterables, **kwargs):
        """Returns a iterator equivalent to map(fn, iter).

        Args:
            fn: A callable that will take as many arguments as there are
                passed iterables.
            timeout: The maximum number of seconds to wait. If None, then there
                is no limit on the wait time.

        Returns:
            An iterator equivalent to: map(func, *iterables) but the calls may
            be evaluated out-of-order.

        Raises:
            TimeoutError: If the entire result iterator could not be generated
                before the given timeout.
            Exception: If fn(*args) raises for any values.
        """
        timeout = kwargs.get('timeout')
        if timeout is not None:
            end_time = timeout + time.time()

        fs = [self.submit(fn, *args) for args in itertools.izip(*iterables)]

        # Yield must be hidden in closure so that the futures are submitted
        # before the first iterator value is required.
        def result_iterator():
            try:
                for future in fs:
                    if timeout is None:
                        yield future.result()
                    else:
                        yield future.result(end_time - time.time())
            finally:
                for future in fs:
                    future.cancel()
        return result_iterator()


    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.shutdown(wait=True)
        return False

```

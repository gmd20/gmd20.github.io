

http://en.wikipedia.org/wiki/MESI_protocol



Memory Barriers

MESI in its naive, straightforward implementaton exhibits two particular low-performance behaviours; firstly, when writing to an invalid cache line, there is a long delay while the line is fetched from another CPU, secondly, moving cache lines to the invalid state is time consuming.

Consequently, CPUs implement store buffers and invalidate queues.

A store bufferis used when writing to an invalid cache line. Since the write will proceed anyway, the CPU issues a read-invalid message (so that it is given the cache line in question and so that all other CPUs invalidate that line) and then pushes the write into the store-buffer, to be executed when the cache line finally arrives. (A CPU will when trying to read cache lines scan its own store buffer, in case it has something ready to write to the cache).

Consequently, a CPU can from its point of view have written something, but it isn't yet in the cache and so other CPUs *cannot see this* - they cannot scan the store buffer of other CPUs.

With regard to invalidation, CPUs implement invalidate queues, whereby incoming invalidate requests are instantly acknowledged but not in fact acted upon - they instead simply enter an invalidation queue, their processing occurs as soon as possible (but not necessarily instantly). As such a CPU can have in its cache a line which is invalid, but where it doesn't yet know that line is invalid - the invalidation queue contains the invalidation which hasn't yet been acted upon. (The invalidation queue is on the other "side" of the cache; the CPU can't scan it, as it can the store buffer).

As a result, memory barriers are required. A store barrier will flush the store-buffer (ensuring all writes have entered that CPUs cache). A read barrier will flush the invalidation queue (ensuring all writes by other CPUs become visible to the flushing CPU).

So MESI in practise doesn't quite work - not a problem if you're single threaded, but definitely a problem if not.

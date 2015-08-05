##### 一个线程一个 lua 虚拟机模型

任然会出现单个 lua 虚拟机的内存过高，而 lua5.3 的 gc 是增量 gc，当 lua 虚拟机内存过大的时候，下一次触发 GC 时的内存就会更大，久而久之容易出现内存泄露，其实不是泄露，是 GC 不及时，而 lua5.3 又把分代 GC 给去掉了，分代 GC 能更及时地做一次 full gc操作，所以单个 lua 虚拟机的内存不应该过大. skynet 很容易做到这一点，某个服务只加载若干数据库实体，服务要访问这些实体数据可以通过通信来达到共享，这就是 actor 模型的本质：在通信的过程中共享数据。一个数据中心只加载若干数据实体，而不是所有的数据实体，有部分实在需要共享的话，那就通过发消息方式来获取，这就是 actor 模型.


http://stackoverflow.com/questions/5092134/whats-the-difference-between-generational-and-incremental-garbage-collection

generational GC is always incremental, because it does not collect all unreachable objects during a cycle. Conversely, an incremental GC does not necessarily employ a generation scheme to decide which unreachable objects to collect or not.

##### A generational GC
divides the unreachable objects into different sets, roughly according to their last use - their age, so to speak. The basic theory is that objects that are most recently created, would become unreachable quickly. So the set with 'young' objects is collected in an early stage.

An incremental garbage collector is any garbage-collector that can run incrementally (meaning that it can do a little work, then some more work, then some more work), instead of having to run the whole collection without interruption. This stands in contrast to old stop-the-world garbage collectors that did e.g. a mark&sweep without any other code being able to work on the objects. But to be clear: Whether an incremental garbage collector actually runs in parallel to other code executing on the same objects is not important as long as it is interruptable (for which it has to e.g. distinguish between dirty and clean objects).

##### An incremental GC 
may be implemented with above generational scheme, but different methods can be employed to decide which group of objects should be sweeped.

A generational garbage collector differentiates between old, medium and new objects. It can then do copying GC on the new objects (keyword "Eden"), mark&sweep for the old objects and different possibilities (depending on implementation) on the medium objects. Depending on implementation the way the generations of objects are distinguished is either by region occupied in memory or by flags. The challenge of generational GC is to keep lists of objects that refer from one generation to the other up to date.

http://www.jopdesign.com/doc/nbgc.pdf
http://www.ibm.com/developerworks/java/library/j-rtj4/index.html
http://www.memorymanagement.org/glossary/i.html#incremental.garbage.collection
http://www.memorymanagement.org/glossary/g.html#generational.garbage.collection




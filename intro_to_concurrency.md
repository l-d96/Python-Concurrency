# Intro

The dictionary definition of concurrency is simultaneous occurrence. In Python,
the things that are occurring simultaneously are called by different names
(thread, task, process) but at a high level, they all refer to a sequence of
instructions that run in order.

Each one simultaneous thing can be stopped at certain points, and the CPU or brain that is
processing them can switch to a different one. The state of each one is saved
so it can be restarted right where it was interrupted.

Only multiprocessing actually runs these trains of thought at literally the
same time. Threading and asyncio both run on a single processor and therefore
only run one at a time. They just cleverly find ways to take turns to speed up
the overall process. Even though they don’t run different trains of thought
simultaneously, we still call this concurrency.

The way the threads or tasks take turns is the big difference between threading
and asyncio. In threading, the operating system actually knows about each
thread and can interrupt it at any time to start running a different thread.
This is called pre-emptive multitasking since the operating system can pre-empt
your thread to make the switch.

Pre-emptive multitasking is handy in that the code in the thread doesn’t need
to do anything to make the switch. It can also be difficult because of that “at
any time” phrase. This switch can happen in the middle of a single Python
statement, even a trivial one like x = x + 1.

Asyncio, on the other hand, uses cooperative multitasking. The tasks must
cooperate by announcing when they are ready to be switched out. That means that
the code in the task has to change slightly to make this happen.

## What is Parallelism?

With multiprocessing, Python creates new processes. A process here can be
thought of as almost a completely different program, though technically they’re
usually defined as a collection of resources where the resources include
memory, file handles and things like that. One way to think about it is that
each process runs in its own Python interpreter.

Because they are different processes, each of your trains of thought in a
multiprocessing program can run on a different core. Running on a different
core means that they actually can run at the same time

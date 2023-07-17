# Intro

The dictionary definition of concurrency is simultaneous occurrence. In Python,
the things that are occurring simultaneously are called by different names
(thread, task, process) but at a high level, they all refer to a sequence of
instructions that run in order.

Each one simultaneous thing can be stopped at certain points, and the CPU or
brain that is processing them can switch to a different one. The state of each
one is saved so it can be restarted right where it was interrupted.

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

# When is concurrency useful?

Concurrency can make a big difference for two types of problems. These are
generally called CPU-bound and I/O-bound.

I/O-bound problems cause your program to slow down because it frequently must
wait for input/output (I/O) from some external resource. They arise frequently
when your program is working with things that are much slower than your CPU,
like filesystem and network connections

On the flip side, there are classes of programs that do significant
computation without talking to the network or accessing a file. These are the
CPU-bound programs, because the resource limiting the speed of your program is
the CPU, not the network or the file system.

Because the operating system is in control of when your task gets
interrupted and another task starts, any data that is shared between
the threads needs to be protected, or thread-safe.

There are several strategies for making data accesses thread-safe
depending on what the data is and how you’re using it. One of them is
to use thread-safe data structures like Queue from Python’s queue
module.

These objects use low-level primitives like threading.Lock to ensure
that only one thread can access a block of code or a bit of memory at
the same time. You are using this strategy indirectly by way of the
ThreadPoolExecutor object.

Another strategy to use here is something called thread local storage.
threading.local() creates an object that looks like a global but is
specific to each individual thread.

{{python
thread_local = threading.local()


def get_session():
    if not hasattr(thread_local, "session"):
        thread_local.session = requests.Session()
    return thread_local.session
}}



`local()` is in the threading module to specifically solve this problem.
It looks a little odd, but you only want to create one of these
objects, not one for each thread. The object itself takes care of
separating accesses from different threads to different data.

Threads can interact in ways that are subtle and hard to detect. These
interactions can cause race conditions that frequently result in
random, intermittent bugs that can be quite difficult to find.

## Race Conditions

Race conditions are an entire class of subtle bugs that can and
frequently do happen in multi-threaded code. Race conditions happen
because the programmer has not sufficiently protected data accesses to
prevent threads from interfering with each other. You need to take
extra steps when writing threaded code to ensure things are
thread-safe.

What’s going on here is that the operating system is controlling when
your thread runs and when it gets swapped out to let another thread
run. This thread swapping can occur at any point, even while doing
sub-steps of a Python statement. 

{{python

import concurrent.futures


counter = 0


def increment_counter(fake_value):
    global counter
    for _ in range(100):
        counter += 1


if __name__ == "__main__":
    fake_data = [x for x in range(5000)]
    counter = 0
    with concurrent.futures.ThreadPoolExecutor(max_workers=5000) as executor:
        executor.map(increment_counter, fake_data)

}}

each of the threads is accessing the same global variable counter and
incrementing it. Counter is not protected in any way, so it is not
thread-safe.

In order to increment counter, each of the threads needs to read the
current value, add one to it, and the save that value back to the
variable. That happens in this line: counter += 1.

Because the operating system knows nothing about your code and can swap
threads at any point in the execution, it’s possible for this swap to
happen after a thread has read the value but before it has had the
chance to write it back. If the new code that is running modifies
counter as well, then the first thread has a stale copy of the data and
trouble will ensue.

# Async IO

The general concept of asyncio is that a single Python object, called
the event loop, controls how and when each task gets run. The event
loop is aware of each task and knows what state it’s in. In reality,
there are many states that tasks could be in, but for now let’s imagine
a simplified event loop that just has two states.

The ready state will indicate that a task has work to do and is ready
to be run, and the waiting state means that the task is waiting for
some external thing to finish, such as a network operation.

Your simplified event loop maintains two lists of tasks, one for each
of these states. It selects one of the ready tasks and starts it back
to running. That task is in complete control until it cooperatively
hands the control back to the event loop.

When the running task gives control back to the event loop, the event
loop places that task into either the ready or waiting list and then
goes through each of the tasks in the waiting list to see if it has
become ready by an I/O operation completing. It knows that the tasks in
the ready list are still ready because it knows they haven’t run yet.

Once all of the tasks have been sorted into the right list again, the
event loop picks the next task to run, and the process repeats. Your
simplified event loop picks the task that has been waiting the longest
and runs that. This process repeats until the event loop is finished.

An important point of asyncio is that the tasks never give up control
without intentionally doing so. They never get interrupted in the
middle of an operation. This allows us to share resources a bit more
easily in asyncio than in threading. You don’t have to worry about
making your code thread-safe.

Now let’s talk about two new keywords that were added to Python: async
and await. In light of the discussion above, you can view await as the
magic that allows the task to hand control back to the event loop. When
your code awaits a function call, it’s a signal that the call is likely
to be something that takes a while and that the task should give up
control.

It’s easiest to think of async as a flag to Python telling it that the
function about to be defined uses await. There are some cases where
this is not strictly true, like asynchronous generators, but it holds
for many cases and gives you a simple model while you’re getting
started.

One exception to this that you’ll see in the next code is the async
with statement, which creates a context manager from an object you
would normally await. While the semantics are a little different, the
idea is the same: to flag this context manager as something that can
get swapped out.

# multiprocessing in a nutshell

Up until this point, all of the examples of concurrency in this
article run only on a single CPU or core in your computer. The
reasons for this have to do with the current design of CPython and
something called the Global Interpreter Lock, or GIL.

This article won’t dive into the hows and whys of the GIL. It’s
enough for now to know that the synchronous, threading, and asyncio
versions of this example all run on a single CPU.

multiprocessing in the standard library was designed to break down
that barrier and run your code across multiple CPUs. At a high
level, it does this by creating a new instance of the Python
interpreter to run on each CPU and then farming out part of your
program to run on it.

As you can imagine, bringing up a separate Python interpreter is not
as fast as starting a new thread in the current Python interpreter.
It’s a heavyweight operation and comes with some restrictions and
difficulties, but for the correct problem, it can make a huge
difference.

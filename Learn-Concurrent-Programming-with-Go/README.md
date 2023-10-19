# Learn Concurrent Programming with Go
## Stepping into concurrent programming
Amdahl’s law tells us that the performance scalability of a fixed-size problem is limited by the non-parallel parts of
an execution. Gustafson’s law tells us that if we keep on finding ways to keep our extra resources busy, the speedup
should continue to increase and not be limited by the serial part of the problem.

- stack space
- registers
- program counter

### Processes
Each operating system process has its own memory space, isolated from other processes. Typically, a process would work
independently with minimal interaction with other processes. Processes provide isolation at the cost of consuming more
resources.

The downside of this is that we end up consuming more memory. In addition, starting up processes takes a bit longer,
since we need to allocate the memory space and other system resources.

#### fork()
Using this call we can create a copy of an execution. When we make this system call, from another executing
process, the operating system makes a complete copy of the memory space and the process’s resource handlers. This
includes registers, stack, file handlers and even the program counter.

Copy on write (COW in short) is an optimization introduced to the fork() system call. It reduces the time wasted by not
copying the entire memory space. For systems using this optimization, whenever fork() is called, both child and parent
process share the same memory pages. If then one of the processes tries to modify the contents of a memory page, that
page is copied to a new page.

### Threads
Creating a thread is much faster (sometimes 100 times faster) and the thread consumes a lot less system resources than a
process. Conceptually, threads are another execution context (kind of a micro-process) within a process.

When you have more than one thread in a single process, we say that the process is multithreaded.

When we create a new thread, the operating system needs to create only enough resources to manage the stack space,
registers, and a program counter.

We also need for each thread to have its own program counter. A program counter is simply a pointer to the instruction that
the CPU will execute next. Since threads will very likely execute different parts of our program, each thread needs to
have a separate instruction pointer as well.

Depending on the thread implementation, whenever a thread terminates, it does not necessarily terminate the entire
process. In Go when the main thread of execution terminates, the entire process also terminates even if other threads
are still running.

IEEE attempted to standardize thread implementations using a standard called POSIX threads (pthreads, for short).
Creation, management, and synchronization of these threads is done through the use of a standard POSIX threads API.

When we run jobs concurrently, we can never guarantee the execution order of those jobs. When our main function creates
the 5 goroutines and submits them, the operating system might pick up the executions in a different order than the one
we created them with.

#### user-level threads
From an operating system point of view, a process containing user-level threads will appear as having just one thread of
execution. The OS doesn’t know anything about user-level threads. The main advantage of user-level threads is
performance. Context switching a user-level thread is faster than context-switching a kernel-level one. This is because
for kernel-level context switches, the OS needs to intervene and choose the next thread to execute. When we can switch
execution without invoking any kernel, `the executing process can keep hold of the CPU without the need to flush its
cache and slow us down.`

`The downside of using user-level threads is when they execute code that invokes blocking IO calls.` If any other
user-level threads are present in the same process, they will not get to execute until the read operation is complete.
`The OS sees the single kernel-level thread, containing all the user-level threads, as a single execution. Thus, the OS
executes the kernel-level thread on a single processor.`

Go attempts to provide a hybrid system that gives us the great performance of user-level threads without most of the
downsides. It achieves this by using `a set of kernel-level threads, each managing a queue of goroutines.` The system
that Go uses for its goroutines is sometimes called M:N threading model. This is when you have M user-level threads
(goroutines) mapped to N kernel-level threads.

`We can also call Go scheduler ourselves in our code to try to get the scheduler to context-switch to another
goroutine.`
In concurrency lingo, this is usually called a yield command. It’s when a thread decides to yield control so that
another thread gets its turn running on the CPU.

### Concurrency and Parallelism
Many developers use the terms concurrency and parallelism interchangeably, sometimes referring to them as the same
concept. We can think of `concurrency as an attribute of the program code` and `parallelism as a property of the
executing program.` Concurrency is about planning how to do many tasks at the same time. Parallelism is about performing
many tasks at the same time.

## Thread communication using memory sharing 
Threads of execution working together toward solving a common problem require some form of communication. This is what
is known as inter-thread communication (ITC), or inter-process communication (IPC) to refer to processes. The type of
communication falls under two main classes:
- memory sharing
- message passing

In this architecture the processor uses a system bus when it needs to read or write from main memory. `Before a processor
uses the bus, it listens to make sure the bus is idle`, and not in use by another processor. Once the bus is free, the
processor places a request for a memory location and goes back to listening and waiting for a reply on the bus.
Microchip engineers worry that `cache-coherence will be the limiting factor` as they scale the number of processor cores.
With many more processors, implementing cache-coherence will become a lot more complex and costly and might eventually
limit the performance. This is what is known as the `Coherency wall.`

`Threads sharing the same memory space, we saw how each thread has its own stack space but shares the main memory of the
process.` Go’s compiler is smart enough to realize when we are sharing memory between goroutines. When it notices this,
it allocates `memory on the heap instead of the stack, even though our variables might look like they are local ones,
belonging on the stack. Escape analysis are the compiler algorithms. Anytime a variable is shared outside the scope of a
function’s stack frame, the variable is allocated on the heap.`

We can tell that a variable escaped to heap memory by `asking the compiler to show the optimization decisions`. We can
do this by using the compile time option `-m`.

Atomic comes from the Greek word atomos meaning indivisible. In computer science when we mention `an atomic operation,
we mean an operation that cannot be interrupted.`

`A critical section in our code is a set of instructions that should be executed without interference` from other
executions affecting the state used in that section. When this interference is allowed to happen, `race conditions`
may arise.

We can run the `Go compiler with the -race command line flag`. With this flag the compiler adds special code to all
memory accesses. This code tracks when different goroutines are reading and writing in memory.

## Synchronization with mutexes
If we had a system with just one processor, we could implement the mutex by just disabling interrupts while a thread is
holding the lock.

Implementation of mutexes involves support from `hardware to provide an atomic test and set operation`. With this
operation, an execution can check a memory location and if the value is what it expects, it updates the memory to a
locked flag value.

When deciding how and when to use, mutexes it’s best to focus on which resources we should protect and discovering where
a critical section starts and ends. This is then balanced on minimizing the number of Lock() and Unlock() calls.

`Note that while correct uses of TryLock do exist, they are rare, and the use of TryLock is often a sign of a deeper
problem in a particular use of mutexes.`

`Readers-writer mutexes give us a variation on standard mutexes in that we only block concurrency when we need to update
a shared resource.` Keep in mind that if multiple goroutines are just reading shared data without updating it, there is
no need for this exclusive access; after all, concurrent reading of shared data does not cause any interference. This
implementation of the read-write lock is `read-preferring`. This means that if we have a consistent number of readers
goroutines hogging the read part of the mutex, a writer goroutine would be unable to acquire the mutex.

## Conditational variables and semaphores
Semaphores go one step further than mutexes in that they `allow us to control how many concurrent goroutines we allow to
execute a certain section at the same time`. In addition, semaphores can be used as a way to store a signal to an
execution.

The cond.Wait() function performs two operations atomically: `It releases the mutex. It blocks the current execution`,
effectively putting the goroutine to sleep. The key to understanding condition variables is to grasp the concept that
the Wait() function releases the mutex and suspends the execution in an atomic manner.

We need to ensure that when we call the signal or broadcast function, there is another goroutine waiting for it;
`otherwise the signal or broadcast is not received by any goroutine and it’s missed`. i.e. we should call these
functions only when we’re holding the associated mutex. In this way we know for sure that the main goroutine is in a
waiting state. This is because the mutex is only released when the goroutine calls Wait().
`Always use Signal(), Broadcast() and Wait() while holding the mutex lock to avoid synchronization problems.`

When we have multiple goroutines suspended on a condition variable’s Wait(), Signal() will only wake up an arbitrary one
of these goroutines. The call Broadcast() on the other hand will wake up all goroutines that are suspended on a Wait().

`Write-starvation` happens when we’re not able to update our shared data structures because the reader parts of the
execution are continuously accessing them, `blocking access to the writer`.

Semaphores give us a different type of concurrency control; in that we can specify the number of concurrent executions
that are permitted. Definition A semaphore with only one permit is sometimes called a binary semaphore.

`Calling Wait() on a condition variable atomically unlocks the mutex and suspends the current execution`.
Calling Signal() resumes the execution of one suspended goroutine that has called Wait(). Calling Broadcast() resumes
the execution of all suspended goroutines that have called Wait(). If we call Signal() or Broadcast() and there are
`no goroutines suspended on a Wait() call, the signal or broadcast is missed`.

## Synchronizing with wait groups and barriers
Wait groups and barriers are two synchronization abstractions that work on groups of goroutines. We typically use `wait
groups to wait for a group of tasks to complete`. On the other hand, we use `barriers to synchronize many goroutines
at a common point`.

Go comes bundled with a WaitGroup implementation in its sync package. It contains the three functions that allow us:
- Done() Decrements the wait group size counter by 1
- Wait() Blocks until the wait group counter size is 0
- Add(delta int) Increments the wait group size counter by delta

Wait groups are great for synchronizing when a task has been completed. What if we need to coordinate our goroutines
before we start a task? We might also need to align different executions at different points in time. Barriers give us
the ability to synchronize groups of goroutines at specific points in our code. Barriers are different from wait groups
in that they combine the wait group’s Done() and Wait() operations together into one atomic call.
`A barrier that can be reused is sometimes called a cyclic barrier.`

This pattern of loading work, waiting for it to complete, and collecting results is a typical application for barriers.
However, it is mostly useful when creating new executions is a fairly expensive operation — for example, `when we use
kernel-level threads. Using this pattern, you save the time taken to create new threads on every load cycle.` We can
implement barriers also using condition variables.

## Communication using message passing
Message passing is another way to have interthread communication (ITC), which is when we have goroutines sending or
waiting for messages to and from other goroutines. `Using message passing, each goroutine has only its own isolated
memory to work with.`

`By default, Go’s channels are synchronous. A sender will block if there isn’t a goroutine consuming its message and
similarly a receiver will also block if there isn’t a goroutine reading its message.` Although channels are synchronous,
we can configure them so that they `store a number of messages before they block again`. When we use a
buffered channel, the sender goroutine will `not block as long as there is space available in the buffer.`

Once the receiver goroutine consumes all the messages and the buffer empties, the receiver goroutine will again block.
A receiver will block in cases:
- we don’t have a sender
- the sender is producing messages at a slower rate than the receiver can read them

`Go’s channels are bidirectional by default.` This means that a goroutine can act as both a receiver and a sender of
messages. However, `we can assign a direction to a channel so that the goroutine using the channel can only send or
receive messages.` if we try to use a receiver’s channel to send messages, `we would get a compilation error`.

`In software development, a sentinel value is a predefined value that signals to an execution, a process, or anr
algorithm to terminate.` In the context of multithreading and a distributed system, sometimes this is referred to
as `a poison pill message`.

Instead of using this sentinel value message, Go gives us the ability to `close a channel`. We can do this in code by
calling the close(channel) function. Once we close a channel, we shouldn’t send any more messages to it, because doing
so raises errors. If we try to receive messages from a `closed channel`, we will get messages containing the `default
value` for the channel’s data type. 

Go gives us a couple of ways to handle closed channels. The best method is to `read an open channel flag whenever we
consume from the channel`. This flag is set to false only when the channel has been closed.

## Selecting channels
`Channels are first-class objects, which means that we can store them as variables, pass or return them from functions,
or even send them on a channel.`

`When using select, if multiple cases are ready, a case is chosen at random. Your code should not rely on the order in
which the cases are specified.`

The select statement gives us the default case for exactly this scenario. `The instructions under the default case will
get executed if none of the other cases is available.` This gives us the ability to try to access one or more channels,
but if none is ready, we can do something else.

`We can use the close operation, on a channel, to act like a signal being broadcasted to all consumers.`

### time.Timer - channel functionality
Thankfully, the time.Timer type in Go provides us with this functionality and we don’t have to implement our own timer
goroutine. We can create one of these timers by calling time.After(duration). This would return a channel on which a
message is sent after the duration time elapses.
`When we use the call time.After(duration), the returned channel will receive a message containing the time when the
message was sent.`

`In Go, we can assign nil values to channels. This has the effect of blocking the channel from sending or receiving
anything.` The same logic applies to select statements. Trying to send to or receive from a nil channel on a select
statement has the same effect of blocking the case using that channel.

`We will end up needlessly looping, on the closed channel select case, receiving 0 every time.`
`WARNING When we use a select case on a closed channel, that case will always execute.`

`Assigning a nil value to the channel variable after the receiver detects that a channel has been closed has the effect
of disabling that case statement. This allows the receiving goroutine to read from the remaining open channels.`

This pattern of `merging channel data into one stream is referred to as a fan-in pattern`. Using the select statement to
merge a different source only works when we have a fixed number of sources.

The terms tightly and loosely coupled software refer to how closely dependent modules are on each other.
- Tightly coupled software means that when we change one component, it will have a ripple effect on many other parts
of the software, usually requiring changes as well.
- In loosely coupled software, components tend to have clear boundaries and few dependencies on other modules.

`Writing loosely coupled software while using memory sharing is more difficult than when using message passing.` This is
because when we change the way we update the shared memory from one execution, it will have a significant effect on the
rest of the application. This is not to say that all code that uses message passing is loosely coupled. Nor is it the
case that all software using memory sharing is tightly coupled. It is just easier to come up with a loosely coupled
design of an application using message passing. This is because we can define simple boundaries of each concurrent
execution with clear input and output channels.

Message passing will degrade the performance of our application if we are spending too much time passing messages
around. Since we pass copies of messages from one goroutine to another, we suffer the performance penalty of spending
the time to copy the data in the message. This extra performance cost is noticeable if the messages are large or if they
are numerous.
- One scenario is when the message size is too large.
- The other scenario is when our executions are very chatty. This is when concurrent executions need to send many
messages to each other.

### Overview
- When multiple channel operations are combined using the select statement, the operation that is unblocked first gets
executed.
- A channel can be made to have non-blocking behavior by using the default case on the select statement.
- Combining the channel operation with a Timer channel on a select statement results in blocking on a channel up to the
specified timeout.
- The select statement can be used not just for receiving messages but also for sending.
- Trying to send to or receive from a nil channel results in blocking the execution.
- Select cases can be disabled when we use nil channels.
- Message passing produces simpler code that is easier to understand. Tightly coupled code results in applications
in which it is difficult to add new features.
- Code written in a loosely coupled way is easier to maintain.
- Loosely coupled software with message passing tends to be simpler and more readable than using memory sharing.
- Concurrent applications using message passing might consume more memory because each execution has its own isolated
state instead of a shared one.
- Concurrent applications requiring the exchange of large chunks of data might be better off using memory sharing
because copying this data for message passing would greatly degrade performance.
- Memory sharing is more suited for applications that would exchange a huge number of messages if they were to use
message passing.

## Programming with channels
`Go’s own mantra is not to communicate by shared memory but to instead share memory by communicating.` Since memory
sharing is more prone to race conditions and requires complex synchronization techniques, when possible we should avoid
it and instead use message passing.

Literally, immutable means unchangeable. In computer programming, we use immutability when we initialize structures
without giving the ability to modify them. When the programming requires changes to these structures, we create a new
copy of the structure containing the required changes, leaving the old copy as is.

Creating a copy when we need to update shared data leaves us with a problem: How do we share the new, updated data that
is now in a separate location in memory? `We need a model to manage and share this new modified data. This is where
message passing and CSP come in handy.`

`CSP, short for communicating sequential processes, is a formal language used to describe concurrent systems. Instead of
using memory sharing, it is based on message passing via channels.`

The key difference when using the CSP model is that executions are not sharing memory. Instead, they pass copies of
data to each other. `Like using immutability, if each execution is not modifying shared data, there is no risk of
interference and thus we avoid most race conditions.` If each execution has its own isolated state, we can eliminate
data race conditions without needing to use complex synchronization logic using mutexes, semaphores, or condition
variables.

One key difference between the CSP model and Go’s implementation is that, in Go, channels are first-class objects,
meaning we can pass them around in functions or even in other channels. This gives us more programming flexibility.

When we are using this model of concurrency in Go, there are two main guidelines to follow:
- `Try to only pass copies of data on channels.` This implies that you shouldn’t pass pointers on channels.
- As much as possible, try `not to mix message passing patterns with memory sharing`.

This pipeline pattern gives us the ability to have executions that can be easily plugged together. Each execution is
represented by a function starting a goroutine accepting input channels as arguments and returning the output channels
as return values.

`In Go, a fan out concurrency pattern is when multiple goroutines read from the same channel. In this way, we can
distribute work among a set of goroutines. Since concurrent processing is nondeterministic, some messages will get
processed quicker than others, resulting in messages being processed in a different order. Thus, the fan out pattern
makes sense only if we don’t care about the order of the incoming messages.`

`In Go, a fan in concurrency pattern occurs when we merge the content from multiple channels into one.`

Instead of fan out, we can use a broadcast pattern, one that replicates messages to a set of output channels. In the
broadcast implementation, we move on to read the next message only after the current message has been
sent to all the channels. `A slow consumer from this broadcast implementation would slow all consumers to the same
rate.`

### Overview
- Communication sequential processes (CSP) is a formal language concurrency model that uses message passing through
synchronized channels.
- Executions in CSP have their own isolated state and do not share memory with other executions.
- A quit channel pattern can be used to notify goroutines to stop their execution.
- Having a common pattern where a goroutine accepts input channels and returns the outputs allows us to easily connect
various stages of a pipeline.
- A fan in pattern merges multiple input channels into one. This merged channel is closed only after all input
channels are closed.
- Fan out is when multiple goroutines are reading from the same channel. In this case, messages on the channel are
`load balanced among the goroutines`.
- The fan out pattern makes sense only when the order of the messages is not important With the broadcast pattern, the
contents of an input channel are replicated to multiple channels.
- In Go, having channels behave as first-class objects means that we can modify the structure of our message passing
concurrent program in a dynamic fashion while the program is executing.

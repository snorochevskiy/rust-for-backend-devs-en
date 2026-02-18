# Threads and I/O

## Concurrency and Parallelism

Before we dive into the topic of asynchrony in Rust, we need to pause for a moment to clarify some terminology. In English, there are two distinct concepts: parallelism and concurrency.

**Parallelism** refers to the simultaneous execution of code on different processors or different cores of a single processor.

Things get a bit more complicated with **concurrency**. When we say code is concurrent, we mean that it is designed in a way that allows for parallel execution.

For example, if we run a multi-threaded program on a processor with only one core, all threads will execute one after another (interleaved), and there will be no "real" physical parallelism. However, all the problems associated with true parallel execution can still occur. For instance, a data race is entirely possible even when running on a single-core processor.

In short: concurrency is about the logical level of parallel execution, while parallelism refers to the physical level.

The asynchronous runtimes discussed in this chapter execute code concurrently, but not necessarily in parallel. That said, the requirements for writing such code are as strict as if it were running on a physical parallel level.

## The Problems with Threads

As we know, backend applications handle many requests. To keep an application responsive, incoming requests must be processed concurrently rather than waiting for each other to finish. The simplest way to achieve this is to handle each request in a separate thread. However, a large number of active threads combined with heavy I/O operations (which are common in backend apps) can become a serious performance bottleneck.

The issue is that OS threads are a "heavy" and expensive resource:

* Creating a new thread requires a system call, which is slow.
* Context switching between OS threads is a time-consuming and costly operation.

><details>
> 
> <summary>Why is a system call slow?</summary>
>
> System calls (syscalls) are the mechanism through which a program running in user space can invoke functionality from the operating system kernel.
> 
> System calls vary slightly depending on the OS and CPU architecture, but the general principle remains the same.
> 
> For example, on Linux with the x86-64 architecture, a system call to create a new thread involves these steps:
> 
> 1. The program writes the ID of the desired system call into the `RAX` register.\
>    (All available syscalls, their IDs, and arguments can be found in the Linux documentation).
> 2. Arguments are passed via registers RDI, RSI, RDX, R10, R8, and R9 in that order.
> 3. The return address from the `syscall` is placed in the RCX register.
> 4. The program executes the syscall instruction, which triggers the system call. In 32-bit Linux, an interrupt (number 60) was used; it still works today, but the `syscall` instruction is preferred.
> 5. After `syscall` is executed, the thread "sleeps," and the CPU switches to kernel mode. In kernel mode, the processor has access to kernel memory and hardware. To start a new thread, the kernel creates a new entry in the thread table and allocates space for the new thread's stack.
> 6. Once the kernel finishes its work, the processor switches back to user mode and resumes the execution of the previously sleeping thread.
> 
> All these context-switching operations between user space and the kernel are quite time-consuming and can negatively impact program performance.
>
> </details>

> <details>
> 
> <summary>Why is switching between threads slow?</summary>
> 
> The execution of threads on CPU cores is managed by the scheduler — a subprogram within the OS kernel. The scheduler decides the sequence and duration for which threads run on the CPU.
> 
> Once a thread has exhausted its time slice (the time allocated for its execution), it is interrupted. Its state (register values) is saved to the thread's stack. The record in the kernel's thread table is updated. Then, the state of the next thread in the queue is restored from its stack, and that thread begins execution.
> 
> This sequence is far from instantaneous. It consumes significant CPU time: roughly one microsecond on a modern processor running Linux.
> 
> f we have 10 threads and each needs to receive its time slice at least once per second, the CPU will spend 10 microseconds on context switching and 999,990 microseconds on execution. That’s only 0.001% of CPU time spent on switching.\
> However, if we have 10,000 threads, the CPU will spend 10,000 microseconds just on switching and 990,000 microseconds on execution. This means 1% of all CPU time is wasted on overhead.\
> The situation is worsened by the fact that the processor tries to give each thread a time slice much more frequently than once per second, meaning context switches happen (and cost) far more often.
> 
> </details>

Consequently, if your program has only a few threads that are mostly performing calculations, OS threads are perfectly fine. But if you have thousands of threads performing frequent I/O operations, a substantial portion of your CPU's power will be wasted just juggling those threads. Clearly, the performance hit will be significant.

## Blocking and Non-blocking I/O

Application backends are typically I/O-bound: they handle HTTP connections, interact with the file system, databases, external queues, distributed caches, and so on. From a threading perspective, there are two types of I/O operations: blocking and non-blocking.

Blocking I/O operations work as follows:

1. A thread makes a system call requesting a blocking I/O operation (for example, reading bytes from an open network connection).
2. The thread then "goes to sleep" and is marked in the thread table as waiting to be woken up after the I/O operation is completed.
3. At some point, the kernel performs the actual data reading from the device and places it into a buffer for the program to read.
4. The entry for the thread waiting for this read operation is updated in the thread table. Now the thread is ready to continue its work.
5. The scheduler allocates a time slice to the thread when its turn comes.
6. The thread wakes up and reads the data already present in the buffer.

Blocking I/O operations are convenient to use. However, when writing backend applications that serve a large number of concurrent requests, using blocking I/O leads to frequent context switching between threads, which significantly impacts performance.

Non-blocking I/O works a bit differently. First, it should be noted that a non-blocking API is practically useless for working with a single I/O device, such as just one network socket.\
(This does not necessarily mean a physical device: for example, several tens of thousands of network sockets can be open on a single network card).

So, let's look at the general outline of how non-blocking I/O works using epoll as an example. epoll is an API for non-blocking I/O in the Linux operating system. Depending on the operating system, the non-blocking API will differ: Linux uses select, poll, epoll, or io_uring, Windows uses IOCP, BSD/MacOS uses kqueue, etc.

Suppose we are building an echo server that accepts a network connection, reads bytes from it, and sends those same bytes back.

1. We receive several connections from clients, and each connection is represented by a network socket. Instead of reading input data for each socket immediately, we simply take the descriptors of these sockets and put them in an array.
2. We pass all descriptors as a single array to the corresponding system call (epoll_wait), specifying which events we are waiting for on these sockets. In our case, it will be an event indicating that data from the socket is ready to be read.
3. The epoll_wait system call blocks the calling thread, and upon waking up, it returns a list of descriptors for which data is ready to be read.
4. We iterate through the list of ready descriptors (received from epoll_wait) and, for each of them, first read all available bytes and then immediately write them back to the socket. After that, we close the socket.
5. Then, we call the epoll_wait system call again for the remaining descriptors. This continues until all of them are processed.

![](img/threads_and_io-non_blocking.svg)

---

<details>

<summary>epoll Example</summary>

For those who have never seen a non-blocking API in action, here is an example in C: a TCP echo server using the [epoll](https://man7.org/linux/man-pages/man7/epoll.7.html) mechanism on Linux. All error handling is omitted for brevity.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/epoll.h>

#define MAX_EVENTS 10

int main(void) {
  int listen_fd = socket(AF_INET, SOCK_STREAM, 0); // Create server socket

  int options = 1; // Allow address reuse
  setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &options, sizeof(options));

  // Bind the socket to port 8081 and start listening
  struct sockaddr_in addr;
  addr.sin_family = AF_INET;
  addr.sin_addr.s_addr = INADDR_ANY;
  addr.sin_port = htons(8081);
  bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr));
  listen(listen_fd, SOMAXCONN);

  // Create epoll non-blocking I/O multiplexer
  int epoll_fd = epoll_create1(0);

  // Add the server socket to epoll
  struct epoll_event ev;
  ev.events = EPOLLIN;
  ev.data.fd = listen_fd;
  epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev);

  // Loop and wait for a descriptor ready for work
  while (1) {
    struct epoll_event events[MAX_EVENTS];
    // Wait for ready I/O descriptors
    int ready_num = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);

    // Iterate through all descriptors ready for processing
    for (int i = 0; i < ready_num; i++) {
      if (events[i].data.fd == listen_fd) {
        // If the ready descriptor is our server socket,
        // it means a new network connection has arrived.
        // Accept it and add the new connection to epoll.
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);
        int conn_fd = accept(listen_fd, (struct sockaddr *)&client_addr, &client_len);

        printf("New: %s:%d\n", inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));

        ev.events = EPOLLIN;
        ev.data.fd = conn_fd;
        epoll_ctl(epoll_fd, EPOLL_CTL_ADD, conn_fd, &ev);
      } else {
        // Otherwise, it's I/O from a client
        int client_fd = events[i].data.fd;
        char buf[1024];
        ssize_t read_bytes = read(client_fd, buf, sizeof(buf));
        if (read_bytes <= 0) {
          // If everything has been read from the client, close
          // the connection and remove its descriptor from epoll
          close(client_fd);
          epoll_ctl(epoll_fd, EPOLL_CTL_DEL, client_fd, NULL);
        } else {
          // Otherwise, immediately write the read bytes back to the client
          write(client_fd, buf, read_bytes);
        }
      }
    }
  }

  close(listen_fd);
  close(epoll_fd);
  return 0;
}
```

As you can see, the entire workflow is built around an infinite loop waiting for ready descriptors using the `epoll_wait` call, which prevents writing simple linear code.

</details>

---

To emphasize:

* With a blocking API, each socket is handled by a separate thread, and this thread goes to sleep on every blocking read or write call.
* With a non-blocking API, we use <ins>one</ins> thread to iteratively work with a batch of sockets, and this single thread goes to sleep until at least one socket from the batch is ready for further work.

However, there is a significant downside: while the blocking API allows writing code in a simple and clear linear style, the non-blocking API requires a specific approach to bridge the gap between the program's linear business logic and the centralized I/O management.

There are several such approaches.

## Working with Non-blocking APIs

There are several standard architectural approaches to separating the I/O event polling loop from the program logic:

* Event loop and callbacks
* Actor model
* Fibers

Let’s take a closer look at each of them.

### Event Loop and Callbacks

If you happened to write JavaScript before Promises were introduced, you likely remember that the results of all I/O operations were handled using callbacks.

> [!NOTE]
> An HTTP request would look something like this:
> 
> ```javascript
> // Create an HTTP request object
> const xhttp = new XMLHttpRequest();
> // Assign a callback to the onreadystatechange field
> xhttp.onreadystatechange = function() { // Callback function
>   if (this.readyState == 4 && this.status == 200) {
>     console.log(this.responseText);
>   }
> };
> xhttp.open("GET", "http://somehost/some_resource");
> xhttp.send();
> ```

The idea is as follows:

We run an infinite loop that accepts "requests" for I/O operations. Usually, a queue is used to deliver these requests to the loop. Each request contains information about the I/O operation itself and a callback function to be executed once the operation finishes.

This way, the thread executing the program logic simply sends a request (e.g., to perform an HTTP request) and attaches a callback containing the code that should process the result. After sending the request, the thread is not blocked; it continues with other tasks.

Later, the request is pulled from the queue by the I/O event loop, which:

1. Opens a network connection.
2. Adds the connection descriptor to a list of descriptors waiting for a "ready to write" signal.
3. Waits for the connection to be ready for writing.
4. Writes the HTTP request.
5. Waits for the connection to be ready for reading the response.
6. Reads the response.
7. Closes the connection.
8. Invokes the callback function, passing the HTTP response into it.

![](img/threads_and_io-callback.svg)

The advantage of this approach is its simplicity of implementation. No special language constructs are required: almost all mainstream languages provide both queues and the ability to pass functions as arguments.

The obvious disadvantage: if the program logic requires a series of sequential I/O operations, the code turns into a "ladder" of callbacks.

```rust
http_call("http://xxx/service_1", |response| {
    // do something with response
    http_call("http://xxx/service_2", |response| {
       // do something with response
       http_call("http://xxx/service_3", |response| {
           // do something with response
       });
    });
});
```

> [!NOTE]
> One variation of the event loop approach worth mentioning is Reactive Programming.
> 
> Reactive programming frameworks are also built around an event loop but provide an API that allows you to write "callbacks" in a way that looks relatively linear. This helps solve the "callback hell" problem that arises with the "classic" event loop.
> 
> Reactive programming gained popularity in Java, where concurrency issues became as acute as in other languages, yet the language initially lacked an async-await mechanism. This forced developers to seek alternative approaches. However, starting with Java 21, virtual threads were added to the language, significantly reducing the need for the reactive approach.

### Actor Model

Another common way to integrate an event loop with application business logic is the Actor Model.

Strictly speaking, the actor model wasn't created specifically to solve non-blocking I/O problems. It offers an alternative approach to writing multi-threaded systems and provides a mechanism that simplifies building applications for networked clusters. However, the way components interact in actor-based programs makes it relatively easy to move the non-blocking I/O loop into a separate component that is easy to interact with.

The core idea is that a program is represented as a set of actors managed by an actor system. Each actor consists of:

* A "Mailbox" — a queue where messages addressed to the actor arrive. Actors do not call each other like functions; they send messages.
* An actor function used to process the next message from the mailbox.
* State: Actors can have internal state (fields) that can change as the actor operates.

An actor system is a framework responsible for delivering messages to mailboxes and periodically executing an actor's function on one of the OS threads.

With this architecture, you can encapsulate all non-blocking I/O logic into a dedicated I/O actor. Then:

* Actors wishing to perform an I/O operation send a corresponding message to the I/O actor.
* The I/O actor pulls I/O requests from its mailbox and processes them using epoll or another non-blocking API. The results are then sent back as messages to the actors that initiated the request.

Let’s illustrate with a diagram how a simple backend application built on an actor system might look. The two main actors we are interested in are:

* The I/O Actor, which handles I/O operations.
* A Worker Actor, which receives user requests and, to respond to them, calls another backend service via HTTP. The HTTP interaction is handled through the I/O actor.

![](img/threads_and_io-actors.svg)

The actor model is excellent for applications whose logic fits within the framework of asynchronous message passing. For example, Erlang/OTP has proven itself as a platform with superb scalability and fault tolerance (the backend of the popular WhatsApp messenger is built on it). However, for writing programs with strictly linear logic, the actor model may feel cumbersome.

### Fibers

This model is the most flexible and modern, appearing under various names: coroutines, green threads, fibers, or user-space threads.

The idea is as follows: if creating OS threads is expensive and switching between them is slow, we can create our own scheduler and lightweight user-space threads within our program. From here on, we will call these lightweight threads "fibers." Naturally, we will run our fibers on top of OS threads, but since fibers are created and switched by our own scheduler in user space, we can avoid the long and costly context switches of OS threads.

A fiber can be implemented in several ways, but it must allow for pausing execution at specific points and resuming from where it left off.

> [!TIP]
> <details>
> 
> <summary>How can you build a simple fiber?</summary>
> 
> If you have never dealt with user-space threads, it might be difficult to imagine how to create your own thread and, furthermore, how to make it stoppable.
> 
> The simplest fiber can be implemented as an array/vector of closures such that the result of one closure becomes the input argument for the next one: `[()->A, A->B, B->C]`.
> 
> Let's look at an example of a simple fiber built from `Fn(T)->R` objects.
> 
> ```rust
> use std::{any::Any, collections::VecDeque, marker::PhantomData};
> 
> // Abstraction for a fiber element. Needed to abstract away from
> // specific types in the Fn(T)->R closure.
> // This allows storing closures of different types in a single vector.
> // For example: [Fn(String)->i32, Fn(i32)->bool]
> trait Stage {
>     fn exec(&self, arg: Box<dyn Any>) -> Box<dyn Any>;
> }
> 
> // A wrapper around the Fn(T)->R closure
> struct Func<T: 'static, R: 'static>(Box<dyn Fn(T) -> R>);
> 
> // A fiber implemented as a chain of closures abstracted via Stage.
> // The chain itself behaves like a closure Fn(())->R, where () is the argument
> // type of the first closure, and R is the result type of the last one.
> // i.e., [()->String, String->i32, i32->bool] reduces to ()->bool
> struct Fiber<R> {
>     // The chain of closures that make up the fiber
>     stages: VecDeque<Box<dyn Stage>>,
>     // The resulting type of the entire fiber
>     result_type: PhantomData<R>,
> }
> 
> impl<R: 'static> Fiber<R> {
>     // Creates a fiber from a closure
>     pub fn from(f: impl Fn(()) -> R + 'static) -> Fiber<R> {
>         let func = Func(Box::new(f));
>         let stage: Box<dyn Stage> = Box::new(func);
>         let mut stages = VecDeque::new();
>         stages.push_back(stage);
>         Fiber {
>             stages,
>             result_type: PhantomData::<R>,
>         }
>     }
> 
>     // Adds another closure to the end of the fiber chain
>     pub fn compose<R2: 'static>(self, f: impl Fn(R) -> R2 + 'static) -> Fiber<R2> {
>         let func = Func(Box::new(f));
>         let mut stages = self.stages;
>         stages.push_back(Box::new(func));
>         Fiber {
>             stages,
>             result_type: PhantomData::<R2>,
>         }
>     }
> 
>     // Executes the entire fiber
>     pub fn run(mut self) -> R {
>         let mut arg: Box<dyn Any> = Box::new(());
>         while let Some(stage) = self.stages.pop_front() {
>             arg = stage.exec(arg);
>         }
>         *arg.downcast().unwrap()
>     }
> }
> 
> impl<T: 'static, R: 'static> Stage for Func<T, R> {
>     // Executes the closure representing a single link in the fiber
>     fn exec(&self, arg: Box<dyn Any>) -> Box<dyn Any> {
>         let t: T = *arg.downcast().unwrap();
>         let r = self.0.as_ref()(t);
>         Box::new(r)
>     }
> }
> 
> fn main() {
>     let fiber = Fiber::from(|()| 1)
>         .compose(|a| a + 1)
>         .compose(|a| a * 2)
>         .compose(|a| format!("result: {a}"));
>     println!("{}", fiber.run()) // result: 4
> }
> ```
> 
> What interests us most here is the `Fiber::run` method. In it, the closures that make up the fiber are executed one after another. It is precisely in the gaps between these closure calls that we can "pause" the fiber's execution.
> 
> For example:
> 
> ```rust,noplayground
> // Fiber execution will return its result wrapped in this enum
> enum RunResult<T: 'static> {
>     struct Suspended { // fiber is paused
>         arg: Any,
>         fiber: Fiber<T>
>     },
>     Finish(T), // fiber has finished its work
> }
> ...
> pub fn run(mut self) -> RunResult<R> {
>     let mut arg: Box<dyn Any> = Box::new(());
>     while let Some(stage) = self.stages.pop_front() {
>         arg = stage.exec(arg);
>         if fiber execution stop condition {
>             return RunResult::Suspended { arg, fiber: self};
>         }
>     }
>     RunResult::Finish(*arg.downcast().unwrap())
> }
> ```
> 
> Thus, during fiber execution, we can check a certain condition and, if necessary, stop the fiber, returning the remaining part of the closure chain that hasn't been executed yet. Having this remaining chain and the intermediate value where execution was stopped, we can always resume the fiber later.
> 
> </details>

A typical fiber scheduler usually has two OS thread pools at its disposal:

* Threads for executing the fibers themselves.
* A thread for performing I/O operations (I/O thread) initiated by fibers.

With such a scheduler specifically for our fibers, we can implement our own I/O API that works like this:

1. An I/O operation is called within fiber X.
2. The scheduler removes this fiber from execution on the OS thread and puts it into a waiting queue. It then picks another fiber ready for execution from the queue and resumes it on the OS thread.
3. The I/O request is sent to a dedicated I/O thread, which processes I/O requests in a loop using epoll or another similar API.
4. When the dedicated I/O thread completes the operation requested by fiber X, it saves the result for that fiber and marks the fiber itself as ready to resume.
5. At some point, the fiber scheduler resumes fiber X on an OS thread. By this time, the I/O result is already available to the fiber.

![](img/threads_and_io-fibers.svg)

Fibers and coroutines are exactly what we will be discussing further in this section.

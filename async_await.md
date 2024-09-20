# Async/Await Done Right

Several weeks ago, Taylor gave an excellent presentation on how async/await is implemented in Javascript.

<!-- pause -->

There's more than one way that async/await can be implemented, though.

<!-- end_slide -->

## There's more than one way to pluck a buzzard 🪶

When it comes to async/await, there are three dominant strategies in use today.

<!-- pause -->

1. No sugar. Interface with the executor or operating system directly. Until recently, this was how most systems worked.

<!-- pause -->

2. Greedy task execution, where async task hits a callback with the result when complete.

<!-- pause -->

3. Lazy task execution, where a task result is polled for.

<!-- pause -->

The first option cannot rightly be considered `async/await`. It is somewhat challenging, and can be hard to get right.
Talyor presented the second option. I am going to present the third, and make a case for why it is the best approach.

<!-- end_slide -->

## First, let's get on the same terms 📚

### Syntax Sugar

Special code that expands to more code, or that is effectively treated by an interpreter as if it were expanded.

```js
// from this

async function foo() {
  // do stuff...

  return bar;
}

// to this

function foo(onDone) {
  spawnGreenThreadSomehow(() => {
    // do stuff...

    onDone(bar);
  });
}
```

### Task

A unit of work in an async program.

### Executor

A program that drives an event loop of tasks to completion.

### Stack

A part of the CPU that can store values. Retrieval of these values is very fast. A program's stack size is often
relatively small.

<!-- end_slide -->

### Heap

Where all the rest of the memory is. Allocations here can take an infinite amount of time in the worst case.

### Pointer

A 32 or 64 bit reference to a location on the heap.

A _fat pointer_ is a 64 or 128 ponter that includes a normal pointer and a length. Fat pointers are used when the memory
size of a value cannot be determined prior to compilation.

A pointer can also reference a _virtual address_, which is a static location in the currently running binary. The term
_function pointer_ generally refers to a pointer that references such an address. Things are normally setup so that,
some way or other, the CPU performs a `jmp` instruction to that address, reading the following bytes of memory as
executable instructions.

> Values on the stack can also be referenced via a pointer but there is often language-specific nuance to this process,
> so I will not delve into it.

### Boxing

The practice of allocating a place on the heap for a value and storing a pointer to it on the current context (usually
the stack).

### Closure

A value made up of a function pointer and the values or pointers that were captured in that function's scope.

### Channel

A way of sending and receiving data where the current thread will be blocked unil a value is ready. The gold standard
here is the golang implementation.

<!-- end_slide -->

## Time to get sweet 🍭

What is the difference between greedy callbacks and lazy polling?

I alluded to this earlier, but the most fundamental difference is in the syntactical sugar. While callback-based systems
desugar into a function and a callback, polling-based systems will tend to desugar like this:

```js
// from this

async function foo() {
  await somethingLong();
  return "Hello";
}

// to this

function foo() {
  return new FooPromise();
}

class FooPromise {
  state1 = null;
  state2 = null;

  poll() {
    if (this.state2) {
      return "Hello";
    } else if (this.state1) {
      if (this.state1.poll()) {
        this.state2 = true;
      }
      return null;
    } else {
      this.state1 = somethingLong();
      return null;
    }
  }
}
```

<!-- pause -->

Note that the promise does not start executing until it is polled. This is why this async/await model is called "lazy".

<!-- end_slide -->

## Concurrency

But if the promise does not do anything until it is polled, how does one run two promises at the same time?

<!-- pause -->

Actually, it's pretty easy:

```js
class AllPromise {
  promises;
  results = [];

  constructor(promises) {
    this.promises = promises;
  }

  poll() {
    let completedCount = 0;

    for (let index = 0; index < this.promises.length; index++) {
      if (this.results[index]) {
        completedCount++;
        continue;
      }

      let result = this.promises[index].poll();
      if (result) {
        this.results[index] = result;
        completedCount++;
      }
    }

    if (completedCount == this.promises.length) return this.results;
    else return null;
  }
}

function all(promises) {
  return new AllPromise(promises);
}

async function allTheFoos() {
  const [result1, result2, result3] = await all([foo(), foo(), foo()]);
}
```

<!-- end_slide -->

## An Executor

Given the previous interfaces for what a promise looks like, at it's simplest, an executor would look like this:

```js
function blockAndDrive(promise) {
  while (true) {
    let result = promise.poll();
    if (result) return result;
  }
}

blockAndDrive(allTheFoos());
```

<!-- pause -->

This example is not ideal because the CPU will be benched out at 100% capcacity while tasks are running, but you get the
idea.

<!-- end_slide -->

## How to actually run async

By default, most operating system don't actually run stuff async. In fact, pretty much all filesystem apis are blocking
in nature. This means that the entire program will be halted, taken out of the current CPU context (so that other
programs can have a turn to do their computing), and put back on sometime after the call is finished.

```js
function foo() {
  const result = readFile("readme.md"); // the program is entirely blocked by this call

  // ...
}
```

<!-- pause -->

To solve this, most systems spawn an OS thread to do the work for them. Here is an example:

```js
function asyncReadFile(path) {
  return new ReadFile(path);
}

class ReadFile {
  path;
  didSpawn = false;
  result = null;

  constructor(path) {
    this.path = path;
  }

  poll() {
    if (!this.didSpawn) {
      spawnOsThread(() => {
        this.result = readFile(this.path);
      });

      didSpawn = true;
      return null;
    }

    return result;
  }
}
```

<!-- end_slide -->

## Unblocking the CPU

As mentioned previously, especially when paired with the previous `asyncReadFile` implementation, the CPU will be
blocked. A readFile operation is expensive. Not only does it make a potentially large heap allocation, but the worker
thread will be blocked, and depending on the number of CPU cores and amount of other programs running, the program might
even deadlock.

The solution here is to create a centralized system for the worker threads, so that the executor can sync it's calls
`poll` with the completion of worker threads.

```js
class Runtime {
  workerSenders = [];
  completeTaskReciever;

  constructor(workerCount) {
    const (completeTaskSender, completeTaskReciever) = channel()

    for (let index = 0; index < workerCount; index++) {
      const (sender, receiver) = channel();

      workerSenders[index] = spawnOsThread(() => {
        while (true) {
          const closure = reciever.recv();
          if (!closure) break;

          closure();
          completeTaskSender.send(`worker ${index}`);
        }
      }) 
    }

    this.completeTaskReciever = completeTaskReciever;
  }

  spawnGreenThread(closure) {
    const sender = this.chooseWorker(); // this part here is pretty interesting... will revist
    sender.send(closure);
  }

  blockTillNextPoll() {
    completeTaskReciever.recv();
  }
}
```

<!-- end_slide -->

## A smarter executor 🧠

Using our new runtime, we can modify our executor and `asyncReadFile` to use our smart green threads.

```js
const RUNTIME = new Runtime(8);

function blockAndDrive(promise) {
  while (true) {
    let result = promise.poll();
    if (result) return result;

    RUNTIME.blockUntilNextPoll();
  }
}
```

```js
class ReadFile {
  // ...

  poll() {
    if (!this.didSpawn) {
      RUNTIME.spawnGreenThread(() => {
        this.result = readFile(this.path);
      });

      didSpawn = true;
      return null;
    }

    return result;
  }
}
```

<!-- end_slide -->

## Truly async

Although most syscalls are not asyncronous at their lowest level, there is a new one on linux that is. It is called
`epoll` and it is for monitoring file descriptors for changes. Here's some information pulled from `man epoll`:

---

The following system calls are provided to create and manage an epoll instance:

- epoll_create(2) creates a new epoll instance and returns a file descriptor referring to that instance. (The more
  recent epoll_create1(2) extends the functionality of epoll_create(2).)
- Interest in particular file descriptors is then registered via epoll_ctl(2), which adds items to the interest list of
  the epoll instance.
- epoll_wait(2) waits for I/O events, blocking the calling thread if no events are currently available. (This system
  call can be thought of as fetching items from the ready list of the epoll instance.)

---

It's pretty simple and we can implement it with relative ease in our runtime.

```js
class Runtime {
  epollDescriptor;

  constructor() {
    // ...

    this.epollDescriptor = epollCreate();

    spawnOsThread(() => {
      while (true) {
        epollWait(this.epollDescriptor);
        completeTaskSender.send("epoll");
      }
    });
  }

  watchFile(descriptor) {
    epollCtl(epollDescriptor, descriptor);
  }
}
```

<!-- end_slide -->

From my understanding, for various technical reasons, epoll can really only be effectively used with network file
descriptors, and not normal files. I suppose this makes sense.

```js
async function readFromTcpSocket(someTcpSocket) {
  return new ReadFromTcpSocketPromise(someTcpSocket);
}

class ReadFromTcpSocketPromise {
  socket;
  didWatch = false;
  data = new SomeSortOfGrowableByteBuffer();
  thisData = new Uint8Array(1024);

  constructor(socket) {
    this.socket = socket;
  }

  poll() {
    if (!this.didWatch) {
      RUNTIME.watchFile(this.socket.fileDescriptor);
      this.didWatch = true;
    }

    while (this.socket.hasData) {
      this.socket.read(this.thisData);
      this.data.append(this.thisData);
    }

    if (this.socket.didReachEof) {
      closeFile(this.socket.fileDescriptor);
      return this.data.arrayBuffer();
    }

    return null;
  }
}
```

<!-- end_slide -->

## Pushing it parallel

As long as you create the runtime with more than 1 thread, you could do parallel computation with relative ease by
spawning multiple green threads.

Although this is simple from an async perspective, there are a vast number of more language-specific constraints to
consider in this realm.

<!-- end_slide -->

## Worker stealing

<!-- end_slide -->

## Memory layout

One advantage that polling-based async has over callback-based async is the memory layout.

- It has a strict top-down and bottom up control flow.
- The call-stack is not infinite, constantly growing for the duration of the program.
  - Of course, you can infuse complexity everywhere by

<!-- end_slide -->

## Performance

Callback based async ends up having to spawn more green thread than polling-based operations.

This is because in eager callback-based async promises must start running immediately. In polling-based async, a green
thread is only spawned when parallel computation is required, or when the OS does not provide an async syscall for a
given api.

<!-- pause -->

Especially when paired with a garbage collection system, all callback-based promises will need to be boxed. This further
decreases performance.

In a polling-based system, most promises can be kept on the stack.

<!-- end_slide -->

## More predictable

Because of the top-in and bottom-out nature of polling and lazy execution, it is very predictable. No queue, microqueue,
and other subtle oddities.

<!-- end_slide -->

## What do you want me to do?

Nothing really. Javascript will likey never switch to a polling-based async model. But perhaps you could learn rust, as
they have done an excellent job implementing a polling-based async model.

<!-- pause -->

### On returning null

It is not a good idea to have `poll` return `null` until the promise has resolved because if a promise is supposed to
return null, it would really break things.

In rust, an enum is used:

```rust
enum Poll<T> {
  Pending,
  Ready(T)
}
```

# Don't Block the Event Loop (or the Worker Pool)

## Should you read this guide?

If you're writing anything more than a brief command-line script, this should help you write faster, more-secure applications.

This targets Node.js servers, but the concepts apply to complex Node.js applications too. Where OS details vary, it is Linux-centric.

## Summary

Node.js runs JavaScript in the Event Loop (initialization and callbacks), and offers a Worker Pool for expensive tasks like file I/O. It scales well, sometimes better than heavyweight approaches like Apache. The secret: it uses few threads to handle many clients, spending your system's time and memory on clients rather than per-thread overheads (memory, context-switching). But with few threads, you must structure your application to use them wisely.

A good rule of thumb for a speedy server: _Node.js is fast when the work associated with each client at any given time is "small"_.

This applies to callbacks on the Event Loop and tasks on the Worker Pool.

## Why should I avoid blocking the Event Loop and the Worker Pool?

Node.js uses few threads to handle many clients. Two types: one Event Loop (aka the main loop, main thread, event thread), and a pool of `k` Workers in a Worker Pool (aka the threadpool).

If a thread takes a long time to execute a callback (Event Loop) or a task (Worker), we call it "blocked". While blocked working for one client, it cannot handle any other clients. Two reasons not to block either:

1. Performance: regular heavyweight activity on either thread hurts your server's _throughput_ (requests/second).
2. Security: if certain input can block a thread, a malicious client could submit this "evil input" to block your threads and starve other clients: a Denial of Service attack.

## A quick review of Node

Node.js uses the Event-Driven Architecture: it has an Event Loop for orchestration and a Worker Pool for expensive tasks.

### What code runs on the Event Loop?

Node.js applications first run an initialization phase, `require`'ing modules and registering event callbacks. They then enter the Event Loop, responding to client requests by executing the appropriate callback. The callback executes synchronously, and may register asynchronous requests that continue after it completes. Those requests' callbacks also execute on the Event Loop.

The Event Loop also fulfills the non-blocking asynchronous requests made by its callbacks, e.g., network I/O.

### What code runs on the Worker Pool?

Node.js's Worker Pool is implemented in libuv, which exposes a general task submission API.

Node.js uses the Worker Pool for "expensive" tasks: I/O for which the operating system provides no non-blocking version, and particularly CPU-intensive tasks.

The Node.js module APIs that use the Worker Pool:

1. I/O-intensive
   1. DNS: `dns.lookup()`, `dns.lookupService()`.
   2. File System: All file system APIs except `fs.FSWatcher()` and those that are explicitly synchronous use libuv's threadpool.
2. CPU-intensive
   1. Crypto: `crypto.pbkdf2()`, `crypto.scrypt()`, `crypto.randomBytes()`, `crypto.randomFill()`, `crypto.generateKeyPair()`.
   2. Zlib: All zlib APIs except those that are explicitly synchronous use libuv's threadpool.

In many applications these APIs are the only sources of Worker Pool tasks. Applications and modules using a C++ add-on can submit other tasks.

When you call one of these APIs from an Event Loop callback, the Event Loop pays minor setup costs entering the Node.js C++ bindings and submitting a task to the Worker Pool. These are negligible compared to the task itself, which is why the Event Loop offloads it. Node.js passes a pointer to the corresponding C++ function in the bindings.

### How does Node.js decide what code to run next?

Abstractly, the Event Loop and the Worker Pool maintain queues for pending events and pending tasks, respectively.

In truth, the Event Loop maintains no queue. Instead it asks the operating system to monitor a collection of file descriptors, using a mechanism like epoll (Linux), kqueue (OSX), event ports (Solaris), or IOCP (Windows). These descriptors correspond to network sockets, watched files, and so on. When the OS says a descriptor is ready, the Event Loop translates it to the appropriate event and invokes the associated callback(s).

In contrast, the Worker Pool uses a real queue of tasks. A Worker pops a task, works on it, and when finished raises an "At least one task is finished" event for the Event Loop.

### What does this mean for application design?

In a one-thread-per-client system like Apache, each pending client gets its own thread. If a thread handling one client blocks, the operating system interrupts it and gives another client a turn, ensuring clients that need little work aren't penalized by clients that need more.

Because Node.js handles many clients with few threads, if a thread blocks on one client's request, pending requests may not get a turn until it finishes its callback or task. _The fair treatment of clients is thus the responsibility of your application_: don't do too much work for any client in a single callback or task.

This is part of why Node.js can scale well, but it also makes you responsible for fair scheduling. The next sections cover fair scheduling for the Event Loop and the Worker Pool.

## Don't block the Event Loop

The Event Loop notices each new client connection and orchestrates the response. All requests and responses pass through it. So if it spends too long at any point, all current and new clients will not get a turn.

Never block the Event Loop: each JavaScript callback should complete quickly. This also applies to your `await`'s, `Promise.then`'s, and so on.

Reason about the "computational complexity" of your callbacks. If a callback takes a constant number of steps regardless of its arguments, every pending client gets a fair turn. If the step count varies with the arguments, think about how long the arguments might be.

Example 1: A constant-time callback.

```js
app.get("/constant-time", (req, res) => {
  res.sendStatus(200);
});
```

Example 2: An `O(n)` callback. Quick for small `n`, slower for large `n`.

```js
app.get("/countToN", (req, res) => {
  const n = req.query.n;

  // n iterations before giving someone else a turn
  for (let i = 0; i < n; i++) {
    console.log(`Iter ${i}`);
  }

  res.sendStatus(200);
});
```

Example 3: An `O(n^2)` callback. Still quick for small `n`, but for large `n` much slower than the previous `O(n)` example.

```js
app.get("/countToN2", (req, res) => {
  const n = req.query.n;

  // n^2 iterations before giving someone else a turn
  for (let i = 0; i < n; i++) {
    for (let j = 0; j < n; j++) {
      console.log(`Iter ${i}.${j}`);
    }
  }

  res.sendStatus(200);
});
```

### How careful should you be?

Node.js uses the Google V8 engine, which is quite fast for many common operations. Exceptions are regexps and JSON operations, discussed below.

For complex tasks, consider bounding the input and rejecting inputs that are too long. Then, even with large complexity, the callback cannot exceed the worst-case time on the longest acceptable input. You can evaluate that cost and decide whether it's acceptable in your context.

### Blocking the Event Loop: REDOS

One common way to block the Event Loop disastrously is by using a "vulnerable" regular expression.

#### Avoiding vulnerable regular expressions

A regular expression (regexp) matches an input string against a pattern. We usually think of a match as a single pass through the input --- `O(n)` time, where `n` is the input length. Often a single pass is all it takes. But some matches require an exponential number of trips through the input --- `O(2^n)` time: if the engine needs `x` trips to determine a match, it needs `2*x` after one more character is added. Since trips relate linearly to time, this blocks the Event Loop.

A _vulnerable regular expression_ is one on which your engine might take exponential time, exposing you to REDOS on "evil input". Whether a pattern is vulnerable (i.e. the engine might take exponential time on it) is actually a difficult question to answer, and varies depending on whether you're using Perl, Python, Ruby, Java, JavaScript, etc., but here are some rules of thumb that apply across all of these languages:

1. Avoid nested quantifiers like `(a+)*`. V8's regexp engine can handle some of these quickly, but others are vulnerable.
2. Avoid OR's with overlapping clauses, like `(a|a)*`. Again, these are sometimes-fast.
3. Avoid using backreferences, like `(a.*) \1`. No regexp engine can guarantee evaluating these in linear time.
4. If you're doing a simple string match, use `indexOf` or the local equivalent. It will be cheaper and will never take more than `O(n)`.

If you aren't sure whether your regular expression is vulnerable, remember that Node.js generally doesn't have trouble reporting a _match_ even for a vulnerable regexp and a long input string. The exponential behavior is triggered when there is a mismatch but Node.js can't be certain until it tries many paths through the input string.

#### A REDOS example

An example vulnerable regexp exposing its server to REDOS:

```js
app.get("/redos-me", (req, res) => {
  const filePath = req.query.filePath;

  // REDOS
  if (filePath.match(/(\/.+)+$/)) {
    console.log("valid path");
  } else {
    console.log("invalid path");
  }

  res.sendStatus(200);
});
```

This regexp is a (bad!) way to check for a valid path on Linux. It matches strings that are a sequence of "/"-delimited names, like "/a/b/c". It is dangerous because it violates rule 1: it has a doubly-nested quantifier.

If a client queries with filePath `///.../\n` (100 /'s followed by a newline character that the regexp's "." won't match), then the Event Loop will take effectively forever, blocking the Event Loop. This client's REDOS attack causes all other clients not to get a turn until the regexp match finishes.

So be leery of using complex regular expressions to validate user input.

#### Anti-REDOS Resources

Some tools check your regexps for safety:

- safe-regex
- rxxr2.

However, neither of these will catch all vulnerable regexps.

Another approach is a different regexp engine. The node-re2 module uses Google's blazing-fast RE2 regexp engine. But be warned: RE2 is not 100% compatible with V8's regexps, so check for regressions if you swap it in, and node-re2 doesn't support particularly complicated regexps.

To match something "obvious" like a URL or a file path, find an example in a regexp library or use an npm module, e.g. ip-regex.

### Blocking the Event Loop: Node.js core modules

Several Node.js core modules have synchronous expensive APIs, including:

- Encryption
- Compression
- File system
- Child process

These are expensive because they involve significant computation (encryption, compression), require I/O (file I/O), or potentially both (child process). They are intended for scripting convenience, not for use in the server context. On the Event Loop they take far longer to complete than a typical JavaScript instruction, blocking the Event Loop.

In a server, _you should not use the following synchronous APIs from these modules_:

- Encryption:
  - `crypto.randomBytes` (synchronous version)
  - `crypto.randomFillSync`
  - `crypto.pbkdf2Sync`
  - You should also be careful about providing large input to the encryption and decryption routines.
- Compression:
  - `zlib.inflateSync`
  - `zlib.deflateSync`
- File system:
  - Do not use the synchronous file system APIs. For example, if the file you access is in a distributed file system like NFS, access times can vary widely.
- Child process:
  - `child_process.spawnSync`
  - `child_process.execSync`
  - `child_process.execFileSync`

This list is reasonably complete as of Node.js v9.

### Blocking the Event Loop: JSON DOS

`JSON.parse` and `JSON.stringify` are also potentially expensive. Though `O(n)` in input length, for large `n` they can take surprisingly long.

If your server manipulates JSON objects, particularly those from a client, be cautious about the size of the objects or strings you work with on the Event Loop.

Example: JSON blocking. We create an object `obj` of size 2^21, `JSON.stringify` it, run `indexOf` on the string, then `JSON.parse` it. The `JSON.stringify`'d string is 50MB. It takes 0.7 seconds to stringify, 0.03 seconds to indexOf on the 50MB string, and 1.3 seconds to parse.

```js
let obj = { a: 1 };
const iterations = 20;

// Expand the object exponentially by nesting it
for (let i = 0; i < iterations; i++) {
  obj = { obj1: obj, obj2: obj };
}

// Measure time to stringify the object
let start = process.hrtime();
const jsonString = JSON.stringify(obj);
let duration = process.hrtime(start);
console.log("JSON.stringify took", duration);

// Measure time to search a string within the JSON
start = process.hrtime();
const index = jsonString.indexOf("nomatch"); // Always -1
duration = process.hrtime(start);
console.log("String.indexOf took", duration);

// Measure time to parse the JSON back to an object
start = process.hrtime();
const parsed = JSON.parse(jsonString);
duration = process.hrtime(start);
console.log("JSON.parse took", duration);
```

Some npm modules offer asynchronous JSON APIs:

- JSONStream, which has stream APIs.
- Big-Friendly JSON, which has stream APIs as well as asynchronous versions of the standard JSON APIs using the partitioning-on-the-Event-Loop paradigm outlined below.

## Complex calculations without blocking the Event Loop

Suppose you want complex calculations in JavaScript without blocking the Event Loop. Two options: partitioning or offloading.

### Partitioning

_Partition_ your calculations so each runs on the Event Loop but regularly yields (gives turns) to other pending events. In JavaScript it's easy to save an ongoing task's state in a closure, as shown in example 2 below.

For example, suppose you want the average of the numbers `1` to `n`.

Example 1: Un-partitioned average, costs `O(n)`

```js
for (let i = 0; i < n; i++) {
  sum += i;
}

const avg = sum / n;
console.log("avg: " + avg);
```

Example 2: Partitioned average, each of the `n` asynchronous steps costs `O(1)`.

```js
function asyncAvg(n, avgCB) {
  // Save ongoing sum in JS closure.
  let sum = 0;
  function help(i, cb) {
    sum += i;
    if (i == n) {
      cb(sum);
      return;
    }

    // "Asynchronous recursion".
    // Schedule next operation asynchronously.
    setImmediate(help.bind(null, i + 1, cb));
  }

  // Start the helper, with CB to call avgCB.
  help(1, function (sum) {
    const avg = sum / n;
    avgCB(avg);
  });
}

asyncAvg(n, function (avg) {
  console.log("avg of 1-n: " + avg);
});
```

You can apply this principle to array iterations and so forth.

### Offloading

For something more complex, partitioning is a poor option: it uses only the Event Loop, so you won't benefit from the multiple cores almost certainly on your machine. _Remember, the Event Loop should orchestrate client requests, not fulfill them itself._ For a complicated task, move the work off the Event Loop onto a Worker Pool.

#### How to offload

Two destination Worker Pools to offload to:

1. The built-in Node.js Worker Pool, via a C++ addon. On older Node versions build it with NAN, on newer ones with N-API. node-webworker-threads offers a JavaScript-only way to access the Node.js Worker Pool.
2. Your own Worker Pool dedicated to computation, rather than the Node.js I/O-themed one. The most straightforward ways are Child Process or Cluster.

Do _not_ simply create a Child Process for every client. You can receive client requests more quickly than you can create and manage children, and your server might become a fork bomb.

#### Downside of offloading

Offloading incurs overhead in the form of _communication costs_. Only the Event Loop can see your application's "namespace" (JavaScript state). From a Worker you cannot manipulate a JavaScript object in the Event Loop's namespace; instead you serialize and deserialize any objects you wish to share. The Worker then operates on its own copy and returns the modified object (or a "patch") to the Event Loop.

For serialization concerns, see the section on JSON DOS.

#### Some suggestions for offloading

Distinguish between CPU-intensive and I/O-intensive tasks: they have markedly different characteristics.

A CPU-intensive task only makes progress when its Worker is scheduled onto one of your machine's logical cores. With 4 logical cores and 5 Workers, one Worker cannot make progress, so you pay overhead (memory and scheduling) for it with no return.

I/O-intensive tasks query an external service provider (DNS, file system, etc.) and wait for its response. While waiting, a Worker has nothing to do and can be de-scheduled by the operating system, giving another Worker a chance to submit its request. Thus, _I/O-intensive tasks make progress even while the associated thread is not running_. External providers like databases and file systems are highly optimized to handle many pending requests concurrently: a file system, for example, examines pending reads and writes to merge conflicting updates and retrieve files in an optimal order.

If you rely on only one Worker Pool, e.g. the Node.js one, the differing characteristics of CPU-bound and I/O-bound work may harm your performance.

So you might maintain a separate Computation Worker Pool.

#### Offloading: conclusions

For simple tasks, like iterating over an arbitrarily long array, partitioning might be a good option. If your computation is more complex, offloading is better: the communication costs of passing serialized objects between the Event Loop and the Worker Pool are offset by the benefit of multiple cores.

However, if your server relies heavily on complex calculations, consider whether Node.js is really a good fit. It excels for I/O-bound work, but for expensive computation it might not be the best option.
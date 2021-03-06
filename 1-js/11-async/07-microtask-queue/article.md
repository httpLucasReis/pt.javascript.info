
# Microtasks and event loop

Promise handlers `.then`/`.catch`/`.finally` are always asynchronous.

<<<<<<< HEAD
Even when a Promise is immediately resolved, the code on the lines *below* your `.then`/`.catch`/`.finally` will still execute first.

Here's the code that demonstrates it:
=======
Even when a Promise is immediately resolved, the code on the lines *below* `.then`/`.catch`/`.finally` will still execute before these handlers.

Here's a demo:
>>>>>>> e074a5f825a3d10b0c1e5e82561162f75516d7e3

```js run
let promise = Promise.resolve();

promise.then(() => alert("promise done"));

alert("code finished"); // this alert shows first
```

If you run it, you see `code finished` first, and then `promise done`.

That's strange, because the promise is definitely done from the beginning.

Why did the `.then` trigger afterwards? What's going on?

# Microtasks

Asynchronous tasks need proper management. For that, the ECMA standard specifies an internal queue `PromiseJobs`, more often referred to as the "microtask queue" (ES8 term).

As stated in the [specification](https://tc39.github.io/ecma262/#sec-jobs-and-job-queues):

- The queue is first-in-first-out: tasks enqueued first are run first.
- Execution of a task is initiated only when nothing else is running.

Or, to say more simply, when a promise is ready, its `.then/catch/finally` handlers are put into the queue; they are not executed yet. When the JavaScript engine becomes free from the current code, it takes a task from the queue and executes it.

That's why "code finished" in the example above shows first.

![](promiseQueue.svg)

Promise handlers always go through this internal queue.

If there's a chain with multiple `.then/catch/finally`, then every one of them is executed asynchronously. That is, it first gets queued, then executed when the current code is complete and previously queued handlers are finished.

**What if the order matters for us? How can we make `code finished` run after `promise done`?**

Easy, just put it into the queue with `.then`:

```js run
Promise.resolve()
  .then(() => alert("promise done!"))
  .then(() => alert("code finished"));
```

Now the order is as intended.

## Event loop

In-browser JavaScript, as well as Node.js, is based on an *event loop*.

"Event loop" is a process when the engine sleeps and waits for events, then reacts on those and sleeps again.

Examples of events:
- `mousemove`, a user moved their mouse.
- `setTimeout` handler is to be called.
- an external `<script src="...">` is loaded, ready to be executed.
- a network operation, e.g. `fetch` is complete.
- ...etc.

Things happen -- the engine handles them -- and waits for more to happen (while sleeping and consuming close to zero CPU).

![](eventLoop.svg)

As you can see, there's also a queue here. A so-called "macrotask queue" (v8 term).

When an event happens, while the engine is busy, its handling is enqueued.

For instance, while the engine is busy processing a network `fetch`, a user may move their mouse causing `mousemove`, and `setTimeout` may be due and so on, just as painted on the picture above.

Events from the macrotask queue are processed on "first come – first served" basis. When the engine browser finishes with `fetch`, it handles `mousemove` event, then `setTimeout` handler, and so on.

So far, quite simple, right? The engine is busy, so other tasks queue up.

Now the important stuff.

**Microtask queue has a higher priority than the macrotask queue.**

In other words, the engine first executes all microtasks, and then takes a macrotask. Promise handling always has the priority.

For instance, take a look:

```js run
setTimeout(() => alert("timeout"));

Promise.resolve()
  .then(() => alert("promise"));

alert("code");
```

What's the order?

1. `code` shows first, because it's a regular synchronous call.
2. `promise` shows second, because `.then` passes through the microtask queue, and runs after the current code.
3. `timeout` shows last, because it's a macrotask.

It may happen that while handling a macrotask, new promises are created.

Or, vice-versa, a microtask schedules a macrotask (e.g. `setTimeout`).

For instance, here `.then` schedules a `setTimeout`:

```js run
Promise.resolve()
  .then(() => {
    setTimeout(() => alert("timeout"), 0);
  })
  .then(() => {
    alert("promise");
  });
```

Naturally, `promise` shows up first, because `setTimeout` macrotask awaits in the less-priority macrotask queue.

As a logical consequence, macrotasks are handled only when promises give the engine a "free time". So if we have a promise chain that doesn't wait for anything, then things like `setTimeout` or event handlers can never get in the middle.


## Unhandled rejection

<<<<<<< HEAD
Remember "unhandled rejection" event from the chapter <info:promise-error-handling>?

Now, with the understanding of microtasks, we can formalize it.

**"Unhandled rejection" is when a promise error is not handled at the end of the microtask queue.**
=======
Remember the `unhandledrejection` event from the article <info:promise-error-handling>?

Now we can see exactly how JavaScript finds out that there was an unhandled rejection.

**An "unhandled rejection" occurs when a promise error is not handled at the end of the microtask queue.**
>>>>>>> e074a5f825a3d10b0c1e5e82561162f75516d7e3

For instance, consider this code:

```js run
let promise = Promise.reject(new Error("Promise Failed!"));

window.addEventListener('unhandledrejection', event => {
  alert(event.reason); // Promise Failed!
});
```

<<<<<<< HEAD
We create a rejected `promise` and do not handle the error. So we have the "unhandled rejection" event (printed in browser console too).

We wouldn't have it if we added `.catch`, like this:
=======
But if we forget to add `.catch`, then, after the microtask queue is empty, the engine triggers the event:
>>>>>>> e074a5f825a3d10b0c1e5e82561162f75516d7e3

```js run
let promise = Promise.reject(new Error("Promise Failed!"));
*!*
promise.catch(err => alert('caught'));
*/!*

// no error, all quiet
window.addEventListener('unhandledrejection', event => alert(event.reason));
```

Now let's say, we'll be catching the error, but after `setTimeout`:

```js run
let promise = Promise.reject(new Error("Promise Failed!"));
*!*
setTimeout(() => promise.catch(err => alert('caught')));
*/!*

// Error: Promise Failed!
window.addEventListener('unhandledrejection', event => alert(event.reason));
```

<<<<<<< HEAD
Now the unhandled rejction appears again. Why? Because `unhandledrejection` triggers when the microtask queue is complete. The engine examines promises and, if any of them is in "rejected" state, then the event is generated.

In the example, the `.catch` added by `setTimeout` triggers too, of course it does, but later, after `unhandledrejection` has already occurred.

## Summary

- Promise handling is always asynchronous, as all promise actions pass through the internal "promise jobs" queue, also called "microtask queue" (v8 term).
=======
Now, if we run it, we'll see `Promise Failed!` first and then `caught`.

If we didn't know about the microtasks queue, we could wonder: "Why did `unhandledrejection` handler run? We did catch and handle the error!"

But now we understand that `unhandledrejection` is generated when the microtask queue is complete: the engine examines promises and, if any of them is in the "rejected" state, then the event triggers.

In the example above, `.catch` added by `setTimeout` also triggers. But it does so later, after `unhandledrejection` has already occurred, so it doesn't change anything.
>>>>>>> e074a5f825a3d10b0c1e5e82561162f75516d7e3

    **So, `.then/catch/finally` are called after the current code is finished.**

    If we need to guarantee that a piece of code is executed after `.then/catch/finally`, it's best to add it into a chained `.then` call.

- There's also a "macrotask queue" that keeps various events, network operation results, `setTimeout`-scheduled calls, and so on. These are also called "macrotasks" (v8 term).

<<<<<<< HEAD
    The engine uses the macrotask queue to handle them in the appearance order.

    **Macrotasks run after the code is finished *and* after the microtask queue is empty.**
=======
Promise handling is always asynchronous, as all promise actions pass through the internal "promise jobs" queue, also called "microtask queue" (ES8 term).

So `.then/catch/finally` handlers are always called after the current code is finished.
>>>>>>> e074a5f825a3d10b0c1e5e82561162f75516d7e3

    In other words, they have lower priority.

<<<<<<< HEAD
So the order is: regular code, then promise handling, then everything else, like events etc.
=======
In most Javascript engines, including browsers and Node.js, the concept of microtasks is closely tied with the "event loop" and "macrotasks". As these have no direct relation to promises, they are covered in another part of the tutorial, in the article <info:event-loop>.
>>>>>>> e074a5f825a3d10b0c1e5e82561162f75516d7e3

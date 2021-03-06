# async-hofs

![CircleCI](https://circleci.com/gh/raywhite/async-hofs.svg?style=shield&circle-token=ba7a6a30ff093e5402b3af43b6889c2b222ce366)

> JS Promise & async / await higher order functions utils.

## About

This repo contains utilities (mostly [higher order functions](https://en.wikipedia.org/wiki/Higher-order_function)) intending to help with common [asynchronous tasks](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) where `async` / `await` or [promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) are used.

The entire module expects a standard `Promise` implementation to be available (**Node v4+**), and also doesn't itself use `async` / `await` in any of it's source - so as to not require transpilation when used as a dependancy (it's intended to be used as a dep for [google cloud functions](https://cloud.google.com/functions/docs/writing/)), which at the time of writing this, is pinned to **Node v6.11.5 LTS**.

## Setup

To clone run `git clone https://github.com/raywhite/async-hofs`. 

Tests are written using [ava](https://github.com/avajs/ava), and can be run with `npm run test:node`. The full test suite includes linting with `eslint .` (`npm run test:lint`), and can be run with `npm run test`.

To install as a dep, run `npm install @raywhite/async-hofs --save`.

## API

### compose(*..fns*) => *fn*

- **...fns** - (`...Function`) - any number of functions.
- **fn** - (`Function`) - the composed function.

While async functions are expected, synchronous functions will also be composed. Note that the composed function will alway return a promise. `compose` will compose functions from  **right to left**.

### sequence(*...fns*) => *fn*

- **...fns** - (`...Function`) - any number of functions.
- **fn** - (`Function`) - the composed function.

While async functions are expected, synchronous functions will also be composed. Note that the composed function will alway return a promise. `sequence` will compose functions from  **left to right**. 

- **...fns** (`...Function`)  

### createAsyncFnPool(*fn*, *[concurrency = 1]*) => *pool*

- **fn** - (`Function`) - an `async` function to be invoked - where it requires parameters, used `Array.prototype.bind`.
- **concurrency** - (`Number`) - how many times to spawn the `async` function - defaults to `1`.
- **pool** - (`Promise`)

Wraps an `async` function, and takes an optional concurrency. `fn` will be used to create a "green" thread (think of it like a goroutine or something)... and it will limit the concurrency with which that function is called. Consider the following example:

```js
const { createAsyncFnPool } = require('async-hofs')

const sleep = async function (value) {
  await new Promise(r => setTimeout(r, Math.random() * 16))
  return value
}

const inputs = [1, 2, 3, 4, 5, 6]
const outputs = []

const thread = async function () {
  while (inputs.length) {
    const value = await sleep(inputs.shift())
    outputs.push(value)
  }
}

const fn = async function () {
  await createAsyncFnPool(thread, 2)
  console.log.call(console, outputs)
}

fn().catch(console.error.bind(console))
```

### createRetrierFn(*fn*, *[limit = 2]*) => *retrier*

Wraps an `async` function so that it will be attempted `limit` times before it actually rejects.

**TODO:** At present this function fires of the original function as soon as the previous attempt failed - it should ideally support a linear and incremental backoff (ie. allowing it to wait *x* milliseconds before making another attempt)- and the simplest way to allow for this would be to make it accept a **curve** function and **increments** as params.

Where the wrapped function rejects multiple times (exceeding the limit), the error that it finally rejects with will always be value that the last attempt rejected with.

- **fn** - (`Function`) - an `async` function to be wrapped for retrying.
- **limit** - (`Number`) - the number of times to retry - defaults to `2`.
- **retrier** - (`Function`) - the wrapped function.

### clock(*fn*, *[concurrency = 1]*) => *clocked*

Given a function `fn` and an optional `concurrency`, this function will return a version of `fn` that will schedule invocation so as to allow a maximum of `concurrency` concurrent invocations of that function. This is intended for use case where you don't want to exceed some memory or IO limit, or create a mutex (for instance to prevent concurrent access to files).

**NOTE:** this method is aliased as `createCLockedFn` - which was really just a more verbose name.

- **fn** - (`Function`) - an `async` function to lock / release.
- **concurrency** - (`Number`) - the number of concurrent invocations allowed - defaults to `1`.
- **clocked** - (`Function`) - the concurrency locked function.
  - **clocked.pending** - a getter for the number of invocations currently running.
  - **clocked.queued** - a getter for the number of calls awaiting invocation.

### bechmark(*fn*, *[precision = 'ms']*, *[...args]*) => *res*

The returned value (`res`) is a `Promise` that resolves with a tuple in the form (`time`, `value`) where `value` is the value resolved by calling `fn`, and `time` is the measured execuition time of `fn` with a precision of `precision`. Where `fn` rejects, `benchmark` itself with reject with the same value ツ.
 
- **fn** - (`Function`) - the async function to be invoked.
- **precision** - (`String`) - a constant (`s|ms|ns`) representing the precision of the timing.
- **args** - (`...Mixed`) - extras arguments to pass to the `fn` invokcation.
- **res** - (`Array`) - the `time` and `value` tuple.

### buffer(*readable*, *[limit = 1000 * 1024]*) => *buf*

Given a stdlib `stream.Readable`, this function will continue to read from the stream until the `end` event is emitted by the stream, and then resolve the returned promise. The returned promise will reject if the `limit` is exceeded, and will also reject with any errors emitted by the underlying stream.

**NOTE:** This funciton will actually consume the stream, meaning that the stream shouldn't also be consumed by another function, unless the event handlers are attached prior to calling `buffer`. Importantly, `buffer` itself can't actually consume a stream that is or was being consumed by `buffer` - so subsequent calls to `buffer` using the same stream will error.

- **readable** - (`stream.Readable`) - the readable stream to be buffered.
- **limit** - (`Number`) - the max number of bytes to buffer.
- **buf** - (`Promise`) - resolves with the buffer contents.

### constant *buffer.LIMIT_EXCEEDED*

The value of the error `message` and `type` upon rejection of the promise returned by `buffer` where the reason for rejection was exceeding of the `limit` parameter. Should be used for asserting whether or not this was the type of error.

### constant *buffer.SECOND_STREAM_CONSUMER*

The value of the error `message` and `type` upon rejection of the promise returned by `buffer`, where the reason for rejection was `buffer` being called more than once with the same readable. Should be used for asserting whether or not this was the type of error.

## License

&bull; **MIT** &copy; Ray White, 2017-2018 &bull;

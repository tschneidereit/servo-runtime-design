# Explainer for Tasks

## Outline

The explainer proceeds as follows:

<!-- TOC depthFrom:1 depthTo:6 orderedList:true updateOnSave:true withLinks:true -->

1. [Explainer for Tasks](#explainer-for-tasks)
    1. [Outline](#outline)
    2. [Overview](#overview)
    3. [Motivation](#motivation)
    4. [Task Communication with Streams](#task-communication-with-streams)
        1. [Tasks as TransformStreams](#tasks-as-transformstreams)
    5. [Error Handling](#error-handling)
    6. [Self-Spawning](#self-spawning)
    7. [API](#api)
        1. [External](#external)
        2. [Internal](#internal)
    8. [Nested Task Hierarchies](#nested-task-hierarchies)
    9. [Web Worker based Polyfills](#web-worker-based-polyfills)
    10. [Higher-level Abstractions](#higher-level-abstractions)
    11. [Resource Usage Considerations](#resource-usage-considerations)

<!-- /TOC -->

## Overview

The most important concept of the Servo Runtime is the Task. Superficially similar to [Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API.html), Task is the Servo Runtime's representation of JavaScript execution threads.

Both Tasks and Web Workers use message passing based interfaces. Instead of `postMessage` and `onmessage` callbacks, Tasks provide bidirectional communications channels based on [`ReadableStream`](https://streams.spec.whatwg.org/#rs-model) and [`WritableStream`](https://streams.spec.whatwg.org/#ws-model).

## Motivation

When writing fast and responsive applications, it's crucially important to never block the main thread and to make good use of the available processing resources. In JavaScript, the former is commonly achieved by splitting up work into small chunks and yielding to the event loop frequently. `Promises` and more recently `async/await` make this a relatively straightforward process.

The latter, distributing work across multiple CPU cores, is used much less pervasively. One reason is that JavaScript lacks multithreading abstractions with good usability and composability. Tasks provide an API and semantics that make them easy to use and composable, making it feasible to distribute complex workloads across multiple threads. Their tight integration with [`event streams`](event-streams.md) makes it natural to distribute event handling across threads, largely eliminating the concept of a main thread that needs to be carefully kept responsive.

## Task Communication with Streams

A `Task` instance exposes a bidirectional channel as a pair of `readable` and `writable` properties. These are instances of `ReadableStream` and `WritableStream`, respectively. Sending messages to the task means writing into the `writable`, receiving messages means reading from the `readable`. Specifically, calling `read()` on the `readable`'s `reader` and thereby getting a Promise signals interest in the task's next message.

```js
let task = await Task.spawn("some_task");
let writer = task.writable.getWriter();
writer.write('some message');
let reader = task.readable.getReader();
let message = await reader.read();
```

As with all streams, reading and writing requires first acquring a reader or writer, respectively. While this may seem like redundant boilerplate, it makes it possible to treat tasks as transform streams as described in the next section. For the `readable`, it isn't usually required to directly interact with the reader anyway, as the same pattern of signaling interest using async iteration applies as for [`event streams`](event-streams.md):

```js
let task = await Task.spawn("some_task");
let writer = task.writable.getWriter();
writer.write('some message');
handleMessages(task.readable);

function handleMessages(readable) {
    for await (let message of readable) {
        // Handle the message.
    }
}
```

### Tasks as TransformStreams

Because task communication is exposed as `readable` and `writable` properties, a task can be used as a `TransformStream`. This way, running computationally expensive operations on another thread becomes trivial in many cases. Consider a common scenario: reading data from disk, applying some kind of transformation such as decompressing or converting to a different format, and writing it out to a socket. Based on an existing JS zip decoder library, here's a streaming, parallelized zip decoder:

```js
// streaming_zip.js
import {Decoder} from 'zip';

async function main({readable, writable}) {
    let decoder = new Decoder();
    let writer = writable.getWriter();
    for await (let chunk of readable) {
        writer.write(decoder.decode(chunk));
    }
}
```

And here's how to use it:

```js
// main.js
import {load} from "sys/io"; // Note: this API isn't finalized.

async function main() {
    return await loadAndDecode('some.zip');
}

async function loadAndDecode(file) {
    for await of (load(file).pipeThrough(await Task.spawn('streaming_zip'))) {
        // Use decoded data.
    }
}
```

It's important to realize that the processing pipeline from file loading to data decoding runs without touching the thread in which it was set up. The `pipeThrough` operation can directly connect the `ReadableStream` returned by `load` to the `WritableStream` exposed by the Task. Both streams are locked, making their interactions unobservable from the outside, so their communication can skip the current thread.

See the [API](#api) section for details, and the [Higher-level Abstractions](#higher-level-abstractions) section for alternative ways to use Tasks.

## Error Handling

It's important that uncaught runtime errors aren't just swallowed silently. For that reason, an exception in a Task that's not caught within the Task itself causes the Task to be aborted and the Task's `readable` stream to be errored with the exception. If the exception isn't handled in the thread that spawned the Task (be it the initial thread or another Task), then that thread is aborted, too, and the exception is propagated further.

An exception sent via a Task's `readable` stream is considered unhandled if either the stream isn't read from in the same turn of the event loop the error was sent in, or the Promise returned by `readable.read()` doesn't have a `catch` handler attached. Commonly the code that spawned the Task will have started an async iteration over the stream, in which case the exception is considered handled if the iterating function or any function awaiting its results contains a `catch` block.

When an exception is propagated, it's wrapped into an error indicating Task abort. If an exception is propagated up through multiple levels of a [nested Task tree](#nested-task-hierarchies), the full chain of Task aborts can be reconstructed by recursing over the nested error objects.

In the previous example, an error during file loading or in the zip decoder Task would cause the application to be aborted. To handle errors, the `loadAndDecode` function can be modified:

```js
async function loadAndDecode(file) {
    try {
        for await of (load(file).pipeThrough(new Task('streaming_zip'))) {
            // Use decoded data.
        }
    } catch (error) {
        console.log(`Error encountered during file processing: ${error}`);
        return error.code;
    }
    
    return 0;
}
```

If a Task is aborted that has itself spawned further Tasks, those sub-Tasks would be left without a parent to report errors to. At that point, any potentially fallible operations cannot be relied upon anymore, so it wouldn't make sense to continue them. To avoid immediate hard shutdown of sub-Tasks, a Task failure sent to the Task's parent contains a list of sub-Tasks. The parent has the option to adopt the sub-Tasks to keep them running, or to signal that they should shut down in an orderly fashion. If the parent doesn't explicitly adopt the sub-Tasks, they're considered abandoned and get shut down immediately.

TODO: define API.

Note that this weakens a Task's encapsulation by exposing implementation details about its threading strategy. If a Task requires harder encapsulation guarantees, it needs to ensure that it properly handles all potential exceptions, thus avoiding a forced shutdown and error propagation.

## Self-Spawning

Not all functionality is adequately exposed through a channel based interface. Instead, it's often preferable to have an object-oriented API exposed through a class. Tasks can be abstracted away behind normal modules exporting an object-oriented API, with the functionality internally implemented as messages on the channel.

Normally this'd require two files: one to implement the module, one for the Task. To make this pattern easier, `Task` provides a `Task.spawnSelf()` function that loads the containing file into a Task. With that, our streaming zip decoder example becomes:

```js
// streaming_zip.js
import {Decoder} from 'zip';

export class Decoder {
    constructor() {
        this.readable = new ReadableStream();
        this.writable = new WritableStream();
        Task.spawnSelf().then(task) {
            this.readble.pipeThrough(task).pipeTo(this.writable);
        }
    }
}

async function main({readable, writable}) {
    let decoder = new Decoder();
    let writer = writable.getWriter();
    for await (let chunk of readable) {
        writer.write(decoder.decode(chunk));
    }
}


// main.js
import {Decoder} from 'streaming_zip';

async function main() {
    return await loadAndDecode('some.zip');
}

async function loadAndDecode(file) {
    try {
        for await of (load(file).pipeThrough(new Decoder())) {
            // Use decoded data.
        }
    } catch (error) {
        console.log(`Error encountered during file processing: ${error}`);
        return error.code;
    }
    
    return 0;
}
```

## API

### External

Streams plus signalling promises for one-time events.

The primary way to create a Task is using the `Task.spawn` async constructor function, passing a module name to load. Module resolution happens the same as for `import` statements. The reason for not using `new Task` is that creating the `readable` and `writable` channels can only happen after creating the Task and loading the given module, which is an asynchronous operation.

As discussed [above](#error-handling), uncaught exceptions from within Tasks are propagated as errors on the `readable` stream. Similarly, Task shutdown is signalled through the `readable` and `writable` streams being closed.

In [TypeScript](https://www.typescriptlang.org/) syntax, `Task` has the following definition:

```ts
declare class Task {
    static spawn(moduleSrc: string): Promise<Task>;
    static spawnSelf(): Promise<Task>;
    close();
    readable: ReadableStream;
    writable: WritableStream;
}
```

### Internal

Tasks expect JavaScript code loaded into them to contain a symbol `main` that's an async function. This function is invoked with a channel object when initializing the Task. The Task is considered complete and will be shut down once the `main` function finishes with a `return`. During the Task's operation, it'll typically use `await` to suspend itself until some external event, such as a message coming in from the channel's `readable`.

The signature of a Task module thus is:

```ts
async function main({readable, writable}: {ReadableStream, WritableStream}): Promise<void>;
```

## Nested Task Hierarchies

JavaScript code running in a Task can itself spawn other Tasks, forming nested trees of Tasks. This enables the implementation of complex components that are heavily parallelized internally without exposing this complexity to client code.

## Web Worker based Polyfills

A polyfill based on browsers' Web Workers is possible, but requires a full implementation of streams, including transferable ReadableStreams.

TODO: describe in more detail.

## Higher-level Abstractions

Class synthesis and multiplexing

## Resource Usage Considerations

Greenthreading

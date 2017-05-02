# Explainer for Event Streams

## Outline

The explainer proceeds as follows:

<!-- TOC depthFrom:1 depthTo:6 orderedList:true updateOnSave:true withLinks:true -->

1. [Explainer for Event Streams](#explainer-for-event-streams)
    1. [Outline](#outline)
    2. [Overview](#overview)
    3. [Motivation](#motivation)
    4. [Listening to Events](#listening-to-events)
    5. [Canceling Default Behavior](#canceling-default-behavior)
    6. [Multiple Listeners](#multiple-listeners)
    7. [Queueing Behavior and Registering Interest](#queueing-behavior-and-registering-interest)
    8. [Events and Tasks](#events-and-tasks)

<!-- /TOC -->

## Overview

In the Servo Runtime, event sources are represented as [`ReadableStream`](https://streams.spec.whatwg.org/#rs-model) instances. An event emitter that can emit multiple types of events will have a read-only accessor for each event type, reading which returns a new stream.

## Motivation

Events are fundamentally streams of notification objects, so `ReadableStream` is a natural representation for them.

This representation enables uniform handling of events of all kinds, be it input events such as mouse move or click notifications, I/O events such as a read notification for a chunk in a file loading operation, or custom events provided by application logic. All of these can, using standard operations, be piped through arbritrary stream combinators. And, crucially, they can be piped to [`tasks`](tasks.md), thereby moving their processing to another thread.

## Listening to Events

In its most common form, event listening will be expressed as an [async iteration](https://jakearchibald.com/2017/async-iterators-and-generators/) over the event stream:

```js
for await (const click of events.mouseClick) {}
```

As for all `ReadableStreams`, it's also possible to manually read from an event stream by acquiring a reader and reading from it:

```js
const clickStreamReader = events.mouseClick.getReader();
let click = await clickStreamReader.read();
```

The async iteration form is not only much more convenient, it also has the advantage that the internally acquired reader will be properly closed once the consumer stops listening (by breaking out of the iteration).

## Canceling Default Behavior

Similar to DOM events, streamed events are cancelable using [`preventDefault`](https://developer.mozilla.org/en-US/docs/Web/API/Event/preventDefault).

An important aspect is when default behavior will apply. That is, how much time does a consumer have to prevent default behavior? This isn't immediately obvious because a consumer might not be reading from an event stream at the time the event is dispatched, or it might be on another thread which is slow to respond. There could potentially be multiple events waiting in the same consumer's queue.

The window of opportunity for preventing default behavior closes once the consumer's thread yields back to the event loop after draining its [microtask queue](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/). That means that to be able to prevent default behavior, a consumer has to read from the event stream without interruption, because otherwise it might yield to the event loop without a read operation pending on the stream when the event is enqueued.

This behavior enables predictable behavior for both the event producer and consumer: the producer can be sure that a consumer cannot infinitely delay event handling, and the consumer can be sure that it doesn't miss any events, accidentally letting default behavior slip through.

## Multiple Listeners

Event streams are commonly exposed as non-memoizing accessors. That is, reading from a property that exposes an event stream twice yields two independent streams: `events.mouseClick !== events.mouseClick`.

This way, all event consumers are by default completely independent from each other: they'll not be able to interfere with each other, either by locking the stream, making it inaccessible by other consumers, or by delaying event delivery.

## Queueing Behavior and Registering Interest

An event stream Is only considered consumed if it is locked and either has an active reader or is [piped to a `WritableStream`](https://streams.spec.whatwg.org/#rs-pipe-to). That is, acquiring a reader means registering interest in the event type. What happens if a stream isn't consumed depends on the type of event.

In some cases, events will just be enqueued until they're read. This'll cause backpressure, potentially throttling event production.

In other cases, such as mouse or key events, not consuming an event stream will be interpreted as disinterest, and events will just be skipped completely.

## Events and Tasks

For many event types, handling them with low latency is crucial. By distributing event processing across multiple threads, risk of heavy computation delaying event handling can be minimized.

The [Tasks model](tasks.md) supports two different ways to parallelize event processing: by sending event streams to other tasks, or by treating a task as a transform stream to pipe an event stream to.

Since `ReadableStream` is transferable, an event stream can be sent to another task simply by writing it into the associated writable stream. After that, the event stream can be consumed without being blocked by the current thread:

```js
// In main.js:
// [..]
let eventsTask = await Task.spawn("events_task");
let writer = eventsTask.writable.getWriter();
writer.write({type: 'mouseClick', stream: events.mouseClick});

// In events_task.js
async function main({readable}) {
    for await (const eventStream of readable) {
        switch (eventStream.type) {
            case 'mouseEvent':
                handleMouseEvents(eventStream);
            default:
                console.warn(`Unknown event type ${eventStream.type}`);
        }
    }
}

async function handleMouseEvents(stream) {
    for await (const event of stream) {
        // Handle event.
    }
}
```

Additionally, [tasks can be used as transform streams](tasks.md##tasks-as-transformstreams), which means it's trivial to delegate handling of an event type to a thread solely focused on handling that specific event type:

```js
// In main.js
// [..]
events.mouseClick.pipeTo((await Task.spawn('mouse_handler')).writable);

// In mouse_handler.js
async function main({readable}) {
    for await (const event of readable) {
        // Handle event.
    }
}
```

# Explainer for Event Streams

## Outline

The explainer proceeds as follows:

<!-- TOC depthFrom:1 depthTo:6 orderedList:true updateOnSave:true withLinks:true -->

1. [Explainer for Event Streams](#explainer-for-event-streams)
    1. [Outline](#outline)
    2. [Overview](#overview)
    3. [Motivation](#motivation)

<!-- /TOC -->

## Overview

In the Servo Runtime, event sources are represented as [`ReadableStream`](https://streams.spec.whatwg.org/#rs-model) instances. An event emitter that can emit multiple types of events will have a read-only accessor for each event type, reading which returns a new stream.

## Motivation

Events are fundamentally streams of notification objects, so `ReadableStream` is a natural representation for them.

This representation enables uniform handling of events of all kinds, be it input events such as mouse move or click notifications, I/O events such as a read notification for a chunk in a file loading operation, or custom events provided by application logic. All of these can, using standard operations, be piped through arbritrary stream combinators. And, crucially, they can be piped to [`tasks`](tasks.md), thereby moving their processing to another thread.

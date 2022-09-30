<style>
@import url('https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@100;400;500&display=swap');
@import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@100&display=swap');
section {
    font-family: 'Noto Sans TC',-apple-system,BlinkMacSystemFont,'Segoe UI',Helvetica,Arial,sans-serif,'Apple Color Emoji','Segoe UI Emoji';
    font-weight: 100;
}
section strong {
    font-weight: 400;
}
section h1 {
    font-weight: 500;
}
section code {
    font-family: 'JetBrains Mono',SFMono-Regular,Consolas,Liberation Mono,Menlo,monospace;
    font-weight: 100;
}
</style>

<style scoped>
section {
    justify-content: center;
}
h1 {
    text-align: center;
    font-size: 3em;
}
p {
    text-align: center;
    font-size: 1em;
}
</style>

# What's different in 3.0 RxJava

2022-02-23
Wayne Yang

---
<!-- _paginate: false -->

<style scoped>
section {
    justify-content: center;
}
h1 {
    text-align: center;
    font-size: 2.5em;
}
</style>

# Introduction

---
<!-- paginate: true -->

<style>
section {
    justify-content: flex-start;
}
</style>

# Version 2.x

The 2.x version is end-of-life as of **February 28, 2021**.No further development, support, maintenance, PRs and updates will happen. The Javadoc of the very last version, **2.2.21**, will remain accessible.

---

# Maven coordinates

RxJava 3 lives in the group `io.reactivex.rxjava3` with artifact ID `rxjava`. Official language/platform adaptors will also be located under the group `io.reactivex.rxjava3`.

### Gradle import

```groovy
dependencies {
  implementation 'io.reactivex.rxjava3:rxjava:3.1.3'
}
```

---

# Java 8 (1/3)

For a long time, RxJava was limited to Java 6 API due to how Android was lagging behind in its runtime support. This changed with the Android Studio 4 where a process called [desugaring](https://developer.android.com/studio/write/java8-support-table) is able to turn many Java 7 and 8 features into Java 6 compatible ones transparently.

This allowed us to increase the baseline of RxJava to Java 8 and add official support for many Java 8 constructs:

 - `Stream`: use `java.util.stream.Stream` as a source or expose sequences as **blocking** `Streams`.
 - Stream `Collector`s: aggregate items into collections specified by [standard transformations](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collector.html).

---

# Java 8 (2/3)

 - `Optional`: helps with the **non-null**ness requirement of RxJava
 - `CompletableFuture`: consume `CompletableFuture`s non-blockingly or expose single results as `CompletableFuture`s.
 - Use site non-null annotation: helps with some functional types be able to return null in specific circumstances.

However, some features won't be supported:

 - `java.time.Duration`: would add a lot of overloads; can always be decomposed into the `time` + `unit` manually.
 - `java.util.function`: these can't throw `Throwable`s, overloads would create bloat and/or ambiguity

---

# Java 8 (3/3)

Consequently, one has to change the project's compilation target settings to Java 8:

```groovy
android {
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}
```

---

# Package structure

RxJava 3 components are located under the `io.reactivex.rxjava3` package (RxJava 1 has `rx` and RxJava 2 is just `io.reactivex`). This allows version 3 to live side by side with the earlier versions. In addition, the core types of RxJava (`Flowable`, `Observer`, etc.) have been moved to `io.reactivex.rxjava3.core`.

---
<!-- _paginate: false -->

<style scoped>
section {
    justify-content: center;
}
h1 {
    text-align: center;
    font-size: 2.5em;
}
</style>

# Behavior changes

---

# More undeliverable errors

With RxJava 2.x, the goal was set to not let any errors slip away in case the sequence is no longer able to deliver them to the consumers for some reason. Despite our best efforts, errors still could have been lost in various race conditions across many dozen operators.

Fixing this in a 2.x patch would have caused too much trouble, therefore, the fix was postponed to the, otherwise already considerably changing, 3.x release. Now, canceling an operator that delays errors internally will signal those errors to the global error handler via `RxJavaPlugins.onError()`.

---

# Connectable source reset

The purpose of the connectable types (`ConnectableFlowable` and `ConnectableObservable`) is to allow one or more consumers to be prepared before the actual upstream gets streamed to them upon calling `connect()`. This worked correctly for the first time, but had some trouble if the upstream terminated instead of getting disconnected. In this terminating case, depending on whether the connectable was created with `replay()` or `publish()`, fresh consumers would either be unable to receive items from a new connection or would miss items altogether.

With 3.x, connectables have to be reset explicitly when they terminate. This extra step allows consumers to receive cached items or be prepared for a fresh connection.

---

# Flowable.publish pause

The implementation of `Flowable.publish` hosts an internal queue to support backpressure from its downstream. In 2.x, this queue, and consequently the upstream source, was slowly draining on its own if all of the resulting `ConnectableFlowable`'s consumers have cancelled. This caused unexpected item loss when the lack of consumers was only temporary.

With 3.x, the implementation pauses and items already in the internal queue will be immediately available to consumers subscribing a bit later.

---

# Processor.offer null-check

Calling `PublishProcessor.offer()`, `BehaviorProcessor.offer()` or `MulticastProcessor.offer` with a null argument now throws a `NullPointerException` instead of signaling it via `onError` and thus terminating the processor. This now matches the behavior of the `onNext` method required by the [Reactive Streams specification](https://github.com/reactive-streams/reactive-streams-jvm#2.13).

---

# MulticastProcessor.offer fusion-check

`MulticastProcessor` was designed to be processor that coordinates backpressure like the `Flowable.publish` operators do. It includes internal optimizations such as operator-fusion when subscribing it to the right kind of source.

Since users can retain the reference to the processor itself, they could, in concept, call the `onXXX` methods and possibly cause trouble. The same is true for the `offer` method which, when called while the aforementioned fusion is taking place, leads to undefined behavior in 2.x.

With 3.x, the `offer` method will throw an `IllegalStateException` and not disturb the internal state of the processor.

---

# Group abandonment in groupBy (1/2)

The `groupBy` operator is one of the peculiar operators that signals reactive sources as its main output where consumers are expected to subscribe to these inner sources as well. Consequently, if the main sequence gets cancelled (i.e., the `Flowable<GroupedFlowable<T>>` itself), the consumers should still keep receiving items on their groups but no new groups should be created. The original source can then only be cancelled if all of such inner consumers have cancelled as well.

---

# Group abandonment in groupBy (2/2)

However, in 2.x, nothing is forcing the consumption of the inner sources and thus groups may be simply ignored altogether, preventing the cancellation of the original source and possibly leading to resource leaks.

With 3.x, the behavior of groupBy has been changed so that when it emits a group, the downstream has to subscribe to it synchronously. Otherwise, the group is considered "abandoned" and terminated. This way, abandoned groups won't prevent the cancellation of the original source. If a late consumer still subscribes to the group, the item that triggered the group creation will be still available.

---

# Backpressure in groupBy (1/2)

The `Flowable.groupBy` operator is even more peculiar in a way that it has to coordinate backpressure from the consumers of its inner group and request from its original `Flowable`. The complication is, such requests can lead to a creation of a new group, a new item for the group that itself requested or a new item for a completely different group altogether. Therefore, groups can affect each other's ability to receive items and can hang the sequence, especially if some groups don't get to be consumed at all.

---

# Backpressure in groupBy (2/2)

This latter can happen when groups are merged via `flatMap` where the number of individual groups is greater than the `flatMap`'s concurrency level (default 128) so fresh groups won't get subscribed to and old ones may not complete to make room. With `concatMap`, the same issue can manifest immediately.

Since RxJava is non-blocking, such silent hangs are difficult to detect and diagnose (i.e., no thread is blocking in `groupBy` or `flatMap`). Therefore, 3.x changed the behavior of `groupBy` so that if the immediate downstream is unable to receive a new group, the sequence terminates with `MissingBackpressureException`.

---

# Window abandonment in window (1/2)

Similar to `groupBy`, the `window` operator emits inner reactive sequences that should still keep receiving items when the outer sequence is cancelled (i.e., working with only a limited set of windows). Similarly, when all window consumers cancel, the original source should be cancelled as well.

---

# Window abandonment in window (2/2)

However, in 2.x, nothing is forcing the consumption of the inner sources and thus windows may be simply ignored altogether, preventing the cancellation of the original source and possibly leading to resource leaks.

With 3.x, the behavior of all `window` operators has been changed so that when it emits a group, the downstream has to subscribe to it synchronously. Otherwise, the window is considered "abandoned" and terminated. This way, abandoned windows won't prevent the cancellation of the original source. If a late consumer still subscribes to the window, the item that triggered the window creation may be still available.

---

# CompositeException cause generation

In 1.x and 2.x, calling the `CompositeException.getCause()` method resulted in a generation of a chain of exceptions from the internal list of aggregated exceptions. This was mainly done because Java 6 lacks the suppressed exception feature of Java 7+ exceptions. However, the implementation was possibly mutating exceptions or, sometimes, unable to establish a chain at all. Given the source of the original contribution of the method, it was risky to fix the issues with it in 2.x.

With 3.x, the method constructs a cause exception that when asked for a stacktrace, generates an output without touching the aggregated exceptions (which is IDE friendly and should be navigable).

---

# Parameter validation exception change

Some standard operators in 2.x throw `IndexOutOfBoundsException` when the respective argument was invalid. For consistency with other parameter validation exceptions, the following operators now throw `IllegalArgumentException` instead:

 - `skip`
 - `skipLast`
 - `takeLast`
 - `takeLastTimed`

---

# From-callbacks upfront cancellation

In 2.x, canceling sequences created via `fromRunnable` and `fromAction` were inconsistent with other `fromX` sequences when the downstream cancelled/disposed the sequence immediately.

In 3.x, such upfront cancellation will not execute the given callback.

---

# Using cleanup order

The operator `using` has an `eager` parameter to determine when the resource should be cleaned up: `true` means before-termination and `false` means after-termination. Unfortunately, this setting didn't affect the cleanup order upon donwstream cancellation and was always cleaning up the resource before canceling the upstream.

In 3.x, the cleanup order is now consistent when the sequence terminates or gets cancelled: `true` means before-termination or before canceling the upstream, `false` means after-termination or after canceling the upstream.

---
<!-- _paginate: false -->

<style scoped>
section {
    justify-content: center;
}
h1 {
    text-align: center;
    font-size: 2.5em;
}
</style>

# API changes

---

# Functional interfaces (1/2)

RxJava 2.x introduced a custom set of functional interfaces in `io.reactivex.functions` so that the use of the library is possible with the same types on Java 6 and Java 8. A secondary reason for such custom types is that the standard Java 8 function types do not support throwing any checked exceptions, which in itself can result in some inconvenience when using RxJava operators.

Despite RxJava 3 being based on Java 8, the issues with the standard Java 8 functional interfaces persist, now with possible desugaring issues on Android and their inability to throw checked exceptions. Therefore, 3.x kept the custom interfaces, but the `@FunctionalInterface` annotation has been applied to them (which is safe/ignored on Android).

---

# Functional interfaces (2/2)

One small drawback with the custom `throws Exception` in the functional interfaces is that some 3rd party APIs may throw a checked exception that is not a descendant of `Exception`, or simply throw `Throwable`.

Therefore, with 3.x, the functional interfaces as well as other support interfaces have been widened and declared with `throws Throwable` in their signature.

```java
@FunctionalInterface
interface Function<@NonNull T, @NonNull R> {
    R apply(T t) throws Throwable;
}
```

---
<!-- _paginate: false -->

<style scoped>
section {
    justify-content: center;
}
h2 {
    text-align: center;
    font-size: 2em;
}
</style>

## New Types

---

# Supplier

RxJava 2.x already supported the standard `java.util.concurrent.Callable` whose `call` method is declared with `throws Exception` by default. Unfortunately, when our custom functional interfaces were widened to `throws Throwable`, it was impossible to widen `Callable` because no in Java, implementations can't widen the `throws` clause, only narrow or abandon it.

Therefore, 3.x introduces the `io.reactivex.rxjava3.functions.Supplier` interface that defines the widest throws possible:

```java
interface Supplier<@NonNull R> {
    R get() throws Throwable;
}
```

---

# Converters (1/2)

In 2.x, the `to()` operator used the generic `Function` to allow assembly-time conversion of flows into arbitrary types. The drawback of this approach was that each base reactive type had the same `Function` interface in their method signature, thus it was impossible to implement multiple converters for different reactive types within the same class. To work around this issue, the `as` operator and `XConverter` interfaces have been introduced in 2.x, which interfaces are distinct and can be implemented on the same class. Changing the signature of `to` in 2.x was not possible due to the pledged binary compatibility of the library.

---

# Converters (2/2)

From 3.x, the `as()` methods have been removed and the `to()` methods now each work with their respective `XConverter` interfaces (hosted in package `io.reactivex.rxjava3.core`):

 - `Observable.to(Function<Observable<T>, R>)` -> `Observable.to(ObservableConverter<T, R>)`
 - `Maybe.to(Function<Flowable<T>, R>)` -> `Maybe.to(MaybeConverter<T, R>)`
 - `Completable.to(Function<Completable, R>)` -> `Completable.to(CompletableConverter<R>)`
 - ...

---
<!-- _paginate: false -->

<style scoped>
section {
    justify-content: center;
}
h2 {
    text-align: center;
    font-size: 2em;
}
</style>

## Moved components

---

# Disposables

Moving to Java 8 and Android's desugaring tooling allows the use of static interface methods instead of separate factory classes. The support class `io.reactivex.disposables.Disposables` was a prime candidate for moving all of its methods into the `Disposable` interface itself (`io.reactivex.rxjava3.disposables.Disposable`).

---

# DisposableContainer

Internally, RxJava 2.x uses an abstraction of a disposable container instead of using `CompositeDisposable` everywhere, allowing a more appropriate container type to be used. This is achieved via an internal `DisposableContainer` implemented by `CompositeDisposable` as well as other internal components. Unfortunately, since the public class referenced an internal interface, RxJava was causing warnings in OSGi environments.

In RxJava 3, the `DisposableContainer` is now part of the public API under `io.reactivex.rxjava3.disposables.DisposableContainer` and no longer causes OSGi issues.

---

# API promotions

The RxJava 2.2.x line has still a couple of experimental operators (but no beta) operators, which have been promoted to standard with 3.x.

https://github.com/ReactiveX/RxJava/wiki/What's-different-in-3.0#api-promotions

---

# API additions

RxJava 3 received a considerable amount of new operators and methods across its API surface.

https://github.com/ReactiveX/RxJava/wiki/What's-different-in-3.0#api-additions

---

# Java 8 additions

Now that the API baseline is set to Java 8, RxJava can now support the new types of Java 8 directly, without the need of an external library (such as the [RxJavaJdk8Interop](https://github.com/akarnokd/RxJavaJdk8Interop) library, now discontinued).
 - `java.util.function.Function`
 - `java.util.Optional`
 - `java.util.stream.Stream`
 - `java.util.concurrent.CompletionStage`
 - `java.util.stream.Collector`

https://github.com/ReactiveX/RxJava/wiki/What's-different-in-3.0#java-8-additions

---

# API renames

 - `startWith`
 - `onErrorResumeNext` with source
 - `zipIterable`
 - `combineLatest` with array argument
 - `Single.equals`
 - `Maybe.flatMapSingleElement`

https://github.com/ReactiveX/RxJava/wiki/What's-different-in-3.0#api-renames

---

# API signature changes

 - `Callable` to `Supplier`
 - `Maybe.defaultIfEmpty`
 - `concatMapDelayError` parameter order
 - `Flowable.buffer` with boundary source
 - `Maybe.flatMapSingle`

https://github.com/ReactiveX/RxJava/wiki/What's-different-in-3.0#api-signature-changes

---

# API removals (1/2)

 - `getValues`
 - `Maybe.toSingle(T)`
 - `subscribe(4 arguments)`
 - `Single.toCompletable()`
 - `Completable.blockingGet()`
 - Some test support methods
 - `replay` with Scheduler
 - `dematerialize()`

---

# API removals (2/2)

 - `onExceptionResumeNext`
 - `buffer` with boundary supplier
 - `combineLatest` with varags
 - `zip` with nested source
 - `fromFuture` with scheduler
 - `Observable.concatMapIterable` with buffer parameter

https://github.com/ReactiveX/RxJava/wiki/What's-different-in-3.0#api-removals

---
<!-- _paginate: false -->

<style scoped>
section {
    justify-content: center;
}
h1 {
    text-align: center;
    font-size: 2.5em;
}
</style>

# Interoperation

---

# Reactive Streams

RxJava 3 still follows the Reactive Streams specification and as such, the `io.reactivex.rxjava3.core.Flowable` is a compatible source for any 3rd party solution accepting an `org.reactivestreams.Publisher` as input.

`Flowable` can also wrap any such `org.reactivestreams.Publisher` in return.

---

# RxJava 1.x

RxJava is more than 7 years old at this moment and many users are still stuck with 3rd party tools or libraries only supporting the RxJava 1 line.

To help with the situation, and also help with a gradual migration from 1.x to 3.x, an external interop library is provided: [RxJavaInterop](https://github.com/akarnokd/RxJavaInterop)

---

# RxJava 2.x

Migration from 2.x to 3.x could also be cumbersome as well as difficult because the 2.x line also amassed an ecosystem of tools and libraries of its own, which may take time to provide a native 3.x versions.

There is limited interoperation between the `Flowable`s through the Reactive Streams `Publisher` interface (although not recommended due to extra overheads), however, there is no direct way for a 2.x `Observable` to talk to a 3.x `Observable` as they are completely separate types.

To help with the situation, and also help with a gradual migration from 2.x to 3.x, an external interop library is provided: [RxJavaBridge](https://github.com/akarnokd/RxJavaBridge)

---

# References
 - [ReactiveX/RxJava: RxJava – Reactive Extensions for the JVM](https://github.com/ReactiveX/RxJava)
 - [What's different in 3.0 · ReactiveX/RxJava Wiki](https://github.com/ReactiveX/RxJava/wiki/What's-different-in-3.0)

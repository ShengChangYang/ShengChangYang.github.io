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

# JNI tips

2021-06-09
Wayne Yang

---
<!-- paginate: true -->

<style>
section {
    justify-content: flex-start;
}
</style>

# JNI
JNI is the Java Native Interface.
> It defines a way for the bytecode that Android compiles from managed code (written in the Java or Kotlin programming languages) to interact with native code (written in C/C++).

<br/>

**Note**: Because Android compiles Kotlin to ART-friendly bytecode in a similar manner as the Java programming language, you can apply the guidance on this page to both the Kotlin and Java programming languages

---

# General tips
Try to minimize the footprint of your JNI layer. There are several dimensions to consider here:
 - Minimize marshalling of resources across the JNI layer.
 - Avoid asynchronous communication between code written in a managed programming language and code written in C++ when possible.
 - Minimize the number of threads that need to touch or be touched by JNI.
 - Keep your interface code in a low number of easily identified C++ and Java source locations to facilitate future refactors.
   > Consider using a JNI auto-generation library as appropriate.

---

# JavaVM and JNIEnv (1/2)
The JavaVM provides the "invocation interface" functions, which allow you to create and destroy a JavaVM.
> In theory you can have multiple JavaVMs per process, but Android only allows one.

<br/>

The JNIEnv provides most of the JNI functions. Your native functions all receive a JNIEnv as the first argument.

---

# JavaVM and JNIEnv (2/2)
The JNIEnv is used for thread-local storage. For this reason, **you cannot share a JNIEnv between threads**. You should share the JavaVM, and use `GetEnv` to discover the thread's JNIEnv.

<br/>

The C declarations of JNIEnv and JavaVM are different from the C++ declarations. The `"jni.h"` include file provides different typedefs.

---

# Threads (1/2)
All threads are Linux threads, scheduled by the kernel. They're usually started from managed code, but they can also be created elsewhere and then attached to the JavaVM.

<br/>

For example, a thread started with `pthread_create()` or `std::thread` can be attached using the `AttachCurrentThread()` or `AttachCurrentThreadAsDaemon()` functions. Until a thread is attached, it has no JNIEnv, and **cannot make JNI calls**.

---

# Threads (2/2)
Attaching a natively-created thread causes a `java.lang.Thread` object to be constructed and added to the "main" `ThreadGroup`, making it visible to the debugger. Calling `AttachCurrentThread()` on an already-attached thread is a no-op.

> Android does not suspend threads executing native code. If garbage collection is in progress, or the debugger has issued a suspend request, Android will pause the thread the next time it makes a JNI call.

<br/>

Threads attached through JNI **must call** `DetachCurrentThread()` **before they exit**.

---

# jclass, jmethodID, and jfieldID (1/3)
If you want to access an object's field from native code, you would do the following:
 - Get the class object reference for the class with `FindClass`
 - Get the field ID for the field with `GetFieldID`
 - Get the contents of the field with something appropriate, such as `GetIntField`

<br/>

> If performance is important, it's useful to look the values up once and cache the results in your native code. Because there is a limit of one JavaVM per process, it's reasonable to store this data in a static local structure.

---

# jclass, jmethodID, and jfieldID (2/3)
The class references, field IDs, and method IDs are guaranteed valid until the class is unloaded. Classes are only unloaded if all classes associated with a ClassLoader can be garbage collected, which is rare but will not be impossible in Android.

---

# jclass, jmethodID, and jfieldID (3/3)
If you would like to cache the IDs when a class is loaded, and automatically re-cache them if the class is ever unloaded and reloaded:

```java
/*
 - We use a class initializer to allow the native code to cache some
 - field offsets. This native function looks up and caches interesting
 - class/field/method IDs. Throws on failure.
 */
private static native void nativeInit();

static {
    nativeInit();
}
```

---

# Local and global references (1/4)
Every argument passed to a native method, and almost every object returned by a JNI function is a "local reference". This means that it's valid for the duration of the current native method in the current thread. **Even if the object itself continues to live on after the native method returns, the reference is not valid.**

This applies to all sub-classes of `jobject`, including `jclass`, `jstring`, and `jarray`.

---

# Local and global references (2/4)
The only way to get non-local references is via the functions `NewGlobalRef` and `NewWeakGlobalRef`.

The `NewGlobalRef` function takes the local reference as an argument and returns a global one. The global reference is guaranteed to be valid until you call `DeleteGlobalRef`.

This pattern is commonly used when caching a jclass returned from `FindClass`, e.g.:

```c++
jclass localClass = env->FindClass("MyClass");
jclass globalClass = reinterpret_cast<jclass>(env->NewGlobalRef(localClass));
```

---

# Local and global references (3/4)
The return values from consecutive calls to `NewGlobalRef` on the same object may be different. To see if two references refer to the same object, you must use the `IsSameObject` function.

You must not assume object references are constant or unique in native code. (Do not use `jobject` values as keys.)

---

# Local and global references (4/4)
Programmers are required to "not excessively allocate" local references. You should free them manually with `DeleteLocalRef` instead of letting JNI do it for you.
> The implementation is only required to reserve slots for 16 local references, so if you need more than that you should either delete as you go or use `EnsureLocalCapacity`/`PushLocalFrame` to reserve more.

<br/>

One unusual case deserves separate mention. If you attach a native thread with `AttachCurrentThread`, the code you are running will never automatically free local references until the thread detaches. In general, any native code that creates local references in a loop probably needs to do some manual deletion.

---

# UTF-8 and UTF-16 strings (1/2)
The Java programming language uses UTF-16. For convenience, JNI provides methods that work with Modified UTF-8 as well. The modified encoding is useful for C code because it encodes `\u0000` as `0xc0 0x80` instead of `0x00`.
> If possible, it's usually faster to operate with UTF-16 strings. Android currently does not require a copy in `GetStringChars`, whereas `GetStringUTFChars` requires an allocation and a conversion to UTF-8.

<br/>

Note that **UTF-16 strings are not zero-terminated**, and `\u0000` is allowed, so you need to hang on to the string length (`GetStringLength`) as well as the `jchar` pointer.

---

# UTF-8 and UTF-16 strings (2/2)
**Don't forget to** `Release` **the strings you** `Get`.
> The string functions return `jchar*` or `jbyte*`, which are C-style pointers to primitive data rather than local references. They are guaranteed valid until `Release` is called, which means they are not released when the native method returns.

<br/>

**Data passed to** `NewStringUTF` **must be in Modified UTF-8 format.**
> Unless you know the data is 7-bit ASCII (which is a compatible subset).

---

# Primitive arrays (1/4)
JNI provides functions for accessing the contents of array objects. While arrays of objects must be accessed one entry at a time (`GetObjectArrayElement`), arrays of primitives can be read and written directly as if they were declared in C.

To make the interface as efficient as possible without constraining the VM implementation, the `Get<PrimitiveType>ArrayElements` family of calls allows the runtime to either return a pointer to the actual elements, or allocate some memory and make a copy.
>  Either way, the raw pointer returned is guaranteed to be valid until the corresponding `Release` call is issued.

---

# Primitive arrays (2/4)
**You must** `Release` **every array you** `Get`. Also, if the `Get` call fails, you must ensure that your code doesn't try to `Release` a NULL pointer later.

You can determine whether or not the data was copied by passing in a non-NULL pointer for the `isCopy` argument.

---

# Primitive arrays (3/4)
The `Release` call takes a mode argument that can have one of three values:
 - `0`
   - Actual: the array object is un-pinned.
   - Copy: data is copied back. The buffer with the copy is freed.
 - `JNI_COMMIT`
   - Actual: does nothing.
   - Copy: data is copied back. The buffer with the copy **is not freed**.
 - `JNI_ABORT`
   - Actual: the array object is un-pinned. Earlier writes are **not** aborted.
   - Copy: the buffer with the copy is freed; any changes to it are lost.

---

# Primitive arrays (4/4)
It is a common mistake to assume that you can skip the `Release` call if `*isCopy` is false.
> This is not the case. If no copy buffer was allocated, then the original memory must be pinned down and can't be moved by the garbage collector.

<br/>

Also note that the `JNI_COMMIT` flag does not release the array, and you will need to call `Release` again with a different flag eventually.

---

# Region calls (1/2)
There is an alternative to calls like `Get<Type>ArrayElements` and `GetStringChars` that may be very helpful when all you want to do is copy data in or out:

```c++
jbyte* data = env->GetByteArrayElements(array, NULL);
if (data != NULL) {
    memcpy(buffer, data, len);
    env->ReleaseByteArrayElements(array, data, JNI_ABORT);
}
```

<br/>

One can accomplish the same thing more simply:

```c++
env->GetByteArrayRegion(array, 0, len, buffer);
```

---

# Region calls (2/2)

This has several advantages:
 - Requires one JNI call instead of 2, reducing overhead.
 - Doesn't require pinning or extra data copies.
 - Reduces the risk of programmer error — no risk of forgetting to call Release after something fails.

<br/>

Similarly, you can use the `Set<Type>ArrayRegion` call to copy data into an array, and `GetStringRegion` or `GetStringUTFRegion` to copy characters out of a `String`.

---

# Exceptions (1/4)
**You must not call most JNI functions while an exception is pending.** Your code is expected to notice the exception (via the function's return value, `ExceptionCheck`, or `ExceptionOccurred`) and return, or clear the exception and handle it.

---

<style scoped>
section.split {
    overflow: visible;
    display: grid;
    grid-template-columns: 45% 55%;
    grid-template-rows: fit-content(10%) fit-content(10%) auto;
    grid-template-areas:
        "slidetitle slidetitle"
        "slidecontent slidecontent"
        "leftpanel rightpanel";
}
section.split h1 {
    grid-area: slidetitle;
}
section.split p {
    grid-area: slidecontent;
}
section.split .ldiv {
    grid-area: leftpanel;
}
section.split .rdiv {
    grid-area: rightpanel;
}
</style>

<!-- _class: split -->

# Exceptions (2/4)
The only JNI functions that you are allowed to call while an exception is pending are:

<div class=ldiv>

 - `DeleteGlobalRef`
 - `DeleteLocalRef`
 - `DeleteWeakGlobalRef`
 - `ExceptionCheck`
 - `ExceptionClear`
 - `ExceptionDescribe`
 - `ExceptionOccurred`
 - `MonitorExit`

</div>

<div class=rdiv>

 - `PopLocalFrame`
 - `PushLocalFrame`
 - `Release<PrimitiveType>ArrayElements`
 - `ReleasePrimitiveArrayCritical`
 - `ReleaseStringChars`
 - `ReleaseStringCritical`
 - `ReleaseStringUTFChars`

</div>

---

# Exceptions (3/4)
Many JNI calls can throw an exception, but often provide a simpler way of checking for failure. For example, if `NewString` returns a non-NULL value, you don't need to check for an exception. However, if you call a method (using a function like `CallObjectMethod`), you must always check for an exception, because the return value is not going to be valid if an exception was thrown.

Note that exceptions thrown by interpreted code do not unwind native stack frames, and Android does not yet support C++ exceptions. The JNI `Throw` and `ThrowNew` instructions just set an exception pointer in the current thread. Upon returning to managed from native code, the exception will be noted and handled appropriately.

---

# Exceptions (4/4)
Native code can "catch" an exception by calling `ExceptionCheck` or `ExceptionOccurred`, and clear it with `ExceptionClear`. As usual, discarding exceptions without handling them can lead to problems.

There are no built-in functions for manipulating the `Throwable` object itself, so if you want to (say) get the exception string you will need to find the `Throwable` class, look up the method ID for `getMessage "()Ljava/lang/String;"`, invoke it, and if the result is non-NULL use `GetStringUTFChars` to get something you can hand to `printf(3)` or equivalent.

---

# Extended checking (1/6)
JNI does very little error checking. Errors usually result in a crash. Android also offers a mode called CheckJNI, where the JavaVM and JNIEnv function table pointers are switched to tables of functions that perform an extended series of checks before calling the standard implementation.

---

# Extended checking (2/6)
The additional checks include:
 - Arrays: attempting to allocate a negative-sized array.
 - Bad pointers: passing a bad jarray/jclass/jobject/jstring to a JNI call, or passing a NULL pointer to a JNI call with a non-nullable argument.
 - Class names: passing anything but the "java/lang/String" style of class name to a JNI call.
 - Critical calls: making a JNI call between a "critical" get and its corresponding release.
 - Direct ByteBuffers: passing bad arguments to `NewDirectByteBuffer`.
 - Exceptions: making a JNI call while there’s an exception pending.

---

# Extended checking (3/6)
The additional checks include:
 - JNIEnv\*s: using a JNIEnv\* from the wrong thread.
 - jfieldIDs: using a NULL jfieldID, or using a jfieldID to set a field to a value of the wrong type (trying to assign a StringBuilder to a String field, say), or using a jfieldID for a static field to set an instance field or vice versa, or using a jfieldID from one class with instances of another class.
 - jmethodIDs: using the wrong kind of jmethodID when making a `Call*Method` JNI call: incorrect return type, static/non-static mismatch, wrong type for ‘this’ (for non-static calls) or wrong class (for static calls).
 - References: using `DeleteGlobalRef`/`DeleteLocalRef` on the wrong kind of reference.

---

# Extended checking (4/6)
The additional checks include:
 - Release modes: passing a bad release mode to a release call (something other than `0`, `JNI_ABORT`, or `JNI_COMMIT`).
 - Type safety: returning an incompatible type from your native method (returning a StringBuilder from a method declared to return a String, say).
 - UTF-8: passing an invalid Modified UTF-8 byte sequence to a JNI call.

<br/>

**Note**: Accessibility of methods and fields is still not checked: access restrictions don't apply to native code.

---

# Extended checking (5/6)
To enable CheckJNI:

```shell
adb shell setprop debug.checkjni 1
```

> This won’t affect already-running apps, but any app launched from that point on will have CheckJNI enabled. (rebooting will disable CheckJNI again)

<br/>

You’ll see something like this in your logcat output the next time an app starts:

```
D Late-enabling CheckJNI
```

> If you’re using the emulator, CheckJNI is on by default.

---

# Extended checking (6/6)
**Real cases**:

<br/>

Samsung J7 (5.1.1)

```
I/art Late-enabling -Xcheck:jni
```

<br/>

Emulator (11 on M1 chip)

```
I Not late-enabling -Xcheck:jni (already on)
```

---

# Native libraries (1/6)
You can load native code from shared libraries with the standard `System.loadLibrary`.

> In practice, older versions of Android had bugs in `PackageManager` that caused installation and update of native libraries to be unreliable. The ReLinker project offers workarounds for this and other native library loading problems.

> If you have only one class with native methods, it makes sense for the call to `System.loadLibrary` to be in a static initializer for that class. Otherwise you might want to make the call from `Application` so you know that the library is always loaded, and always loaded early.

---

# Native libraries (2/6)
There are two ways that the runtime can find your native methods. You can either explicitly register them with `RegisterNatives`, or you can let the runtime look them up dynamically with `dlsym`.

---

# Native libraries (3/6)
To use `RegisterNatives`:
 - Provide a `JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved)` function.
 - In your `JNI_OnLoad`, register all of your native methods using `RegisterNatives`.
 - Build with `-fvisibility=hidden` so that only your `JNI_OnLoad` is exported from your library.

<br/>

The static initializer should look like this:

```java
static {
    System.loadLibrary("fubar");
}
```

---

# Native libraries (4/6)
The JNI_OnLoad function should look something like this if written in C++:

```c++
JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv* env;
    if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }

    // Find your class. JNI_OnLoad is called from the correct class loader context for this to work.
    jclass c = env->FindClass("com/example/app/package/MyClass");
    if (c == nullptr) return JNI_ERR;

    // Register your class' native methods.
    static const JNINativeMethod methods[] = {
        {"nativeFoo", "()V", reinterpret_cast<void*>(nativeFoo)},
        {"nativeBar", "(Ljava/lang/String;I)Z", reinterpret_cast<void*>(nativeBar)},
    };
    int rc = env->RegisterNatives(c, methods, sizeof(methods)/sizeof(JNINativeMethod));
    if (rc != JNI_OK) return rc;

    return JNI_VERSION_1_6;
}
```

---

# Native libraries (5/6)
To instead use "discovery" of native methods, you need to name them in a specific way. This means that if a method signature is wrong, you won't know about it until the first time the method is actually invoked.

---

# Native libraries (6/6)
Any `FindClass` calls made from `JNI_OnLoad` will resolve classes in the context of the class loader that was used to load the shared library. When called from other contexts, `FindClass` uses the class loader associated with the method at the top of the Java stack, or if there isn't one (because the call is from a native thread that was just attached) it uses the "system" class loader. The system class loader does not know about your application's classes, so you won't be able to look up your own classes with `FindClass` in that context. This makes `JNI_OnLoad` a convenient place to look up and cache classes: once you have a valid `jclass` you can use it from any attached thread.

---

# 64-bit considerations
To support architectures that use 64-bit pointers, use a long field rather than an int when storing a pointer to a native structure in a Java field.

---

# Unsupported features/backwards compatibility (1/2)
All JNI 1.6 features are supported, with the following exception:
 - `DefineClass` is not implemented. Android does not use Java bytecodes or class files, so passing in binary class data doesn't work.

---

# Unsupported features/backwards compatibility (2/2)
For backward compatibility with older Android releases, you may need to be aware of:
 - Dynamic lookup of native functions
 - Detaching threads
 - Weak global references
 - Local references
   In Android versions prior to Android 8.0, the number of local references is capped at a version-specific limit. Beginning in Android 8.0, Android supports unlimited local references.
 - Determining reference type with `GetObjectRefType`

---

# FAQ: Why do I get `UnsatisfiedLinkError`? (1/4)
When working on native code it's not uncommon to see a failure like this:

```
java.lang.UnsatisfiedLinkError: Library foo not found
```

---

# FAQ: Why do I get `UnsatisfiedLinkError`? (2/4)
In some cases it means the library wasn't found. In other cases the library exists but couldn't be opened by `dlopen(3)`.

Common reasons why you might encounter "library not found" exceptions:
 - The library doesn't exist or isn't accessible to the app. Use `adb shell ls -l <path>` to check its presence and permissions.
 - The library wasn't built with the NDK. This can result in dependencies on functions or libraries that don't exist on the device.

---

# FAQ: Why do I get `UnsatisfiedLinkError`? (3/4)
Another class of `UnsatisfiedLinkError` failures looks like:

```
java.lang.UnsatisfiedLinkError: myfunc
        at Foo.myfunc(Native Method)
        at Foo.main(Foo.java:10)
```

<br/>

In logcat, you'll see:

```
W/dalvikvm(  880): No implementation found for native LFoo;.myfunc ()V
```

---

# FAQ: Why do I get `UnsatisfiedLinkError`? (4/4)
Some common reasons for this are:
 - The library isn't getting loaded.
 - The method isn't being found due to a name or signature mismatch. This is commonly caused by:
   - For lazy method lookup, failing to declare C++ functions with `extern "C"` (avoid name mangling) and appropriate visibility (`JNIEXPORT`).
   - For explicit registration, minor errors when entering the method signature.

<br/>

Using `javah` to automatically generate JNI headers may help avoid some problems.

---

# FAQ: Why didn't `FindClass` find my class? (1/6)
(Most of this advice applies equally well to failures to find methods with `GetMethodID` or `GetStaticMethodID`, or fields with `GetFieldID` or `GetStaticFieldID`.)

---

# FAQ: Why didn't `FindClass` find my class? (2/6)
Make sure that the class name string has the correct format. JNI class names start with the package name and are separated with slashes, such as `java/lang/String`.

> If you're looking up an array class, you need to start with the appropriate number of square brackets and must also wrap the class with 'L' and ';', so a one-dimensional array of `String` would be `[Ljava/lang/String;`. If you're looking up an inner class, use '$' rather than '.'.

<br/>

In general, using `javap` on the .class file is a good way to find out the internal name of your class.

---

# FAQ: Why didn't `FindClass` find my class? (3/6)
If you enable code shrinking, make sure that you configure which code to keep.
> Configuring proper keep rules is important because the code shrinker might otherwise remove classes, methods, or fields that are only used from JNI.

---

# FAQ: Why didn't `FindClass` find my class? (4/6)
If the class name looks right, you could be running into a class loader issue. `FindClass` wants to start the class search in the class loader associated with your code. It examines the call stack, which will look something like:

```
Foo.myfunc(Native Method)
Foo.main(Foo.java:10)
```

<br/>

The topmost method is `Foo.myfunc`. `FindClass` finds the `ClassLoader` object associated with the `Foo` class and uses that.

---

# FAQ: Why didn't `FindClass` find my class? (5/6)
You can get into trouble if you create a thread yourself (perhaps by calling `pthread_create` and then attaching it with `AttachCurrentThread`). Now there are no stack frames from your application. If you call `FindClass` from this thread, the JavaVM will start in the "system" class loader instead of the one associated with your application, so attempts to find app-specific classes will fail.

---

# FAQ: Why didn't `FindClass` find my class? (6/6)
There are a few ways to work around this:
 - Do your `FindClass` lookups once, in `JNI_OnLoad`, and cache the class references for later use.
 - Pass an instance of the class into the functions that need it, by declaring your native method to take a `Class` argument and then passing `Foo.class` in.
 - Cache a reference to the `ClassLoader` object somewhere handy, and issue `loadClass` calls directly. This requires some effort.

---

# FAQ: How do I share raw data with native code?
You may need to access a large buffer of raw data from both managed and native code. Common examples include manipulation of bitmaps or sound samples. There are two basic approaches.

You can store the data in a `byte[]`.
> This allows very fast access from managed code. On the native side, however, you're not guaranteed to be able to access the data without having to copy it. In some implementations, `GetByteArrayElements` and `GetPrimitiveArrayCritical` will return actual pointers to the raw data in the managed heap, but in others it will allocate a buffer on the native heap and copy the data over.

---

# FAQ: How do I share raw data with native code?
The alternative is to store the data in a direct byte buffer. These can be created with `java.nio.ByteBuffer.allocateDirect`, or the JNI `NewDirectByteBuffer` function.
> Unlike regular byte buffers, the storage is not allocated on the managed heap, and can always be accessed directly from native code (get the address with `GetDirectBufferAddress`). Depending on how direct byte buffer access is implemented, accessing the data from managed code can be very slow.

---

# FAQ: How do I share raw data with native code?
The choice of which to use depends on two factors:
 1. Will most of the data accesses happen from code written in Java or in C/C++?
 1. If the data is eventually being passed to a system API, what form must it be in? (For example, if the data is eventually passed to a function that takes a `byte[]`, doing processing in a direct `ByteBuffer` might be unwise.)

<br/>

If there's no clear winner, use a direct byte buffer. Support for them is built directly into JNI, and performance should improve in future releases.

---

# References
 - [JNI tips](https://developer.android.com/training/articles/perf-jni)
 - [Java Native Interface Specification](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/jniTOC.html)

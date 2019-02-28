# How to structure Fuchsia support for a language

This document describes the structure languages typically when supporting
Fuchsia.

## System calls

The lowest level of Fuchsia support in a language provides access to the
[Zircon system calls](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/docs/syscalls/).
Exposing these system calls lets programs written in the language interact with
the kernel and, transitively, with the rest of the system.

Programs cannot issue system calls directly. Instead, they make system calls by
calling functions in the [vDSO](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/docs/vdso.md),
which is loaded into newly created processes by their creator.

The public entry points for the vDSO are defined in
[syscalls.abigen](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/system/public/zircon/syscalls.abigen).
This file is processed by the [abigen](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/docs/vdso.md#abigen-tool)
tool. When adding a new language, consider using the `-json` flag to dump
[a JSON representation](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/system/host/abigen/syscall_schema.json)
of the Zircon system calls:

```sh
abigen -json syscalls.json zircon/system/public/zircon/syscalls.abigen
```

You can then build a compiler that takes `syscalls.json` as input and produces
bindings appropriate for your language.

## Async

The vast majority of Fuchsia programs act as *servers*. After startup, they wait
in an event loop to receive messages, process those messages (potentially by
sending messages to other processes), and then go back to sleep in their event
loop.

The fundamental building block for event loops in Fuchsia is the
[port](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/docs/objects/port.md)
object. A thread can sleep in a port using
[`zx_port_wait`](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/docs/syscalls/port_wait.md).
When the kernel wakes up the thread, the kernel provides a *packet*, which is a
data structure that describes why the kernel woke up the thread.

Typically, each thread has a single port object in which it sleeps, which a
significant amount of code written in your language will need to interact with.
Rather than expose the port directly, language mantainers usually provide
a library that abstracts over a port and provides asynchronous wait operations.

Most asynchronous wait operations bottom out in
[`zx_object_wait_async`](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/docs/syscalls/object_wait_async.md). Typically, the `port` and `key`
arguments are provided by the library and the `handle` and `signals`
arguments are provided by the clients. When establishing a wait, the clients
also typically provide an upcall (e.g., a closure) for the library to invoke
when the wait completes, at which point the library uses the `key` to recover
the upcall (e.g., from a hash table).

No additional kernel object is needed to wake a thread up from another thread.
You can wake up a thread by simply queuing a user packet to the thread's port
using
[zx_port_queue](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/docs/syscalls/port_queue.md).

### Examples

* [async](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/system/ulib/async)
  (C and C++)
* [fuchsia-async](https://fuchsia.googlesource.com/fuchsia/+/master/garnet/public/rust/fuchsia-async/) (Rust)
* [zxwait](https://fuchsia.googlesource.com/third_party/go/+/master/src/syscall/zx/zxwait/) (Go)

## FIDL

The Zircon kernel itself largely provides memory management, scheduling, and
interprocess communication. Rather than being provided directly by the kernel,
the bulk of the system interface is actually provided through interprocess
communication, typically using [channels](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/docs/objects/channel.md).
The protocols used for interprocess communication are defined in
[Fuchsia Interface Definition Language (FIDL)](../fidl/README.md).

FIDL support for a language typically involves two pieces:

1. A language-specific backend for the FIDL compiler that generates code in the
   target language.
2. A support library written in the target language that is used by the code
   generated by the FIDL compiler.

These pieces are usually not built into the language implementation or runtime.
Instead, the libraries are part of the developer's program and versioned
independently from the language runtime. The stable interface between the
program and the language runtime should be the *system calls* rather than the
FIDL interfaces so that developers can pick the versions of their FIDL
interfaces and the version of their language runtimes independently.

In some cases, the language runtime might need to use FIDL internally. If that
happens, prefer to hide this implementation detail from the developer's program
if possible in your language. The developer might wish to use newer versions of
the same FIDL protocols without conflicting with the version used internally by
the language runtime.

### FIDL compiler backend

The [FIDL compiler](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/system/host/fidl/)
has a single frontend that is used for all languages and multiple backends that
support a diverse assortment of languages. The frontend produces a
[JSON intermediate format](https://fuchsia.googlesource.com/fuchsia/+/master/docs/development/languages/fidl/reference/json-ir.md)
that is consumed by the language-specific backends.

You should create a new backend for the FIDL compiler for your language. The
backend can be written in whatever language you prefer. Typically, language
maintainer choose either Go or the target language.

 * [fidlgen](https://fuchsia.googlesource.com/fuchsia/+/master/garnet/go/src/fidl/compiler/backend) (C++, Rust, and Go)
 * [fidlgen_dart](https://fuchsia.googlesource.com/topaz/+/master/bin/fidlgen_dart) (Dart)

### Generated code

The generated FIDL code varies substantially from one language to another.
Typically the generated code will contain the following types of code:

* Data structure definitions that represent the data structures defined in the
  [FIDL language](https://fuchsia.googlesource.com/fuchsia/+/master/docs/development/languages/fidl/reference/language.md).
* A codec that can serialize and deserialize these data structure into and from
  the [FIDL wire format](https://fuchsia.googlesource.com/fuchsia/+/master/docs/development/languages/fidl/reference/wire-format/README.md).
* Stub objects that represent the server end of a FIDL interfaces. Typically,
  stub object have a *dispatch* method that deserializes a message read from a
  Zircon channel and perform an indirect jump into an implementation of the
  method specied by the message's *ordinal*.
* Proxy objects that represent the client end of a FIDL interface. Typically,
  method calls on proxy objects result in a message being serialized and
  sent over a Zircon channel. Typically, proxy object have a *dispatch* for
  event messages similar to the dispatch method found in stubs for request
  messages.

Some languages offer multiple options for some of these types of generated code.
For example, a common pattern is to offer both *synchronous* and *asynchronous*
proxy objects. The synchronous proxies make use of
[`zx_channel_call`](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/docs/syscalls/channel_call.md)
to efficiently write a message, block waiting for a response, and then read the
response, whereas asynchronous proxies use
[`zx_channel_write`](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/docs/syscalls/channel_write.md),
[`zx_object_wait_async`](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/docs/syscalls/object_wait_async.md),
and
[`zx_channel_read`](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/docs/syscalls/channel_read.md)
to avoid blocking on the remote end of the channel.

Generally, we prefer to use *asynchronous* code whenever possible. Many FIDL
protocols are designed to be used in an asynchronous, feed-forward pattern.

### Support library

When designing the generated code for your language, pay particular attention to
binary size. Sophisticated program often interact with a large number of FIDL
protocols, each of which might define many data structures and interfaces.

One important technique for reducing binary size is to factor as much code as
possible into a FIDL *support library*. For example, the C bindings, all the
serialization and deserialization logic is performed by a routine in a support
library. The generate code contains only a table that describes the wire format
in a compact form.

Typically, the support library is layered on top of the async library, which
itself has no knowledge of FIDL. For example, most support libraries contain a
*reader* object, which manages the asynchronous waiting and reading operations
on channels. The generated code can then be restricted to serialzation,
deserialization, and dispatch.

 * [C](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/system/ulib/fidl)
 * [C++](https://fuchsia.googlesource.com/fuchsia/+/master/garnet/public/lib/fidl/cpp/)
 * [Rust](https://fuchsia.googlesource.com/fuchsia/+/master/garnet/public/lib/fidl/rust/fidl)
 * [Dart](https://fuchsia.googlesource.com/topaz/+/master/public/dart/fidl/)
 * [Go](https://fuchsia.googlesource.com/third_party/go/+/master/src/syscall/zx/fidl/)

## POSIX-style IO

POSIX-style IO operations (e.g., `open`, `close`, `read`, and `write`) are
layered on top of FIDL. If your language has C interop, you can use the
[FDIO library](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/system/ulib/fdio),
which translates familiar POSIX operations into the underlying `fuchsia.io` FIDL
protocol. If your language does not have C interop, you will need to interface
directly with `fuchsia.io` to provide POSIX-style IO.

You can recover the underlying Zircon handles for file descriptors using [`lib/fdio/unsafe.h`](https://fuchsia.googlesource.com/fuchsia/+/master/zircon/system/ulib/fdio/include/lib/fdio/unsafe.h).
Typically, languages have a tiny library that layers on top of the async library
to perform asynchronous waits on file descriptors. This library typically
provdes a less error-prone interface that abstracts these "unsafe" FDIO
functions.